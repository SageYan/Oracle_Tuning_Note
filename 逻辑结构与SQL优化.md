[TOC]

# 逻辑结构与SQL优化



## 存储结构概述

  

```sql
逻辑结构          物理结构
 database
    |
tablespace   ---<  datafile
    |                |
 segment             |
    |                |
 extent              |
    |                ^
oracle block  ---<  OS block

实验：查看操作系统块的大小
[root@telegraf_test ~]# du -ah ca
4.0K	ca/private/ca.csr

查看db block
show parameter db_block_size;
```

## Block

```sql
调用dbms_rowid包来研究块到底能装下多少行数据
select f, b, count(*)
from (select dbms_rowid.rowid_relative_fno(rowid) f,
             dbms_rowid.rowid_block_number(rowid) b
        from test_block_num)
group by f, b;


1. 行迁移的成因与优化
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





## segment & extent

```sql
select segment_name,
       segment_type,
       tablespace_name,
       blocks,
       extents,bytes/1024/1024
from user_segments
where segment_name = 'T'
```



## tablespace

```SQL
查看表空间的总体情况：
SELECT A.TABLESPACE_NAME "表空间名",
     A.TOTAL_SPACE "总空间(G)",
     NVL(B.FREE_SPACE, 0) "剩余空间(G)",
     A.TOTAL_SPACE - NVL(B.FREE_SPACE, 0) "使用空间(G)",
     CASE WHEN A.TOTAL_SPACE=0 THEN 0 ELSE trunc(NVL(B.FREE_SPACE, 0) / A.TOTAL_SPACE * 100, 2) END "剩余百分比%" --避免分母为0
FROM (SELECT TABLESPACE_NAME, trunc(SUM(BYTES) / 1024 / 1024/1024 ,2) TOTAL_SPACE
        FROM DBA_DATA_FILES
       GROUP BY TABLESPACE_NAME) A,
     (SELECT TABLESPACE_NAME, trunc(SUM(BYTES / 1024 / 1024/1024  ),2) FREE_SPACE
         FROM DBA_FREE_SPACE
        GROUP BY TABLESPACE_NAME) B
WHERE A.TABLESPACE_NAME = B.TABLESPACE_NAME(+)
ORDER BY 5;
```



## rowid

```SQL
rowid的意义
select rowid from t where rownum=1;
ROWID
------------------
AAAYPJAAQAAATNDAAA
--以下可定位该行具体在哪个对象、文件、块、行（注：rowid是64进制的）
data object number=AAAYPJ
file              =AAQ
block             =AAATND
row               =AAA
select dbms_rowid.rowid_object('AAAYPJAAQAAATNDAAA') data_object_id#,
dbms_rowid.rowid_relative_fno('AAAYPJAAQAAATNDAAA') rfile#,
dbms_rowid.rowid_block_number('AAAYPJAAQAAATNDAAA') block#,
dbms_rowid.rowid_row_number('AAAYPJAAQAAATNDAAA') row#
from dual;
--再确认对象信息
select owner,object_name from dba_objects where object_id=99273;
select file_name,tablespace_name from dba_data_files where file_id=16;

```



相关的优化案例

```SQL
1. block的size与SQL运行的关系：
block越大，相同数据量的情况下，Oracle产生的逻辑读越小；产生热点块的争用的概率越大！

2.利用分区表化整为零

3.删大量记录后逻辑读不变
查看高水位下的块数：
SQL> select bytes/1024/1024 from user_segments where segment_name='T';

4.row与block的比例
select num_rows,blocks from user_tab_statistics where table_name='T';

5.回收站的对象过多导致查询效率低下

6.表空间频繁扩展与插入性能
SQL> select count(*) from user_extents where segment_name='T_A';


```

