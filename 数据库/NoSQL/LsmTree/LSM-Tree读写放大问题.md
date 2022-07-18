基于 LSM-Tree 的存储系统越来越常见了，如 RocksDB、LevelDB。LSM-Tree 能将**离散**的**随机**写请求都转换成**批量**的**顺序**写请求（WAL + Compaction），以此提高写性能。但也带来了一些问题，即读放大、写放大和空间放大。

# 读、写、空间放大的概念
通常 Compaction 涉及到以下三个放大因子，Compaction 需要在三者之间做取舍。
## 读放大(Read Amplification)
指的是一个简单的Query操作需要进行IO操作读取硬盘的次数，例如一个简单的Query需要读取5个页面，发生了5次IO，那么读放大因子就是5

## 写放大(Write Amplification)
假如每秒写入10MB数据，而观察到磁盘的写入速度是30MB/s，那么写放大因子就是3

写分为立即写和延迟写，比如 redo log是立即写，传统基于B-Tree数据库刷脏页和 LSM Compaction 是延迟写。redo log 使用 direct IO 写时**至少以512字节对齐**，假如 log 记录为100字节，磁盘需要写入512字节，写放大为5。
## 空间放大(Space Amplification)
假设我需要存储10MB数据，但是实际硬盘使用量是30MB，那么空间放大因子就是3

有比较多的因素会影响空间放大：
1. 比如在Compaction 过程中需要临时存储空间
2. 空间碎片，Block中有效数据的比例小
3. 旧版本数据未及时删除等等