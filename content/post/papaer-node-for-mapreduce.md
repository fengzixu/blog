---
title: "Paper Note — MapReduce"
date: 2020-05-04T17:59:42+09:00
draft: false
toc: true
tags: [
    "paper note"
]
categories: [
    "distributed system"
]
---

# Overview

If you are a software engineer, you must hear a tech concept — “MapReduce”. It is quite popular in the processing dataset on the large scale cluster. It was created by Google in 2004. With the development of technology, maybe Google has a new awesome method, but MapReduce has become an important sample for learning the principle of the distributed system.

This blog is a kind of paper note. Because I am learning the 6.824 Distributed System from MIT. Reading the MapReduce paper is one of the tasks of that class. So, if you are not familiar with the MapReduce or Distributed System, I will introduce “What is MapReduce” and “How MapReduce handles the large-scale data-set on distributed cluster” to you from an engineer’s perspective.

# What is the MapReduce
In a simple way, the MapReduce is a kind of programming model and associated implementation.

For the programming model of MapReduce, it is a kind of abstraction for business logic. There are two necessary interfaces were exposed to users. If users want to use a kind of MapReduce system, they need to provide implementations for both of them. Before that, users have to check if their business logic is matched with this programming model.

1. Map:  Map function accept two arguments that stand for a key-value pair. And it generates a set of intermediate key-value pairs by processing the input data.
2. Reduce: Reduce function is more like a kind of “Merge” operation. The input of the Reduce function comes from the output of the Map function. It will merge all intermediate values associated with the same intermediate key.

For the implementation part of MapReduce, It is the responsibility of a distributed system engineer, not developers using this system. They should construct a stable and high-performance distributed computing system for processing the job provided by users.

# What problems MapReduce wants to resolve
The most important reason of Why Google wants to create the MapReduce is to solve very practical requirement from other Google developers who are not the expert of the distributed system.

With the development of the Internet, more and more large-scale dataset needs to be processed. For example, Google needs to process large amounts of crawled documents, web request logs. So, Google developers implemented lots of special computing logic for processing different types of raw data.  But the majority of this computing logic is sequential. Limited by the resources and computing power of a single node, it may not be able to calculate the results in a meaningful time. Of course, those codes are also complex.

So, some Google experts in the distributed system area want to develop a new abstraction that can distribute computing tasks to several computing hosts and hide the complexity of this system for users. This kind of abstraction can save the time of processing the large-scale dataset by computing them in parallel.

Therefore, the problem to be solved by MapReduce is "how to enjoy the excellent computing performance brought by distributed systems without understanding the distributed systems"

# The Programming Model
The MapReduce programming model includes 2 parts

1. Map: Map function will get key-value pairs as the argument. It processes input pairs and produces a set of intermediate key-value pairs. The MapReduce library groups together all intermediate values associated with the same intermediate key as a new key-value pair. It can be passed to the Reduce function as the argument.
2. Reduce: Reduce function also gets the key-value pair as the argument. But the value of this pair is usually a vector object. So the Reduce should merge these values to form a possibly smaller set of values.

# Implementation
Until now, I explained the “What is the MapReduce” and “What is the programming model of MapReduce’” from the user’s perspective.

So, If I am a developer who wants to calculate the number of occurrences of each word in a large collection of documents. I just need to define two key functions like below and specify the input/output file name.

```go
func map(key, value string) {
// key: document name
// value: document content
for each_word range value {
		Emit(each_word, 1)
    }
}

func reduce(key string, values []int) {
// key: word
// values: a list of counts
    sum := 0
    for v range values {
      sum += v
    }
    Emit(key, sum)
}
```

Then, we can submit them to the MapReduce system, wait for a while and take a cup of coffee. Our MapReduce will help us to calculate the result.

Actually, the implementation part of a MapReduce system is quite complex. Because you have to ensure your system is stable and high-performance. You will process lots of problems in the distributed system area. Now, let’s go to the implementation of MapReduce by Google to learn “How Google constructs a distributing computing system”.

## Overview of implementation
Here is a graph that illustrates the complete process of one MapReduce call. You can combine the explanation below and this graph to understand the implementation of a kind of MapReduce system.

![](https://raw.githubusercontent.com/fengzixu/graph-bed/master/map%20reduce.png)

### Input Data
According to the description in MapReduce paper, all input/out files are stored in the GFS(Google File System). It is a kind of distributed file system. A large document will be partitioned into several smaller docs to store, typically 16 megabytes to 64 megabytes. Users can pass the optional argument to control the size of every file slice.

In addition to the raw data, the input for the MapReduce system also includes two functions that are provided by users: Map() and Reduce().

### Worker & Master
The MapReduce system starts up many copies of the programs on a cluster of machines. Although those programs share the same piece of code. Their roles are different.

One of the programs is special, we call it “Master”. The rest of the programs on the host of the cluster are “Worker”. The “Master” is more like the scheduler which is responsible for collecting necessary info from workers and distributing “Map” or “Reduce” tasks to workers. 

For workers, there are two types of tasks they need to do.

#### Map

The worker parses the input data from split data set as the key-value pairs and pass them to the Map() function was defined by users. The Map() function will produce the intermediate key-value pairs. Those intermediate data are stored in the memory temporarily.

Periodically, the buffered intermediate key-value pairs will be flushed to the local disk, partitioned into R regions by the partitioning function. The number of R is equal to the number of Reduce worker. The location info about these intermediate key-value pairs will be reported back to the Master. Because the Master is responsible for dispatching the task. When the Master wants to distribute the Reduce task to the worker, it should notify the location info of data to that worker. 

Several Map tasks usually were performed in parallel on Map workers. The number of parallel executions depends on the maximum value between the number of split files and the number of Map workers.


#### Reduce 

Master can assign a “Reduce” task to the worker. And this task includes the location info of intermediate key-value pairs that the worker needs to handle. After that, the Reduce worker read the buffered data set from the local disk of Map worker by invoking the RPC interface.

After the Reduce worker reads all intermediate data, it sorts them, so that all occurrences of the same key are group together. Maybe you feel confused about “Why the Reduce worker wants to sort intermediate data?”.  The most important reason is the different keys map to the same Reduce task. The arguments of Reduce() function are key and matched value lists separately. Before the Reduce worker invokes the user’s Reduce() function, it should ensure the same key are group together. Moreover, if the memory space is not enough to sort all intermediate data-set, it may need to use the external sort approach.

The Reduce worker iterates over sorted intermediate data set and pass the unique key and corresponding set of intermediate value list to the user’s Reduce() function. The final output will be appended to the file corresponding to this Reduce worker.

### Output Data

After a complete MapReduce call was finished, the final result usually was split into multiple output files. You can choose to merge them into one file. But users usually use those output as the input data set of another MapReduce call. Or they send those output files to another system which can handle the input data was partitioned into multiple files.

### Master Data Structure
The Master node plays an important role in the whole MapReduce system. In addition to distribute Map/Reduce tasks to workers, It is more like a conduit through which the location of intermediate file regions is propagated from the Map tasks to the Reduce tasks. 

So, the Master not only stores the state of every task(idle, processing or complete), but also saves the location info of intermediate key-value pairs were produced by the user’s Map() function.

The Master updates the location info of intermediate key-value pairs when the new Map task was finished. And then, it will post that location info to the Reduce worker.

## Pause For A While
Actually, in sections above, almost all of the words are more like explain “How the MapReduce system process a user’s request”. You can know the whole process of the invocation of MapReduce. But, a production-level MapReduce system needs to resolve more problems. Now, let ’s dive into the MapReduce system and see how the distributed computing cluster it depends on is constructed. 

## Fault Tolerance
Because the MapReduce system runs on a large-scale distributed cluster. Any node of this cluster may encounter any failure. It results in that node was off-line for a while or forever.

As I mentioned before, there are two types of nodes in the MapReduce system. One is Master, another is Worker. Let’s discuss them separately.

### Worker Failure

Like many other distributed systems, there is a heartbeat mechanism between the Master and Worker nodes in the MapReduce system. The Master also stores the healthy status of every worker. Once the Master doesn’t receive response from the worker within amount of time, it marks this worker as “Unreachable” or “failed”.

And then, Master will reset some tasks which are completed or processing by the failed Worker to the idle state, so that those resetting tasks can be re-scheduled. However, the processing of reset tasks is different for Map tasks and Reduce tasks. For Map tasks, whatever processing tasks or completed tasks will be reset.  For Reduce tasks, only the processing tasks need to be reset. The most important reason of don’t reset the completed Reduce tasks is Reduce worker writes the output to the file which stores in the GFS. GFS ensures the data produced by Reduce worker is high-reliable. But the intermediate key-value pairs were produced by the Map tasks were stored in the local disk of Map worker. During the failure happens, the Map worker is unreachable. So the MapReduce system needs to reset it.

![](https://raw.githubusercontent.com/fengzixu/graph-bed/master/worker-failure.png)

So, even if there are some workers are in the unreachable state, MapReduce can use “Reset” and “Re-Execute” approaches to move the process forward.

### Master Failure

Actually, according to the current implementation in MapReduce paper, the Master is single. In that case, if Master was gone, the whole MapReduce system will stop to execute computation works, of course, the user’s request wouldn’t be responded.

But, it is also easy to do the improvement for Master. The Master can periodically write the buffered state info to the GFS system. The GFS system will ensure those state info is high-reliable. Once the Master was gone, another Master copy can continue to handle tasks based on the last checkpoint state.

### Logic Failure

#### If user-specified Map() and Reduce() functions are deterministic

If user-specified Map() and Reduce() functions are deterministic, the MapReduce can ensure that the output of sequential and parallel execution is same. Actually, the MapReduce system utilizes the “atomic commit” mechanism to achieve this property. 

For Map task, it outputs the intermediate key-value pairs into multiple temporary files on the local disk. When it was completed, it reported location info of those temp files to the master. This “report” action is atomic. It means it is impossible to submit only the location information of part of temporary files. Either submit all location information or not submit. And if the Master received a completion message(includes location info) for an already completed Map task, it will ignore this message.

For Reduce task, after it wrote the calculating result to the temporary file, it will rename it. This “Rename” action is also atomic. If there are two same Reduce tasks run on different workers, the final file content just contains the data produced by one of Reduce tasks. The atomic rename was ensured by the file system the MapReduce system relies on.

#### If user-specified Map() and Reduce() functions are non-deterministic

Actually, MapReduce is only a kind of distributed computing system, it uses the powerful computing ability to execute Map() and Reduce() functions provided by users. So, if the logic of Map() and Reduce() functions are non-deterministic, the result must be non-deterministic. 

For example, there is a Map task called Map1. It outputs R partitioning files and stores them on the local disk. This location info was forwarded to the Reduce worker by a Reduce task. But, if the intermediate key-value pairs produced by Map1 were lost due to the failure of the disk, Master will re-run  Map1 task. If the user’s Map() function is non-deterministic, the output of Map() may be different from the previous execution. In that case, the output of the Reduce task must be different.

The above execution process can be regarded as two different MapReduce operations executed in sequential. For the results of different sequential operations, MapReduce can only guarantee that the results of each parallel operation are the same as its corresponding sequential operation results.

## Optimization

### Locality

In 2004, when the MapReduce was created, the bandwidth is not only relatively scarce resource but also a bottleneck of MapReduce.  For the graph of overview implementation, you can see there is much info that needs to be delivered by the network.

1. Master distributes the task to Map worker or Reduce worker
2. Map worker reads input data from the file system which may store the input file on other machines
3. Map worker reports intermediate key-value pair region info to Master
4. Read the intermediate key-value pairs from Map worker to Reduce worker

In Google, there is a unified file system called GFS. All input files used by the MapReduce system were stored by GFS. The GFS usually divides the input file into multiple blocks, the size of each block is usually 64MB. And each block has 3 copies at least which are stored on different machines.  The MapReduce system will share the same cluster with GFS. In that case, we can let Master records complete location info for all blocks of the input file. 

During the scheduling Map task, the Master can check the which input file Map task will process, and try its best to schedule the Map task to that host which includes all blocks the Map task needs. So that the Map task can read the input file from the local disk. Even if the Master doesn’t find the best node, it also schedules the Map task to the host located in the same network switch with the best node. 


### Task Granularity
Usually, the number of Map tasks is equal to the number of split files, we call it M. The number of Reduce task (R) usually was specified by users. Because it is equals to the number of output files.

According to the description of the MapReduce paper, the author suggests the number of tasks(Map + Reduce) is much larger than the number of worker machines in the cluster. There are two benefits.

#### Improve the dynamic load balancing

Actually, the load balancing is a kind of resource management problem. It more focuses on resource usage, other than computing tasks. Wikipedia gives a very clear explanation about load balancing

> In computing, load balancing refers to the process of distributing a set of tasks over a set of resources (computing units), with the aim of making their overall processing more efficient. Load balancing techniques can optimize the response time for each task, avoiding unevenly overloading compute nodes while other compute nodes are left idle.  

Dynamic Load Balancing is one of the approaches to make the resource usage of cluster balanced.

So, Why the author think more number of tasks can improve dynamic load balancing?  Please, think about a scenario like below:

There are 3 worker machines, but 4 tasks. Every task needs to cost 20 resources. Every node has 100 resources. In that case, if we want to distribute tasks to 3 worker machines, there is just a worker who takes two tasks. So the resource usage for every worker node like below:

1. Worker1: 40% (two tasks)
2. Worker2: 20%
3. Worker3: 20%

So, if we reduce the task granularity from 20 to 10 resources. There are 8 tasks. In that case, the resource usage for every worker node like this:

1. Worker1: 30% 
2. Worker2: 30%
3. Worker3: 20%

You can see that the number of tasks is greater than before, the load balancing of worker machines is better. Moreover, when you want to add or remove the worker machine, the smaller task granularity is easy to schedule for dynamic load balancing.

#### Speed up the recovery when a worker fails

The larger the number of tasks, the smaller the granularity of individual tasks. From the resource usage perspective, small tasks are easier to distribute as soon as possible than the big size. If one of the worker machines was down, the tasks it completed can be distributed to several healthy worker nodes to re-execute. 

### Backup Tasks
One MapReduce operation triggered by users usually was divided into multiple smaller Map and Reduce tasks. They were calculated on several worker machines. But, In the real production environment, engineers found some MapReduce operation takes an unusual long time due to one or several “Bad Machines”. We can call them “Straggler”. The “Straggler” usually has some small question which affects the processing of Map or Reduce task, such as bad disk, resource pressure(because there are other tasks run on that worker machine). In that case, this straggler will length the time of the whole MapReduce operation.

So, how MapReduce solve this problem? The answer is simple and general. It comes from a famous statement in the Computer Science area: Using space resources to exchange time.

When the MapReduce operation is close to completion, the Master will copy all tasks which were in the processing state. And Master schedules them to other worker machines.  Due to the “Atomic Commit” feature for Map and Reduce tasks I mentioned above,  whatever either the primary or the backup task executions complete, the task will be marked as the completed state.  Although, this approach typically increases the certain percentage of resource use. In the real production environment, it isn’t more than a few percents. But, it significantly reduces the processing time of the whole MapReduce operation.

After I read this paragraph, I think it is a typical sample of “ Using Space to exchange the time”. It is also a kind of trade-off in software engineering.

# Refinements
Until now, I have explained the basic functionality provided by the MapReduce. But, there are still some extension points can be used in the MapReduce system. It not only improve the user’s experience when they are using the system but also optimize the performance of the whole system. I will pick up some typical refinements to explain.

## Combiner Function
In some cases, each Map task may produce the numerous repetitive intermediate key-value pairs. For example, when you use the MapReduce to count the occurrence of each word in a document file, if the Map task read the content line by line,  it may produce hundreds or thousands of records like <the, 1>.  

As I mentioned before, all intermediate key-value pairs will be sent to the Reduce worker machine and the bandwidth resource is scare in 2004 at least. Too much repetition record increases the time of transferring data. So, can we do the “Combine” action for those repetitive records before they were sent? If we can, the number of transferring data will be significantly reduced.

The author calls this method “Combiner Function” in the MapReduce paper. It is an optional function that runs on the Map worker machine. User can choose to provide or not. It is almost the same as the logic of Reduce. The only difference is “Combiner Function” outputs the result to the temporary file and send it to the Reduce task, the Reduce task writes the result to the single output file.

From this extension point, you can see the bandwidth is the biggest bottleneck for the MapReduce system at that time.

## Skip Bad Records

When you use the MapReduce to process a large-scale data set, you may more focus on if this data set can be processed within a meaningful time successfully. In that case, you can accept to ignore some bad records which may cause the crash of the Map or Reduce task.

So, we can install a signal handler on every worker process and let it catches segmentation violations and bus errors. Before the Map() or Reduce() function is invoked, the MapReduce system will give a unique sequence number for the input argument. When the Map() or Reduce() function are crashed, the signal handler can report this error with the sequence number to the Master. If the Master finds it received too many error reports for the same sequence number, it can skip this record and remove the associated tasks for move the whole MapReduce job forward.

## Local Execution

Debug a program which runs on the distributed system is hard to do. In addition to the number of workers machines are large, We don’t even know which node handles the task we care about

Whatever the task is performed in sequential or parallel, the final result isn't changed, the only difference is the length of processing time. So, Google engineers also developed an alternative MapReduce system in a sequential version. It can run all tasks in sequential and in the local machines. 

When all Map or Reduce tasks are processed in sequential, we can use some debug tools(e.g. gdb) to help us check if there is a bug in our logic of Map() or Reduce() function.

# Performance
In the MapReduce paper,  it mentioned it used two cases to test the performance of MapReduce system. The first one is finding a particular pattern from 1TB data. The second one is sorting the 1TB data. 

The testing cluster has approximately 1800 machines. All machines are in the same hosting facility. The MapReduce job only runs on this cluster when the majority of machines are in idle state, usually on a weekend afternoon. The quota of each machine is like below:

1. CPU: 2 * 2GHz Xeon processors with HyperThreading enabled
2. MEM: 4GB (Around 1-1.5GB was reserved by other tasks running on that machine)
3. Disk: 2 * 160GB IDE disk
4. Network: A Gigabit Ethernet link

## Grep

The grep program looks for a 3 character pattern from the 1TB data set. The input data was split into M =15000 sub-files, every sub-file is 64MB. The final output was written to the single file, so R = 1. 

The whole processing time is around 150 seconds. It contains the 60 seconds of startup overhead. “Startup” means distribute programs to every worker machine, communicate with GFS to open the set of 1000 input files, and to get the information needed for the locality optimization.

![](https://raw.githubusercontent.com/fengzixu/graph-bed/master/peformance-1.jpg)

You can see the graph above. At the peak, the rate of input files scanned is up to the 30GB/s, in that time, there are 1764 workers were assigned the Map task. With the finishing of Map task, the rate starts to decrease and hits the 0 about 80 seconds.

## Sort
Actually, the analysis of performance of Sort program is quite complex. Honestly, I don’t know if the processing time of sorting 1TB data on the MapReduce system is enough good. Because I didn’t run the same sort program on the single worker machine or other distributed system. But there are still some points of performance report we need to know.

The 1TB data set was split into M=15000 files, the size of each split file is 64MB. The final result will be output to R = 4000 files by Reduce task. Here is the performance analysis graph as below

![](https://raw.githubusercontent.com/fengzixu/graph-bed/master/performance-2.jpg)

### The input rate is less than grep

In the normal execution condition, the intermediate key-value pairs of grep program and sort program are totally different. In the grep program, it only needs to record the matched pattern and its number of occurrences from every split file. The size of the intermediate pairs is negligible. It means the majority of intermediate result is useless for grep program. But, for Sort program, it should spend about half their time and I/O bandwidth writing intermediate pairs to the local disk, almost all data.

### The input rate is higher than shuffle rate and output rate

If you remember, in the implementation section of the MapReduce system, I mentioned an optimization for the input rate called “Locality”. The Master tries its best to schedule the Map task to the worker machine where it needs to read the data. So that, the Map task can read the input data from the local disk, not the network. 

The shuffle rate stands for forwarding the intermediate data from the Map task to Reduce task. Those data need to be sent by the network. So its rate is less Thant the input rate.

The shuffle rate is higher than the output rate. Because the final sorted result will be written to two copies. It was decided by the underlying file system(GFS). Because it needs to ensure our data is high-reliable and high-available by replication.

### Backup Optimization

In the no backup tasks condition, you can see that the final completion time of the whole MapReduce job is up to the 1283 seconds. It takes 44% time longer than the normal execution (usually 891 seconds). Because there is a long tail problem. A few abnormal tasks cost too much time when the majority of tasks we’re finished.

### Task killed

In the process of performance testing, the author simulates one of the disaster scenarios “Kill 200 worker process”. In that case, the Master cannot receive the hear-beat package from those 200 workers. The Master will re-deploy the worker process on those worker machines and re-execute completed/processing Map tasks and processing Reduce tasks.

# Conclusion
Honestly, MapReduce is the first paper I read. From the first reading to this paper node was finished, I spent 3 days. Not sure if it is valuable. But I try my best to find some sample in my past work experience to help myself to understand every optimization, every decision done by the author of MapReduce. Think why the author of MapReduce wants to do that is a very interesting thing for me.

There are several points leave deep impression me as below:

## Utilize the trade-off well
During constructing the large distributed computation system, the author of MapReduce encounters many difficult problems, such as limited bandwidth, long-tail problem increases processing time, how to improve the dynamic load balancing for the distributed cluster.  The most interesting thing is there is no perfect solution can solve every problem without any side effect. For example, if you want to resolve the long tail problem, you need to pay for more resources for backup tasks. After we compared the cost between more used resources or longer processing time, we choose the first one finally. Because, at the phase of close to complete the whole MapReduce job, distribute some resources to backup tasks is not a big problem. 

## The awesome system needs a challenging environment to test and optimize
After the basic MapReduce system run on the production environment for several days, many Google engineers start to improve and optimize it. Because more and more problems happen when the data scale increases dramatically or the user’s requirement for the completion time of MapReduce job improves. For example, it must be after many tests that it is found that the split input file size is between 16MB-64MB to get better computing efficiency.

## User First

Any awesome system is for resolving one problem or a kind of problem. The reason of MapReduce was born is for hiding the complexity of the distributed system and let developer who isn’t the expert of distributed system enjoy the benefit from the distributing computation.

# Reference

- [looking_for_clarification_on_googles_mapreduce](https://www.reddit.com/r/compsci/comments/1tdp9c/looking_for_clarification_on_googles_mapreduce)
- [MapReduce Paper](http://static.googleusercontent.com/media/research.google.com/zh-CN//archive/mapreduce-osdi04.pdf)