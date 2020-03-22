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

# System Hierarchy
