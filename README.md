# HOME
仅是根据自己的理解做个记录，欢迎大家指正

# 欢迎到访RocksDB
RocksDB是一个嵌入式的key-value存储C++库，key和value可存储任意字节流。它是facebook在LevelDB基础上开发并且还向后兼容LevelDB的API。

RocksDB使用闪存可以达到极低的延迟。RocksDB使用纯C++实现的 `Log Structured` 数据库引擎作为存储。还有一个java版本正在开发中，叫RocksJava。

RocksDB有高度灵活的配置选项，可适配支持多种场景运行，包括纯内存、闪存、硬盘或是HDFS分布式文件系统。它支持多种压缩算法，还有丰富的实用工具,用来产品支撑和调试。

#特性
* 为有在本地（闪存设备、内存）存储数TB数据的应用服务而设计
* 优化了大小为中小Key-Values的存储效率,在快速存储（闪存设备或内存）上
* 有效利用操作系统的多核