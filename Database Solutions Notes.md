# Problems of Traditional Databases
1. Database import is not transactional and import speed is limited.
2. Traditional database solution cannot solve OLTP and OLAP at the same time.
3. Traditional Database solution (like MySQL) is not distributed. Availibility and Scaling are the issues.

# Requirements of Migrating from Old to New
1. Support loading different data sources.

# Requirements of the NewSQL
1. Performant on OLAP/OLTP.
2. Scaling on Read/Write/Capacity.

# Solutions
## TiDB
TiDB has a raft-based storage engine TiKV (row-based storage), and a raft-based storage engine TiFlash (column-based storage, learner only). TiKV is used for writes. TiFlash is used for analytic workloads.

To embrace Hadoop eco-system, TiDB has a Spark extension called TiSpark. TiSpark is used for quick data load from hadoop eco-system. Also, TiSpark is able to leverage on SparkSQL/Spark  to handle some OLAP workloads in Spark. This I suppose will help customer easily migrate Spark workloads to TiDB.

To support **Isolation** between OLAP and OLTP workload, TiDB uses TiFlash to synchronize DML/DDL changes from TiKV key-based regions to TiFlash partitions. Conceptually, region is consist of multiple partitions to benefit the column scans. TiFlash only contains the learner replicas of the raft groups. The Disk format is saved as **Delta Tree**. And in memory, it builds a B+ Tree index to speed up the reads.

### Next step research
1. Will DDL and long running read trxs affect the replica lag on the TiFlash? Seems not, but why Aurora RO has the limitation.
2. The Region is 96 MB. Is it logical size?
3. Seems TiDB read is pretty heavy with the lack of a global sequence # chain (like aurora volume lsn chain). Please re-check TiDB read process.
4. How can TiDB read from replica, use what timestamp? pemissive locking and optimistic locking results in different read timestamp, why?
5. How TiDB save its metadata, like region info and schema info?
6. Columnar data also has delta, how to use them? Check parquet file format, delta treeand  other columnar data format.
7. Research on different benchmarks, TPC-C TPC-H, CH-benCHmark
8. Learn Index merge join, hash join (with index), etc.  
