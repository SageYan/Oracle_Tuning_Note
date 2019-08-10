[TOC]



# Oracle数据库体系结构概述

## 数据库的组成：

实例 + 一组文件

```powershell
实例：内存 + 进程
一组文件： 参数文件、控制文件、数据文件、归档文件、日志文件、配置文件等
```

## 实例概述：

用户进程写sql语句，服务器进程处理sql，我们对数据库的操作要通过实例（内存和后台进程）：

### 内存

内存包含了6个模块：

#### 1.share pool

```powershell

library cache（库高速缓存区）：缓存运行的sql语句，调用过的pl/sql块（原因：系统运行的大量的sql语句是相似的，重复的。oracle为了加速sql的运行。）
注释：
sql语句的后台运行流程：
1.共享池扫描，在内存中，命中历史上运行过的sql，做软解析。作用是减少sql语句的重解析。
2.  解析(解析出执行计划）
       (a.硬解析，解析崭新的sql语句：
        1语法分析、
        2语义分析，对用户对象的存在性，有效性解析、
        3安全审核、
        4优化，组件优化器筛选执行计划，对比所有计划的成本cost，选择cost最低的计划,最后将sql文  本和执行计划写入共享池）
   执行   [按照执行计划（全表扫描，索引访问）访问后台对象]
   获取  （从数据库里面找到的数据返回给客户端）
 
data dictionary cache（数据字典高速缓存区，行高速缓存row cache，以行为单位缓存最近所访问的字典行数据）：
作用加速sql语句的解析，because ORACLE DB在解析sql时会频繁访问数据字典，减少从硬盘IO。       
```

#### 2.database buffer cache

```powershell
database buffer cache（数据库缓冲区高速缓存），目的是减少IO的瓶颈。
将数据库数据块镜像读入缓存，数据库所有的操作都是对缓存中的镜像进行的。
LRU（最近最少使用列表）管理buffercache，将进入buffer cache最早，使用最少的冷块优先释放，加载新块。
以8K为单位做IO，缺点浪费内存空间，优点减少物理IO。
```

#### 3.large pool  

```powershell
存放与sql和pl/sql块不想关的块
备份还原恢复时使用的块。                                      
```



#### 4.java pool          

前端应用有编译Java的需求，需开启Java pool

#### 5.streams pool（流池）

流复制的需求

#### 6.logbuffer    

记录所有数据块的变化，记录redo条目，为了减少log写盘的次数，缓冲日志写盘的IO瓶颈。



### 后台进程

#### 1.PMON 

```powershell
process monitor 进程监视器  
         1.进程监视，负责重新启动意外终止的非核心后台进程
         2.清理意外终止的链接在服务器端遗留的垃圾资源。（User process异常终止，修改了内存的数据等）
         3.将实例的信息注册到监听，周期60s
         4.在集群环境，以60s为单位收集当前节点的cpu压力。
                                                                       
```

#### 2.DBWn  

```powershell
数据同步，将内存里的脏数据写入硬盘。
         1.检查点
         2.脏数据达到阈值
         3.扫描当前buffercache 发现没有空闲空间可用
         4.time out
         5.集群环境的ping请求，rcping request触发数据写，触发多实例的数据同步
         6.表级别的drop和truncate
         7.tablespace的read-only
         8.tablespace offline
         9.热备份 begin backup
```

#### 3.SMON  

```powershell
系统监控程序    
         1.空间管理  定期合并空闲空间 回收临时段
         2.实例恢复  前滚  回滚  释放资源（系统断电，实例意外终止了。
                                       1commit的数据，数据没有写完，利用日志使交易重现，叫前滚。commit的数据是要做永久修改的。
                                       2 没提交的数据，因为DBWn的九种触发写的情况，写入了硬盘。因为没有提交，（联机日志上不是完整的事物）smon取出老镜像，将写入的数据回滚。
                                       3.将锁资源释放 PS 日志就是欠条。）
```

#### 4.CKPT  

```powershell
1.会调度数据写
2.将已经完成的检查点写到数据文件头
3.将已经完成的检查点写到控制文件
```

####  5.LGWR   

```powershell
把redo写到联机日志 a commit 会触发日志写
                 b logbuffer 的1/3
                 c time out
                 d 任何一次数据写之前
```

