# athena-best-practices

## Athena Architecture
Athena is a managed serverless data analytics engine which is capable of analyzing vast volume of data, cross source data. Athena leverages on Presto to deal with DML, and Hive to deal with DDL.

## Athena Performance Tuning 
### data partitioning
the amount of data scanned by each query can be restricted by partitioning data, thus performance will be improved and cost will be reduced as well.
Athena leverages Hive for partitioning data. To create a table with partitions, PARTITIONED BY must to used to define keys by which to partition data when create table using the CREATE TABLE statement.

by using tehe column in the WHERE clous, the partitions that are scanned in a query can be restricted.

| Query | Non-Partitioned Table | Cost | Partitioned Table | Cost | Savings |
| ----- | Run Time | Data Scanned | Run Time | Data Scanned | ---- | ---- |
| SELECT count(*) FROM linetime WHERE l_shipdate = '1996-09-01' | 9.71 seconds | 74.1GB | 2.16 seconds | 29.06 MB | $0.0001 | 99% cheaper, 77% faster |
