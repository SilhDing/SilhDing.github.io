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

## System Design

In this section, we are going to give an brief introduction of our system.

### Terminology

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

### Hierarchy and Components

We used a simplified version of industry standard distributed system. The picture below shows a brief structure of the system.

![system](system.svg)

Please note that each element in the picture (e.g., RM, GFD) would be a physical machine or a virtual machine. To simplify the system, we make RM and GFD on one physical machine. A physical machine may also contains a LFD and several replicas. 

A LFD will be responsible for detecting replica failures or accepting new replicas in its own cluster (in our project, a cluster is in fact a physical machine and a LFD or a replica is an exclusive process).

