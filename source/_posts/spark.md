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

Shuffle mechanism is used to redistribute the partitioned RDDs to satisfy certain conditions. For example, in order to detect domain-specific "stop" words, we need to count the document frequency of each word, which may requires `reduceByKey` operations. **Shuffle is an expensive operation.** It normally requires a lot of disk IO and network IO costs. Also, each node (worker) will not be independent anymore, so one slow work can make the whole process slow.

### Transformations and Actions

[Transformations](https://spark.apache.org/docs/latest/rdd-programming-guide.html#transformations) and [actions](https://spark.apache.org/docs/latest/rdd-programming-guide.html#actions) are different, and you should be aware of how they behave differently.

Note that, if there is no actions, Spark could records the dependency information between RDDS without evaluating them, which is called "lazy evaluation". Thus, several distinct steps could be merged and be executed seamlessly. This is very helpful in boosting computing performance. Thus, do not invoke actions unless required.

## Ideas

Here are some ideas I used for this task. They show great improvement in the performance.

### Reorder the Workflow

Properly reordering the work flow may lead to lower computing cost. The work flow listed previously is obviously not the best one. For example,

- We can remove English common words when we are finding and standardizing valid tokens; **then we have less words in `Reduce` operation later**.
- We can remove domain-specific common words and typo in one step, as both needs to know the document frequency of the words.

You could also try to further improve the work flow. But remember, always try to decrease the workload for next step, especially for `Reduce` operation which requires shuffle.

### Consider Intermediate Results

This trick is super helpful and effective.

At first, let's talk about a sorting algorithm. We all know "merge sort" algorithm, which divides the array into many smaller subarrays, sort these subarrays then merge. It shows a principle called "divide and conquer".

**This principle will give us some ideas in designing the program.** Say now we have multiple WET files, each containing many documents, and we want to count the corpus frequency (total appearance) and document frequency (number of documents that contain this word) of each word. A straightforward solution is:

1. parse words in each document from each WET file and generate a tuple: `[word_name, appearance_in_doc, 1]` with map functions ("1" refer the document frequency is 1);
2. reduce all tuples by `word_name` from step 1 and get statistics.

This method will works well and give you accurate results. But it is not optimal. A better solution would be:

1. parse words in all documents from a single WET, and **within the WET file**, generate a tuple for each **distinct** word: `[word_name, appearance_in_WET, document_appearance_in_WET]`; do this for each WET file.
2. reduce all tuples from step 1 and get statistics.

Why the second one is better? Because **we have preprocessed each WET** before getting the final results. In the first solution, `Reduce` have larger workload as a distinct word may have many corresponding tuples (e.g., if there are 50000 words in the corpus, we then will have 50000 pair before reducer works, but we may only have 2000 distinct words). If we have reduce the words (i.e., eliminate tuples with same `word_name` by merging them) in each WET file, the reducer surely has less work to do, and we will see a significant improvement in computing performance.

### Distribute Workload Evenly

If there is an action which needs `Reduce` and shuffle, the reducer needs to wait all mappers to finish their work before the reducer could start its job.

Is it possible that there is a mapper takes much longer to finish its job, which slow down the whole pipeline? In this case, some mappers might be idle and reduce cannot start its job either.

The answer is yes, and it is even highly possible. Suppose when we have 4 workers and 5 WET files. The first step is to initialize 5 RDDs (corresponding 5 WET files) and distribute them to 4 workers. However, even under the best allocation strategy, there is always one worker who has 2 WET files to process. As we mentioned earlier, it may take a long time to process just a single WET file. Thus, we will expect a period of time when three workers are idle and one working (which is called "straggler").

This poor strategy is highly possible to lead to waste of resources. Normally, if the size of RDD is smaller (and number of RDD increases), we are less likely to see a worker become the "straggler". For example, we could make each document as a RDD. However, it also leads to more overhead which consumes storage and computing resources as well.

## Benchmark

To process 400 WET files (~200 GB) with 16 AWS `m4.xlarge` slaves instance, the original and naive program needs to take **more than 3 hours**, while the improved program (which used the tricks above) only takes **within 35 minutes**.

Though these tricks seem vert intuitive and easy to implement, they might not come to your mind when you write Spark programs the first time. Spark in fact provides us with interactive and informative UIs, showing the real-time workflow and metrics of each worker. **It is an important source for you to spot the bottleneck of your program and figure out a solution.** We will talk about it at the end of this post.

# Iterative ML Model Training

Remember what I said at the beginning of this post: Spark has some additional advantages over MapReduce. The most important features that Spark brings to us is **in-memory processing**.

This feature will be more prominent when the task is iterative: we may need to access or process the same data multiple times. Resilient Distributed Datasets (RDDs) enable multiple map operations in memory, while Hadoop MapReduce has to write interim results to a disk, which may involve some addition IO cost.

Thus, here comes the second task: given a lot of training data and multiple nodes (instances), how can we efficiently train our machine learning model. The ML model itself may has a large amount of parameters.
