# A Spillable State Backend for Apache Flink

## Introduction

`HeapKeyedStateBackend` is one of the two `KeyedStateBackend` in Flink, since state lives as Java objects on the heap in `HeapKeyedStateBackend` and the de/serialization only happens during state snapshot and restore, it outperforms `RocksDBKeyeStateBackend` when all data could reside in memory.

However, along with the advantage, `HeapKeyedStateBackend` also has its shortcomings, and the most painful one is the difficulty to estimate the maximum heap size (Xmx) to set, and we will suffer from GC impact once the heap memory is not enough to hold all state data. There’re several (inevitable) causes for such scenario, including (but not limited to):

* Memory overhead of Java object representation (tens of times of the serialized data size).
* Data flood caused by burst traffic.
* Data accumulation caused by source malfunction.

To resolve this problem, we proposed a new `SpillableKeyedStateBackend` to support spilling state data to disk before heap memory is exhausted. We will monitor the heap usage and choose the coldest data to spill, and reload them when heap memory is regained after data removing or TTL expiration, automatically.

Similar to the idea of [Anti-Caching approach](http://www.vldb.org/pvldb/vol6/p1942-debrabant.pdf) proposed for database, the main difference between supporting data spilling in `HeapKeyedStateBackend` and adding a big cache for `RocksDBKeyedStateBackend` is now memory is the primary storage device than disk. Data is either in memory or on disk instead of in both places at the same time thus saving the cost to prevent inconsistency, and rather than starting with data on disk and reading hot data into cache, data starts in memory and cold data is evicted to disk.

More details please refer to [FLIP-50](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=125307861), and the upstreaming work is in progress through [FLINK-12692](https://issues.apache.org/jira/browse/FLINK-12692).
We setup this repository as a preview version for those who want to try it out before Apache Flink officially supports it.

## How to build the binary

Please note that we only support Flink 1.10.x and later releases.

You can use the below commands to build a binary to run with Flink 1.10.x releases:

```
git clone https://github.com/realtime-storage-engine/flink-spillable-statebackend.git
cd flink-spillable-statebackend/flink-statebackend-heap-spillable
git checkout origin/release-1.10
mvn clean package -Dflink.version=<flink version>
```

And the below commands against current Flink master:

```
git clone https://github.com/realtime-storage-engine/flink-spillable-statebackend.git
cd flink-spillable-statebackend/flink-statebackend-heap-spillable
mvn clean package -Dflink.version=<flink.version>
```

The JAR file will be generated in `target` directory with name of `flink-statebackend-heap-spillable-<flink version>`.
For example, `flink-statebackend-heap-spillable-1.10.1.jar` for Flink 1.10.1.

## How to deploy

First of all, copy the compiled JAR file into the `lib` directory of your Flink deployment

And then set your state backend to `SpillableStateBackend`. There are two ways to achieve this:

1. Set `state.backend` to `org.apache.flink.runtime.state.heap.SpillableStateBackendFactory` in flink-conf.yaml
    
2. Set via java API

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStateBackend(new SpillableStateBackend(checkpointPath));
```

And you need to add the below dependency to your application before compilation:

```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-statebackend-heap-spillable</artifactId>
  <version>${flink.version}</version>
</dependency>
```

## Configurations

| Key | Default | Type | Description |
| -------------     | ------ | --------- | ---- |
|state.backend.spillable.chunk-size|512MB|MemorySize|Size of a chunk (mmap file) which should be a power of two|
|state.backend.spillable.heap-status.check-interval|1min|Duration|Interval to check heap status|
|state.backend.spillable.gc-time.threshold|2s|Duration|If garbage collection time exceeds the threshold, state will be spilled|
|state.backend.spillable.spill-size.ratio|0.2|Float|Ratio of retained state to spill in a turn|
|state.backend.spillable.load-start.ratio|0.1|Float|If memory usage is under this watermark, state  will be loaded into memory |
|state.backend.spillable.load-end.ratio|0.3|Float|Memory usage can't exceed this watermark after state is loaded|
|state.backend.spillable.trigger-interval|1min|Duration|Interval to trigger continuous spill/load|
|state.backend.spillable.cancel.checkpoint|true|Boolean|Whether to cancel running checkpoint before spill. Cancelling checkpoints can release the spilled states and make them gc faster|

## Performance

We use a `WordCount` benchmark to compare performance of `HeapKeyedStateBackend`, `SpillableKeyedStateBackend` and `RocksDBKeyedStateBackend`.
You can find the source code of the benchmark as well as how to execute it in the `flink-spillable-benchmark` module of this project.

With the below hardware environment and job setup:

* Machine
	* 96 Cores, Intel(R) Xeon(R) Platinum 8163 CPU @ 2.50GHz
	* 512GB memory
	* 12 x 7.3T HDD storage
* Job
	* One slot per TaskManager
	* Job parallelism: 1
	* Disable slot sharing and operator chaining
	* Checkpoint interval: 1min
	* 20 Million keys, and each key is a 16-length `String`
	* Word sending rate: 1M words/s
	
* StateBackend
	* `HeapKeyedStateBackend`: 10GB heap
	* `SpillableStateBackend`: 3GB heap, with the default configuration
	* `RocksDBStateBackend`: 3GB block cache, disable `state.backend.rocksdb.memory.managed`, all other configurations with default setting

we got the below result:

| StateBackend | TPS | Description |
| -------------     | ---------- | ---- |
|`HeapKeyedStateBackend`|720K records/s||
|`SpillableStateBackend`|130K records/s|47 out of 128 (36.71%) key groups are spilled to disk|
|`RocksDBStateBackend`|60K records/s||

