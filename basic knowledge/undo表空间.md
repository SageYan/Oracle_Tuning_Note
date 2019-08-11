# undo表空间

## undo表空间中的老镜像的作用：

为事务提供回退
为事务提供恢复
提供读一致性
提供对DML数据的闪回

```SQL
select tablespace_name from dba_tablespaces where contents='UNDO';
注释：undo表空间里面有多个rollback segement
select SEGMENT_NAME,TABLESPACE_NAME, STATUS  from dba_rollback_segs;
实验：【监控用户对undo的使用】
desc v$transaction;  事物信息
select s.username,t.XIDUSN,t.USED_UBLK from v$session s,v$transaction t where s.saddr=t.SES_ADDR;
注释：若有事物（比如for循环更新数据等）没提交，缓存了很多镜像在undo，会导致另外的查询变慢，因为要全部读取修改之前的老镜像,
     导致其他事务查询历史数据的效率变差。
begin
  for i in 1..300000 loop
   update t02 set X=X+1;
   end loop;
   end;
  /
```



## rollback segment的管理方法

```sql
show parameter undo_management;
auto
show parameter undo_tablespace;    
--查看目前使用的undo表空间，在自动管理模式下，如果undo表空间有问题，数据库是打不开的，报错03113。在生产环境里，通常会有两个undo表空间，在其中一个写满时，切换！
   
   
实验：--切换undo 表空间
create undo tablespace undotbs2 datafile '/u01/app/oracle/oradata/lianxi/undotbs02.dbf' size 10M;
--新的undo表空间，undo表空间也是datafile 收到日志的保护；
select SEGMENT_NAME,TABLESPACE_NAME, STATUS  from dba_rollback_segs;
查看rollback segment的状态；
连接到普通用户Scott，更新表的数据：
update e01 set sal=sal+1;
select s.username,t.XIDUSN,t.USED_UBLK from vsession s,vtransaction t where s.saddr=t.SES_ADDR;查看回滚段使用情况
select name from v$rollname where usn=1;查看使用的回滚段的名字
切换undo表空间
alter system set undo_tablespace='UNDOTBS2' ；
select SEGMENT_NAME,TABLESPACE_NAME, STATUS  from dba_rollback_segs;
只有正在使用的回滚段online  其他的undotbs1的段都offline

--产生环境常用两个undotablespace用于切换undo，防止undo过满导致性能下降！
     
     
实验：切换手动管理
alter system set undo_management=manual scope=spfile;

SCOTT>update e01 set sal=sal+1;
update e01 set sal=sal+1
      *
ERROR at line 1:
ORA-01552: cannot use system rollback segment for non-system tablespace 'USERS'
除了系统回滚段，没有可用的回滚段
select SEGMENT_NAME,TABLESPACE_NAME, STATUS  from dba_rollback_segs;查看
alter table e01 move tablespace system;将表移动到system表空间  操作可以执行

SCOTT>create table ee01 as select * from emp;
create table ee01 as select * from emp
                                  *
ERROR at line 1:
ORA-01552: cannot use system rollback segment for non-system tablespace 'TEST1'
SCOTT>create table ee01 as select * from emp where 1=0;

Table created.
注释：此操作能成功因为11g的新特性：
SQL> sho parameter segment;

NAME                                 TYPE                   VALUE
------------------------------------ ---------------------- ------------------------------
deferred_segment_creation            boolean                TRUE
rollback_segments                    string
transactions_per_rollback_segment    integer                5
如果把deferred_segment_creation关闭，也不能成功，延迟段的分配
创建回滚段
create rollback segment rsg1 tablespace undotbs1;
alter rollback segment rsg1 online;
SQL> show parameter rollback; 

NAME                                 TYPE                   VALUE
------------------------------------ ---------------------- ------------------------------
fast_start_parallel_rollback         string                 LOW
rollback_segments                    string
transactions_per_rollback_segment    integer                5  --每个回滚段最多的事务数
alter system set rollback_segments='RSG1' scope=spfile;   写入参数

create rollback segment rbs1 tablespace undotbs1;
create rollback segment rbs2 tablespace undotbs1;
create rollback segment rbs3 tablespace undotbs1;
alter system set rollback_segments='RSG1','RBS1','RBS2','RBS3' scope=spfile;

--这种方法在自动管理回滚段的时，不能打开数据库时使用手动管理打开数据库！

```





## 提供对DML数据的闪回

```SQL
闪回查询：
select * from e01 as of timestamp(sysdate-10/1440);
闪回数据：
alter table e01 enable row movement;
flashback table e01 to timestamp(sysdate-10/1440);

SQL> show parameter undo;

NAME                                 TYPE                   VALUE
------------------------------------ ---------------------- ------------------------------
undo_management                      string                 AUTO
undo_retention                       integer                900
undo_tablespace                      string                 UNDOTBS1
闪回版本查询：应对短时间间内的大量修改
select versions_startscn,versions_endscn,versions_operation,versions_xid,sal
from e01 versions between scn minvalue and maxvalue
where empno=7369;
或
select versions_starttime,versions_endtime,versions_operation,versions_xid,sal
from e01 versions between scn minvalue and maxvalue
where empno=7369;
select * from e01 as of scn 1652947 ;
select * from e01 as of scn 1652946 ;  --找到改变之前的数据
flashback table e01 to scn 1652946;
--注释：闪回不了drop truncate  因为他们不undo


实验：追加日志模式
闪回事务查询sys才可以
sys>alter database add supplemental log data;   打开追加日志模式

update e01 set sal=sal*1.23 where deptno=30;

select versions_startscn,versions_endscn,versions_operation,versions_xid,sal
from e01 versions between scn minvalue and maxvalue
where empno=7499；  --拿到versions_xid

sys> select undo_sql from flashback_transaction_query where xid='0100090038030000';
拿到反向sql语句
执行反向sql  提交  数据返回

```



## ORA-01555

```sql 
设置事务隔离级别为串行读
set transaction isolation level serializable;
简单地说，serializable就是使事务看起来象是一个接着一个地顺序地执行。
只看到本次命令发出之前本窗口所提交的数据，其他窗口修改提交的数据不看，使用undo；

实验：【避免01555】
desc dba_tablespaces;
select  TABLESPACE_NAME, RETENTION from dba_tablespaces where CONTENTS ='UNDO';
查看retention是否强制
alter tablespace undotbs2 retention guarantee; 
--打开后，强制保留预定时间，保证查询不出错，不保证修改能进行；不开，修改能进行，查询可能出错！

undo数据文件的移动： 和移动日志文件相同
undo扩容：和永久表空间相同

```



