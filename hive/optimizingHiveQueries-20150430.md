## Tips for efficient Hive queries

Hive on Hadoop is a great data processing tool which is easy to use given its SQL-like syntax. Some tips to optimize Hive queries are described in this article.

Typically there are 3 areas where you can optimize you Hive queries:

- data layout (partitions, buckets)
- data sampling 
- data processing (map join, parallel execution) 


### Data layout tips

#### Partitioning Hive tables

The idea is to partition the data based on some dimension that is often used in queries in the *where* statement (ex: `select * from user_table where region = 'Europe'`. Note that it is also possible to partition data based on several dimensions. Partitioning the data largely reduces the number of read data, and so reduces the number of mappers, I/O operations and time to answer the query.
To partition a table one can just use the following statement
```
CREATE TABLE user_table
...
PARTITIONED BY (region STRING)
```
Be careful: do not partition if the cardinality (number of unique values) of the column is too high which would results in too many partitions, especially if there is a high risk that the column used for partitioning will not be used as filter in all queries. Concretely, there is a directory per partition and then subdirectories for subpartitions (if you use more than 1 column for partitioning), creating a huge overhead if you need to parse all these directories and files. Moreover, HDFS uses large block size of typically 64 MB or more, which means that each file, even with a few bytes of data, will have to allocate that block size on HDFS, potentially resulting in a waste of disk space.


#### Bucketing Hive tables
Bucketing is useful if there is a column that is frequently used for join operations. To create buckets (it is just a way to hash data and store it by hash results):

```
CREATE TABLE user_table
...
CLUSTERED BY (country) INTO 64 BUCKETS
```
and to add data from another existing table

```
set hive.enforce.bucketing=true;
INSERT OVERWRITE TABLE user_table
SELECT ...
FROM ...
```
Then to make sure you only join the relevant data

```
set hive.optimize.bucketmapjoin=true
SELECT /*+MAPJOIN*/a.*,b.*
FROM user_table a JOIN country_attributes b
ON a.country = b.country
```

### Sampling

#### Bucket sampling
In the exploration phase of the data, one may want to address only part of the data. Sampling may be complicated if you need to join tables because if you select data randomly from 2 tables, what results from the *join* statement may be empty. That is another case where the bucketing explained above is great.
```
SELECT a.*, b.*
FROM user_table (bucket 30 out of 64 on country) a,
     country_attributes TABLESAMPLE(bucket 30 out of 64 on country) b
WHERE a.country = b.country
```

#### Block or random sampling
If you need to sample data from 1 table in a random way, one can use the following
```
SELECT *
FROM user_table TABLESAMPLE(1 PERCENT)
```

### Parallel processing
The idea is to parallelize stages that are sequentially done by Hive by default. This is particularly interesting when you have such queries
```
SELECT a.*
FROM
(
	SELECT ...
	FROM ...
) a
JOIN
(
	SELECT ...
	FROM ...
) b
ON
( a.country = b.country)
GROUP BY a.postcode
```
We have the following *sequential* steps:

- Stage 1: *a* table
- Stage 2: *b* table
- Stage 3: join

With the option `set hive.exec.parallel=true;`, stage 1 and stage 2 are done in parallel and stage 3 afterwards.
