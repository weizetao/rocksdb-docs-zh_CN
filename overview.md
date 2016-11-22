#RocksDB Basics 基础

##1. 介绍
RocksDB项目起始于facebook尝试去开发一个能在快速存储设备上（尤其是Flash），让数据存储能发挥出淋漓尽致性能的数据库。由纯C++编写，存储key-values。支持原子读写。

RocksDB提供灵活并丰富的配置项，来适配多样的运行环境，包括纯内存、闪存、硬盘、HDFS等。它还支持多种压缩算法，和用来项目支持和调试的工具。

RocksDB从代码上借鉴了开源的leveldb以及HBase。是在leveldb1.5版本基础上开发的。

##2. 假定与目标
###性能：
RocksDB核心设计点是高性能，for fast storage and server workloads.
它应该能够充分发挥flash和ram的读写速率的潜力。
它应该支持高效的查表操作以及区间遍历操作。
它应该可以通过可配置项友好支持不同业务场景，如大量随机读、大量更新、或者两者的混合都有。
它的架构应该支持easy tuning of Read Amplification, Write Amplification and Space Amplification.

###产品支持
RocksDB应该设计成拥有好用的内置工具和实用的程序来帮助你部署和调试生产环境。大部分重要的参数都是可配置的，以致可以适配应用在不同的硬件环境的不同程序上。

###向后兼容
新版本必须向后兼容，方便使用者版本升级。

##3. Higth Level Architecture
RocksDB是一个存储二进制形式的keys和values的嵌入式key-value存储。RocksDB存储的数据都是有序的，常用的操作是`Get(key)`,`Put(key)`,`Delete(key)`,`Scan(key)`。   

RocksDB由memtable,sstfile,和logfile三个基础构件组成。memtable是保存在内存里的--新写入的数据直接插入到memtable，并选择性的写入Logfile。logfile是个顺序写入的文件。当metatable填满的时候，将其中数据刷写到一个sstfile文件，此时对应的logfile可以被安全删除。sstfile存储的数据非常便于keys的查找。

sstfile的格式介绍，请点[这里](https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format)。

##4. 特性

### Gets,Iterrators and Snapshots
Keys和Values都被视为单纯的字节流，并且key和value都没有大小的限制。`Get`接口允许应用获取单个key-value。`MultiGet`接口允许应用批量获取keys。All the keys-values returned via a MultiGet call are consistent with one-another.

数据库里的所有数据都是按序排列存储的。应用程序可以指定排序规则（排序时的比较算法）。`Iterator`接口允许应用在数据库上执行`RangeScan`的遍历操作。`Iterator` API能够直接从指定的key开始扫描，也可以从指定的key反向扫描。每个`Iterator`创建时都生成一个 consistent-point-in-time 视图，用来保证数据遍历时的一致性。

`Snapshot` API允许应用创建 `point-in-time` 数据库视图。`Get`和`Iterator` 接口都可以指定某个 Snapshot进行读取数据。在这样的场景下，Snapshot 和 Iterator 同样都支持point-in-time数据库视图，但它们的实现机制并不相同。耗时短的短线遍历操作适合使用Iterator，长线遍历操作则适合使用Snapshot。每个Iterator都维护着被point-in-time视图涉及的文件的引用计数，这些文件不会被删除直到iteraotr被释放。另一方面，Snapshot并不会阻止文件的删除，而是由合并进程检测snapshot的存在并保证不会删除任何一个被snapshot引用的key。

重新加载RocksDB库（重启服务）会导致Snapshot 失效。

###Prefix Iterators 前缀迭代器
大多数的LSM引擎不能支持高校的`RangeScan`接口，因为它需要去查看每个数据文件。但在数据库里大部分应用并不都是随机上的range遍历操作；而是通过指定一个前缀进行遍历。RocksDB对这个使用场景提供了更好的支持。应用可以配置一个`prefix_extractor`来指定前缀，RocksDB用它存储前缀的索引。当iterator指定一个前缀进行遍历时，可以根据索引忽略掉不包含这个前缀的数据文件。 

	prefix_extractor 就是让所有key的固定前缀长度做一个索引，索引标记了哪个前缀在哪些数据文件里。

###Updates 更新
`PUT` API作用是将一条key-value 插入到数据库中，如果这个key已经存在，则覆盖之前的值。`Write`API 允许多条keys-values被原子的写入数据库，数据库保证一次性的全部写入成功，或者全部写入失败。如果多条Keys其中有任意数据已经存在，则覆盖之前的值。

### Persistency 持久化
RocksDB拥有一个事务日志。所有的Puts操作都会存储在内存里的memtable中并同时可选择的写入到事务日志中。每次`Put`都可以通过`WriteOptions`设置是否将Put操作写入到事务日志中。

The WriteOptions may also specify whether or not a sync call is issued to the transaction log before a Put is declared to be committed.

Internally, RocksDB uses a batch-commit mechanism to batch transactions into the transaction log so that it can potentially commit multiple transactions using a single sync call.

### Fault Tolerance 容错
RocksDB使用校验和发现存储里的错误。每个数据块（一般是4K至128K大小）都会做校验和。一个数据块，一旦已经写入存储，将不会再被更新。

RocksDB dynamically detects hardware support for checksum computations and avails itself of that support when available.

###Multi-Threaded Compactions 多线程合并
Compactions are needed to remove multiple copies of the same key that may occur if an application overwrites an existing key. Compactions also process deletions of keys. Compactions may occur in multiple threads if configured appropriately.

当应用覆盖一个已经存在的key时，其实是多了一个该key的副本，value不同，合并需要删除那些重复无用的key。真正的物理删除操作也是在合并阶段执行的。通过配置合并可以在多线程中执行。

The overall write throughput of an LSM database directly depends on the speed at which compactions can occur, especially when the data is stored in fast storage like SSD or RAM. RocksDB may be configured to issue concurrent compaction requests from multiple threads. It is observed that sustained write rates may increase by as much as a factor of 10 with multi-threaded compaction when the database is on SSDs, as compared to single-threaded compactions.

LSM数据库的总写入吞吐量直接取决于compactions发生时的速度，特别是当数据存储在快速存储（如SSD或RAM）中时。 RocksDB可以配置为从多个线程发出并发compaction请求。 可以观察到，当数据库在SSD上时，与单线程compactions相比，开启多线程compaction，持续写入速率可以提升多达10倍。

The entire database is stored in a set of sstfiles. When a memtable is full, its content is written out to a file in Level-0 (L0). RocksDB removes duplicate and overwritten keys in the memtable when it is flushed to a file in L0. Some files are periodically read in and merged to form larger files - this is called compaction.

整个数据库存储在一组sstfiles中。当memtable已满时，其内容将写入Level-0(L0)的文件中。当它被刷新写入到L0文件时，RocksDB会删除memtable中重复的和被覆写过的keys。一些文件被定期的读入并合并形成较大文件，这个过程被称为compation。

RocksDB supports two different styles of compaction. The Universal Style Compaction stores all files in L0 and all files are arranged in time order. A compaction picks up a few files that are chronologically adjacent to one another and merges them back into a new file L0. All files can have overlapping keys.

RocksDB支持两种compaction方式。通用的compation方式是将所有的文件都存储为L0，并按时间排列。一次compaction是挑取一部分按照时间排列相近的文件合并为一个新的L0文件。所有的文件都可能有重叠的key。

The Level Style Compaction stores data in multiple levels in the database. The more recent data is stored in L0 and the oldest data is in Lmax. Files in L0 may have overlapping keys, but files in other layers do not. A compaction process picks one file in Ln and all its overlapping files in Ln+1 and replaces them with new files in Ln+1. The Universal Style Compaction typically results in lower write amplification but higher space amplification than Level Style Compaction.

分级方式compation是将数据库的数据存储在多个级别中。最近的数据存储在L0，越旧的数据存储在越高级别的文件中。L0文件中的数据是有重叠的key的，但其它层级的文件中没有。一次compation过程是挑选一个Ln的文件和与这个文件有重叠的Ln+1所有文件，生成新的Ln+1的文件替换原来的。通用compation方式对比分级compation特点是较低的写放大，但更高的空间放大。

A MANIFEST file in the database records the database state. The compaction process adds new files and deletes existing files from the database, and it makes these operations persistent by recording them in the MANIFEST file. Transactions to be recorded in the MANIFEST file use a batch-commit algorithm to amortize the cost of repeated syncs to the MANIFEST file.

MANIFEST是记录数据库状态的文件。compation过程产生的新文件和被删掉的已有文件，这些操作都会被持续的记录到MANIFEST文件中。记录在MANIFEST文件中的事务使用批量提交算法来将重复同步的成本分摊到MANIFEST文件。

###Avoiding Stalls 避免
Background compaction threads are also used to flush memtable contents to a file on storage. If all background compaction threads are busy doing long-running compactions, then a sudden burst of writes can fill up the memtable(s) quickly, thus stalling new writes. This situation can be avoided by configuring RocksDB to keep a small set of threads explicitly reserved for the sole purpose of flushing memtable to storage.

后台compaction线程也被用于将memtable刷写至文件。如果所有的后台compation线程都正忙于运行长耗时的compation操作时，突然的写入操作可能快速填满memtable，这时就会拒绝新的写入操作。可以通过配置预留小部分线程专门负责memtable的刷新来避免这种场景的发生。

###Compaction Filter 合并过滤器
Some applications may want to process keys at compaction time. For example, a database with inherent support for time-to-live (TTL) may remove expired keys. This can be done via an application-defined Compaction Filter. If the application wants to continuously delete data older than a specific time, it can use the compaction filter to drop records that have expired. The RocksDB Compaction Filter gives control to the application to modify the value of a key or to drop a key entirely as part of the compaction process. For example, an application can continuously run a data sanitizer as part of the compaction.

一些应用希望能在compaction阶段添加处理keys的自定义逻辑。例如，一个支持TTL的数据库需要删除已过期的keys。这时就可以使用应用程序自定义合并过滤器来实现。如果应用程序想要持续的删掉早于特定时间的数据，则可以使用合并过滤器来删除过期的记录。RocksDB 合并过滤器可以让应用程序控制key的更新或删除，让其作为compaction操作的一部分。例如，应用程序可以作为compaction的一部分连续的运行数据清理操作。

###ReadOnly Mode 只读模式
A database may be opened in ReadOnly mode, in which the database guarantees that the application may not modify anything in the database. This results in much higher read performance because oft-traversed code paths avoid locks completely.

当数据保证没有任何更新操作时，可以以只读模式打开数据库。效果就是能够达到更高的读性能，因为避免了频繁的锁操作。

###Database Debug Logs 数据库调试日志
RocksDB writes detailed logs to a file named LOG*. These are mostly used for debugging and analyzing a running system. This LOG may be configured to roll at a specified periodicity.

RocksDB将详细日志写入名为LOG *的文件。 这些主要用于调试和分析正在运行的系统。 该日志可以被配置为定期分割。

###Data Compression 数据压缩
RocksDB supports snappy, zlib, bzip2, lz4 and lz4_hc compression. RocksDB may be configured to support different compression algorithms at different levels of data. Typically, 90% of data in the Lmax level. A typical installation might configure no-compression for levels L0-L2, snappy compression for the mid levels and zlib compression for Lmax.

RocksDB支持 snappy, zlib, bzip2, lz4 and lz4_hc 压缩算法。不同的层级数据可以配置为不同的压缩算法。通常，最高级的数据配置90%的压缩率算法。典型的做法是，第0-2级的数据不进行压缩，中间层级使用snappy压缩，最高级则使用zlib。

###Transaction Logs 事务日志
RocksDB stores transactions into logfile to protect against system crashes. On restart, it re-processes all the transactions that were recorded in the logfile. The logfile can be configured to be stored in a directory different from the directory where the _sstfile_s are stored. This is necessary for those cases in which you might want to store all data files in non-persistent fast storage. At the same time, you can ensure no data loss by putting all transaction logs on slower but persistent storage.

RocksDB 将事务存储至日志文件中以预防系统崩溃。在重启时，重演一次日志中记录的所有事务记录。这个日志可以被配置存储到单独的目录，用来和sstfile区分存储。在需要将所有数据文件存储在非持久化的快速存储上时，这是必须的。同时，为了确保数据不丢失，将事务日志写到速度不快但可持久化的存储上。

	将事务日志写入到机械硬盘上，因为log都是顺序写入，性能不差。
	
###Full Backups, Incremental Backups and Replication 全量备份，增量备份 和 主从复制
RocksDB has support for full backups and incremental backups. RocksDB is an LSM database engine, so, once created, data files are never overwritten, and this makes it easy to extract a list of file-names that correspond to a point-in-time snapshot of the database contents. The API DisableFileDeletions instructs RocksDB not to delete data files. Compactions will continue to occur, but files that are not needed by the database will not be deleted. A backup application may then invoke the API GetLiveFiles/GetSortedWalFiles to retrieve the list of live files in the database and copy them to a backup location. Once the backup is complete, the application can invoke EnableFileDeletions; the database is now free to reclaim all the files that are not needed any more.

RocksDB支持全量备份和增量备份。RocksDB是一个LSM数据库引擎，所以数据文件一旦生成就不会被覆写，使得更容易提取`point-in-time`快照对应的数据文件列表。接口中的`DisableFileDeletions`选项可以让RocksDB不删除数据文件，Compations照常执行，但无用的数据文件不会被删除了。备份程序可以依据`GetLiveFiles/GetSortedWalFiles`接口获取到当前有效的数据文件列表并拷贝到本分目录。当备份完成，程序可以再启用EnableFileDeletions，这样数据库就又开始回收无用的文件了。

Incremental Backups and Replication need to be able to find and tail all the recent changes to the database. The API GetUpdatesSince allows an application to tail the RocksDB transaction log. It can continuously fetch transactions from the RocksDB transaction log and apply them to a remote replica or a remote backup.

增量备份和主从同步需要找到发生变化了的数据。GetUpdatesSince接口允许应用程序收集事务日志的尾部。它可以持续的获取RocksDB的事务日志的事务并将它们应用到远程从属副本或者远程备份。

A replication system typically wants to annotate each Put with some arbitrary metadata. This metadata may be used to detect loops in the replication pipeline. It can also be used to timestamp and sequence transactions. For this purpose, RocksDB supports an API called PutLogData that an application may use to annotate each Put with metadata. This metadata is stored only in the transaction log and is not stored in the data files. The metadata inserted via PutLogData can be retrieved via the GetUpdatesSince API.

replication系统通常希望用元数据注释每个Put，该元数据可以用于检测replication管道中的循环，也可以用于时间戳和事务顺序。为此，RocksDB支持一个称为PutLogData的API，应用程序可以使用该API来为每个Put添加元数据。 此元数据仅存储在事务日志中，不存储在数据文件中。 通过PutLogData插入的元数据可以通过GetUpdatesSince API检索。

RocksDB transaction logs are created in the database directory. When a log file is no longer needed, it is moved to the archive directory. The reason for the existence of the archive directory is because a replication stream that is falling behind might need to retrieve transactions from a log file that is way in the past. The API GetSortedWalFiles returns a list of all transaction log files.

RocksDB事务日志创建在数据库目录，当一个日志文件变成不再需要时，它会被转移到存档目录。存档目录存在的原因是因为主从复制的从属端需要根据过去式的日志文件检索事务。GetSortedWalFiles API返回事务日志的列表。

###Support for Multiple Embedded Databases in the same process 一个程序支持嵌入多个数据库
A common use-case for RocksDB is that applications inherently partition their data set into logical partitions or shards. This technique benefits application load balancing and fast recovery from faults. This means that a single server process should be able to operate multiple RocksDB databases simultaneously. This is done via an environment object named Env. Among other things, a thread pool is associated with an Env. If applications want to share a common thread pool (for background compactions) among multiple database instances, then it should use the same Env object for opening those databases.

RocksDB的一个常见用例是应用程序将其数据进行逻辑分区或分片。 这种技术有利于应用程序负载平衡和从故障快速恢复。 这意味着单个服务器进程应该能够同时操作多个RocksDB数据库。 这通过名为Env的环境对象完成。 除此之外，线程池与Env相关联。 如果应用程序想要在多个数据库实例之间共享公共线程池（用于后台合并），那么它应该使用相同的Env对象来打开这些数据库。

Similarly, multiple database instances may share the same block cache.

同样的是，多个数据库实例会共享相同的区块缓存。

###Block Cache -- Compressed and Uncompressed Data  块缓存--压缩和非压缩数据
RocksDB uses a LRU cache for blocks to serve reads. The block cache is partitioned into two individual caches: the first caches uncompressed blocks and the second caches compressed blocks in RAM. If a compressed block cache is configured, then the database intelligently avoids caching data in the OS buffers.

RocksDB使用LRU缓存来提供读服务。块缓存分为两种独立的缓存：一个缓存未压缩的数据，一个缓存压缩过的数据，在RAM里。如果配置了缓存压缩，数据库会智能的避免使用OS缓冲器缓存数据。

###Table Cache 表缓存
The Table Cache is a construct that caches open file descriptors. These file descriptors are for sstfiles. An application can specify the maximum size of the Table Cache.

表缓存是缓存已打开的文件描述符的一个结构体，这些文件都是sstfiles。应用程序可以指定表缓存的最大大小。

###External Compaction Algorithms  外部合并算法
The performance of an LSM database depends significantly on the compaction algorithm and its implementation. RocksDB has two supported compaction algorithms: LevelStyle and UniversalStyle. We would also like to enable the large community of developers to develop and experiment with other compaction policies. For this reason, RocksDB has appropriate hooks to switch off the inbuilt compaction algorithm and has other APIs to allow applications to operate their own compaction algorithms.

LSM数据库的性能表现显然取决于合并算法和它的实现。RocksDB支持两种合并算法：LevelStyle 和 UniversalStyle。我们还希望看到其它较大的开发团体来测验其它的合并策略。所以，RocksDB可以停用内置的合并算法并通过APIs允许应用程序使用自己的合并算法。

Options.disable_auto_compaction, if set, disables the native compaction algorithm. The GetLiveFilesMetaData API allows an external component to look at every data file in the database and decide which data files to merge and compact. The DeleteFile API allows applications to delete data files that are deemed obsolete.

Options.disable_auto_compaction，一旦被设定，将取消原生的合并算法。GetLiveFilesMetaData API 允许外部组件去检索每个数据文件并决定合并策略。DeleteFile API支持应用程序删除它们认为无效的数据文件。

###Non-Blocking Database Access 数据库非阻塞接入
There are certain applications that are architected in such a way that they would like to retrieve data from the database only if that data retrieval call is non-blocking, i.e., the data retrieval call does not have to read in data from storage. RocksDB caches a portion of the database in the block cache and these applications would like to retrieve the data only if it is found in this block cache. If this call does not find the data in the block cache then RocksDB returns an appropriate error code to the application. The application can then schedule a normal Get/Next operation understanding that fact that this data retrieval call could potentially block for IO from the storage (maybe in a different thread context).

一些应用程序想要设计为能够通过某种方式来实现非阻塞的检索数据，即数据检索不必从存储设备读取数据。RocksDB缓存部分数据在块缓存，应用程序希望只检索块缓存的数据。如果这次请求在块缓存中未找到数据，RocksDB需要返回一个错误码给程序。应用程序根据错误码来决定是否再执行 Get/Next操作去存储设备上再去检索。

###Stackable DB 可堆叠数据库
RocksDB has a built-in wrapper mechanism to add functionality as a layer above the code database kernel. This functionality is encapsulated by the StackableDB API. For example, the time-to-live functionality is implemented by a StackableDB and is not part of the core RocksDB API. This approach keeps the code modularized and clean.

RocksDB提供内置的封装机制，用于在数据库核心层之上添加功能。这些功能由StackableDB API提供。例如，time-to-live（时效性）就是通过StackableDB实现的，但并不是核心层的API。这样可以保持代码的模块化和简洁性。

###Backupable DB 可备份数据库
One feature implemented using the StackableDB interface is BackupableDB, which makes backing up RocksDB simple. You can read more about it here: How to backup RocksDB?

使用StackableDB实现的一个功能是 BackupableDB，这使得RocksDB备份变得简单。了解更多可以阅读这里：[如何备份RocksDB](https://github.com/facebook/rocksdb/wiki/How-to-backup-RocksDB%3F)

###Memtables:
####Pluggable Memtables:
The default implementation of the memtable for RocksDB is a skiplist. The skiplist is an sorted set, which is a necessary construct when the workload interleaves writes with range-scans. Some applications do not interleave writes and scans, however, and some applications do not do range-scans at all. For these applications, a sorted set may not provide optimal performance. For this reason, RocksDB supports a pluggable API that allows an application to provide its own implementation of a memtable. Three memtables are part of the library: a skiplist memtable, a vector memtable and a prefix-hash memtable. A vector memtable is appropriate for bulk-loading data into the database. Every write inserts a new element at the end of the vector; when it is time to flush the memtable to storage the elements in the vector are sorted and written out to a file in L0. A prefix-hash memtable allows efficient processing of gets, puts and scans-within-a-key-prefix.

memtable默认实现方式是跳表。跳表是个有序集合，当工作面上使用range-scans交织写入时，这个数据机构非常有必要。但有的应用不使用内部写入和扫描，并且也没有range-scans的操作。对于这些应用，一个有序集合并不能提供良好的表现。基于这个原因，RocksDB通过pluggalbe API支持应用自定义memtable的实现。三种memtable的实现：跳表、动态数组、前缀哈希。动态数组memtable适合批量加载数据到数据库中，每次写入将新元素插入到数组的末尾; 当要将memtable刷新到存储时，数组中的元素被排序后写到L0中的文件中。 前缀哈希的memtable能够让gets，puts和scans-within-a-key-prefix等操作效率更好。

#####Memtable Pipeline Memtable管道
RocksDB supports configuring an arbitrary number of memtables for a database. When a memtable is full, it becomes an immutable memtable and a background thread starts flushing its contents to storage. Meanwhile, new writes continue to accumulate to a newly allocated memtable. If the newly allocated memtable is filled up to its limit, it is also converted to an immutable memtable and is inserted into the flush pipeline. The background thread continues to flush all the pipelined immutable memtables to storage. This pipelining increases write throughput of RocksDB, especially when it is operating on slow storage devices.

一个数据库的memtables个数是可配置的。当一个memtable写满，它就会被变成一个不可变更的memtable并由后台线程将其刷新至存储上。与此同时，新的数据写入到一个新的memtable中，当新分配的memtable又被写满后，它也会被转变为一个不可变更的memtable并插入到刷新管道里。后台线程负责将管道里所有不可改变的memtable刷新到存储上。这个管道提升了RocksDB的写吞吐，尤其存储是比较慢速的设备（HDD）的时候。

####Memtable Compaction 
When a memtable is being flushed to storage, an inline-compaction process removes duplicate records from the output steam. Similarly, if an earlier put is hidden by a later delete, then the put is not written to the output file at all. This feature reduces the size of data on storage and write amplification greatly. This is an essential feature when RocksDB is used as a producer-consumer-queue, especially when the lifetime of an element in the queue is very short-lived.

当memtable刷新至存储时，一个内部合并进程会将重复的记录从输出流中删掉。类似的，如果早先的put被稍后的隐藏删除，则put不会被写入到输出流。这个特性大大减少了写入存储的数据大小和写放大问题。当使用producer-consumer-queue时，这是个基本的特性，尤其是当队列里的元素生命周期非常短的时候。

###Merge Operator 合并操作
RocksDB natively supports three types of records, a Put record, a Delete record and a Merge record. When a compaction process encounters a Merge record, it invokes an application-specified method called the Merge Operator. The Merge can combine multiple Put and Merge records into a single one. This powerful feature allows applications that typically do read-modify-writes to avoid the reads altogether. It allows an application to record the intent-of-the-operation as a Merge Record, and the RocksDB compaction process lazily applies that intent to the original value. This feature is described in detail in Merge Operator

RocksDB原生支持三种操作记录，`Put` `Delete` `Merge`。当compaction遇到Merge记录时，会调用应用程序特殊的合并操作。Merge会将多个Put和Merge合并为一条记录。This powerful feature allows applications that typically do read-modify-writes to avoid the reads altogether. It allows an application to record the intent-of-the-operation as a Merge Record, and the RocksDB compaction process lazily applies that intent to the original value. This feature is described in detail in Merge Operator

##5. TOOLS 工具
There are a number of interesting tools that are used to support a database in production. The sst_dump utility dumps all the keys-values in a sst file. The ldb tool can put, get, scan the contents of a database. ldb can also dump contents of the MANIFEST, it can also be used to change the number of configured levels of the database. It can be used to manually compact a database.

这里有很多有意思的工具，来更好的支持线上数据库。`sst_dump`可以转存一个sst文件所有的kyes-values。`ldb`工具可以put,get,scan数据库内容。ldb还可以转存MANIFEST的内容，还可以改变配置的levels。可以手动触发一次compact。

##6. Tests 测试
There are a bunch of unit tests that test specific features of the database. A make check command runs all unit tests. The unit tests trigger specific features of RocksDB and are not designed to test data correctness at scale. The db_stress test is used to validate data correctness at scale.

这里有一堆单元测试来测试数据库的特定特性。make check命令执行所有的单元测试用例。这些单元测试时为了触发RocksDB的特定功能，并不是设计用来验证数据的正确定的。`db_stress` 被用于在规模上验证数据的正确性。

##7. Performance 性能
RocksDB performance is benchmarked via a utility called db_bench. db_bench is part of the RocksDB source code. Performance results of a few typical workloads using Flash storage are described here. You can also find RocksDB performance results for in-memory workload here.

评估RocksDB的性能标准工具叫 `db_bench`。db_bench是RocksDB源代码的一部分。在闪存上的性能结果请看[这里](https://github.com/facebook/rocksdb/wiki/Performance-Benchmarks)，完全内存的性能表现请看[这里](https://github.com/facebook/rocksdb/wiki/RocksDB-In-Memory-Workload-Performance-Benchmarks).

Author: Dhruba Borthakur


