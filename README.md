# athena-best-practices

## Athena Architecture
Athena is a managed serverless data analytics engine which is capable of analyzing vast volume of data, cross source data. Athena leverages on Presto to deal with DML, and Hive to deal with DDL.

## Athena Performance Tuning 
### data partitioning
the amount of data scanned by each query can be restricted by partitioning data, thus performance will be improved and cost will be reduced as well.
Athena leverages Hive for partitioning data. To create a table with partitions, PARTITIONED BY must to used to define keys by which to partition data when create table using the CREATE TABLE statement.

by using tehe column in the WHERE clous, the partitions that are scanned in a query can be restricted.
-athena bp1

### how to dicide which columns to partition on
| Hive Style Partitioning | Non Hive Style Partitioning |
| ----------------------- | --------------------------- |
| mys3bucket/myprefix/year=2020/month=06/day=15 | mys3bucket/myprefix/2020/06/15 |
