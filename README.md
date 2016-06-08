# HOME
工作之余，首次翻译

# 欢迎到访RocksDB
RocksDB是一个嵌入式的key-value存储C++库，key和value可存储任意字节流。它是facebook在LevelDB基础上开发并且还向后兼容LevelDB的API。

RocksDB使用闪存可以达到极低的延迟。RocksDB使用纯C++实现的 `Log Structured` 数据库引擎作为存储。还有一个java版本正在开发中，叫RocksJava。

RocksDB有高度灵活的配置选项，可适配支持多种场景运行，包括纯内存、闪存、硬盘或是HDFS分布式文件系统。它支持多种压缩算法，还有丰富的实用工具,用来产品支撑和调试。
