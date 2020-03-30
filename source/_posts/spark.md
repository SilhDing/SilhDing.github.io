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

In this project, I will use Apache Spark to finish two tasks:

1. Use Spark to preprocess an existing corpus crawled from the Internet to generate output in a specific format: extract, load, and transform (ELT).
2. Write a scalable distributed iterative ML program using Apache Spark.

The two tasks listed above, in fact, seem not challenging at the first glance. Yes, you are right. By briefly studying the official documents and some example programs, you can finish the job easily. **However, please consider some special requirements or scenarios before drawing any conclusions**:

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
5. Remove "stop" words (stop words refer to common words in English, e.g., "a", "the" and domain-specific common words, which appear in more than 90% of all documents);
6. Remove typo words. Typo words can be identified by checking the number of documents that the word appears in. If the number is below a limit, we may treat this word as a typo.
7. sort all words by their lexicographical order and give them index number starting from 0.
8. Generate "word:count" pair for all words and computer statistic data of processed corpus.

All steps listed above, in fact, do not require complex algorithms. You may find that we could parallelize some operations, e.g., discard invalid tokens, as each machine can independently finish this job. Some other operations may need `Reduce` operation, e.g., removing typo, as we have to know the document frequency of this word across the entire corpus.

But, what makes the program so slow?

## Performance Analysis

### Shuffle

Shuffle mechanism is used to redistribute the partitioned RDDs to satisfy certain conditions. For example, in order to detect domain-specific "stop" words, we need to count the document frequency of each word, which may requires `reduceByKey` operations. **Shuffle is an expensive operation.** It normally requires a lot of disk IO and network IO costs. Also, each node (worker) will not be independent anymore, so one slow work can make the whole process slow.

### Transformations and Actions

[Transformations](https://spark.apache.org/docs/latest/rdd-programming-guide.html#transformations) and [actions](https://spark.apache.org/docs/latest/rdd-programming-guide.html#actions) are different, and you should be aware of how they behave differently.

Note that, if there is no actions, Spark could records the dependency information between RDDS without evaluating them, which is called "lazy evaluation". Thus, several distinct steps could be merged and be executed seamlessly. This is very helpful in boosting computing performance. Thus, do not invoke actions unless required.

## Ideas

Here are some ideas I used for this task. They show great improvement in the performance.

### Reorder the work flow

Properly reordering the work flow may lead to lower computing cost. The work flow listed previously is obviously not the best one.
