[TOC]



# 表连接与SQL优化

## 三大经典表连接

Nested Loops Join

Hash join

Merge sort join

## 1 表的访问次数

### NL

驱动表被访问0或1次，被驱动表被访问0或N次；N是有驱动表返回的结果集的条数决定的。

```sql
set linesize 1000
alter session set statistics_level=all ;

SELECT /*+ leading(t1) use_nl(t2) */ * FROM t1, t2
WHERE t1.id = t2.t1_id AND t1.n in(17, 19);

select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));
-------------------------------------------------------------------------------------
| Id  | Operation          | Name | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
-------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |      |      1 |        |      2 |00:00:00.01 |    2019 |
|   1 |  NESTED LOOPS      |      |      1 |      2 |      2 |00:00:00.01 |    2019 |
|*  2 |   TABLE ACCESS FULL| T1   |      1 |      2 |      2 |00:00:00.01 |       8 |
|*  3 |   TABLE ACCESS FULL| T2   |      2 |      1 |      2 |00:00:00.01 |    2011 |
-------------------------------------------------------------------------------------

--starts显示了T2被访问了两次
```

### Hash join

驱动表被访问0到1次，被驱动表被访问0到1次；绝大部分情况下，两表各被访问1次

```sql
--Hash Join中 t2表只会被访问1次或0次(驱动表被访问1次，被驱动表被访问1次）
alter session set statistics_level=all ;
set linesize 1000
SELECT /*+ leading(t1) use_hash(t2) */ * FROM t1, t2
WHERE t1.id = t2.t1_id;

select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));

----------------------------------------------------------------------------------------------------------------
| Id  | Operation          | Name | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |      |      1 |        |    100 |00:00:00.03 |    1019 |       |       |          |
|*  1 |  HASH JOIN         |      |      1 |    100 |    100 |00:00:00.03 |    1019 |   960K|   960K| 1233K (0)|
|   2 |   TABLE ACCESS FULL| T1   |      1 |    100 |    100 |00:00:00.01 |       7 |       |       |          |
|   3 |   TABLE ACCESS FULL| T2   |      1 |  85905 |    100K|00:00:00.01 |    1012 |       |       |          |
----------------------------------------------------------------------------------------------------------------
```

### merge sort join

表的访问次数与hash join一样

## 2 驱动表的顺序与性能

### NL

```SQL
SELECT /*+ leading(t1) use_nl(t2)*/ *
FROM t1, t2
WHERE t1.id = t2.t1_id
AND t1.n = 19;
-------------------------------------------------------------------------------------
| Id  | Operation          | Name | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
-------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |      |      1 |        |      1 |00:00:00.01 |    1014 |
|   1 |  NESTED LOOPS      |      |      1 |      1 |      1 |00:00:00.01 |    1014 |
|*  2 |   TABLE ACCESS FULL| T1   |      1 |      1 |      1 |00:00:00.01 |       8 |
|*  3 |   TABLE ACCESS FULL| T2   |      1 |      1 |      1 |00:00:00.01 |    1006 |
-------------------------------------------------------------------------------------
SELECT /*+ leading(t2) use_nl(t1)*/ *
FROM t1, t2
WHERE t1.id = t2.t1_id
AND t1.n = 19;
-------------------------------------------------------------------------------------
| Id  | Operation          | Name | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
-------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |      |      1 |        |      1 |00:00:00.60 |     701K|
|   1 |  NESTED LOOPS      |      |      1 |      1 |      1 |00:00:00.60 |     701K|
|   2 |   TABLE ACCESS FULL| T2   |      1 |  85905 |    100K|00:00:00.01 |    1006 |
|*  3 |   TABLE ACCESS FULL| T1   |    100K|      1 |      1 |00:00:00.57 |     700K|
-------------------------------------------------------------------------------------
--查看buffer 发现语句2开销比1大很多
```

### Hash join

```SQL
set linesize 1000
alter session set statistics_level=all;
SELECT /*+ leading(t1) use_hash(t2)*/ *
FROM t1, t2
WHERE t1.id = t2.t1_id
and t1.n=19;
select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));
----------------------------------------------------------------------------------------------------------------
| Id  | Operation          | Name | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |      |      1 |        |      1 |00:00:00.04 |    1013 |       |       |          |
|*  1 |  HASH JOIN         |      |      1 |      1 |      1 |00:00:00.04 |    1013 |   960K|   960K|  395K (0)|
|*  2 |   TABLE ACCESS FULL| T1   |      1 |      1 |      1 |00:00:00.01 |       7 |       |       |          |
|   3 |   TABLE ACCESS FULL| T2   |      1 |  85905 |    100K|00:00:00.01 |    1006 |       |       |          |
----------------------------------------------------------------------------------------------------------------
set linesize 1000
SELECT /*+ leading(t2) use_hash(t1)*/ *
FROM t1, t2
WHERE t1.id = t2.t1_id
and t1.n=19;
select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));
----------------------------------------------------------------------------------------------------------------
| Id  | Operation          | Name | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem |
----------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |      |      1 |        |      1 |00:00:00.06 |    1013 |       |       |          |
|*  1 |  HASH JOIN         |      |      1 |      1 |      1 |00:00:00.06 |    1013 |    11M|  2469K|   16M (0)|
|   2 |   TABLE ACCESS FULL| T2   |      1 |  85905 |    100K|00:00:00.01 |    1005 |       |       |          |
|*  3 |   TABLE ACCESS FULL| T1   |      1 |      1 |      1 |00:00:00.01 |       8 |       |       |          |
----------------------------------------------------------------------------------------------------------------

--查看Omem 发现语句2的内存开销比1大很多！
```

### Merge Sort join

驱动表的顺序无影响

## 3 表连接是否有排序

NL 和 hash join 无排序

Merge sort Join 有排序

## 4 各种连接的使用限制

1 Hash join不支持连接条件是 < ,>,<> 和like的情景

2 Merge Sort Join 不支持连接条件是 <> 和 like 的情景

3 NL没有限制



## 案例（三刀三斧四式）

### Nested Loops Join

注意：NL只适合在OLTP场景。通俗的讲：应用有大量访问，而且只返回很少记录。

#### 1 对驱动表的限制条件要考虑建立索引

```SQL
CREATE INDEX t1_n ON t1 (n);
---有了限制条件的索引，Nested Loops Join性能略有提升
set linesize 1000
alter session set statistics_level=all ;
SELECT /*+ leading(t1) use_nl(t2) */ *
FROM t1, t2
WHERE t1.id = t2.t1_id AND t1.n = 19;
select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));
```

#### 2 被驱动的连接条件要考虑创建索引

```sql
CREATE INDEX t2_t1_id ON t2(t1_id);
----表连接性能有了大幅度提升
alter session set statistics_level=all ;
SELECT /*+ leading(t1) use_nl(t2) */ *
FROM t1, t2
WHERE t1.id = t2.t1_id
AND t1.n = 19;
select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));
-------------------------------------------------------------------------------------------------------------
| Id  | Operation                     | Name     | Starts | E-Rows | A-Rows |   A-Time   | Buffers | Reads  |
-------------------------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT              |          |      1 |        |      1 |00:00:00.01 |       7 |      1 |
|   1 |  NESTED LOOPS                 |          |      1 |      1 |      1 |00:00:00.01 |       7 |      1 |
|   2 |   NESTED LOOPS                |          |      1 |      1 |      1 |00:00:00.01 |       6 |      1 |
|   3 |    TABLE ACCESS BY INDEX ROWID| T1       |      1 |      1 |      1 |00:00:00.01 |       3 |      0 |
|*  4 |     INDEX RANGE SCAN          | T1_N     |      1 |      1 |      1 |00:00:00.01 |       2 |      0 |
|*  5 |    INDEX RANGE SCAN           | T2_T1_ID |      1 |      1 |      1 |00:00:00.01 |       3 |      1 |
|   6 |   TABLE ACCESS BY INDEX ROWID | T2       |      1 |      1 |      1 |00:00:00.01 |       1 |      0 |
-------------------------------------------------------------------------------------------------------------
```

#### 3  结果集小的做驱动表 ，大的做被驱动表



### Hash Join

Hash join更适合OLAP场景。通俗的讲就是最终放回的记录较多。

明确hash join的限制条件，比如连接条件是 < > like

两表无索引时，SQL会倾向于hash join

#### 1 两表的限制条件有索引（建索引是要看最终结果的返回量）

#### 2 小的结果集驱动大的结果集

```sql
set linesize 1000
alter session set statistics_level=all;
SELECT  *
FROM t1, t2
WHERE t1.id = t2.t1_id;
select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));



--使用dbms_stats.set_table_stats骗过oracle，使得t2作驱动表
EXEC  dbms_stats.set_table_stats(user, 'T1', numrows => 20000000  ,numblks => 1000000);
EXEC  dbms_stats.set_table_stats(user, 'T2', numrows => 1  ,numblks => 1);

set linesize 1000
alter session set statistics_level=all;
SELECT  *
FROM t1, t2
WHERE t1.id = t2.t1_id;
select * from table(dbms_xplan.display_cursor(null,null,'allstats last'));


```

#### 3 保证PGA能够容纳hash运算

### Merge sort Join

适用OLAP，通俗的讲适用于吞吐量较大的操作。

当心连接条件的限制

#### 1 看数据返回量，决定是否创建连接条件的索引

#### 2 使用连接条件索引消除部分排序，不能完全消除

```sql
CREATE INDEX idx_t1_id ON t1(id);
set linesize 1000
set autotrace traceonly
SELECT /*+ leading(t1) use_merge(t2)*/ *
FROM t1, t2
WHERE t1.id = t2.t1_id;

          1  recursive calls
          0  db block gets
       1020  consistent gets
          0  physical reads
          0  redo size
      13935  bytes sent via SQL*Net to client
        589  bytes received via SQL*Net from client
          8  SQL*Net roundtrips to/from client
          1  sorts (memory)
          0  sorts (disk)
        100  rows processed
```

#### 3 避免多余的字段，导致排序过大

#### 4 增大PGA



### NL统计信息收集不准确 导致执行计划出错

 查看dbms_xplan.display_cursor中的：

E-Rows | A-Rows  预估值和实际值相差是否过多！



