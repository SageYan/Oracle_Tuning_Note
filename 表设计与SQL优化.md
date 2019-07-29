[TOC]

# 表设计与SQL优化

## 分区表

### 1)  范围分区

```SQL
1 创建范围分区
SQL> create table range_part_tab (id number,deal_date date,area_code number,contents varchar2(4000))
partition by range (deal_date)
(
partition p1 values less than (TO_DATE('2015-02-01', 'YYYY-MM-DD')),
partition p2 values less than (TO_DATE('2015-03-01', 'YYYY-MM-DD')),
partition p3 values less than (TO_DATE('2015-04-01', 'YYYY-MM-DD')),
partition p4 values less than (TO_DATE('2015-05-01', 'YYYY-MM-DD')),
partition p5 values less than (TO_DATE('2015-06-01', 'YYYY-MM-DD')),
partition p6 values less than (TO_DATE('2015-07-01', 'YYYY-MM-DD')),
partition p7 values less than (TO_DATE('2015-08-01', 'YYYY-MM-DD')),
partition p8 values less than (TO_DATE('2015-09-01', 'YYYY-MM-DD')),
partition p9 values less than (TO_DATE('2015-10-01', 'YYYY-MM-DD')),
partition p10 values less than (TO_DATE('2015-11-01', 'YYYY-MM-DD')),
partition p11 values less than (TO_DATE('2015-12-01', 'YYYY-MM-DD')),
partition p12 values less than (TO_DATE('2016-01-01', 'YYYY-MM-DD')),
partition p_max values less than (maxvalue));
--插入随机数据
insert into range_part_tab (id,deal_date,area_code,contents)
select rownum,
       to_date( to_char(sysdate-365,'J')+TRUNC(DBMS_RANDOM.VALUE(0,365)),'J'),
       ceil(dbms_random.value(590,599)),
       rpad('*',400,'*')
  from dual
connect by rownum <= 100000;
```

### 2) list分区

```sql
SQL> create table list_part_tab (id number,deal_date date,area_code number,contents varchar2(4000))
partition by list (area_code)
(
partition p_591 values  (591),
partition p_592 values  (592),
partition p_593 values  (593),
partition p_594 values  (594),
partition p_595 values  (595),
partition p_596 values  (596),
partition p_597 values  (597),
partition p_598 values  (598),
partition p_599 values  (599),
partition p_other values  (DEFAULT));
```

### 3) HASH分区

```SQL
create table hash_part_tab (id number,deal_date date,area_code number,contents varchar2(4000))
partition by hash (deal_date)
PARTITIONS 12;

```

### 4) 复合分区

```sql
create table range_list_part_tab (id number,deal_date date,area_code number,　contents varchar2(4000))
 partition by range (deal_date)
 subpartition by list (area_code)
 subpartition TEMPLATE --子分区的表空间
 (subpartition p_591 values  (591),
  subpartition p_592 values  (592),
  subpartition p_593 values  (593),
  subpartition p_594 values  (594),
  subpartition p_595 values  (595),
  subpartition p_596 values  (596),
  subpartition p_597 values  (597),
  subpartition p_598 values  (598),
  subpartition p_599 values  (599),
  subpartition p_other values (DEFAULT))
(
 partition p1 values less than (TO_DATE('2015-02-01', 'YYYY-MM-DD')),
 partition p2 values less than (TO_DATE('2015-03-01', 'YYYY-MM-DD')),
 partition p3 values less than (TO_DATE('2015-04-01', 'YYYY-MM-DD')),
 partition p4 values less than (TO_DATE('2015-05-01', 'YYYY-MM-DD')),
 partition p5 values less than (TO_DATE('2015-06-01', 'YYYY-MM-DD')),
 partition p6 values less than (TO_DATE('2015-07-01', 'YYYY-MM-DD')),
 partition p7 values less than (TO_DATE('2015-08-01', 'YYYY-MM-DD')),
 partition p8 values less than (TO_DATE('2015-09-01', 'YYYY-MM-DD')),
 partition p9 values less than (TO_DATE('2015-10-01', 'YYYY-MM-DD')),
 partition p10 values less than (TO_DATE('2015-11-01', 'YYYY-MM-DD')),
 partition p11 values less than (TO_DATE('2015-12-01', 'YYYY-MM-DD')),
 partition p12 values less than (TO_DATE('2016-01-01', 'YYYY-MM-DD')),
 partition p_max values less than (maxvalue))；


查询分区的segment信息：
SET LINESIZE 666
set pagesize 5000
column segment_name format a20
column partition_name format a20
column segment_type format a20
select segment_name,
partition_name,
segment_type,
bytes / 1024 / 1024 "字节数(M)",
tablespace_name
from user_segments
where segment_name IN('RANGE_PART_TAB','NORM_TAB');
```

## 分区优势

### 1) 减少访问路径

```SQL
通过分区列过滤分区
```

### 2) 方便数据清理

```SQL
alter table part_tab_trunc truncate partition p1 ;
alter table part_tab_drop drop partition p1 ;
```

### 3) 操作灵活

```sql
split分区：
create table part_tab_split (id int,col2 int ,col3 int ,contents varchar2(4000))
partition by range (id)
(
partition p1 values less than (10000),
partition p2 values less than (20000),
partition p_max values less than (maxvalue));

alter table part_tab_split SPLIT PARTITION P_MAX  at (30000) into (PARTITION p3  ,PARTITION P_MAX);
alter table part_tab_split SPLIT PARTITION P_MAX  at (40000) into (PARTITION p4  ,PARTITION P_MAX);
alter table part_tab_split SPLIT PARTITION P_MAX  at (50000) into (PARTITION p5  ,PARTITION P_MAX);

add分区
create table part_tab_add (id int,col2 int,col3 int,contents varchar2(4000))
partition by range (id)
(
partition p1 values less than (10000),
partition p2 values less than (20000),
partition p3 values less than (30000),
partition p4 values less than (40000),
partition p5 values less than (50000),
partition p_max values less than (maxvalue));

add之前删掉默认分区
alter table part_tab_add  drop partition p_max;
alter table part_tab_add  add PARTITION p6 values less than (60000);
alter table part_tab_add  add PARTITION p_max  values less than (maxvalue);



exchange分区
--该操作会导致全局索引失效（要考虑including indexes 或者重建索引），具体见索引章节
--分区表的exchange

create table part_tab_exch (id int,col2 int,col3 int,contents varchar2(4000))
partition by range (id)
(
partition p1 values less than (10000),
partition p2 values less than (20000),
partition p3 values less than (30000),
partition p4 values less than (40000),
partition p5 values less than (50000),
partition p_max values less than (maxvalue));

create index idx_part_exch_col2 on part_tab_exch(col2) local;
create index idx_part_exch_col3 on part_tab_exch (col3);

--创建普通表
create table normal_tab (id int,col2 int,col3 int,contents varchar2(4000));
create index idx_norm_col2  on normal_tab (col2);

--其中including indexes  可选，为了保证全局索引不要失效
alter table part_tab_exch exchange partition p1 with table normal_tab including indexes update global indexes;


```

### 4) 其他知识

```sql
1. 分区与rowid
--分区表的数据转移分区之后，由于所在分区segment改变，导致rowid改变
--以下这个enable row movement必须操作，否则会出现ORA-014402:更新分区关键字列导致分区更改
alter table part_tab_rowid   enable row movement;
update part_tab_rowid set id=12 where id=1;

2. 分区与统计信息
--传统的analyze收集的分区统计信息不准确
--分区表的统计信息可以按照分区收集
exec dbms_stats.gather_table_stats(ownname => 'LJB',tabname => 'RANGE_PART_TAB',　partname =>'p_201512', estimate_percent => 10,method_opt=>
'for all indexed columns',cascade=>TRUE) ;
---收集完p_201512分区的统计结构后，可以通过如下方式查询到结果：
col partition_count format 99999
col num_rows format 99999999
col partition_name format a28
col table_name format a30
col last_analyzed format date

select table_name,
partition_name,
last_analyzed,
partition_position,
num_rows
from user_tab_statistics t
where table_name ='RANGE_PART_TAB';


3. 分区相关的数据字典
--查看分区类型，是否有子分区，分区总数
select partitioning_type,
subpartitioning_type,
partition_count
from user_part_tables
where table_name ='RANGE_PART_TAB';

--该表在那列上见分区
select column_name,object_type,column_position 
from user_part_key_columns where name ='&TABLENAME'

--查询分区表各分区的大小与分区名
select partition_name,
segment_type,
bytes
from user_segments
where segment_name ='RANGE_PART_TAB';

--查询分区表统计信息收集情况
select table_name,
partition_name,
last_analyzed,
partition_position,
num_rows
from user_tab_statistics t
where table_name ='RANGE_PART_TAB'

--查询分区表索引情况
select table_name,
index_name,
last_analyzed,
blevel,
num_rows,
leaf_blocks,
distinct_keys,
status
from user_indexes
where table_name ='RANGE_PART_TAB';

--查询分区表各索引大小
select segment_name,segment_type,sum(bytes)/1024/1024
from user_segments
where segment_name in
(select index_name
from user_indexes
where table_name ='RANGE_PART_TAB')
group by segment_name,segment_type ;

--查询分区表索引段的分配情况
select segment_name
partition_name,
segment_type,
bytes
from user_segments
where segment_name in
(select index_name
from user_indexes
where table_name ='RANGE_PART_TAB');

--查询分区表索引相关统计信息
select t2.table_name,
t1.index_name,
t1.partition_name,
t1.last_analyzed,
t1.blevel,
t1.num_rows,
t1.leaf_blocks,
t1.status
from user_ind_partitions t1, user_indexes t2
where t1.index_name = t2.index_name
and t2.table_name='RANGE_PART_TAB';
```



## 全局临时表

### 分类：

​          1）基于会话的全局临时表

```SQL
SQL> create global temporary table T_TMP_session on commit preserve rows as select  * from dba_objects where 1=2;
SQL> select table_name,temporary,duration from user_tables  where table_name='T_TMP_ SESSION';
```

​           2）基于事务的全局临时表

```sql
SQL> create global temporary table t_tmp_transaction on commit delete rows as select * from dba_objects where 1=2;
SQL> select table_name, temporary, DURATION from user_tables  where table_name= 'T_TMP_TRANSACTION';
```

### 优势：

```powershell
1）高效地删除记录
2）针对不同的会话数据独立
```

## 主外键的设计

外键的列必须添加索引：

1）避免死锁

2）连接查询更高效

## 压缩技术

### 1）表的压缩

```sql
ALTER TABLE t MOVE COMPRESS;
execute dbms_stats.gather_table_stats(ownname=>user, tabname=>'t');
--压缩表，占更少的块，减少IO
```

### 2）压缩索引

```sql
ALTER index  idx2_object_union  rebuild  COMPRESS;
execute dbms_stats.gather_index_stats(ownname=>user, indname=>'idx2_object_union');
SELECT t.index_name,t.compression,t.leaf_blocks,t.blevel  FROM user_indexes t WHERE index_name = 'IDX2_OBJECT_UNION';
--占更少的块，减少IO
```



## 相关的优化案例

1）分区ddl导致索引失效

```sql 
1 alter table &t truncate partition p1 
导致全局索引失效
如何避免：alter table &t truncate partition p1 update global indexes;
--另外drop split exchange都会导致global index失效

2 只有split在默认分区有数据的情况下，导致局部索引失效
需要 alter index &indexname rebuild;
```

2) 规划失败导致默认分区数据暴涨

通过split默认分区，重新规划分区

3）为何分区表的性能不如不同分区

```
分区表的分区索引虽然比全局索引小；但是索引的检索效率是由B tree的高度（level）决定的。
所以：分区表的局部索引检索效率为 level*分区数，所以没有普通表的效率高
```

4）全局临时表的案例

```SQL
1. 不能对全局临时表收集统计信息
会导致执行计划出错

2. 接口程序优化
用普通表作为数据传输的接口表（数据处理完成，数据删除）效率低下
使用临时表效率提高

3. 作为中间结果运算的表，经常删除，导致redo增加
将中间表改成临时表
--以下创建视图，方便后续直接用select * from v_redo_size进行查询，查询redo size
create or replace view v_redo_size as
select a.name,b.value
from v$statname a,v$mystat b
where a.statistic#=b.statistic#
and a.name='redo size';
```

5）监控异常的表设计

```SQL
（1）监控失效分区索引
prompt <p>查询当前用户下，失效-普通索引
select t.index_name,
t.table_name,
blevel,
t.num_rows,
t.leaf_blocks,
t.distinct_keys
from user_indexes t
where status = 'INVALID';

prompt <p>查询当前用户下，失效-分区索引
select t1.blevel,
t1.leaf_blocks,
t1.INDEX_NAME,
t2.table_name,
t1.PARTITION_NAME,
t1.STATUS
from user_ind_partitions t1, user_indexes t2
where t1.index_name = t2.index_name
and t1.STATUS = 'UNUSABLE';

（2）监控未建分区的大表
prompt <p>当前用户下，表大小超过10个GB未建分区的
select segment_name,
segment_type,
sum(bytes) / 1024 / 1024 / 1024 object_size
from user_segments
WHERE segment_type = 'TABLE'
group by  segment_name, segment_type
having sum(bytes) / 1024 / 1024 / 1024 >= 10
order by object_size desc;

（3）监控分区数过多的表
prompt <p>当前用户下，分区最多的前10个对象
select *
from (select  table_name, count(*) cnt
from user_tab_partitions
group by  table_name
order by cnt desc)
where rownum <= 10;
prompt <p>当前用户下，分区个数超过100的表
select  table_name, count(*) cnt
from user_tab_partitions
having count(*) >= 100
group by table_name, table_name
order by cnt desc;
--或者如下更方便
select table_name, partitioning_type, subpartitioning_type
from user_part_tables
where partition_count > 100;


（4）监控分区表各分区大小严重不均匀情况
set linesize 266
col table_name format a20
select table_name,
max(num_rows),
trunc(avg(num_rows),0),
sum(num_rows),
case when sum(num_rows),0 then 0,else trunc(max(num_rows) / sum(num_rows),2) end,
count(*)
from user_tab_partitions
group by table_name
having max(num_rows) / sum(num_rows) > 2 / count(*);
也可用来作为判断查询当前用户下有因为疏于分区管理导致大量数据进了默认分区的参考。
select table_name,
partition_name,
num_rows
from user_tab_partitions
where table_name = 'RANGE_PART_TAB'
order by num_rows desc;


（5）监控当前有多少带子分区的分区表
select table_name,
partitioning_type,
subpartitioning_type,
partition_count
from user_part_tables
where subpartitioning_type <> 'NONE';
select count(*) from user_part_tables where  subpartitioning_type <> 'NONE';



--监控哪些全局临时表被收集统计信息
prompt <p>当前用户下，被收集统计信息的临时表
select owner,
table_name,
t.last_analyzed,
t.num_rows,
t.blocks
from user_tables t
where t.temporary = 'Y'
and last_analyzed is not null;


--查看当前数据库哪些对象外键没建索引
select table_name,
constraint_name,
cname1 || nvl2(cname2, ',' || cname2, null) ||
nvl2(cname3, ',' || cname3, null) ||
nvl2(cname4, ',' || cname4, null) ||
nvl2(cname5, ',' || cname5, null) ||
nvl2(cname6, ',' || cname6, null) ||
nvl2(cname7, ',' || cname7, null) ||
nvl2(cname8, ',' || cname8, null) columns
from (select b.table_name,
b.constraint_name,
max(decode(position, 1, column_name, null)) cname1,
max(decode(position, 2, column_name, null)) cname2,
max(decode(position, 3, column_name, null)) cname3,
max(decode(position, 4, column_name, null)) cname4,
max(decode(position, 5, column_name, null)) cname5,
max(decode(position, 6, column_name, null)) cname6,
max(decode(position, 7, column_name, null)) cname7,
max(decode(position, 8, column_name, null)) cname8,
count(*) col_cnt
from (select substr(table_name, 1, 30) table_name,
substr(constraint_name, 1, 30) constraint_name,
substr(column_name, 1, 30) column_name,
position
from user_cons_columns) a,
user_constraints b
where a.constraint_name = b.constraint_name
and b.constraint_type = 'R'
group by b.table_name, b.constraint_name) cons
where col_cnt > ALL
(select count(*)
from user_ind_columns i
where i.table_name = cons.table_name
and i.column_name in (cname1, cname2, cname3, cname4, cname5,
cname6, cname7, cname8)
and i.column_position <= cons.col_cnt
group by i.index_name)


--查询所有含外键的表：
select count(*),TABLE_NAME,c_constraint_name from (
select a.table_name,
substr(a.constraint_name, 1, 30) c_constraint_name,
substr(a.column_name, 1, 30) column_name,
position,
b.owner,
b.constraint_name,
b.constraint_type
from user_cons_columns a, user_constraints b
where a.constraint_name = b.constraint_name
and b.constraint_type = 'R' )
group by TABLE_NAME,c_constraint_name


--监控表中有没有过时类型的字段
select table_name,
column_name,
data_type
from user_tab_columns
where data_type in ( 'LONG','CHAR');
```

