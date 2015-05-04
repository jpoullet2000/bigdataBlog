## How to configure HBase on low memory cluster

<!-- Ref: http://product.hubspot.com/blog/hbase-tutorial-5-tips-for-running-on-low-memory-ec2 -->

### Reduce the number of regions per server
Before getting into math, let's recall briefly what the memstore is in HBase. The memstore holds in-memory modifications to the Store before it is flushed to the Store as HFile, in other words the data that are coming in are first stored in memory, and when their volume reaches a certain size (typically 128MB on recent HBase versions), the data are flushed on disk.

Now, let's do some computation. You have 1 memstore per region. Suppose that each memstore is 128 MB. If you have 100 regions for each region server, you would have `100*128MB`, i.e. about 13 GB RAM. Keep in mind that only 40% is actually used for memstores, which means that each of the region server would at least need `13 GB/0.4` about 33 GB RAM. Of course, you might need a bit less RAM since not all of your memstores will be full at all times. 

Suppose you have chosen 100 regions because you need to store 1 TB with 10 GB regions. What can you do if you just have 32 GB RAM and do not want to monitor all your nodes 24/7? (note that if you need to store 1 TB you need 3 TB disk space if you chose the default replication of 3, remind that if the data are compressed on disk they are not in memory).    

- you can increase your region size, say 20GB each (`hbase.hregion.max.filesize`)
- you can decrease you memstore size, say 64MB (`hbase.hregion.memstore.flush.size`)
- you can increase the heap fractions used for the memstores. If you load is write-heavy, say maybe up to 50% of the heap (`hbase.regionserver.global.memstore.upperLimit`, `hbase.regionserver.global.memstore.lowerLimit`)

### Steal memory from other services 
A typical configuration calls for 1GB RAM for a datanode, which is often not needed. You can cut datanode head down to 400 MB, giving 624MB extra to HBase. This solution will not probably safe you much. 

### Tune or disable MSLAB
The MSLAB feature adds 2MB of heap overhead by default for each region. You can tune this buffer down with `hbase.hregion.memstore.mslab.chunksize`. The lower you go the less effective it is, but the less memory overhead as well. To disable it completely, set `hbase.hregion.memstore.mslab.enabled` to false.

### Reduce caching
Caching and batching are used to reduce the effect of network latency on large scans, but they also require more memory on both the client and server side. 

### Limit concurrent mappers
Lowering the `hbase.regionserver.handler.count` helps to limit the number of active connections taking memory. 

