# athena-best-practices

## Athena Architecture
Athena is a managed serverless data analytics engine which is capable of analyzing vast volume of data, cross source data. Athena leverages on Presto to deal with DML, and Hive to deal with DDL.

## Athena Performance Tuning 
### data partitioning
the amount of data scanned by each query can be restricted by partitioning data, thus performance will be improved and cost will be reduced as well.
Athena leverages Hive for partitioning data. To create a table with partitions, PARTITIONED BY must to used to define keys by which to partition data when create table using the CREATE TABLE statement.

by using tehe column in the WHERE clous, the partitions that are scanned in a query can be restricted.
-athena bp1

#### how to dicide which columns to partition on
| Hive Style Partitioning | Non Hive Style Partitioning |
| ----------------------- | --------------------------- |
| mys3bucket/myprefix/year=2020/month=06/day=15 | mys3bucket/myprefix/2020/06/15 |

- columns that are used as filters are good candidates for partitioning
- as the number of partitions in your table increases, the higher the overhead of retrieving and processing the partition metadata, the smaller your files
- if your data is heavily skewed to one partitoin value, and most queries use that value, then the overhead may wipe out the initial benefit
- [link for details](https://docs.aws.amazon.com/athena/latest/ug/partitions.html) 


### data bucketing
- bucketing the data within a single partition is another way to partition data
- with bucketing, you can specify one or more columns containing rows that you want to group together, and put those rows into multiple buckets
- allows you to quey only the bucket that you need to read when the buckted columns value is specified, which can dramatically reduce the number of data to read
- within Athena, you can specify the bucketed column inside your Create Table statement by specifying CLUSTERED BY (<buckted columns>) INTO <number of bucktes> BUCKETS
- the number of buckets should be so that the files are of optimal size

```SQL
CREATE TABLE user_info_bucketed(user_id BIGINT, firstname STRING, lastname STRING)
COMMENT 'A bucketed copy of user_info'
PARTITIONED BY(ds STRING)
CLUSTERED BY(user_id) INTO 256 BUCKETS;
```
### Compress and split files
- compressing data can speed up queries significantly, as long as the files are either an optimal size, or the files are splittable
- splittable files allow the execution engine in Athena to split the reading of a file by multiple readers in increase parallelism
- if you have a single unsplittable file, then only a single reader can read the file, while all other readers site idle. Not all compression algorithms are splittable
- generally, the higher the compression ration of an algorithm, the more CPU is required to compress and decompress data
 
| Algorithm | Splittable? | Compression ratio | Compress + Decompress speed |
| --------- | ----------- | ----------------- | --------------------------- |
| Gzip | No| High | Medium |
| bzip2 | Yes | Very High | Slow |
| LZO | No | Low | Fast |
| Snappy | No | Low | Very Fast |

- Most popular columnar formats - Parquet and ORC. Default algorithm Parquet applies is Snappy, it can also support GZIP and LZO. Default algorithm ORC applies is ZLIB, it can also support Snappy as well as no compression.
- Generally, better compression ratios or skipping blocks of data means reading fewer bytes from Amazon S3, leading to better query performance
- One parameter that could be tuned is the block size or stripe size. The block size in Parquet or stripe size in ORC represent the maximum number rows that can fit into one block in terms of size in bytes. Default: 128MB for Parquet, 64MB for ORC.
- if block size/stripe size is small - more data being scanned by the query

### Optimize ORDER BY clause
- the ORDER BY clause returns data in sorted order
- in order to do this, Presto must send all rows of data to a single worker node first and them sort them
- it is advisable to look at the top pr bottom N values while using ORDER BY clause, then use a LIMIT clause to reduce the cost of the sort significantly by pushing the sorting and limiting to individual worker nodes, rather than the sorting being done in an single worker.

Athena-bp2
 
### Optimize JOIN clause
- Presto does not support join reordering yet so, it will perform joins from left to right.
- You should specify the tables from largest to smallest while ensuring two tables are not specified together the will result in a cross join

Athena-bp3

### Optimize Group By Cluase
- the GROUP BY operator distributes rows based on the GROUP BY colmns to worker nodes, which hold the GROUP BY values in memory
- the GROUP BY columns are looked up in memory and the values are compared as the rows are being ingested.
- the values are then aggregated together when the GROUP BY column matches.
- When using GROUP BY in your query, order the columns by the cardinality by the highest cardinality (that is, most number of unique values, distributed evenly) to the lowest.
- example: 
 ```SQL
 select state, gender, count(*) from tablename GROUP BY state, gender
 ```
- another optimization will be to use numbers instead of strings, as numbers require less memory to store and are faster to compare than strings
- the numbers represent the locdation of the grouped column name ins the SELECT statement
- example: 
 ```SQL
 select state, gender, count(*) from tablename GROUP BY 1, 2
 ```
- a 3rd optimization is to limit the number of columns within the SELECT statement to reduce the amount of memory required to store, as rows are held in memory and aggregated for the GROUP BY clause.

### Optimize LIKE operator
- regular expressions are better to use instead of the LIKE clause when you are filtering for multiple values on a string column
- this is particularly useful when you are comparing a long list of values

 Athena bp4
### Use approximate functions
- when exploring larger datasets, a common use case is to find the count of distinct values for a certain column usign COUNT(DISTINCT column)
- if you are looking for which webpages to dive deep into, then using APPROX_DISTINCT() is feasible
 
 Athena bp5

### Use LIMIT clause
- instead of selecting all the columns while running a query, LIMIT the final SELECT statement to only the columns that you need
- this reduces the number of data that needs to be processed through the entire query execution pipeline
- this is also helpful when you perform multiple joins or aggregations on a query

 Athena bp5

### Optimize file size
- Queries run more efficiently when reading data can be parallelized and when blocks of data can be read sequentially
- if files are too small (generally less than 128MB), the execution engine might be spending additional time.
- if a file is not splittable and the files are too large, the query processing waits until a single reader has completed reading the entire file. That can reduce parallelism
- one remedy to solve the small file problem is to use the S3DistCP utility on Amazon EMR
  * [S3DistCP Detail](https://docs.aws.amazon.com/emr/latest/ReleaseGuide/UsingEMR_s3distcp.html)
- the below are few benefits of having larger files:
  * Faster listing
  * Fewer amazon s3 requests
  * Less metadata to manager

### Generic internal error: Value 82183 exceeds MAX_SHORT
- the customer sees an error message that says 'transient issue' but the stack says 'value 82183 exceed MAX_SHORT'
- SOLUTION: download the EMR logs to find the corrupted file or change the data type of the column in INT.

### Cannot find query status of DDL
```log
 QUERY NOT_FOUND msg: "Cannot find query status of DDL: 56b07e5d-1c21-4723-80af-c9fbf3ff01e5"
```
- the query was not able to get DDL status
- known limitation while running "MSCK repair" or "show partitions" - due to too many partitions in the table or too many objects in the partition
- the walkaround is to write a script using:
```SQL
 ALTER TABLE table_name ADD [if not exists] PARTITION
```

