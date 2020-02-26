---
title: "Project: A Distributed, Reliable and Fault-tolerant Banking System"
date: 2019-12-01 17:53:35
tags:
    - "design"
categories:
    - "technique" 
    - "course"
    - "project"
---

Designning and implementing a distributed and fault-tolerant system is always interesting but **challenging** and **complicated**.

In this team project, we designed and implemented a comprehensive distributed banking system. However, the application part is simple in our project as we want to mainly focus on the design and implementation of distributed system and principles of strong consistency. Below are some highlights or features of our system:

- Industrial standard architecture (but simplified) of ditributed system;
- Support **active replication** and **passive replications** schemes;
- Implement total ordering to ensure strong consistency;
- Tolerant to changes of membership (i.e., able to handle the cases where new replicas join or existing replicas fail);
- Implement web UI for each component;
- Support some basic banking operations, e.g., create an account, deposit, withdraw, etc.
- ...

# Design Overview

In this section, we are going to give an brief introduction of our system.

## Terminology

| Name | Note |
| ----------- | ----------- |
| RM | replication manager |
| GFD | global fault detector |
| LFD | local fault detector |
| Address | the IP and port number of a service (e.g. the RM, GFD, LFD or a replica) |
| Replica status | whether the replica is **NEW** (a new replica that is syncing up) or **READY** (the replica is up and running). |
| Local replica status | the replica status of all the replicas that are managed by a LFD. |
| Global replica status | the replica status of all the replicas that are managed by a GFD. |
| Replication configuration | the addresses of reachable replicas, the replication style, the primary replica if passive, and etc.|
| Reachable replica | in the distributed system settings, it is hard to say if a replica is “running” or not, since it can happen that a replica is still running but can’t be reached by a LFD or GFD. To avoid such ambiguity, we would say that a replica is “unreachable”.|
|Service info | the replication configuration in the client’s point of view, including the addresses of replicas that the client needs to reach. In passive replication, a client only needs to reach the primary replica but in active replication, it needs to reach all active replicas.|

## Hierarchy and Components

We used a simplified version of industry standard distributed system. The picture below shows a brief structure of the system.

![system](system.svg)

Please note that each element in the picture (e.g., RM, GFD) would be a physical machine or a virtual machine or even a process. To simplify the system, we make RM and GFD on one physical machine. A LFD and several replicas would be included in a cluster, which, in our case, is in fact a physical machine.

We will give detailed description of each component in the picture above.

### Replication Manager (RM)

The replication manager maintains a table of all replicas and their addresses (called replica status table), and also the replication style (either passive or active) and which is the primary replica in passive replication.

The replication manager also maintains a sequence number counter that monotonically increases.

The global fault detector periodically polls the status of all replicas and reports changes to the replication manager. The changes can be addition or removal of replicas. The replication manager is responsible for reacting appropriately to the changes. For example, in passive replication, if the primary replica is removed from the membership, the replication manager is responsible for choosing a new primary replica.

The replication manager assumes that the global fault detector never crashes.

### Global Fault Detector (GFD)

The global fault detector maintains a list of addresses of local fault detectors and the status of all replicas at the last check. **It will never directly contact any replicas**.

The global fault detector periodically asks all local fault detectors for replica status. The interval (a.k.a. the fault detection interval) is configured via a configuration file or command line argument. If there are any changes between the new status and the previous one, it reports these changes to the replication manager.

That a replica becomes unreachable and that a local fault detector becomes unreachable both mean related replicas are removed from the membership. The GFD doesn’t distinguish between these two cases.

If a local fault detector is unreachable for one time, it is immediately removed from the list and all associated replicas under that local fault detector are considered as leaving the cluster. This is to avoid a local fault detector from fluctuating between reachable and unreachable due to temporary network malfunctioning.

### Local Fault Detector (LFD)

The local fault detector maintains a list of addresses of replicas that it monitors (which can be only one).

Each time the global fault detector queries replica status, it asks all the replicas on its list for their status and returns the result to the global fault detector.

If a replica is unreachable for one time, it is immediately removed from the list.

Please note that we used a 2-tier mechanism for failure detection (LFD and GFD), which could be efficient when there are many replicas across the whole system. A single fault detector, which manages all replicas, would show poor scalability.

### Replica

Replicas maintains the state of the application and responds to queries of the local fault detector.

When a new replica is added to the cluster, the replica will notify its LFD, and LFD will later noptify GFD about this update. When the replication manager detects the addition of this replica via the report of the GFD, it sends an `updateReplicationConfig` to the new replica as well as all other replicas to trigger synchronization (the new replica can download checkpoints and logs from any other replicas).

# Interaction

A critical part in a distributed system is **how can your system cope with a new replica or failed replica**. In this section, we'll cover some cases you might be interested in.

## Startup

In order to simplify our implementation, all components in our system will be started manually. The GFD and RM will be first started. GFD is configured with the address of the RM and its monitoring list is initially empty.

When a LFD is started, the LFD calls /addLocalFaultDetector of the GFD to add itself to the monitoring list of the GFD.

When a replica is started, the replica calls /addReplica of the LFD to add itself to the monitoring list of the LFD.

Next time that GFD polls replica status, it will detect the presence of the new replicas or absence of old replicas and reports this change to the RM.

## Replica Status Polling

The GFD performs replica status polling to know the liveness of all replicas and reports any changes since the last check to the replication manager.

The frequency of the GFD performing this polling is determined by the fault detection interval, which is part of the configuration file of GFD and can be modified by the RM.

### Detect a Failed Replica

When a replica fails or becomes unreachable, LFD will first be aware the issue and notify the GFD about this update. The GFD would notify the RM and RM would later send the new replication configuration that has removed the failed replica to all reachable replicas.

You may refer to the UML sequence diagram for more accurate description.

**Here should be an UML diagram.**

### Detect a New Replica

When a replica is started (note it could be a previously failed replica and now it recovers), it will first report to its LFD (a new replica will always configured with the address of its LFD). LFD will deliver this message to GFD, who will also notify RM if there are changes in replica configuration.

On the RM's side, when a new replica is detected, the RM broadcasts the replication configuration containing the new replica to all reachable replicas. After knowing other replicas, the new replica can download checkpoints and logs from any up and running replica.

When the primary replica is unreachable in passive replication, the RM chooses a new primary and broadcasts the new configuration to all reachable replicas. (We will give more information for this part later)

The sequence interaction diagram shows how the GFD conducts replica status polling and what would happen if a new replica is detected.

**Here should be an UML diagram.**

## Client Operation

When a client initializes a request, the request will go through many components and trigger various behaviors, which could be **highly complicated and interactive**. Our goal in processing clients' requests is to guarantee strong consistency across all replicas by implementing **total ordering**. In addition, we are also suposed to design and implement different workflows under different replication style/schemes (active or passive). We will skip this section and give more explaination when we talk about replication styles later.


# Request Handling

Up to now we already know how the system keeps detecting new or failed replicas. However, you may still have questions:

1. How do you handle a request while ensuring strong consistency?
2. Consider a case where there are multiple clients that may send reqeusts simultaneously. How do you implement total ordering to guarantee consistency?
3. Consider a case where the whole system have been running for a while and have handled some request. Now a new replica joins. How does it synchronize to the most recent state. In other words, how do you implement logging and checkpointing?

No worry. Before we start, let's dive deep into the replication style.

## Replication Style

In this project, we support two different replication styles.

### Active Style

In active replication style, once a client send a request, **ALL** replicas will have to execute the request and, if necessary, make state changes. Active replication style promises faster receovery from faults but it normally requires more memory and processing costs.

If many client send requests simultaneously, it is highly possible that differnet replicas will receive different order of requests. In this case, strong consistency might be compromised, and we need **total ordering** to address this concern. We will talk about it later.

### Passive Style

In passive replication style, only the **primary replica** will receive and handle this request. For other replicas (a.k.a **secondary replicas**), they will only get checkpoints from primary replica periodically (the frequency is determined by a parameter called *checkpointing interval* which can be modified while the system is running). Passive replication style requires less memory and processing costs, but may take longer to receover from faults/failures.

Total ordering will be trivial here as there is only one replica which really executes requests. However, it may complicate other parts. For example, if the primary replica fails, how can we determine a new replica as the primary replica? Some consensus protocols might be helpful (e.g., PAXOS), but remember: *implementing a consensus protocol would be highly complicated and painful.*

No worry. We will introduce our solution later.






