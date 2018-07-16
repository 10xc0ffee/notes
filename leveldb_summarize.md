# Leveldb Notes

### Leveldb 简介

Leveldb是一个单机key-value数据库library。Comparator，filter，cache，以及各种性能相关的参数都可以在创建DB的时候由参数传入。基于LSM，leveldb倾向于优化写请求，更适合写请求远大于读请求的负载。

### Leveldb 架构

|==============================DB==============================|

|====VersionSet====|

|====CurVersion====||====MemTable====||====ImmTable====||====WAL====|

|====TableCache====||====BlockCache====|

|=============================Disk=============================|

Leveldb架构如上，由五六个重要的数据结构组成。DB提供用户接口，所有的存在Snapshot，Read，Write，Delete，Iterate对外接口都在这层提供。VersionSet追踪了所有SSTable的历史变动。CurVersion包含当前SSTable的索引，所有读操作访问table cache都要先经过它来定位SSTable。WAL记录写操作的oplog。MemTable使用跳表提供了快速写和无锁读特性。TableCache默认使用LRU将SSTable映射到内存中，提供快速的查找和遍历。

其中最中间的这层构成了整个数据库当前的视图。对于Leveldb的三种operations：

* Write主要涉及WAL的append，MemTable的insert
* Read主要涉及MemTable和ImmTable的get，以及CurVersion的table find和TableCache的get
* Compaction主要涉及ImmTable的serialize或SSTables的多路归并，和VersionSet的meta change


### Writes in leveldb
Leveldb的写操作包括Upsert和Delete两种类型。和其他log系统一样，写操作将操作（WriteBatch）写到log file之后就返回用户成功。后台的Compaction task负责Immtable持久化和多sstable的归并。数据同时会在内存以跳表的形式存在。

前台写WriteBatch会被加入FIFO，队首所在用户线程负责batch这些写请求，并通过notify其他用户线程同一批落盘的写请求的完成。这种做法降低了写请求的平均等待时间，原因如下，

* 每次只有一个线程在写log file，这个过程不需要加锁，给新的Write请求进入队列的空隙。
* batch write对于较小的log record连续append来说，更给有利于提高磁盘吞吐。

后台Compaction对前台写有负反馈。为了保证memtable的性能，其占用内存是有软硬threshold的。当达到软threshold的时候，前台写会sleep 1秒, 以让出CPU等待immtable的Compaction。这种思路有利于避免长尾。

### Compaction 作用


### Compaction 策略


### Reads in leveldb


### Snapshot 实现
Leveldb支持MVCC，对外可以提供snapshot isolation。其中Version即是写请求产生的Sequence Id。用户在DB的生命周期内创建了某个SeqId的snapshot，对于DB来说，仅代表Compaction不会merge并删除该SeqId之后的sstable。

### 关于Leveldb 数据安全
涉及fdatasync和fsync的使用。POSIX说fdatasync会sync和fd相关的data和保证数据准确性的meta。我发现Leveldb使用的是fdatasync if possible。
要注意SyncDirIfManifest，Manifest 文件持久化了新加入的sstable并去除了多余的sstable，为了保证Manifest的data sync完成后其中记录的sstable都存在，需要在之前调用父目录的fdatasync。
另外unlink和rename在filesystem层不一定保序，这导致需要SetCurrentManifest在rename之后调用SyncDirIfManifest，防止redundant的sstable被删除而Manifest却没更新。

### Leveldb代码
Leveldb的代码比较容易理解，特别是几个抽象Memtable, VersionSets, Version, Iterator, 使整个架构很清晰。对函数调用前置条件的注释和Assertion很值得借鉴。

不好阅读的地方也有。VersionSets和CurrentVersion，TableCache的互相引用，DB Mutex作为参数横跨很长的调用链。这些导致我对一些细节没法很清晰的察觉，需要进一步熟悉代码才行。