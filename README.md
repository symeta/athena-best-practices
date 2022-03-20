# athena-best-practices

## Athena Architecture
- Athena is a **managed serverless data analytics engine** 
- which is capable of analyzing **vast volume of data**, **cross source data**. 
- Athena leverages on **Presto to deal with DML**, 
- and **Hive to deal with DDL**.


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

## Athena Deep Dive Trouble-Shooting
### Query exhausted resources at this scale factor
- Athena uses Presto as a query engine which runs different stages of a query in memory
- for a small number of queries, and for certain operators, Presto brings all the data into a single node, and it may fail because it cannot spill pages to disk when memory is exhausted. the service team has rectified this for 'Group By' operator.
- in absence of spill to disk functionality on all operators, we recommend customers:
  * re-organize queries: 
    *(1) instead of select *, only include the columns that the customer needs
    *(2) optimize ORDER BY clause
    *(3) optimize Joins
    *(4) partition data
  * convert data into Parquet/ORC formats & consider partitioning data
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
- [detail link](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL#LanguageManualDDL-AddPartitions)

### Hive cursor error: Please reduce your request rate
```log
HIVE_CURSOR_ERROR: please reduce your request rate. (Service: Amazon S3: Status Code: 503; Eooro Code: SlowDown; Request ID: FE72543AC6E4116A)
```
- this error si because of S3 throttling, which means you have exceeded the request rates.
- here, the customer needs to partition the bucket to accommodate the higher request rate.
- if they are not consistently hitting these throttles, re-trying would take care of this, but Athena does not automatically retry.
- customer can follow best practices mentioned here: [Optimizing Amazon S3 Performance](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html)
- recommend storing and formatting the data in such a way that it is optimized for Big Data processing engines, which includes using columnar formats like Parquet or ORC.
- also, make sure that the file size is at least 256MB and not greater than 1GB.

### Hive partition schema mismatch
- customer might see error 'HIVE_PARTITION_SCHEMA_MISMATCH' when crawling a partitioned table in Glue.
- until the option of "InheritFromTable" was implemented, the customer ahd to update partitions manually
- to solve this error, we can edit the crawler and enable the setting that says [Update all new and existing partitions with metadata from the table], after that crawl the table again.
- in addition, you can set a crawler configuration option to InheritFromTable
- this option makes sure that partitions inhert from metadata properties such as their classification, input format, output format, SerDe information, and schema from their parent table.

### Hive cursor error - Cannot read value
- first, tell the customer to make sure that the columns data type and create table statement matches.
- this usually happens when we create a table using Glue crawler when the crawler reads the column as String instead of Bigint.
- to rectify this, once Glue crawler creates the table metadata, you can edit the table's column type and save it.
- after changing the column from type String to  BIG INT, then re-running the crawler again gives a new error message that says 'Internal Service Exception'
- previewing the table in Athena gives HIVE_PARTITION_SCHEMA_MISMATCH error
- this is because Glue Crawler Hive representation only supports String for Map type while Athena's Map type supports all primitive types as keys.
- when we try to update the type in Data Catalog and then recrawl, it causes 'InternalServiceException' because Crawler thinks it's invalid.

### Error: 'Hive write close error' while running CTAS
```log
'HIVE_WRITER_CLOSE_ERROR: Error committing write: java.lang.IllegalStateException: Reached max limit of upload attempts for part. 
 You may need to manually clean the data at location 's3://bucket/Unsaved/2019/02/14/tables/abc' before retring. 
 Athena will not delete data in your account.'
```
- usually occurs when Athena is trying to upload large number of files at once on the S3 bucket location which is causing S3 bucket throttling
- Athena uploads results in chunks in parallel and upload for some chunks may be canceled and retried if the upload isn't going well.
- may by a transient issue related to how long a particular part is taking to upload to S3, try to re-run the query
- might be due to the limit on the size of each part in the multi part upload
- reduce ammount of data scanned in query by adding partitions or use bucketing along with partitioning.
