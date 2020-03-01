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

### Replication Manager

The replication manager maintains a table of all replicas and their addresses (called replica status table), and also the replication style (either passive or active) and which is the primary replica in passive replication.

The replication manager also maintains a sequence number counter that monotonically increases.

The global fault detector periodically polls the status of all replicas and reports changes to the replication manager. The changes can be addition or removal of replicas. The replication manager is responsible for reacting appropriately to the changes. For example, in passive replication, if the primary replica is removed from the membership, the replication manager is responsible for choosing a new primary replica.

The replication manager assumes that the global fault detector never crashes.

### Global Fault Detector

The global fault detector maintains a list of addresses of local fault detectors and the status of all replicas at the last check. **It will never directly contact any replicas**.

The global fault detector periodically asks all local fault detectors for replica status. The interval (a.k.a. the fault detection interval) is configured via a configuration file or command line argument. If there are any changes between the new status and the previous one, it reports these changes to the replication manager.

That a replica becomes unreachable and that a local fault detector becomes unreachable both mean related replicas are removed from the membership. The GFD doesn’t distinguish between these two cases.

If a local fault detector is unreachable for one time, it is immediately removed from the list and all associated replicas under that local fault detector are considered as leaving the cluster. This is to avoid a local fault detector from fluctuating between reachable and unreachable due to temporary network malfunctioning.

### Local Fault Detector

The local fault detector maintains a list of addresses of replicas that it monitors (which can be only one).

Each time the global fault detector queries replica status, it asks all the replicas on its list for their status and returns the result to the global fault detector.

If a replica is unreachable for one time, it is immediately removed from the list.

Please note that we used a 2-tier mechanism for failure detection (LFD and GFD), which could be efficient when there are many replicas across the whole system. A single fault detector, which manages all replicas, would show poor scalability.

### Replica

Replicas maintains the state of the application and responds to queries of the local fault detector.

When a new replica is added to the cluster, the replica will notify its LFD, and LFD will later noptify GFD about this update. When the replication manager detects the addition of this replica via the report of the GFD, it sends an `updateReplicationConfig` to the new replica as well as all other replicas to trigger synchronization (the new replica can download checkpoints and logs from any other replicas).

# Interaction

In this project, we use **HTTP-based** messages for communication between different components. That is to say, each component may act as a HTTP server, a HTTP client, or both. A clear and rigid workflow is also critical in order to make sure the system behaves as we expect. We will cover some cases that you might be interested in and explain the mechanism/workflow behind them.

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

![polling_1](polling_1.svg)

### Detect a New Replica

When a replica is started (note it could be a previously failed replica and now it recovers), it will first report to its LFD (a new replica will always configured with the address of its LFD). LFD will deliver this message to GFD, who will also notify RM if there are changes in replica configuration.

On the RM's side, when a new replica is detected, the RM broadcasts the replication configuration containing the new replica to all reachable replicas. After knowing other replicas, the new replica can download checkpoints and logs from any up and running replica.

When the primary replica is unreachable in passive replication, the RM chooses a new primary and broadcasts the new configuration to all reachable replicas. (We will give more information for this part later)

The sequence interaction diagram shows how the GFD conducts replica status polling and what would happen if a new replica is detected.

![polling_2](polling_2.svg)

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

## Total Ordering

Here are many ways to implement total ordering in active replication style. Let's list some:

1. Ask every client to grab a ***sequence number*** from RM or another agency every time it wants to send a request. The request and its assigned sequence number will then be delivered to all replicas, and replica will handle requests in the order of request's sequence number (like what we do in TCP protocol).
2. Ask all clients to send requests to RM (or another agency) and let the RM decide the order of requests to process. Replicas will get requests from RM.

However, these schemes might be fragile if we consider some extreme cases:

1. In the first scheme, what if a client grabs a sequence number and holds it for a long time before sending a request? What may happen is that replicas will block in order to wait for that request's arrival.
2. In the second scheme, what if we have many replicas sending requests? Scalability would be an issue and the RM (or the agency receiving all requests) would become the bottleneck.

We would use a new scheme instead in our peoject. This scheme only involves replicas themselves and it is easy to implement.

### Polite Client Assumption

First, we have the following assumptions:

- The RM first notifies all clients of the new replication configuration, then notifies all replicas of the new config.
- Whenever the client knows there is a replica whose state is NEW (which means the synchronization for this NEW replica is undergoing and have not finished), it stops sending requests until all replicas are READY. ***You will later see that this assumption is strong and it greatly simplifies the implementation of total ordering.***

### Handling Client Requests

Let’s first talk about how a replica handles a client request.

By *Polite Client Assumption*, we know that a new replica never receives client requests. A replica only receives requests when it is in ready state.

The procedure is:

1. Put the client request in the ***RequestPool*** and waits for it to complete. When this thread is waiting, other threads do consensus and request processing work. When everything is done, this thread is woken up with the result.
2. Return the result to the client.

The RequestPool is a bridge between the client side and the consensus side. The client side doesn’t care how the consensus is done, and the consensus side doesn’t care where the request comes from and where it goes to.

So now we have a **clear separation** between client request handling and consensus protocol.

### Executor Thread

Before talking about consensus protocol, let’s first assume that we already have a total order by running consensus. Now how do we actually process those requests?

The truth is that **the replica has a long-running Executor thread. The thread runs a while(true) loop that checks if it has any work to do repeatedly**.

It has an internal counter `nextProcLid` that is the next log ID to process. We use log ID (or Lid) to refer to the *index of a message in the total order*.

It also needs to read a global variable `lastCommitLid` that is the last committed log ID (which means you can process all the logs up to this lid, but not those after this lid). For now, we can assume that `lastCommitLid` is always the ID of the last log entry in its log. Later, we’ll talk about why it is not.

The thread work as follows:

1. Check if `nextProcLid <= lastCommitLid` and there is a request in the RequestPool with the log ID equal to `nextProcLid`. If false, then we have processed all committed requests, so we have nothing to do.
2. If false, we sleep 100 ms and go to step 1. If true, we process that request, hand over the response to the RequestPool and increment `nextProcLid`. By giving the response to the RequestPool, the Executor thread wakes up the client request thread that is waiting for it.
3. Go to step 1. We want to process as many as possible.

The RequestPool has an API that allows us to find a request by the log ID. The log ID is not assigned by the client request thread. It is set by the consensus protocol.

Now we have already talked about how to handle client requests and how to process a request. All the “request-related” things has been discussed. The only thing left is how to achieve consensus on the order of requests.

In the rest of the section, we will focus our eyes only on the request ID (which is generated uniquely by the client) and the log ID. We will ignore the content of the requests.

### Prosper and Acceptor

A replica can be either a ***proposer*** or an ***acceptor***, but not both.

- A proposer dictates the order of messages.
- An acceptor accepts whatever the proposer says.

The proposer has a Proposer thread running in the background.

The Proposer thread has an internal counter `nextProposeLid` that is the next log id to propose.

The Proposer thread also has an internal hash map `lastLidMap` whose key is a replicaId, and whose value is the lastLid of that replica (the ID of the last log that replica has). 

The proposer periodically executes this:

1. Get all unassigned requests from the RequestPool. An unassigned request has no lid. After it is proposed, it is assigned a lid. So we ensure that a request can be proposed only once.
2. Assign `nextProposeLid` to an unassigned request and then increment `nextProposeLid`, repeat for all unassigned requests. This is how the requests in the RequestPool are assigned. 
3. Append these `requestId` to the local log that it has. `requestId` is the unique identifier generated by the client in the request. The index of the requestId in the log is its lid.
4. Send `appendLog(lastCommitId, logs)` to all ready acceptors; logs start from `lastLidMap[replicaId]+1` for each replica. After doing `appendLog`, all ready replicas are informed of the assignments. But these new logs are not committed yet.
5. Set `lastCommitId` to the last lid in its log. After informing all replicas, we can commit these logs. This value will be updated to all replicas in the next round.

Periodically, even if there is no unassigned request, the proposer will still send an `appendLog(lastCommitId, [])` to all replicas (to inform them of the new commit ID, for example).

The acceptor reacts to appendLog request from the proposer by doing the following things:

- Assign `lastCommitId` to the value in `appendLog`.
- If the RequestPool has the requests, assign the lids to the requests.
- If not, tell the RequestPool to save the pair `(lid, requestId)` somewhere and assign the lids to those requests when it sees the requests later.

By doing so, the Executor thread in the acceptor will eventually process the request.

### New Proposer Confirmation

**When a new replica joins the cluster, the RM assigns a `replicaRank` to the replica.** The replicaRank is an integer increasing from 0. It marks when a replica joins the cluster. New replicas have greater ranks.

Whenever a replica receives an `updateReplicationConfig` request from the RM, ***it recognizes the replica with the smallest rank as the proposer***.

Since a new proposer will appear only AFTER the old proposer crashes, there will never be two proposers at the same time. There will be times when there is no proposer at all, but it’s okay. Eventually, there will be a proposer.

We also know that the transition is one-directional: an acceptor can become a proposer, but a proposer can never become an acceptor (it’s a dictator for life).

Whenever there is a proposer change, the new proposers and the acceptors will perform a proposer confirmation process.

1. The proposer sends `confirm(lastLid)` to all ready acceptors (a) to tell all replicas the last log ID that the proposer has and (b) to ask them to tell the proposer the last log ID that they have. The new proposer might not be the one who has the longest log.
2. Each acceptor responds with the latest log ID it has to the proposer; if an acceptor has longer logs, it also put these additional logs in the response. 
3. Upon receiving all responses, for all acceptors’ responses,
    - If an acceptor has a longer log, the proposer will append the missing part to its log. Now the proposer has the longest log.
    - Set lastLidMap[replicaId] to the lid in the response. Next time the proposer will send logs to this replica from this position.
4. If the RequestPool has no unassigned requests for now, put a ***NoOp(do-nothing)*** request in the pool. The proposer might have some assigned but uncommitted requests. If we don’t commit them, the clients might be blocked. However, the way we commit them is not to modify the `lastCommitLid` directly, but to propose a NoOp request, which in turn triggers the `appendLog` procedure, committing the NoOp request and all the uncommitted requests ahead of it.

After the new proposer finishes the confirmation, it will start the Proposer thread. There might be some unassigned requests left when it was an acceptor, the new proposer will propose these requests eventually. No requests will be lost.

Up to now, you might be confused with the description above. Let's address some questions again.

> **Q: Why do we need a `lastCommitLid`?**

There are **three parts** in the log:

- the first part is what is committed and processed;
- the second part is what is committed but unprocessed (the executor hasn’t process them though it is able to do so);
- the third part is what is uncommitted (the executor is not allowed to process them).

Why can’t we assume all logs are committed?

Think of this scenario of three replicas: the proposer appended the log X to acceptor A, but before appending to acceptor B, crashed. Acceptor A crashed simultaneously (a.k.a. two simultaneous faults). Acceptor B becomes the new proposer.

You see the problem?

Acceptor B has no possibility knowing that X is in the log. It may propose something different, for example an unassigned request Y, before re-proposing X. But since acceptor A knows X, it might already have returned the response to the client.

Now we have an **inconsistency**: the client thought X happened before Y, but in the new “world”, the new proposer said Y happened before X!

The way to avoid this from happening is to commit (allowing processing a request) ***only after all replicas know the log***, which is a typical consensus behavior (you may compare it with 2-phase commit or PAXOS). That’s why we need a `lastCommitLid`.

In other words, uncommitted logs are changeable during proposer changes.
 

>  **Q: Why do we need a NoOp request?**

This issue raises from committing.

Let’s look at a new proposer. It has some assigned but uncommitted requests. These uncommitted requests block the clients who are waiting for responses. And in turn, since the clients are blocked, they can’t send new requests.

And now comes the problem: the proposer can commit only when receiving new requests! It's deadlock.

To avoid it from happening, if the proposer has no unassigned requests, it proposes a NoOp request that does nothing. By the time the NoOp request is committed, all the requests before the NoOp are committed as well! Remember lastCommitLid means that **ALL** the requests before and equal to that lid are committed.

### New Replica

Let’s first recap the ***Polite Client Assumption***. No client interactions during new replica synchronization.

Upon receiving updateReplicationConfig, the proposer checks if there is a NEW replica that it sees for the first time. If so, it starts the following synchronization step.

1. The proposer first asks if the NEW replica is still in the NEW state. The latency of information propagation can cheat. That’s why we need to confirm the state of the new replica.
2. If the replica says READY, the proposer internally marks it as READY and finish. If the replica says NEW, do the following. By internally marking as READY, the proposer will ignore any state of that replica in later updateReplicationConfig. There might be some delay for it to become ready.
3. **The proposer waits until all the requests are processed**. This is to ensure that its state is the newest state. If it has some requests left undone, its state is earlier than latest.
4. The proposer checkpoints the state. It contains the balance of all accounts and the last processed log ID and the last committed log ID.
5. The proposer sends the checkpoint to the new replica.
6. The new replica accepts and recovers the state from the checkpoint. **Now the new replica is in the same state as the proposer.**
7. The new replica switches to READY state. If the old proposer crashes, the new proposer will ask if the new replica is still new first (this is a corner case that we might ignore).
8. The proposer internally marks the replica as READY and will never perform the synchronization on that replica again, even if later `updateReplicationConfig` says it is still NEW (due to the latency of fault detectors).


## Checkpointing

# Web UI

## GFD

## LFD

## RM

## Replica







