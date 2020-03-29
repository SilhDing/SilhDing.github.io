---
title: "Project: Apache Spark - Process Large Web Crawl Data and Train ML Model"
date: 2019-02-26 22:05:41
tags:
    - "design"
    - "distributed"
categories:
    - "technique"
    - "course"
    - "project"
---

<img src="spark.png" alt="drawing" width="500"/>

<br>

Apache Spark is an open source, distributed parallel computing framework, which is very powerful in processing and analyzing a large amount of data. It also has some additional advantages over MapReduce.

In this project, we will use Apache Spark to finish two tasks:

1. Use Spark to preprocess an existing corpus crawled from the Internet to generate output in a specific format: extract, load, and transform (ELT).
2. Write a scalable distributed iterative ML program using Apache Spark.

The two tasks listed above, in fact, seem not challenging at the first glance. Yes, you are right. By briefly studying the official documents and some example programs, we can finish the job easily. **However, please consider some special requirements or scenarios before drawing any conclusions**:

- There is an upper limit for the running time of the program. In other words, your program should preform efficiently in terms of time.
- We have limited computing resources, i.e., memory, CPU cores, etc.
- Program should reliable and tolerant for failures (though Spark has some built-in mechanisms for fault-tolerance)

You will gradually see that programming with Spark is never easy, especially when you wish your program has high performance.

Please note that, in this project, I also used HDFS and Amazon EC2 to manage our machines and distribute data/files, which, however, is not our focus, so I just exploited some existing framework/scripts for this part.

# Preprocessing with Spark

In this task, I was given an open repository of web crawled data (we only choose WET format file). These data should be processed following the steps below:

1. Retrieve all documents (payloads) with some third-party Python APIs;
2. Discard all tokens that are not valid (e.g., invalid words contain characters rather than English letters and some simple characters);
3. Standardize all valid tokens from step 2 (including stripping away some invalid leading or tailing characters, convert all characters to lower-case, etc.);
4. Discard some documents whose length is lower than a limit;
5. Remove "stop" words ;
