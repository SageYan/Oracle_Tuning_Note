[TOC]



# Oracle 优化参考手册

## 数据采集

* 查看警报日志
* 通过数据字典采集日志
* 通过AWR报告

```sql
1 show parameter background
2 以dba_XXX,user_XXX,all_XXX为开头的数据字典,数据变化不大，静态的
  以v$XXX 开头的动态性能视图可作为Oracle性能分析的根据
3 SQL> @?/rdbms/admin/awrrpt.sql
  查看数据库快照：
SQL> desc dba_hist_snapshot
SQL> select SNAP_ID,STARTUP_TIME from dba_hist_snapshot order by 1;
  手动生成快照
SQL> exec dbms_workload_repository.create_snapshot()
  导出快照
  首先查看数据库的逻辑目录
  SQL> @?/rdbms/admin/awrextr.sql
  目标端需要存在逻辑目录，用来导入快照
  SQL> @?/rdbms/admin/awrload.sql
  SQL> @?/rdbms/admin/awrrpti.sql

```

## 针对全表扫描，优化IO

减少和分散

查看热点盘：分散热点盘中的数据

SQL> select FILE#,PHYRDS,PHYWRTS,PHYBLKRD,PHYBLKWRT from v$filestat;



### 减少IO

合理使用索引

消除段的碎片，压缩高水位

```sql
如果需要将表的空间进行缩小，有两个办法：
（1）move
（2）shrink

1、对于空间的要求，shrink不需要额外的空间，move需要两倍的空间

2、shrink的算法是从segment的底部开始，移动row到segment的顶部，移动的过程相当于delete/insert操作的组合，在这个过程中会产生大量的undo和redo信息。

3、move是直接移动数据块的位置，鉴于上面的原因，在使用shrink的时候，耗时可能非常长，通常慢于move。

对于一般系统，可采用shrink方式（仅仅适用于堆表,且位于自动段空间管理的表空间(堆表包括:标准表,分区表,物化视图容器,物化视图日志表)）：

（1）启用行记录转移(enable row movement)        
alter table wxk_big_tab_bak enable row movement ;

（2）这里先执行compact 再 到空闲时间执行 cascade
ALTER TABLE wxk_big_tab_bak SHRINK SPACE COMPACT
ALTER TABLE wxk_big_tab_ba SHRINK SPACE cascade 
--compact:仅仅是缩小表和索引，并不移动高水位线，不释放空间
--cascade:缩小表及其索引，并移动高水位线，释放空间
alter table app_info disable row movement; --关闭行移动
（3） 整理完碎片后最好重新收集统计信息：
begin 
dbms_stats.gather_table_stats(ownname => 'chunqiu',tabname => 'app_info',cascade => true);
end;

对于重要系统，最好还是采用move

（1）move前最好逻辑备份待整理的表；
（2）--查看索引
select index_name,table_name,tablespace_name,index_type,status  from dba_indexes  where table_owner='WXK'
（3）对于大表，建议开启并行和nologging
 alter table wxk_big_tab_bak MOVE nologging parallel 2;
（4）整理完毕后重建相关的索引
 alter index SYS_C0011093 rebuild nologging parallel 2;
（5）恢复表和索引的并行度、logging
 alter table wxk_big_tab_bak logging parallel 1;
 alter table app_info disable row movement; --关闭行移动
（6） 收集统计信息
```



```sql
1识别全表扫描
select OBJECT_OWNER,OBJECT_NAME,OPERATION,OPTIONS FROM v$sql_plan where OPTIONS='FULL' and OBJECT_OWNER='SCOTT';

2确定表的数据量:
select count(*) from scott.ob1;

3查看表的碎片情况:
set serverout on
declare
V_UNFORMATTED_BLOCKS NUMBER;	
V_UNFORMATTED_BYTES NUMBER;	
V_FS1_BLOCKS NUMBER;		
V_FS1_BYTES NUMBER;		
V_FS2_BLOCKS NUMBER;		
V_FS2_BYTES NUMBER;		
V_FS3_BLOCKS NUMBER;		
V_FS3_BYTES NUMBER;		
V_FS4_BLOCKS NUMBER;		
V_FS4_BYTES NUMBER;		
V_FULL_BLOCKS NUMBER;		
V_FULL_BYTES NUMBER;
begin
dbms_space.space_usage(
'SCOTT',
'OB1',
'TABLE',
V_UNFORMATTED_BLOCKS,
V_UNFORMATTED_BYTES,	
V_FS1_BLOCKS,		
V_FS1_BYTES,		
V_FS2_BLOCKS,		
V_FS2_BYTES,		
V_FS3_BLOCKS,		
V_FS3_BYTES,		
V_FS4_BLOCKS,		
V_FS4_BYTES,		
V_FULL_BLOCKS,		
V_FULL_BYTES
);
dbms_output.put_line('0~25%: '||V_FS1_BLOCKS);
dbms_output.put_line('25~50%: '||V_FS2_BLOCKS);
dbms_output.put_line('50~75%: '||V_FS3_BLOCKS);
dbms_output.put_line('75~100%: '||V_FS4_BLOCKS);
end;
/

4.消除碎片，优化全表扫描：
打开sql跟踪功能：
set autot trace exp   看看表的查询运行成本
如果是dynamic sampling   则分析表analyze table ob1 compute statistics;
再运行sql查看成本

消除碎片   alter table ob1 move;  要保证表空间里面有足够的空闲
再分析analyze table ob1 compute statistics;获得统计信息
再运行sql查看成本
alter table ob1 move tablespace xxxx;
```

### 分散IO

在表空间下使用多个数据文件

使用段的分区技术

### 提高io的吞吐量

```shell
【提高io的吞吐量】：_db_file_optimizer_read_count   针对全表扫描，修改此参数提高，批量读取块的数量
SQL> select indx,ksppinm from x$ksppi where ksppinm like '%read_count%';

      INDX KSPPINM
---------- -----------------------------
      1156 _db_file_optimizer_read_count

SQL> select KSPPSTVL from X$KSPPCV where INDX=1156;

KSPPSTVL
----------
8



SQL> alter session set "_db_file_optimizer_read_count"=16;    加上双引号是因为_是隐含参数的命名

Session altered.
注释：放大IO吞吐量受到主机系统的IO吞吐量的限制。

##PS:事件10053和trace文件的标记
10053事件就提供了这样的功能。它产生的trace文件提供了Oracle如何选择执行计划，为什么会得到这样的执行计划信息。
1）标记trace文件
SQL> alter session set tracefile_identifier='10053';
2）查看当前trace文件
SELECT u_Dump.Value || '/' || Lower(Db_Name.Value) || '_ora_' ||
       V$process.Spid ||
       Nvl2(V$process.Traceid, '_' || V$process.Traceid, NULL) || '.trc' "Trace File"
  FROM V$parameter u_Dump
 CROSS JOIN V$parameter Db_Name
 CROSS JOIN V$process
  JOIN V$session
    ON V$process.Addr = V$session.Paddr
 WHERE u_Dump.Name = 'user_dump_dest'
   AND Db_Name.Name = 'db_name'
   AND V$session.Audsid = Sys_Context('userenv', 'sessionid');
3）打开10053事件
ALTER SESSION SET EVENTS='10053 trace name context forever, level 1';
4）执行操作
5）关闭事件
ALTER SESSION SET EVENTS '10053 trace name context off';



【使用并型查询提高全表扫描的效率】：
select /*+parallel (ob1 8)*/ * from ob1;
8需要有8颗CPU才能开8个parallel，    sql强制执行

select username,program from v$session where username='SCOTT';
查看链接的Scott的会话级别连接。
```

## 内存优化

```sql 
select MEMORY_SIZE,ESTD_DB_TIME from v$memory_target_advice; 内存优化数据字典
在内存自动管理模式下通过：v$pgastat v$sgainfo 查看

[root@oratest shm]# ipcs -sm  查看内存信号量


SYS>show spparameter memory;

从spfile中删除11g内存自动管理的参数:
SQL> alter system reset memory_target scope=spfile sid='*';
alter system set pga_aggregate_target=200m scope=spfile;
alter system set sga_target=500m scope=spfile;
SQL> startup force


PGA的管理（10g内存风格）
select  PID,PGA_USED_MEM/1024/1024 from v$process;
后台进程查看的PGA使用情况
SQL> select p.pid,b.name,p.PGA_USED_MEM/1048576 mb from v$bgprocess b,v$process p where p.addr=b.paddr;
根据会话查找pga使用情况：
SQL> select PGA_USED_MEM/1048576 from v$process where addr in (select paddr from v$session where username='SCOTT');
查看pga的调整建议：
SQL> select PGA_TARGET_FOR_ESTIMATE/1048576,ESTD_PGA_CACHE_HIT_PERCENTAGE from v$pga_target_advice; 
pga_aggregate_target -->不是所有进程总和的上限，而是单个pga扩张的上限

控制pga手动管理的参数：workarea_size_policy 
修改此参数为manual手动管理，进行大表查询排序，不断手动放大排序区，在1M之后，排序成本不下降。因为此时收到系统IO的制约（以系统最大IO往TEMP表空间写排序结果）；
当排序区放大到足够容纳内存中排序，可见排序成本明显下降！


SGA的自动管理
SQL> select * from v$sgainfo;
SQL> desc v$sga_target_advice;
在自动管理模式下，通过如下参数控制各个内存池收缩的最小值：
shared_pool_size
db_cache_size
large_pool_size
jave_pool_size
streams_pool_size

eg:SQL>  alter system set db_cache_size=200M;
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
SGA手动管理：
alter system set SGA_TARGET=0;
1）shared_pool内存中存放SQL pl/sql语句
使用SQL文本做模糊匹配
select SQL_TEXT from v$sqlarea where SQL_TEXT like 'select * from scott.%';
使用SQL的hash值
SQL> grant select on V_$MYSTAT to SCOTT;
SQL> select sid from v$mystat where rownum=1;
SQL> select SQL_HASH_VALUE,PREV_HASH_VALUE from v$session where sid=22;
SQL> select SQL_HASH_VALUE,PREV_HASH_VALUE from v$session where username='SCOTT';
SQL> select SQL_TEXT from v$sqlarea where hash_value=3842168405;
spid通过top命令确定
select sql_text from v$sqlarea where hash_value in (select  sql_hash_value from v$session s ,v$process p where p.addr=s.paddr and p.spid=3672);

alter system flush shared_pool;清空shared_pool
 
##绑定变量优点：
游标无需进行重解析！
SQL> var b1 number
SQL> select ename from emp where empno =: b1;

no rows selected

SQL> exec :b1:=7499;

PL/SQL procedure successfully completed.

SQL> select ename from emp where empno =: b1;

ENAME
--------------------
ALLEN

如何确定重复解析的SQL：
select substr(sql_text ,1,25) sqltext ,count(*) from v$sqlarea having count(*) >5 group by substr(sql_text ,1,25); 确定多次出现的SQL
select SQL_TEXT from v$sqlarea where SQL_TEXT like 'select * from scott.%';
绑定变量的缺点：
键值分布极度倾斜，并且表拥有列的统计信息（计算列的键值分布的均匀情况）；绑定变量使得优化器执行计划选择出错
eg:
构造键值分布倾斜的表
SQL> select object_id,count(*) from ob1 group by object_id order by 2;

 OBJECT_ID   COUNT(*)
---------- ----------
         1         39
         2        360
         3       3600
         4      36000
         5      46958
SQL> create index ob1_idx on ob1(object_id);

Index created.

SQL> analyze table ob1 compute statistics;

Table analyzed.

SQL> set autot trace exp        
SQL> select * from ob1 where object_id=1;

Execution Plan
----------------------------------------------------------
Plan hash value: 2526347053

---------------------------------------------------------------------------------------
| Id  | Operation                   | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |         | 17391 |  1681K|   283   (0)| 00:00:04 |
|   1 |  TABLE ACCESS BY INDEX ROWID| OB1     | 17391 |  1681K|   283   (0)| 00:00:04 |
|*  2 |   INDEX RANGE SCAN          | OB1_IDX | 17391 |       |    34   (0)| 00:00:01 |
---------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("OBJECT_ID"=1)

SQL> select * from ob1 where object_id=2;

Execution Plan
----------------------------------------------------------
Plan hash value: 2526347053

---------------------------------------------------------------------------------------
| Id  | Operation                   | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |         | 17391 |  1681K|   283   (0)| 00:00:04 |
|   1 |  TABLE ACCESS BY INDEX ROWID| OB1     | 17391 |  1681K|   283   (0)| 00:00:04 |
|*  2 |   INDEX RANGE SCAN          | OB1_IDX | 17391 |       |    34   (0)| 00:00:01 |
---------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("OBJECT_ID"=2)

SQL> select * from ob1 where object_id=3;

Execution Plan
----------------------------------------------------------
Plan hash value: 2526347053

---------------------------------------------------------------------------------------
| Id  | Operation                   | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |         | 17391 |  1681K|   283   (0)| 00:00:04 |
|   1 |  TABLE ACCESS BY INDEX ROWID| OB1     | 17391 |  1681K|   283   (0)| 00:00:04 |
|*  2 |   INDEX RANGE SCAN          | OB1_IDX | 17391 |       |    34   (0)| 00:00:01 |
---------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("OBJECT_ID"=3)

SQL> select * from ob1 where object_id=4;

Execution Plan
----------------------------------------------------------
Plan hash value: 2526347053

---------------------------------------------------------------------------------------
| Id  | Operation                   | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |         | 17391 |  1681K|   283   (0)| 00:00:04 |
|   1 |  TABLE ACCESS BY INDEX ROWID| OB1     | 17391 |  1681K|   283   (0)| 00:00:04 |
|*  2 |   INDEX RANGE SCAN          | OB1_IDX | 17391 |       |    34   (0)| 00:00:01 |
---------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("OBJECT_ID"=4)

SQL> select * from ob1 where object_id=5;

Execution Plan
----------------------------------------------------------
Plan hash value: 2526347053

---------------------------------------------------------------------------------------
| Id  | Operation                   | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |         | 17391 |  1681K|   283   (0)| 00:00:04 |
|   1 |  TABLE ACCESS BY INDEX ROWID| OB1     | 17391 |  1681K|   283   (0)| 00:00:04 |
|*  2 |   INDEX RANGE SCAN          | OB1_IDX | 17391 |       |    34   (0)| 00:00:01 |
---------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("OBJECT_ID"=5)
   注意：全部走索引，重新收集一下列的统计信息
   
SQL> analyze table ob1 compute statistics for all indexed columns;

Table analyzed.

SQL> select * from ob1 where object_id=4;

Execution Plan
----------------------------------------------------------
Plan hash value: 3207514228

--------------------------------------------------------------------------
| Id  | Operation         | Name | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |      | 36000 |  3480K|   347   (1)| 00:00:05 |
|*  1 |  TABLE ACCESS FULL| OB1  | 36000 |  3480K|   347   (1)| 00:00:05 |
--------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter("OBJECT_ID"=4)
   注意：假如执行计划还是不准确，表的碎片可能很严重，需要消除碎片
   SQL> alter table ob1 move;  move之后需要重建索引
   SQL> alter index OB1_IDX rebuild;
   重新收集统计信息和索引统计信息
   SQL> analyze table ob1 compute statistics;
   SQL> analyze table ob1 compute statistics for all indexed columns;
   
SQL> var b1 number
SQL> select * from ob1 where object_id=:b1;

Execution Plan
----------------------------------------------------------
Plan hash value: 2526347053

---------------------------------------------------------------------------------------
| Id  | Operation                   | Name    | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |         | 17391 |  1681K|   278   (0)| 00:00:04 |
|   1 |  TABLE ACCESS BY INDEX ROWID| OB1     | 17391 |  1681K|   278   (0)| 00:00:04 |
|*  2 |   INDEX RANGE SCAN          | OB1_IDX | 17391 |       |    34   (0)| 00:00:01 |
---------------------------------------------------------------------------------------
注意：使用绑定变量后，执行计划是走索引的，如变量等于5的时候计划计划有误！

##共享池保留区
SQL> show parameter shared_pool_reserved_size 

NAME                                 TYPE                   VALUE
------------------------------------ ---------------------- ------------------------------
shared_pool_reserved_size            big integer            6920601

SQL> alter session set events 'immediate trace name heapdump level 2'; 转储共享池的空间到用户跟踪文件

Session altered.

SQL> select indx,ksppinm from  x$ksppi where ksppinm like '%reserved%';

      INDX KSPPINM
---------- ---------------------------------------
       136 _simulator_reserved_obj_count
       137 _simulator_reserved_heap_count
       180 shared_pool_reserved_size
       181 _shared_pool_reserved_pct
       182 _shared_pool_reserved_min_alloc

SQL> select KSPPSTVL from x$ksppcv where INDX=182;

KSPPSTVL
-----------------
4400       
因为sharedpool里面分配的hash bucket比较碎，用来针对不同大小的SQL；容易产生碎片！
主要用来解决共享池内存碎片缓存SQL不了的问题！最小分配单位是4400个字节


##共享池使得对象常驻内存
SQL> desc dbms_shared_pool  --使用此包来实现
1）查看内存缓存的对象
SQL> desc v$db_object_cache  ---用来查看内存缓存的对象
SQL> col OWNER for a20
SQL> col NAME for a20 
SQL> col TYPE for a30
SQL> select owner,name,type from v$db_object_cache where type in ('PACKAGE','BODY');
SQL> alter system flush shared_pool;  ---清理共享池之后只有10个对象缓存


2）执行系统包，常驻内存
SQL> exec dbms_shared_pool.keep('sys.DBMS_LOGSTDBY');

PL/SQL procedure successfully completed.
SQL> exec dbms_shared_pool.unkeep('sys.DBMS_LOGSTDBY'); 释放


SQL>  select owner,name,type,kept  from v$db_object_cache where type in ('PACKAGE','BODY');查询

##PS：模糊匹配相关的系统v$视图;
SQL> select NAME from v$fixed_table where name like 'V$%ADVICE%';

NAME
------------------------------------------------------------
V$SHARED_POOL_ADVICE
V$JAVA_POOL_ADVICE
V$STREAMS_POOL_ADVICE
V$PX_BUFFER_ADVICE
V$MEMORY_TARGET_ADVICE
V$SGA_TARGET_ADVICE
V$DB_CACHE_ADVICE
V$MTTR_TARGET_ADVICE
V$PGA_TARGET_ADVICE_HISTOGRAM
V$PGA_TARGET_ADVICE


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
缓冲区高速缓存管理 buffer cache
show parameter db_cache_size
PS： 表在内存中缓存的块
SQL> select data_object_id from user_objects where object_name='EMP';

DATA_OBJECT_ID
--------------
         87108
SQL> select FILE#,BLOCK# ,DIRTY  from v$bh where OBJD=87108;
SQL> select FILE_ID,BLOCK_ID ,BLOCKS from dba_extents where SEGMENT_NAME  ='OB1'; 查看对象存储情况！

系统buffer_cache建议
select SIZE_FOR_ESTIMATE,ESTD_PHYSICAL_READS,name from v$db_cache_advice;
SIZE_FOR_ESTIMATE ESTD_PHYSICAL_READS NAME
----------------- ------------------- ----------------------------------------
               32               24438 DEFAULT
               64               22331 DEFAULT
高速缓冲池有三种：default，
               db_keep_cache:趋向于常驻内存，适用于缓存经常访问，不大的对象
               db_recycle_cache：不做长时间缓存。不经常访问的大对象
查看对象属性，缓存在内存的哪里：
SQL> select table_name,buffer_pool  from user_tables;
SQL> alter table DEPT storage(buffer_pool keep);修改

查看buffer cache命中
SQL> select NAME,1-PHYSICAL_READS/(DB_BLOCK_GETS+CONSISTENT_GETS) from v$buffer_pool_statistics;
配置 db keep cache
缩小buffer cache
SQL> alter system set db_cache_size=330M;
分配 keep cache
SQL> alter system set db_keep_cache_size=8M;
清空buffer cache
SQL> alter system flush BUFFER_CACHE;
操作keep缓存表
select * from scott.dept;
查看命中
SQL> select NAME,1-PHYSICAL_READS/(DB_BLOCK_GETS+CONSISTENT_GETS) from v$buffer_pool_statistics;

NAME                                     1-PHYSICAL_READS/(DB_BLOCK_GETS+CONSISTENT_GETS)
---------------------------------------- ------------------------------------------------
KEEP                                                                                .5625
DEFAULT                                                                        .932216976


buffer cache中的锁
查看等待事件等级：
SQL> select distinct  WAIT_CLASS#,WAIT_CLASS from v$event_name order by 1;

WAIT_CLASS# WAIT_CLASS
----------- --------------------------------------------------------------------------------------------------------------------------------
          0 Other
          1 Application
          2 Configuration
          3 Administrative
          4 Concurrency
          5 Commit
          6 Idle
          7 Network
          8 User I/O
          9 System I/O
         10 Scheduler

WAIT_CLASS# WAIT_CLASS
----------- --------------------------------------------------------------------------------------------------------------------------------
         11 Cluster
         12 Queueing
查看用户等待事件
SQL>  select event ,WAIT_CLASS# from v$session where username='SCOTT' and  WAIT_CLASS# <> 6;

锁的内存地址
SQL>  select event ,WAIT_CLASS#,p1,p1raw from v$session where username='SCOTT' and  WAIT_CLASS# <> 6;   p1raw是锁的内存地址
select FILE#,DBABLK,OBJ,TCH  from x$bh where HLADDR=p1raw;  看锁管理的块！
 
```



## DML 优化

```SQL
1) insert 数据装载加速
append + nologing
为了提高插入的速度，我们可以对表关闭写log功能。 SQL 如下：
sql> alter table table_name NOLOGGING; 
插入/修改,完数据后，再修改表写日志：  
sql> alter table table_name LOGGING;
SQL> insert /*+ APPEND */  into test1 select * from dba_objects;
```



## 消除行迁移

```sql
1）查看行迁移的情况
SQL> analyze table ob1 compute statistics;

Table analyzed.

SQL>  select BLOCKS,TABLE_NAME,EMPTY_BLOCKS ,AVG_ROW_LEN ,CHAIN_CNT from user_tables where TABLE_NAME='OB1';

    BLOCKS TABLE_NAME                                                   EMPTY_BLOCKS AVG_ROW_LEN  CHAIN_CNT
---------- ------------------------------------------------------------ ------------ ----------- ----------
      1247 OB1                                                                    33          99          0

2）消除行迁移
注意：如果一张表经常出现行迁移，证明表结构有问题；
	 方法一：
	 SQL> analyze table t compute statistics; 
     SQL> select table_name,num_rows,CHAIN_CNT from user_tables where table_name='T'; 
     SQL> alter table t move ; 
　　 SQL> analyze table t compute statistics; 
　   SQL>　select table_name,num_rows,CHAIN_CNT from user_tables where table_name='T'; 
　　 Index 需要rebuild 或者删除重建 
	  
	 方法二：
	 消除行迁移的方法是利用rowid备份行迁移的数据，delete原始表中的行迁移数据，重新insert
     SQL>@?/rdbms/admin/utlchain.sql
     将表中的chained row移动到chained_rows表
     sql> analyze table test_table_name list chained rows;
     查看
     SQL> select table_name,head_rowid from chained_rows where table_name = 'T01';
     创建中间表
     CREATE TABLE t02 AS SELECT * FROM t01 WHERE ROWID IN (SELECT HEAD_ROWID FROM CHAINED_ROWS WHERE TABLE_NAME = 'T01');
     删除行迁移的数据
     从已经存在的表student_infor中删除链接行和迁移行
     DELETE FROM t01 WHERE ROWID IN (SELECT HEAD_ROWID FROM CHAINED_ROWS WHERE TABLE_NAME = 'T01');
     插入中间表的数据
     INSERT INTO t01 SELECT * FROM t02;  
```



​    

## 索引

```sql
索引的逻辑分类：
单列索引和复合索引
1）查看
SQL> select INDEX_NAME,INDEX_TYPE ,UNIQUENESS from user_indexes where table_name='EMP';
SQL> create index i_emp_ename on emp(ename);
SQL> create index i_emp_ename_sal on emp(ename,sal);
非唯一键索引和唯一键索引
create unique index i_emp_ename on emp(ename);

**alter table  emp add constraint uk_ename unique (ename);
**说明唯一键会自动生成唯一索引
**eg:
     SQL> insert into emp (empno,ename) values (1,'SCOTT');
     insert into emp (empno,ename) values (1,'SCOTT')
     *
     ERROR at line 1:
     ORA-00001: unique constraint (SCOTT.UK_ENAME) violated
     
     插入失败，禁用唯一键
     SQL> alter table emp modify constraint UK_ENAME disable;
     
     Table altered.
     
     SQL> insert into emp (empno,ename) values (1,'SCOTT');
     
     1 row created.
     启动唯一键失败，启用不检查现有数据还是失败，因为Oracle自动创建索引起不来，创建一个索引，就过去了！
     SQL> alter table emp modify constraint UK_ENAME enable;
     alter table emp modify constraint UK_ENAME enable
     *
     ERROR at line 1:
     ORA-02299: cannot validate (SCOTT.UK_ENAME) - duplicate keys found
     
     
     SQL> alter table emp modify constraint UK_ENAME enable novalidate;
     alter table emp modify constraint UK_ENAME enable novalidate
     *
     ERROR at line 1:
     ORA-02299: cannot validate (SCOTT.UK_ENAME) - duplicate keys found
     
     
     SQL> create index i_ename on emp(ename);
     
     Index created.
     
     SQL> alter table emp modify constraint UK_ENAME enable novalidate;
     
     Table altered.
     
     SQL> select INDEX_NAME,INDEX_TYPE ,UNIQUENESS from user_indexes where table_name='EMP';
     
     INDEX_NAME                                                   INDEX_TYPE                                             UNIQUENESS
     ------------------------------------------------------------ ------------------------------------------------------ ------------------
     PK_EMP                                                       NORMAL                                                 UNIQUE
     I_ENAME                                                      NORMAL                                                 NONUNIQUE



函数索引
SQL> create index i_ename_UP on emp (UPPER(ENAME));
应用程序域索引

索引的物理分类：
非分区索引和分区索引
create index global_index on student_range_part(credit) global
partition by range (credit)
(partition index_part1 values less than (60),
partition index_part2 values less than (80),
partition index_partmax values less than (maxvalue)
);
全局索引要求最高分区（即最后一个分区）必须有一个值为 maxvalue 的最大上限值，这样可以确保底层表的所有行都能放在这个索引中；
一般情况下，大多数分区操作（如删除一个旧分区）都会使全局索引无效，除非重建全局索引，否则无法使用

BTREE索引：不记录NULL，将关键字和rowid的组合放在索引段
常规BTREE
反键BTREE

bitmap位图索引：减少重复关键字在索引段中的存储，适合创建在重复值多的列


+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

索引的结构
analyze index I_ENAME validate structure;   -----分析索引结构有效性
SQL> select HEIGHT,NAME,BLOCKS,BR_BLKS,LF_BLKS,LF_ROWS,BR_ROWS from  index_stats ;           

    HEIGHT NAME                                                             BLOCKS    BR_BLKS    LF_BLKS    LF_ROWS    BR_ROWS
---------- ------------------------------------------------------------ ---------- ---------- ---------- ---------- ----------
         1 I_ENAME                                                               8          0          1         15          0
         
索引合并；合并索引碎片，不能降低二元高度
SQL> alter index I_ENAME coalesce;
analyze index I_ENAME validate structure; 


+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
影响索引使用的因素：
注意：使用count（*） 计数时，走索引的条件是需要索引的所在列要有非空约束！
1 查询返回的数据量
2 隐式的数据类型的转换
3 初始化参数影响索引使用
  --SQL> alter session set optimizer_index_cost_adj=90;  取值为1~10000
  此参数默认100即100%；
  成本比较如下： 全表扫描成本 VS  索引访问成本*optimizer_index_cost_adj  索引权重参数越小，索引越重要！
  
  --optimizer_mode=all_rows（按照返回的数据量做执行计划）|first_rows_100(不按照IO做计划；按照快速返回前100行做执行计划，从响应时间的角度做执行计划，因此执行计划以索引为主！适合分页查询)
  
4 索引聚簇因子影响索引访问
查看返回结果，数据块的分布情况！
select dbms_rowid.rowid_block_number(rowid),count(*) from scott.ob1 where object_id<5000 group by dbms_rowid.rowid_block_number(rowid);
查看高水位下数据块的数量：
SQL> select blocks from dba_tables where table_name='OB1'; 
聚簇因子反应表中的数据分布状态：顺序访问索引键值在块上跳转的次数。


+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
索引的使用实例
1 索引扫描算法
  INDEX FAST FULL SCAN 不读取分支和根块！只读叶子块！根据全表扫描的IO批量读取！返回的结果无序！
  INDEX FULL SCAN 结果集有序，根--枝--叶顺序访问索引，单块读取！
  INDEX UNIQUE SCAN ：找到符合条件的键值直接返回！
  INDEX RANGE SCAN ：扫描一个区间内的所有关键字！
  INDEX SKIP SCAN： 1 复合索引 2 前导列在where条件中不出现 3要拥有列的统计信息（analyze table ob1 compute statistics for all indexed columns）
```



## 存储的规划与使用

```
SQL> alter system set db_cache_size =320M;
SQL> alter system set db_16K_cache_size = 12M;
SQL> create tablespace mytbs datafile '/home/oracle/mytbs.tbf' size 10M blocksize 16K;
SQL> select block_size ,tablespace_name from dba_tablespaces;

BLOCK_SIZE TABLESPACE_NAME
---------- ------------------------------------------------------------
      8192 SYSTEM
      8192 SYSAUX
      8192 UNDOTBS1
      8192 TEMP
      8192 USERS
      8192 EXAMPLE
     16384 MYTBS
```



## 压缩表和分区表

```SQL
段的压缩技术：减少IO 消耗CPU；
SQL> alter table ob2 compress  for oltp;
段的分区技术
分区表、索引、分区索引，要利用其性能优势，最基本就是要提取数据时，要通过它首先将数据的范围缩小到一个即使做全盘扫描也不会太慢的情况。
所以SQL一定要有分区上的这个字段的一个WHERE条件，将数据迅速定位到分区内部

range分区：
CREATE TABLE ORDER_ACTIVITIES 
( 
    ORDER_ID      NUMBER(7) NOT NULL, 
    ORDER_DATE    DATE, 
    TOTAL_AMOUNT NUMBER, 
    CUSTOTMER_ID NUMBER(7), 
    PAID           CHAR(1) 
) 
PARTITION BY RANGE (ORDER_DATE) 
(
  PARTITION ORD_ACT_PART01 VALUES LESS THAN (TO_DATE('01- MAY -2003','DD-MON-YYYY')) TABLESPACEORD_TS01,
  PARTITION ORD_ACT_PART02 VALUES LESS THAN (TO_DATE('01-JUN-2003','DD-MON-YYYY')) TABLESPACE ORD_TS02,
  PARTITION ORD_ACT_PART02 VALUES LESS THAN (TO_DATE('01-JUL-2003','DD-MON-YYYY')) TABLESPACE ORD_TS03
)

list分区
CREATE TABLE PROBLEM_TICKETS 
( 
    PROBLEM_ID   NUMBER(7) NOT NULL PRIMARY KEY, 
    DESCRIPTION  VARCHAR2(2000), 
    CUSTOMER_ID  NUMBER(7) NOT NULL, 
    DATE_ENTERED DATE NOT NULL, 
    STATUS       VARCHAR2(20) 
) 
PARTITION BY LIST (STATUS) 
( 
      PARTITION PROB_ACTIVE   VALUES ('ACTIVE') TABLESPACE PROB_TS01, 
      PARTITION PROB_INACTIVE VALUES ('INACTIVE') TABLESPACE PROB_TS02);
      
hash分区
CREATE TABLE emp
(
    empno NUMBER (4),
    ename VARCHAR2 (30),
    sal   NUMBER 
)
PARTITION BY  HASH (empno) PARTITIONS 8
STORE IN (emp1,emp2,emp3,emp4,emp5,emp6,emp7,emp8);

复合分区
create table p_emp5 (id number,ename varchar2(12),deptno number(4))
partition by range (deptno) 
subpartition by hash (id) subpartitions 8 store in (tbs1,tbs3,tbs5,tbs7) 
(partition p1 values less than (100),
partition p2 values less than (200) store in (tbs2,tbs4,tbs6,tbs8),
partition p3 values less than (300) 
(subpartition p3_1 tablespace users,
subpartition p3_2 tablespace example));
```



### 在线重定义表

```sql
将普通堆表进行分区
联机重定义表：ETL

  CREATE TABLE "SCOTT"."OB1"
   (    "OWNER" VARCHAR2(30),
        "OBJECT_NAME" VARCHAR2(128),
        "SUBOBJECT_NAME" VARCHAR2(30),
        "OBJECT_ID" NUMBER,
        "DATA_OBJECT_ID" NUMBER,
        "OBJECT_TYPE" VARCHAR2(19),
        "CREATED" DATE,
        "LAST_DDL_TIME" DATE,
        "TIMESTAMP" VARCHAR2(19),
        "STATUS" VARCHAR2(7),
        "TEMPORARY" VARCHAR2(1),
        "GENERATED" VARCHAR2(1),
        "SECONDARY" VARCHAR2(1),
        "NAMESPACE" NUMBER,
        "EDITION_NAME" VARCHAR2(30)
   ) SEGMENT CREATION IMMEDIATE
  PCTFREE 10 PCTUSED 40 INITRANS 1 MAXTRANS 255
 NOCOMPRESS LOGGING
  STORAGE(INITIAL 65536 NEXT 1048576 MINEXTENTS 1 MAXEXTENTS 2147483645
  PCTINCREASE 0 FREELISTS 1 FREELIST GROUPS 1
  BUFFER_POOL DEFAULT FLASH_CACHE DEFAULT CELL_FLASH_CACHE DEFAULT)
  TABLESPACE "USERS"
  


     
     CREATE TABLESPACE tbs1 DATAFILE '/u01/app/oracle/oradata/ARIS/tbs01.dbf' SIZE 10M AUTOEXTEND ON NEXT 1310720 MAXSIZE 3000M;
     CREATE TABLESPACE tbs2 DATAFILE '/u01/app/oracle/oradata/ARIS/tbs02.dbf' SIZE 10M AUTOEXTEND ON NEXT 1310720 MAXSIZE 3000M;
	 CREATE TABLESPACE tbs3 DATAFILE '/u01/app/oracle/oradata/ARIS/tbs03.dbf' SIZE 10M AUTOEXTEND ON NEXT 1310720 MAXSIZE 3000M;
	 
	 创建中间表
	 CREATE TABLE "SCOTT"."P_OB1"
   (    "OWNER" VARCHAR2(30),
        "OBJECT_NAME" VARCHAR2(128),
        "SUBOBJECT_NAME" VARCHAR2(30),
        "OBJECT_ID" NUMBER,
        "DATA_OBJECT_ID" NUMBER,
        "OBJECT_TYPE" VARCHAR2(19),
        "CREATED" DATE,
        "LAST_DDL_TIME" DATE,
        "TIMESTAMP" VARCHAR2(19),
        "STATUS" VARCHAR2(7),
        "TEMPORARY" VARCHAR2(1),
        "GENERATED" VARCHAR2(1),
        "SECONDARY" VARCHAR2(1),
        "NAMESPACE" NUMBER,
        "EDITION_NAME" VARCHAR2(30)
   ) partition by range(object_id)(
   partition P1 values less than (30000) tablespace tbs1,
   partition P2 values less than (60000) tablespace tbs2,
   partition P3 values less than (maxvalue) tablespace tbs3);
   
   
   
   预检和rowid预检
   SQL>  EXEC DBMS_REDEFINITION.CAN_REDEF_TABLE('SCOTT','OB1',2);
   alter table ob1 add constraint pk_objects primary key (object_id);
   SQL>  EXEC DBMS_REDEFINITION.CAN_REDEF_TABLE('SCOTT','OB1');
   
   开始重定义
   SQL> exec DBMS_REDEFINITION.START_REDEF_TABLE('SCOTT','OB1','P_OB1');
   同步数据
   SQL> exec DBMS_REDEFINITION.SYNC_INTERIM_TABLE (uname => 'SCOTT',orig_table  => 'OB1',int_table  => 'P_OB1');
   SQL> exec DBMS_REDEFINITION.SYNC_INTERIM_TABLE (uname => 'SCOTT',orig_table  => 'OB1',int_table  => 'P_OB1');
   结束重定义
   SQL> EXEC DBMS_REDEFINITION.FINISH_REDEF_TABLE('SCOTT','OB1','P_OB1'); 
   注释：
   在FINISH_REDEF_TABLE之前，可以使用abort_redef_table停止重定义
   SQL> exec DBMS_REDEFINITION.ABORT_REDEF_TABLE ('test1','t_objects','t_objects_temp');
   
   参考：
   https://www.linuxidc.com/Linux/2017-06/145083.htm
   
--经实验，在开始重定义之后在临时表上创建local索引，重定义完成后，主键对应的索引也是分区索引;
alter table t_objects_temp add constraint pk_objects_temp primary key (created, object_id) using index local;
create index i_objects_temp on t_objects_temp(object_id, STATUS) local;
--数据同步完成之后
9,修改索引，约束名称和原表一致
alter index I_OBJECTS_TEMP rename to I_OBJECTS;
alter index PK_OBJECTS_TEMP rename to PK_OBJECTS;
alter table t_objects rename constraint pk_objects_temp to pk_objects;
 
```



### 分区表的索引

```sql
分区表的本地分区索引
create index i_emp_ename on emp(ename) local; ---每个表分区配一个索引分区
全局分区索引
   CREATE [url=]INDEX[/url] INX_TAB_PARTITION_COL1 ON TABLE_PARTITION(COL1)
   GLOBAL PARTITION BY RANGE(COL1)
   PARTITION IDX_P1 values less than (1000000),
   PARTITION IDX_P2 values less than (2000000),
   PARTITION IDX_P3 values less than (MAXVALUE))
```



## IOT表和集簇表

```
IOT：插入慢，查询快，数据存放极度有序，数据存放在索引段
create table iot_emp (empno number constraint PK_EMPNO primary key,
name varchar2(20),sal number(7,2)) organization index;

cluster：加速表连接查询 浪费空间 数据字典用的多
SQL> create cluster my_cls (deptno number(2));
SQL> create index my_cls_idx on cluster my_cls;
SQL> create table c_dept cluster my_cls(deptno) as select * from dept;
SQL> create table c_emp cluster my_cls(deptno) as select * from emp;


```



## sql优化的步骤

```SQL
1.定位高消耗的SQL语句
2.查看SQL语句涉及到的对象的统计信息是否是最新的
  select TABLE_NAME,to_char(last_analyzed ,'yyyy-mm-dd hh24:mi:ss') last_analyzed from user_tables;
3.查看执行计划
4.分析原因
5.将优化的结果永久化！
```



## 优化器

```SQL
RBO：基于SQL上下文，以及对象属性指定执行计划

CBO：基于成本，制定执行计划
select /*+rule*/ * from ob1 where object_id<80000;

成本计算
成本=（IO消耗的时间+CPU消耗的时间）/单块读取的时间

计算方法：
SQL> select * from aux_stats$;
单块读取的时间=IO的探查时间+db_block_size/IO传输速度 约为10ms

IO消耗的时间=批量读取的时间+单块的读取时间
批量读取的时间=IO的次数*一次批量读取的时间
计算方法：IO的次数=高水位以下块的数量/一次批量读取的块数  1340/8个
eg:SQL> select BLOCKS from user_tables where TABLE_NAME='OB1';

    BLOCKS
----------
      1340
	      一次批量读取的时间=IO探查时间+（一次批量读取的块数*8K）/IO传输速度  =10+（8*8K）/4K=26ms
		  IO消耗的时间=（1340/8)*26ms
		  
		  
CPU的时间消耗=SQL在cpu上循环的次数/cpu主频    46966490/3074.07407
计算方法： SQL> explain plan for select * from scott.ob1;
           SQL> select CPU_COST from plan_table; ---循环次数
```



## 统计信息的收集

```shell
统计信息的收集
begin
dbms_stats.GATHER_SCHEMA_STATS(
ownname => 'SCOTT',
options => 'GATHER',
estimate_percent => dbms_stats.auto_sample_size,
method_opt => 'for all columns size repeat',
degree =>  4 );
end;
/

注释：　 
estimate_percent 抽样百分比
method_opt  操作的方法
for table --只统计表　
for all indexed columns --只统计有索引的表列 
for all indexes --只分析统计相关索引 
for all columns
method_opt=>'for all columns size skewonly'
method_opt=>'for all columns size repeat'
method_opt=>'for all columns size auto'
		
		
options 
	   gather——重新分析整个架构（Schema）。 
        gather empty——只分析目前还没有统计的表。 
        gather stale——只重新分析修改量超过10%的表（这些修改包括插入、更新和删除）。 
        gather auto——重新分析当前没有统计的对象，以及统计数据过期（变脏）的对象。注意，使用gather auto类似于组合使用gather stale和gather empty。
		
		
删除表的统计信息		
exec dbms_stats.delete_table_stats('SCOTT','OB1'); 
锁定解锁，用于空闲时收集大表的统计信息
exec dbms_stats.lock_table_stats('SCOTT','EMP');
exec dbms_stats.unlock_table_stats('SCOTT','EMP');


收集表的统计信息：
begin
dbms_stats.gather_table_stats(
OWNNAME=>'SCOTT',
TABNAME=>'EMP',
estimate_percent => dbms_stats.auto_sample_size,
method_opt=>'for all columns size skewonly',
CASCADE=>true);
end;
/


统计信息的导出导入
exec DBMS_STATS.CREATE_STAT_TABLE('SCOTT','EMP_0709'); 
exec DBMS_STATS.EXPORT_TABLE_STATS(OWNNAME => 'SCOTT', STATTAB => 'EMP_0709', TABNAME => 'EMP');
exec DBMS_STATS.IMPORT_TABLE_STATS(OWNNAME => 'SCOTT', STATTAB => 'EMP_0709', TABNAME => 'EMP');
跨用户导入统计信息
1 导入统计信息的表并且跟新C5列
SQL> update emp_0709 set c5='BLACK';
2 执行统计计划导入过程
SQL> exec DBMS_STATS.IMPORT_TABLE_STATS(OWNNAME => 'BLACK' , STATTAB => 'EMP_0709', TABNAME => 'EMP');
```

