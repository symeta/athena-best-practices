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
| Gzip(DEFLATE) | No| High | Medium |
| bzip2 | Yes | Very High | Slow |
| LZO | No | Low | Fast |
| Snappy | No | Low | Very Fast |

- Most popular columnar formats - Parquet and ORC. Default algorithm Parquet applies is Snappy, it can also support GZIP and LZO. Default algorithm ORC applies is ZLIB, it can also support Snappy.
- Generally, better compression ratios or skipping blocks of data means reading fewer bytes from Amazon S3, leading to better query performance
- One parameter that could be tuned is the block size or stripe size. The block size in Parquet or stripe size in ORC represent the maximum number rows that can fit into one block in terms of size in bytes. Default: 128MB for Parquet, 64MB for ORC.
- if block size/stripe size is small - more data being scanned by the query

