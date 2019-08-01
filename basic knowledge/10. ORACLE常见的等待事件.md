[TOC]

# ORACLE常见的等待事件

## 1 buffer busy waits

```powershell
 原因：
 当一个会话试图修改一个数据块，但这个数据块正在被另一个会话修改时。
 当一个会话需要读取一个数据块，但这个数据块正在被另一个会话读取到内存中时。
 备注：数据处理的最小单位是块
 
 处理办法：
 热点快：分区表，分区索引，反向索引
 低效SQL
 数据交叉访问：使得不同的数据功能，模块分布在不同的实例上访问
```

## 2 DB file scattered read

```
最常见的两种情况是全表扫描(FTS：Full Table Scan)和索引快速扫描(IFFS：index fast full scan)
```

3 Direct path read

```sql
磁盘上有大量的临时数据产生，比如排序，并行执行等操作。或者意味着PGA中空闲空间不足。
另外：
但对于一些大表，将其缓冲至buffer cache势必会将buffer cache中的许多其它对象挤出，即ageing out。为了避免这一情况，产生了direct path read,即不需要缓冲到缓存区，而是直接由服务器进程从磁盘获取。
禁用：alter system set event='10949 TRACE NAME CONTEXT FOREVER' scope=spfile;
      alter system set "_serial_direct_read"=never;
```

## 4 Direct path write

```sql
这个等待事件和direct path read正好相反，是会话将一些数据从PGA中直接写入到磁盘文件上，而不经过SGA。
这种情况通常发生在：
a.使用临时表空间排序(内存不足)；
b.数据的直接加载(使用append方式加载数据)；
c.并行DML操作。
```

## 5 Free buffer waits

```sql
(1) data buffer太小，导致空闲空间不够；
(2) 内存中的脏数据太多，DBWR无法及时将这些脏数据写到磁盘中以释放空间。
```

## 6 Library cache lock

```sql
一般可以理解的是alter table或者alter package/procedure会以X模式持有library cache lock，造成阻塞。
但是常见的问题还有以下几种原因：
1)错误的用户名密码：
2)正在执行搜集统计信息，这是大家往往会忽略的，一般会看last_ddl_time，却忽略了last_analyzed，
3)错误的语句解析（failed parse）
4)bug
```

## 7 Library cache pin

```powershell
这个等待事件和library cache lock一样是发生在共享池中并发操作引起的事件。
通常来讲，如果Oracle要对一些PL/SQL或者视图这样的对象做重新编译，需要将这些对象pin到共享池中。
如果此时这个对象被其他的用户特有，就会产生一个library cache pin的等待
```

## 8 Log buffer space

```sql
当log buffer中没有可用空间来存放新产生的redo log数据时，就会发生log buffer space等待事件。如果数据库中新产生的redo log的数量大于LGWR写入到磁盘中的redo log数量，必须等待LGWR完成写入磁盘的操作，LGWR必须确保redo log写到磁盘成功之后，才能在redo buffer当中重用这部分信息。
如果数据库中出现大量的log buffer space等待事件，可以考虑如下办法：
(1)、增加redo buffer的大小。
(2)、提升磁盘的I/O性能。
```

## 9 Log file sequential read

```SQL
这个等待事件通常发生在对redo log信息进行读取时，比如在线redo的归档操作，ARCH进程需要读取redo log的信息，由于redo log的信息是顺序写入的，所以在读取时也是按照顺序的方式来读取的。
```

## 10 Log file switch（archiving needed）

```sql
在归档模式下，这个等待事件发生在在线日志切换(log file switch)时，需要切换的在线日志还没有被归档进程(ARCH)归档完毕的时候。当在线日志文件切换到下一个日志时，需要确保下一个日志文件已经被归档进程归档完毕，否则不允许覆盖那个在线日志信息(否则会导致归档日志信息不完整)。
出现这样的等待事件通常是由于某种原因导致ARCH进程死掉，比如ARCH进程尝试向目的地写入一个归档文件，但是没有成功(介质失效或者其他原因)，这时ARCH进程就会死掉。如果发生这种情况，在数据库的alert log文件中可以找到相关的错误信息
```

## 11 Log file switch(checkpoint incomplete)

```
当一个在线日志切换到下一个在线日志时，必须保证要切换到的在线日志上的记录信息(比如一些脏数据块产生的redo log)被写到磁盘上(checkpoint)，这样做的原因是，如果一个在线日志文件的信息被覆盖，而依赖这些redo信息做恢复的数据块尚未被写到磁盘上(checkpoint) ，此时系统down掉的话，Oracle将没有办法进行实例恢复。

解决的方法是，增加日志文件的大小或者增加日志组的数量。
```

## 12 Log file sync

```sql
引起log file sync的原因：
1.频繁提交或者rollback,检查应用是否有过多的短小的事务，如果有，可以使用批处理来缓解。
2.OS的IO缓慢：解决办法是将日志文件放裸设备上或绑定在RAID 0或RAID 1+0中，而不是绑定在RAID 5中。
3.过大的日志缓冲区（log_buffer ） 过大的log_buffer,允许LGWR变得懒惰，因为log buffer中的数据量无法达不到_LOG_IO_SIZE，导致更多的重做条目堆积在日志缓冲区中。	
当事务提交或者3s醒来时，LGWR才会把所有数据都写入到redo log file中。	由于数据很多，LGWR要用更多时间等待redo写完毕。	
这种情况，可以调小参数_LOG_IO_SIZE参数，其默认值是LOG_BUFFER的1/3或1MB，取两者之中较小的值。	
4.CPU负载高
5.RAC私有网络性能差

判断：
如果log file sync的等待时间很高,而log file parallel write的等待时间并不高,这意味着log file sync的原因并不是缓慢的日志I/O,而是应用程序过多的提交造成。
当log file sync的等待时间和 log file parallel write等待时间基本相同，说明是IO问题造成的log file sync等待事件。

--如果log file sync远大于log file parallel write的等待时间，只会为以下三种情况
1、CPU资源紧张
2、LGWR在申请latch资源时遇到竞争(IMU未使用，RAC环境下不适用)
3、同时提交的进程太多

```

## 13 cursor: pin S wait on X

```sql
cursor: pin S wait on X，这个等待事件主要是由硬解析引起的
--硬解析，软解析，软软解析在共享池中的等待事件

硬解析导致异常等待处理：使用绑定变量(应用层面)，cursor sharing=true 禁用ACS(adaptive cursor sharing)，配置充足的shared pool v$sqlarea sql_text列中相似的sql很多，说明未使用绑定变量
软解析争用处理：调整session cached cursors参数，使软解析变为软软解析
软软解析导致争用处理：cursor pin:s ，使用提示符/**/改变sql hash值，将一条SQL变为多条减少争用；应用层缓存游标，实现一次解析，多次执行
```

## 14 latch：cache buffers chains

```
一般产生CACHE BUFFERS CHAINS的原因有几个方面:

1、buffer cache太少(也说明SQL语句效率低,较多的逻辑读意味着较多的latch get操作，从而增加了锁存器争用。多个进程同时扫描大范围的索引或表时，可能广泛地发生cache buffers chains 锁存器争用);

2、热块挣用。(从oracle9i开始，对latch:cache buffer chains支持只读共享访问，这可以减少部分争用，但并不能完全消除争用。)
```

