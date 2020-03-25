---
title: "Project: A Distributed and Auto-Scalable Online Shop Web Server"
date: 2019-04-21 16:54:56
tags:
    - "design"
    - "distributed"
categories:
    - "technique" 
    - "course"
    - "project"
---

<img src="head.jpeg" alt="drawing" width="500"/>

If you have ever get along with AWS EC2, you must know that the most critical feature it provides is load balancing, which is capable of automatically killing or launching machines (or VMs) according to the real-time metrics (e.g., CPU utilization). In this project, we also wanted to do something similar.

We implemented a distributed web server which can emulated online shopping (browse buy). The system consists of multiple tiers, and is able to scale in or out accordingly, depending on the current QPS.

Please note that it is a small project, and there is, in fact, no any real shopping applications, we as want to mainly pay attention to design and implementation of scalability and system hisrarchy. 

We also want you to keep in mind that:

- Though we talks about "servers" here, each server in fact is a process, which act as a VM and can easily be killed or launched.
- You cannot by anything though our web :)

# System Hierarchy

Refer to the picture below for the syste, hierarchy:

![hierarchy](structure.svg)

## Front-end Server

The responsibility of front-end servers is to receive requests (but not process them). Thus, there should be something like "request pool" to store all requests. Who should be reponsible for maintaining this "pool"? In order to do that, we also divided all front-end servers into 2 categories:

### Master

The master front-end server is supposed to maintain a ***central queue*** which has all requests received by front-end servers. Please note, the master server itself will also receive reqeusts, just like slave servers (explain below). There is only and at least one master in our system. Another critical feature for master server is that it also takes charges of scaling in/out, according to the length of central queue.

### Slave

Slave front-end servers have less workload compared to the master. It will only receive requests and push them into the central queue on master front-end servers. There might be multiple slave masters.

## Middle-end Server

Each middle-end server would continuously get a request from the master front-end server and process it with the cache. In addition, it will also detec whether scale-in or scale-out for middle-tier is necessary. If so, it would notify the master server to kill or launch middle-end servers accordingly.

## Cache and Database

When it comes to database and cache, consistency appears to the top priority. While read-only operations might be executed on cache directly, write operations might cause more trouble as we have to go all the way to the database, and might have to invalidate some cache entries as well. It would become more complex and troublesome if there are mutiple clients or caches in the system.

In this project, we will not do anything about database and cache, not to mention any consistency models. We only wrote a very simple interface to execute some operations on a cache. If you want to learn more about cache-realted technologies, you may refer to my another post on distributed system and file caching. (If you do not see such a post, probably I am still working on it)

# Scalability

## Middle Tier

Scaling on middle tier is **determined** by middle-end servers and **done** by master front-end server.

### Scaling Out

As we mentioned earlier, middle-end servers will continuously poll requests from master server. In fact, when it tries to get a request, it will also check the length of the central queue. If it detects a "too long" queue **for consecutive two times**, we will launch two more servers.

### Scaling In

Consider the case where this is no pending requests in the central queue. Now, a middle-end server cannot get a request, and it will wait until timeout. If there happens to be three consecutive timeouts, master server will be notified to scale in by killing some servers.

## Front Tier

The number of front-end servers is dynamically adjusted by number of the middle-end servers. The master front-end server will check it when the number of middle-end servers change. By doing so, we can always keep an optimal ratio of front-end and middle-end servers, which is helpful in avoiding waste of resource.

## Benchmarking

You may now be aware that we have many parameters in out system, and probably we need to conduct some experiments to determine the optimal values. That's right. We need benchmarking and experiments to answer the following questions:

1. What is the initial number for front-end server and middle-end server (based on the QPS)?
2. What is value of timeout when a middle-end server tries to get a request from the master?
3. What is the optimal ratio of front-end and middle-end servers? The optimal value might different under different load pattern.

Thus, designing a auto-scalable system is not only about algorithma dn strategies. We also need to carefully study the load pattern and conduct some experiments to determine how we should tweark our system.