---
title: "Project: Apache Spark - Process Large Web Crawl Data and Train ML Model"
date: 2019-02-26 22:05:41
tags:
    - "design"
    - "distributed"
    - "Python"
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
- Program should be reliable and tolerant for failures (though Spark has some built-in mechanisms for fault-tolerance).

You will gradually see that programming with Spark is never easy, especially when you wish your program has high performance.

Please note that, in this project, I also used HDFS and Amazon EC2 to manage our machines and distribute data/files, which, however, is not our focus, so I just exploited some existing framework/scripts for this part.

<p style="color:#F13E3E"><b>Note</b>: due to the CMU academic integrity policy, some contents in this post are disabled as they are directly related to my codes or design. Please contact me or leave comments if you want to learn more about this project and <u><b>AND</b></u> will not take this course in the future!</p>

# Preprocessing with Spark

In this task, I was given an open repository of web crawled data (we only choose WET format file). These data should be processed following the steps below:

1. Retrieve all documents (payloads) from some WET files with some third-party Python APIs; a WET file could be large and have many documents;
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

Shuffle mechanism is used to redistribute the partitioned RDDs to satisfy certain conditions. For example, in order to detect domain-specific "stop" words, we need to count the document frequency of each word, which may require `reduceByKey` operations. **Shuffle is an expensive operation.** It normally requires a lot of disk IO and network IO costs. Also, each node (worker) will not be independent anymore, so one slow worker can make the whole process slow.

### Transformations and Actions

[Transformations](https://spark.apache.org/docs/latest/rdd-programming-guide.html#transformations) and [actions](https://spark.apache.org/docs/latest/rdd-programming-guide.html#actions) are different, and you should be aware of how they behave differently.

Note that, if there is no actions, Spark could record the dependency information between RDDs without evaluating them, which is called "lazy evaluation". Thus, several distinct steps could be merged and be executed seamlessly. This is very helpful in boosting computing performance. Thus, do not invoke actions unless required.

## Ideas

Here are some ideas I used for this task. They show great improvement in the performance.

### Reorder the Workflow

<p style="color:#F13E3E">[Contents are disabled]</p>

### Consider Intermediate Results

This trick is super helpful and effective.

At first, let's talk about a sorting algorithm. We all know "merge sort" algorithm, which divides the array into many smaller subarrays, sorts these subarrays then merges. It shows a principle called "divide and conquer". **This principle will give us some ideas in designing the program**.

<p style="color:#F13E3E">[Contents are disabled]</p>

### Distribute Workload Evenly

If there is an action which needs `Reduce` and shuffle, the reducer needs to wait all mappers to finish their work before the reducer could start its job.

Is it possible that there is a mapper takes much longer to finish its job, which slow down the whole pipeline? In this case, some mappers might be idle and reducer cannot start its job either.

The answer is yes, and it is even highly possible. Suppose when we have 4 workers and 5 WET files. The first step is to initialize 5 RDDs (corresponding to 5 WET files) and distribute them to 4 workers. However, even under the best allocation strategy, there is always one worker who has 2 WET files to process. As we mentioned earlier, it may take a long time to process just a single WET file. Thus, we will expect a period of time when three workers are idle and one working (which is called "straggler").

This poor strategy is highly possible to lead to waste of resources. Normally, if the size of RDD is smaller (and number of RDD increases), we are less likely to see a worker become the "straggler". For example, we could make each document as a RDD partition. However, it also leads to more overhead which consumes storage and computing resources as well.

## Benchmark

To process 400 WET files (~200 GB) with 16 AWS `m4.xlarge` slaves instance, the original and naive program needs to take **more than 3 hours**, while the improved program (which used the tricks above) only takes **within 35 minutes**.

Though these tricks seem very intuitive and easy to implement, they might not come to your mind when you write Spark programs the first time. Spark in fact provides us with interactive and informative UIs, showing the real-time workflow and metrics of each worker. **It is an important source for you to spot the bottleneck of your program and figure out a solution.** We will talk about it at the end of this post.

# Iterative ML Model Training

Remember what I said at the beginning of this post: Spark has some additional advantages over MapReduce. The most important features that Spark brings to us is **in-memory processing**.

This feature will be more prominent when the task is iterative: we may need to access or process the same data multiple times. Resilient Distributed Datasets (RDDs) enable multiple map operations in memory, while Hadoop MapReduce has to write interim results to a disk, which may involve some additional IO cost.

Thus, here comes the second task: given a lot of training data and multiple nodes (instances), how can we efficiently train our machine learning model in parallel. The ML model itself may has a large amount of features.

The ML model here is a simple logistic regression, and we are going to use gradient descent to update our model. This model starts with a random guess of the model feature weights and refines them over many iterations. In a single iteration, each data sample in the training set is processed to generate an incremental update to the model features. The model features updates are accumulated and not applied until the end of an iteration. **This algorithm enables us to implement a training workflow which involves many nodes:**

- In an iteration, each node can compute and accumulate gradient independently;
- The reducer can then accumulate updates from all mappers, get the final updates in the current iteration, update the feature weights, and broadcast new feature weights to all mappers;
- Mappers start a new iteration.

This workflow is really suitable for Spark: in each iteration, the nodes need to access the same training data and compute the gradient, as Spark can keep data in memory, nodes do not have to load data from disk through IO (this is what MapReduce does). The whole process would be much faster and more efficient.

However, some problem will appear when you are writing you program: sometimes your program will encounter out-of-memory issue and then crashes, or it has to take a long time to finish its job.

Compared with the first task, this task also forces you to figure out how you should manage the computing resources appropriately. Apparently, out-of-memory issue happens due to the exhaustion of memory. Is this because the memory stores too much data? Another question is, are there any other ways to make my program faster (in addition to the tips mentioned in the first task)?

No worry. I will summarize some useful tricks.

## Join-based Parameters Communication

As we mentioned earlier, the number of ML features may be huge (millions in this task). However, each data sample may not have so many nonzero values (probably only have tens of nonzero values). Also, processing each data sample only requires reading the weight values for its nonzero features and generating updates for its non-zero feature. Thus, **a node does not need to get all feature weights from the ML model.** This practice is critical in this project. In fact, trying to store a full copy of the weights or all updates on any of the machines will case failure: remember, memory is expensive and limited.

In order to take advantage of this feature, we might exploit some methods, such as inverted indices (which is popular in many area such as search engine) to design a better workflow.

## Ideas

### A Better Workflow

We have mentioned that workflow is very important in implementing a Spark program. But, in the previous example, we just reordered some tasks to reduce the work load. For a different task, the workflow might be much harder to design or improve.

<p style="color:#F13E3E">[Contents are disabled]</p>

### Optimization on `join`

#### Partition

When using `join` in your code, remember:

***When joining an RDD of N partitions with an RDD of M partitions, the resulting RDD may contain N*M partitions. You may want to explicitly control the number of partitions for the resulting RDD.***

So, if possible, remember to use some built-in APIs (e.g., `partitionBy`) to partition your RDDs again.

#### Shuffle

Shuffle in always expensive and "join" operation may also involve shuffle. However, ***When two RDDs are already partitioned by the same function, joining them does not require a shuﬄe***.

Why? And what does this indicate?

In fact, every time we create RDDs, we need to use some partitioner function to partition data and distribute data on all nodes. If two RDDs have the same partitioner functions, it indicate that data entries with same key will also reside on the same node; nodes then does not need to communicate with nodes to do "join", which leads to shuffle.

The default partitioner function is hash function. But how can you make sure two RDDs have the same partitioner function?

There are many possible cases where partitioner functions are changed. But the most important one is: ***if you applied any custom partitioning to your RDD (e.g. using `partitionBy`), using `map` or `flatMap` would "forget" that partitioner function.*** This is because Spark cannot know if the key is changed after `map`. Instead, `mapValues` is better as you could only change the value with this function.

**A caveat**: always choose `mapValues` or `flatMapValues` if you don't intend to change the key.

### Manage Your Resources

You may see some out-of-memory (OOM) issue when running your spark program. This is because Spark by default will keep RDD in memory, when we need to create another RDD (by some action), they will also be placed in memory. E.g., in this task, we need to maintain a training data RDD while generating the updates RDD. What if the memory is limited?

<p style="color:#F13E3E">[Contents are disabled]</p>

### Find the Bottleneck

Spark gives you the live monitoring web UI to help you track the task and find the bottleneck.

Normally, the first thing to check is **job**. Clicking a job shows the details of this job, including its event timeline and a DAG visualization. You can easily figure out which operations belong to which stage and the locations where shuffle occurs.

The web UI also shows statistics/metrics for tasks. You may also access status of each machine and check if any nodes are dead already. You can also detect any stragglers by checking the task numbers of each machine from the web UI.

## Benchmark

The improved program could finish 2 iterations within 30 mins with 40GB Criteo dataset as training data (ML model with 88M features) on AWS 16 `c4.xlarge` instances; or 4 iterations within 40 mins with 10 GB KDD2012 dataset (ML model with 54M features) on 16 `c4.xlarge` instances.
