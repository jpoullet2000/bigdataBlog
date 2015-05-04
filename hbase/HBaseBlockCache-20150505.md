## HBase Blockcache

HBase supports block cache to improve read performance. 

### BlockCache contents
BlockCache makes it faster but how and what it does actually keeps in cache.

- **your query results**: each time a get or scan operation occurs, the result is added to the Block Cache (if it was not already)
- **row keys**: the row key corresponding to the value is also added, so keep the row key as small as possible
- **hbase:meta catalog**: this table tells which region servers serves which regions, it can consume several MB of cache if there are a large number of regions.
- **indexes of HFiles**: HFiles contain indexes allowing HBase to seek for data within them without needing to open the entire HFile. The size of an index is a factor of the block size, the size of your row keys, and the amount of data you are storing. For big datasets, the size can exceed 1GB per region server, although it is unlikely that the entire index will be in the cache at the same time.
- **bloom filters**: only if they are used, the main idea of bloom filters is to avoid reading data which are not requested or in other words reduce scans for key existence tests. If a column family supports bloom filters, that means that an extra index (1 byte per entry) is kept which helps cut down on the time necessary to determine if a given colum exists in a given row. Bloom filters are mainly useful when you have a very large number of variably named columns, each cell having a small amount of data. 

The HBase team has published the [results of exhaustive BlockCache testing](https://blogs.apache.org/hbase/entry/comparing_blockcache_deploys) on August 7th 2014, revealing the following guidelines:
- If the dataset fits completely in cache, the default configuration, which uses the *onheap LruBlockCache*, performs best.  GC is half that of the next most performant deploy type, CombinedBlockCache:Offheap with at least 20% more throughput.
- Otherwise, if your cache is experiencing churn running a steady stream of evictions, move your block cache offheap using *CombinedBlockCache in the offheap mode*. See the BlockCache section in the HBase Reference Guide for how to enable this deploy. Offheap mode requires only one third to one half of the GC of LruBlockCache when evictions are happening.


If the data needed for an operation does not fit in memory, using the BlockCache can be counter-productive. To bypass the BlockCache for a given Scan or Get, use the setCacheBlocks(false) method.
You can prevent a specific column family's contents from being cached by setting its BLOCKCACHE configuration to *false*. In HBase shell, use
```
hbase> alter 'myTable', 'myCF', CONFIGURATION => {BLOCKCACHE => 'false'}
```

More info can be found [here](http://www.cloudera.com/content/cloudera/en/documentation/core/latest/topics/admin_hbase_blockcache_configure.html).
