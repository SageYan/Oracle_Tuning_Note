[TOC]



# Alter index coalesce VS shrink space

10g中引入了对索引的shrink功能，索引shrink操作会扫描索引的页块，并且通过归并当前存在的数据将先前已删除记录的空间重新利用；很多书籍亦或者MOS的Note中都会提及SHRINK命令与早期版本中就存在的COALESCE(合并)命令具有完全相同的功能，或者说2者是完全等价的-" alter index shrink space is equivalent to coalesce"，事实是这样的吗？

```brush
/* 测试使用版本10.2.0.4 * /

/* 建立测试用表YOUYUS，高度为3 */

SQL> drop table YOUYUS;
Table dropped.

SQL> create table YOUYUS as select rownum t1,rpad('A',20,'B') t2 from dual connect by level<=999999;
Table created.

SQL> create index ind_youyus on youyus(t1,t2) nologging;
Index created.

SQL> analyze  index IND_YOUYUS validate  structure;
Index analyzed.

/*
大家因该很熟悉 analyze index .. validate structure 命令 ，实际上该命令存在一个兄弟:
analyze  index IND_YOUYUS validate  structure online，
加上online子句后validate structure可以在线操作，但该命令不会填充index_stats临时视图
*/

SQL> set linesize 200;
SQL> set linesize 200;
SQL> select height,
  2         blocks,
  3         lf_blks,
  4         lf_rows_len,
  5         lf_blk_len,
  6         br_blks,
  7         br_rows,
  8         br_rows_len,
  9         br_blk_len,
 10         btree_space,
 11         used_space,
 12         pct_used
 13    from index_stats;

    HEIGHT     BLOCKS    LF_BLKS LF_ROWS_LEN LF_BLK_LEN    BR_BLKS    BR_ROWS BR_ROWS_LEN BR_BLK_LEN BTREE_SPACE USED_SPACE   PCT_USED
---------- ---------- ---------- ----------- ---------- ---------- ---------- ----------- ---------- ----------- ---------- ----------
         3       5376       5154    36979767       7996          9       5153       61784       8028    41283636   37041551         90

/*  可以看到IND_YOUYUS索引的基本结构，在初始状态下其block总数为5376，其中页块共5154  */

/*  我们在表上执行删除操作，均匀删除三分之一的数据 */

SQL> delete YOUYUS where mod(t1,3)=1;
333333 rows deleted.

SQL> commit;
Commit complete.


SQL> select vs.name, ms.value
  2    from v$mystat ms, v$sysstat vs
  3   where vs.statistic# = ms.statistic#
  4     and vs.name in ('redo size','consistent gets');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
consistent gets                                                          45
redo size                                                                 0

SQL> alter index ind_youyus coalesce;

Index altered.

SQL> select vs.name, ms.value
  2    from v$mystat ms, v$sysstat vs
  3   where vs.statistic# = ms.statistic#
  4     and vs.name in ('redo size','consistent gets');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
consistent gets                                                         788
redo size                                                          70649500

/* coalesce 操作产生了大约67MB的redo数据  */

SQL> analyze  index IND_YOUYUS validate  structure;
Index analyzed.

SQL> set linesize 200;
SQL> select height,
  2         blocks,
  3         lf_blks,
  4         lf_rows_len,
  5         lf_blk_len,
  6         br_blks,
  7         br_rows,
  8         br_rows_len,
  9         br_blk_len,
 10         btree_space,
 11         used_space,
 12         pct_used
 13    from index_stats;

    HEIGHT     BLOCKS    LF_BLKS LF_ROWS_LEN LF_BLK_LEN    BR_BLKS    BR_ROWS BR_ROWS_LEN BR_BLK_LEN BTREE_SPACE USED_SPACE   PCT_USED
---------- ---------- ---------- ----------- ---------- ---------- ---------- ----------- ---------- ----------- ---------- ----------
         3       5376       3439    24653178       7996          9       3438       41188       8028    27570496   24694366         90

/* 可以看到执行coalesce(合并)操作后页块数量下降到3439，
而branch枝块和root根块的结构是不会变化的，同时coalesc命令并不释放索引上的多余空间，
但索引结构实际占用的空间BTREE_SPACE下降到了27570496 bytes */

/* 以下为此时ind_youyus索引的treedump * /

[maclean@rh2 ~]$ cat /s01/10gdb/admin/YOUYUS/udump/youyus_ora_5104.trc| \
 grep "level:";cat /s01/10gdb/admin/YOUYUS/udump/youyus_ora_5104.trc|grep leaf|wc -l

branch: 0x130787c 19953788 (0: nrow: 8, level: 2)
   branch: 0x1308c41 19958849 (-1: nrow: 450, level: 1)
   branch: 0x1308eea 19959530 (0: nrow: 447, level: 1)
   branch: 0x1309195 19960213 (1: nrow: 447, level: 1)
   branch: 0x130943e 19960894 (2: nrow: 447, level: 1)
   branch: 0x13096e7 19961575 (3: nrow: 447, level: 1)
   branch: 0x130××× 19962258 (4: nrow: 447, level: 1)
   branch: 0x1309c3b 19962939 (5: nrow: 447, level: 1)
   branch: 0x1309e0f 19963407 (6: nrow: 307, level: 1)
3439

/* 清理测试现场 */

SQL> drop table YOUYUS;
Table dropped.

SQL> create table YOUYUS as select rownum t1,rpad('A',20,'B') t2 from dual connect by level<=999999;
Table created.

SQL> create index ind_youyus on youyus(t1,t2) nologging;
Index created.

SQL> delete YOUYUS where mod(t1,3)=1;
333333 rows deleted.

SQL> commit;
Commit complete.

SQL> conn maclean/maclean
Connected.
SQL> select vs.name, ms.value
  2    from v$mystat ms, v$sysstat vs
  3   where vs.statistic# = ms.statistic#
  4     and vs.name in ('redo size','consistent gets');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
consistent gets                                                          45
redo size                                                                 0

SQL> alter index ind_youyus shrink space;

Index altered.

SQL> select vs.name, ms.value
  2    from v$mystat ms, v$sysstat vs
  3   where vs.statistic# = ms.statistic#
  4     and vs.name in ('redo size','consistent gets');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
consistent gets                                                        2951
redo size                                                          90963340

/* SHRINK SPACE操作产生了86MB的redo数据，多出coalesce时的28%　*/

SQL> analyze  index IND_YOUYUS validate  structure;

Index analyzed.

SQL> set linesize 200;
SQL> select height,
  2         blocks,
  3         lf_blks,
  4         lf_rows_len,
  5         lf_blk_len,
  6         br_blks,
  7         br_rows,
  8         br_rows_len,
  9         br_blk_len,
 10         btree_space,
 11         used_space,
 12         pct_used
 13    from index_stats;

    HEIGHT     BLOCKS    LF_BLKS LF_ROWS_LEN LF_BLK_LEN    BR_BLKS    BR_ROWS BR_ROWS_LEN BR_BLK_LEN BTREE_SPACE USED_SPACE   PCT_USED
---------- ---------- ---------- ----------- ---------- ---------- ---------- ----------- ---------- ----------- ---------- ----------
         3       3520       3439    24653178       7996          9       3438       41188       8028    27570496   24694366         90

/* 以下为此时ind_youyus索引的treedump * /

[maclean@rh2 ~]$ cat /s01/10gdb/admin/YOUYUS/udump/youyus_ora_5125.trc|grep "level:"; \
cat /s01/10gdb/admin/YOUYUS/udump/youyus_ora_5125.trc|grep leaf|wc -l

branch: 0x1309efc 19963644 (0: nrow: 8, level: 2)
   branch: 0x130b2c1 19968705 (-1: nrow: 450, level: 1)
   branch: 0x130b56a 19969386 (0: nrow: 447, level: 1)
   branch: 0x130b815 19970069 (1: nrow: 447, level: 1)
   branch: 0x130babe 19970750 (2: nrow: 447, level: 1)
   branch: 0x130bd67 19971431 (3: nrow: 447, level: 1)
   branch: 0x130b919 19970329 (4: nrow: 447, level: 1)
   branch: 0x130b3bf 19968959 (5: nrow: 447, level: 1)
   branch: 0x1309efe 19963646 (6: nrow: 307, level: 1)
3439

/* 索引结构与coalesce命令维护后相同，但shrink space操作释放了索引上的空闲空间 */

/* 再次清理测试现场 */

SQL> drop table YOUYUS;
Table dropped.

SQL> create table YOUYUS as select rownum t1,rpad('A',20,'B') t2 from dual connect by level<=999999;
Table created.

SQL> create index ind_youyus on youyus(t1,t2) nologging;
Index created.

SQL>  delete YOUYUS where mod(t1,3)=1;
333333 rows deleted.

SQL> commit;
Commit complete.

SQL> conn maclean/maclean
Connected.

SQL> select vs.name, ms.value
  2    from v$mystat ms, v$sysstat vs
  3   where vs.statistic# = ms.statistic#
  4     and vs.name in ('redo size','consistent gets');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
consistent gets                                                          45
redo size                                                                 0

SQL> alter index ind_youyus shrink space compact;

Index altered.

SQL> select vs.name, ms.value
  2    from v$mystat ms, v$sysstat vs
  3   where vs.statistic# = ms.statistic#
  4     and vs.name in ('redo size','consistent gets');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
consistent gets                                                        3208
redo size                                                          90915424

SQL> analyze  index IND_YOUYUS validate  structure;

Index analyzed.

SQL> set linesize 200;
SQL> select height,
  2         blocks,
  3         lf_blks,
  4         lf_rows_len,
  5         lf_blk_len,
  6         br_blks,
  7         br_rows,
  8         br_rows_len,
  9         br_blk_len,
 10         btree_space,
 11         used_space,
 12         pct_used
 13    from index_stats;

    HEIGHT     BLOCKS    LF_BLKS LF_ROWS_LEN LF_BLK_LEN    BR_BLKS    BR_ROWS BR_ROWS_LEN BR_BLK_LEN BTREE_SPACE USED_SPACE   PCT_USED
---------- ---------- ---------- ----------- ---------- ---------- ---------- ----------- ---------- ----------- ---------- ----------
         3       5376       3439    24653178       7996          9       3438       41188       8028    27570496   24694366         90

/* shrink space compact 起到了和coalesce完全相同的作用，但其产生的redo仍要多于coalesce于28% */
```



coalesce与shrink space命令对比重建索引(rebuild index)有一个显著的优点:不会导致索引降级。从以上测试可以看到coalesce与shrink space compact功能完全相同；在OLTP环境中，大多数情况下我们并不希望回收索引上的空闲空间，那么coalesce或者shrink space compact(not shrink space)可以成为我们很好的选择，虽然实际操作过程中2者消耗的资源有不少差别。

