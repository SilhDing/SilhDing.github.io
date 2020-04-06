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

It is very common for us to access resources located on remote machines. We may easily write some applications to use the network to communicate between different components running on different machines, which, however, is tedious and inelegant to insert ad-hoc networking code every place our software needs to access remote resources.

Instead, we could implement remote procedure calls to hide the network complexities and to access the remote resources in a clean abstraction. Users of these RPCs will not need to care anything on networking, and they could also develop more applications on the top of them.

## Requirements

The purpose of this task is to enable clients to use some commands (such as `cat`, `ls`) to access files on remote machine. In order to do this, we need to rewrite the standard C library calls and replace the original ones:

	open, close, read, write, lseek, stat, unlink, getdirentries

We will also implement the non-standard calls `getdirtree` and `freedirtree`. The course will provide us a library containing these two functions.


```c
struct dirtreenode* getdirtree( const char *path );
```
Function `getdirtree` recusively descend through directory hierarchy starting at directory indicated by path string. Allocates and constructs directory tree data structure representing all of the directories in the hierarchy.

```c
void freedirtree( struct dirtreenode* dt );
```
Function `freedirtree` recursively frees the memory used to hold the directory tree structures, including the name strings, pointer arrays, and `dirtreenode` structures.

These two function enables you to use `tree` on the terminal which will print out the file tree structure on the terminal. The library only makes sure you can execute the command locally, and we need to implement a RPC for it.

## Design

This task will be completed in C, so it may involve some basic practice on memory management, networking programming (we use TCP here) and some file operations. In addition, we also need to design some protocols.

### Serialization and Deserialization

<img src="serial.png" alt="drawing" width="600"/>

The picture below shows what will happen once a RPC is called. We need to design how should we:

1. pack RPC name and corresponding parameters into a string (serialization on client side) and send it to server;
2. unpack RPC name and parameters, call corresponding local function, get results, pack results into a string and send back to client (deserialization and serialization on server side);
3. unpack results from server and show locally.

Due to CMU academic policy, we will not discuss any detail on these protocols. Please note that you may need some a little bit more complicated algorithm when dealing with `getdirtree`, as we have to serialize a tree structure into a string and deserialize from a string.

### Concurrency

We must handle the situation where multiple clients are accessing to a single remote machine. A simple solution is using the multi-process mode. Each client will be handled in a specific child process, which has its own file descriptor table and will not conflict with other clients' FD tables.

### Local and Remote File Descriptor

a very subtle but important problem is, for client, how can we distinguish between the local and remote file descriptor.

You may ask, why we have to do this? ***This is because some programs on the client still needs to access local resources.*** For example, a client opens a remote file and the FD is `3` (this is created by the remote machine) and a local file, which returns a FD `3` as well. In this case, if we continue some file operations such as `read`, we don't know which file we exactly need to read. Thus, there must be a mechanism to tell the origin of the FD (i.e., it is from local or remote).

<p style="color:#F13E3E">[Contents are disabled]</p>

# File Cache

In the previous task, we already have a simply distributed system, where clients and servers could communicate via RPCs. Now, we will design and implement a cache in Java between these two components.

Caching is a great technique for improving the performance of a distributed system. It can help reduce data transfers, and improve the latency of operations. This task will continue to use existing binary tools in task 1 and interpose on their C library calls. Instead of directly connecting the server, clients now will only access the proxy (which contains a cache) to execute some operations:

<img src="cache.png" alt="drawing" width="600"/>

<br>

Seems easy, right? Well, let's talk about some details on requirements.

## Requirements

Here are something we need to consider through this task.

1. ***Cache policy***. Remember that a cache always have limited space, so you need to evict some cache entries under a policy you design;
2. ***Concurrency***. The system must support multiple proxies and clients running simultaneously;
3. ***Consistency***.  


[TODO]
