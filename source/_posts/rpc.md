---
title: "Project: A Distributed System with Multiple RPCs and File Fetching Cache"
date: 2019-02-01 23:03:45
tags:
    - "system"
    - "network"
    - "distributed"
categories:
    - "technique"
    - "course"
    - "project"
---

<img src="rpc.png" alt="drawing" width="500"/>

You may have seen many awesome and outstanding distributed systems which work for some real-life applications. Today, we will go all to the way to the system level and see how distributed computing could be implemented here.

This project consists of two tasks:

1. build an RPC system to allow remote file operations (`open`, `read`, `write`, ...); these RPC stubs will be interposed in place of the C library functions that handle file operation. For example, if you execute `cat foo` command on the local machine, instead of opening and printing the contents of a local file, it will access the contents of file `foo` on the remote server machine.
2. based on the task 1, a proxy containing a cache should be implemented and placed between the clients and the server. Multi-clients and multi-proxy will be supported. You should also exploit some cache policies and strategies to guarantee consistency.

This project comes from the CMU course 15-440/640: Distributed System. <font color="#F13E3E">Due to the CMU academic integrity policy, the code will not be publicized and some contents in this post will also be disabled as there are directly related to the design of my solution. </font>

# RPC
