## Hot Restart Store

![](images/enterprise-onlycopy.jpg)
<br></br>

![image](images/NoteSmall.jpg) ***NOTE:*** *This feature is still in progress and will be fully available with the final release of Hazelcast 3.6.*


![image](images/NoteSmall.jpg) ***NOTE:*** *This feature is supported for Hazelcast Enterprise 3.6 or higher.*


This chapter explains the Hazelcast's Hot Restart Store feature which provides fast cluster restarts by storing the states of the cluster members into the disk. This feature is currently provided for Hazelcast map data structure and Hazelcast JCache implementation.

### Hot Restart Store Overview

Hot Restart Store enables you to get your cluster up and running swiftly after a cluster restart that can be caused by a planned shutdown (including rolling upgrades) or a sudden cluster-wide crash (e.g. power outage). For this purpose, required states for Hazelcast clusters and members are introduced. Please refer to the [Managing Cluster and Member States section](#managing-cluster-and-member-states) for information on the cluster and member states.

Hot Restart feature is supported for the following restart types:

- **Restart after a planned shutdown**:
	- The cluster is shutdown completely and restarted with the exact same previous setup and data.

		You can shutdown the cluster completely using the method `HazelcastInstance.getCluster().shutdown()` or manually changing the cluster state to `PASSIVE` and then shutting down each member one by one. When you send the command to shut the cluster down, i.e. `HazelcastInstance.getCluster().shutdown()`, the members that are not in the `PASSIVE` state change their states to `PASSIVE`. Then, each member shuts itself down by calling the method `HazelcastInstance.shutdown()`.

	- Rolling upgrade: The cluster is restarted intentionally member by member, for example to install an operating system patch or a new hardware.

		To be able to shutdown the cluster member by member as part of a planned restart, each member in the cluster should be in the `FROZEN` or `PASSIVE` state. After cluster state is changed to `FROZEN` or `PASSIVE` state, each member can be shutdown manually, by calling the method `HazelcastInstance.shutdown()`. When that member is restarted, it will rejoin to the running cluster. After all members are restarted, cluster state can be changed to `ACTIVE` back.

- **Restart after a cluster crash**: The cluster is restarted after its all members are crashed at the same time due to a power outage, networking interruptions, etc.


During the restart process, before loading data, each member waits until all members present in the partition table are started. During this particular process, no operations are allowed. Once all cluster members are started, Hazelcast changes the cluster state to `PASSIVE` and starts to load data. When all data is loaded, Hazelcast changes the cluster state to its previous known state before shutdown and starts to accept operations which are allowed by the restored cluster state.

If a member fails to start or join to the cluster in time or fails to load its data, then that member will be terminated immediately. After fixing the problems causing the failure, that member can be restarted back. If cluster start cannot be completed in time, then all members will fail to start. Please refer to the [Configuring Hot Restart section](#configuring-hot-restart) for defining timeouts.

In the case of restart after a cluster crash, Hot Restart feature realizes that it was not a clean shutdown and Hazelcast tries to restart the cluster with the last saved data following the process explained in the above paragraphs. In some cases, specifically when cluster crashes while there is an ongoing partition migration process in the cluster, currently it is not possible to restore last saved state.


### Configuring Hot Restart

You can configure Hot Restart programmatically or declaratively. The configuration includes elements to enable/disable the feature, to specify the directory where the Hot Restart data will be stored and to define timeout values.


The following are example configurations for a Hazelcast map and JCache implementation.

**Declarative Configuration**:

An example configuration is shown below.

```xml
<hazelcast>
   ...
   <hot-restart enabled="true">
	   <home-dir>/mnt/hot-restart</home-dir>
	   <validation-timeout-seconds>120</validation-timeout-seconds>
	   <data-load-timeout-seconds>900</data-load-timeout-seconds>
   </hot-restart>
   ...
   <map>
      <hot-restart-enabled>true</hot-restart-enabled>
   </map>
   ...
   <cache>
      <hot-restart-enabled>true</hot-restart-enabled>
   </cache>
   ...
</hazelcast>
```


**Programmatic Configuration**:

The programmatic equivalent of the above declarative configuration is shown below.

```java
HotRestartConfig hotRestartConfig = new HotRestartConfig();
hotRestartConfig.setEnabled(true);
hotRestartConfig.setHomeDir(new File("/mnt/hot-restart"));
hotRestartConfig.setValidationTimeoutSeconds(120);
hotRestartConfig.setDataLoadTimeoutSeconds(900);
config.setHotRestartConfig(hotRestartConfig);

...
MapConfig mapConfig = new MapConfig();
mapConfig.setHotRestartEnabled(true);

...
CacheConfig cacheConfig = new CacheConfig();
cacheConfig.setHotRestartEnabled(true);
```

The following are the descriptions of configuration elements:

- `hot-restart`: The configuration that includes the element `home-dir` used to specify the directory where Hot Restart data will be stored. Its default value is `hot-restart` and it is mandatory to give a value. You can use the default one or specify another directory.
- `validation-timeout-seconds`: Validation timeout for Hot Restart process when validating the cluster members expected to join and the partition table on the whole cluster.
- `data-load-timeout-seconds`: Data load timeout for Hot Restart process. All members in the cluster should complete restoring their local data before this timeout.
- `hot-restart-enabled`: The configuration that is used to enable or disable the Hot Restart feature. This element is used for the supported data structures (in the above examples, you can see that it is included in `map` and `cache`).



### IP Address and Port

Hazelcast relies on the IP address-port pair as a unique identifier of a cluster member. The member must restart with these settings the same as before shutdown. Otherwise, Hot Restart fails.

### Hot Restart Store Design Details

Hazelcast's Hot Restart Store uses the log-structured storage approach. The following is a top-level design description:

- The only kind of update operation on persistent data is _appending_.
- What is appended are facts about events that happened to the data model represented by the store; either a new value was assigned to a key or a key was removed.
- Each record associated with key K makes the previous record associated with the same key stale.
- Stale records contribute to the amount of _garbage_ present in the persistent storage.
- Measures are taken to remove garbage from the storage.

This kind of design focuses almost all of the system's complexity into the garbage collection (GC) process, stripping down the client's operation to the bare necessity of guaranteeing persistent behavior: a simple file append operation. Consequently, the latency of operations is close to the theoretical minimum in almost all cases. Complications arise only during the prolonged periods of maximum load, and this is where the details of the GC process begin to matter.

### Concurrent, Incremental, Generational GC

In order to maintain the lowest possible footprint in the update operation latency, the following properties are built into the garbage collection process:

- There is a dedicated thread which performs the GC. In Hazelcast terms, this thread is called the Collector and the application thread is called the Mutator.
<br></br>
- On each update, there is metadata to be maintained; this is done asynchronously by the Collector thread. The Mutator enqueues update events to the Collector's work queue.
<br></br>
- The Collector keeps draining its work queue at all times, including while going through the GC cycle. Updates are taken into account at each stage in the GC cycle, preventing the copying of already dead records into compacted files.
<br></br>
- All GC-induced I/O competes for the same resources as the Mutator's update operations. Therefore, measures are taken to minimize the amount of I/O done during GC. Additionally measures are taken to achieve a good interleaving of Collector and Mutator operations, minimizing latency outliers perceived by the Mutator.
<br></br>
- I/O minimization is subject to a bet on the Weak Generational Garbage Hypothesis, which states that a new record entering the system is likely to become garbage soon. In other words, a key updated now is more likely than average to be updated again soon.

The I/O minimization scheme was taken from the seminal Sprite LFS paper, [Rosenblum, Ousterhout, _The Design and Implementation of a Log-Structured File System_](http://www.cs.berkeley.edu/~brewer/cs262/LFS.pdf). The following is the outline:

- Data is not written to one huge file, but to many files of moderate size (8 MB) called "chunks".
- Garbage is collected incrementally, i.e. by choosing several chunks, copying all their live data to new chunks, then deleting the old ones.
- I/O is minimized using a collection technique which results in a bimodal distribution of chunks with respect to their garbage content: most files are either almost all live data or all garbage.
- The technique consists of two main principles:
    1. Chunks are selected based on their _Cost-Benefit factor_ (see below).
    2. Records are sorted by age before copying to new chunks.

The Cost-Benefit factor of a chunk consists of two components multiplied together:

1. The ratio of benefit (amount of garbage that can be collected) to I/O cost (amount of live data to be written).
2. The age of the data in the chunk, measured as the age of the youngest record it contains.

The essence is in the second component: given equal amount of garbage in all chunks, it will make the young ones less attractive to the Collector. Assuming the generational garbage hypothesis, this will allow the young chunks to quickly accumulate more garbage.

Sorting records by age will group young records together in a single chunk and will do the same for older records. Therefore the chunks will either tend to keep their data live for a longer time, or quickly become full of garbage.