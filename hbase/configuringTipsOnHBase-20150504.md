## Tips for configuring HBase
<!-- http://www.quora.com/What-are-some-tips-for-configuring-HBase -->
A great documentation is available on the Apache HBase wbsite (http://hbase.apache.org/). You can configure HBase at several levels. 

### System
Increase the default per-process file handle limit [3] in
```
/etc/security/limits.conf
```

### HDFS
Set `dfs.datanode.max.xceivers` to 2047 in
```
$HADOOP_HOME/conf/hdfs-site.xml
```
Set `dfs.datanode.socket.write.timeout` to 0 


### HBase
First, note that the default configuration values are stored at `src/main/resources/hbase-default.xml` in the source tree. For your site-specific configuration values, edit `conf/hbase-site.xml`
Set `hbase.rootdir` to point to the directory in HDFS where HBase will put its data; e.g.
```
hdfs://localhost:9000/hbase
```

### Per-Cluster
`hfile.block.cache.size` controls the amount of region server heap space to devote to the block cache. Currently defaults to 20%.


### Per-Table

- Max File Size: for clusters with lots of data, can be tuned up to 1 GB to result in less regions on the cluster.
- MemStore Flush Size
...

### Per-Family

- Compression
- Bloom filters


## Per-Region Server
`hbase.regionserver.global.memstore.upperLimit` is used to cap the amount of heap room in each region server to reserve for all MemStores served by that region. It defaults to 40% of the heap.
`hbase.hregion.memstore.flush.size` is the threshold for deciding when to flush a single MemStore to disk. It defaults to 64 MB.
`hbase.hregion.memstore.block.multiplier` controls when to start blocking writes to keep the MemStore size sane. It defaults to 2 (multiplied by the memstore.flush.size). For production clusters with lots of RAM that you monitor closely, you can up to something like 8.
`hbase.hregion.max.filesize` determines how big a StoreFile is allowed to grow before splitting a region. Defaults to 256 MB.


## Per-Store
`hbase.hstore.blockingStoreFiles` determines the maximum number of StoreFiles per Store to allow before blocking writes and forcing a compaction. The default is 7, but in production clusters monitored closely, it may make sense to up to 15.

