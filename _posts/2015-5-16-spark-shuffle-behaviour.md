---
layout: post
title: "Spark Shuffle Behaviour"
comments: True
permalink: "spark-shuffle-behaviour"
---

#### Background

The Spark has bottleneck on the shuffling while running jobs with non-trivial number of mappers and reducer. There has been lots of improvement in recent release on shuffling like consolidate file and sort-shuffling from version 1.1+.Here I have explained the `YARN` and  `Spark` parameter that are useful to optomize Spark shuffle performance.

#### Cluster Configuration
 The cluster is Cloudera Enterprise Data Hub Edition Trial 5.3.1 with Spark 1.2.0 and Hadoop 2.5.0 .The following is the container configuration for  the cluster:

* `yarn.scheduler.minimum-allocation-mb = 1GB` , minimum container size
* `yarn.scheduler.maximum-allocation-mb = 4GB` , maximum container size
* `yarn.scheduler.minimum-allocation-vcores = 1`, minimum cores for container
* `yarn.scheduler.maximum-allocation-vcores = 4,`, maximum cores for container
* `yarn.nodemanager.resource.manager.memory-mb = 14GB`, amount of physical memory that can be allocated to containers
* `yarn.nodemanager.resource.manager.cpu-vcores = 18 `, number of cores that can be allocated to containers

From the above configuration, the spark executors memory size cannot be greater than 4GB  and number of cores assigned cannot be more than 4, else yarn ResourceManger could not start the executors.

Moreover, the cluster is using [capacity](http://hadoop.apache.org/docs/r1.2.1/capacity_scheduler.html) scheduler so there is strict limit on user and group on the amount of cluster resources allocated. The cluster has following queue and resource allocation to each queue:

|	Queue Name |      Percentage Resource Allocation |
|:-------------------------:|:---------------------------------------------------------:|
|	dev                     |	20%                                                
|   rootuser             | 30%
|     other                  | 40%

The dev and rootuser queue have user-limit factor of 10, which allows the single user in the queue to use 10 times the configured capacity for the queue.


#### Spark Shuffle Behaviour

The Terasort is well known benchamrk for Hadoop cluster which basic idea is to generate 1 TB random data , sort it (as fast as possible) and validate the sort data. We used similar Spark-Terasort for benachameking and tuning our cluster, which  which is available in [here](https://github.com/ehiggs/spark-terasort) from [Ewan Higgs](https://github.com/ehiggs).Instead of 1 TB data we generate , sort and validate 100 GB data for our test purpose.Below is the command we use to generate the data:

```spark-submit --class com.github.ehiggs.spark.terasort.TeraGen --deploy-mode client --master yarn --num-executors 26 --driver-memory 1g --executor-memory 1g --executor-cores 1 --queue dev /tmp/spark-terasort-1.0-SNAPSHOT-jar-with-dependencies.jar 100g /user/bijay/terasortInput```

The configuration parameters for Spark shufle with default values are:
> `spark.shuffle.consolidateFiles` 	`false`
> `spark.shuffle.spill `	`true`
> `spark.shuffle.spill.compress` 	`true`
> `spark.shuffle.memoryFraction` 	`0.2`
> `spark.shuffle.compress` 	`true`
>`spark.shuffle.file.buffer.kb` 	`32`
> `spark.reducer.maxMbInFlight` 	`48`
>  `spark.shuffle.manager` 	`sort`
> `spark.shuffle.sort.bypassMergeThreshold` 	`200`
> `spark.shuffle.blockTransferService` 	`netty`

Spark Sort:

```spark-submit --class com.github.ehiggs.spark.terasort.TeraSort --deploy-mode client --master yarn --num-executors 30 --driver-memory 1g --executor-memory 1g --executor-cores 2 --queue dev /tmp/spark-terasort-1.0-SNAPSHOT-jar-with-dependencies.jar /user/bijay/terasortInput /user/bijay/terasortOutput```

The resource usage for the sorting job with the **default Spark shuffle configuration** values:

> **Details for Stage 1**                                                  
 
|Parameters|Values
|--------------------|-------------          
| Total task times across all task|3.7 h                          |
|  Input                          | 93.1 GB
| Shuffle Write          | 5.0 GB
| Shuffle Spill (memory )| 307.8 MB
|Shuffle Spill (disk) | 25.4 MB

> **Details for stage 2**

| Parameters | Vlaue
|---------------------|---------------------
|Total task times across all task | 19.3 h
|Output   | 93.1 GB
|Shuffle Read | 5.0 GB
| Shuffle spill (memory) | 135.4 GB
| Shuffle spill (disk) | 4.0 GB

#### TeraSort after changing the Spark Shuffle Configuration

Following changes are made to default Spark Shuffle Configuration:
>`spark.shuffle.consolidateFiles=true`  ` create consolidates files during shuffle `
>`spark.shuffle.memoryFraction=0.4` `Fraction of Java heap to use for aggregation and cogroups during shuffles is increased by 2 times`
>`spark.shuffle.file.buffer.kb=64` `Size of the in-memory buffer for each shuffle file output stream, in kilobytes is increased by 2 times`
>`spark.reducer.maxMbInFlight=96` `Maximum size (in megabytes) of map outputs to fetch simultaneously from each reduce task is increased by 2 times`

 With the above configuration change the metrics for the TeraSort job is shown below:

> **Stage 1**                                                  
 
|Parameters|Values
|--------------------|-------------          
| Total task times across all task|3.6 h                          |
|  Input                          | 93.1 GB
| Shuffle Write          | 5.0 GB
| Shuffle Spill (memory )| 0 MB
|Shuffle Spill (disk) | 0 MB

> **Stage 2**

| Parameters | Vlaue
|---------------------|---------------------
|Total task times across all task | 14.5 h
|Output   | 93.1 GB
|Shuffle Read | 4.8 GB
| Shuffle spill (memory) | 122.5 GB GB
| Shuffle spill (disk) | 3.4 GB

These are the basic `Spark` and `YARN` parameters that can be used to tune to increse the `Spark` performance. There isn't any perfect configuration but it all depends on your jobs and workloads, data distribution and parallelizing of the workflow. You should try with different permuation before finding the one that fits your use case.












 
