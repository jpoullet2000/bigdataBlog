## HBase response times

<!-- http://hadoop-hbase.blogspot.com/2014/08/hbase-client-response-times.html -->

There are several causes to latency in HBase:

- Normal network round-trip time (RTT), internal or external to the Hadoop cluster, *order of ms* 
- Disk read and write time, *order of ms*
- Client retries due to moved regions of splits, *order of s*; HBase can move a region if it considers that the cluster is not well balanced, regions are splitted when they become too large in size (these great features are built in HBase and called *auto-sharding*), note that the time is not dependent on the region size since no data is actually moved, just the ownership is transferred to another region server.
- Garbage collector (GC), *order of 50ms*.
- Server outages, when a region server dies it takes by default 90s to detect, then the server regions are reassigned to other servers, which then have to replay logs (WAL) that could not be run. Depending on the amount of uncommited data, this can take several minutes. So in total it takes a *few minutes*. Note that logs are automatically replayed since recent versions of HBase (hbase.master.distributed.log.replay set to true).
- Cluster overwhelming if too many clients are trying to write data into HBase, it could be too many user queries, flushes, major compactions, etc, so that CPU, RAM or I/O can not follow. Major compactions can take *minutes* or even *hours*. One can limit the number of concurrent mappers by setting `hbase.regionserver.handler.count`. If memstores takes too much room in RAM, reduce their size by setting `hbase.regionserver.global.memstore.size`. Too many flushes (for instance if you have too many column families) may result into too many HFiles (format in which HBase store data in HDFS) and an extra overload can be created for minor compaction.  


Typically, latency will be of a few ms if things are in blockcache, or <20ms when disk seeks are needed. It can take 1 or 2s in cases of GC, region splits/moves. If latency goes higher, this is the result of a major compaction or more serious problem may have occured (server failure, etc).

