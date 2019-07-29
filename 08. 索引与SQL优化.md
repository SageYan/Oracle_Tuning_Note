[TOC]

# 索引与SQL优化

## 索引的特性

1. 高度较低

2. 索引存储列值 ：索引leaf block存放的是列值和对应的rowid

3. 索引本身有序

   

## 索引与SQL优化

```sql
1 索引高度低有利于查询
--查看索引高度
set linesize 1000
set autotrace off
select index_name,
blevel,
leaf_blocks,
num_rows,
distinct_keys,
clustering_factor
from user_ind_statistics
where table_name in( 'T1','T2','T3','T4','T5','T6','T7');

2 索引存储列值 可以优化聚合函数
注意count() 索引列当中不能含有NULL，否者不走索引

3 索引本身有序 
优化排序、优化min/max


4 组合索引的选用
1）单列返回的值多，联合返回的值少
2）考虑前导列的顺序
3) where 条件都是等值查询，顺序不影响
4）组合索引的最佳顺序一般是把等值查询列置前

```

### 索引的扫描类型与分类构造

#### 1  index range scan

#### 2  index unique scan

```sql
--请注意这个INDEX UNIQUE SCAN扫描方式,在唯一索引情况下使用。
```

#### 3  TABLE ACCESS BY USER ROWID

```sql
--请注意这个TABLE ACCESS BY USER ROWID扫描方式,直接根据rowid来访问，最快的访问方式！
select rowid from t where object_id=8;
ROWID
-----
AAAZxiAAGAAAB07AAH

select * from t where object_id=8 and rowid='AAAZxiAAGAAAB07AAH';
```

#### 4  INDEX FULL SCAN

```sql
--并体会与INDEX FAST FULL SCAN的区别
alter table T modify object_id not null;
create  index idx_object_id on t(object_id);
select * from t  order by object_id;
```

#### 5  INDEX FAST FULL SCAN

```sql
alter table T modify object_id not null;
create  index idx_object_id on t(object_id);
select count(*) from t;
```

#### 6  INDEX FULL SCAN (MINMAX)

```sql
create  index idx_object_id on t(object_id);
select max(object_id) from t;
```

#### 7  INDEX SKIP SCAN

```sql
create table t as select * from dba_objects;
update t set object_type='TABLE' ;
update t set object_type='VIEW' where rownum<=30000;
create  index idx_type_id on t(object_type,object_id);
select * from t where object_id=8;

--联合索引，前导列不在where条件中且前导列中重复值较多！
```

#### 8 TABLE ACCESS BY INDEX ROWID

```sql
create table t as select * from dba_objects;
update t set object_id=rownum;
create  index idx_object_id on t(object_id);

select object_id from t where object_id=2 and object_type='TABLE';
--在接下来的试验中，你会看到TABLE ACCESS BY INDEX ROWID消失了。（回表消失）
create  index idx_id_type on t(object_id,object_type);
select object_id from t where object_id=2 and object_type='TABLE';
```



## 索引相关的优化案例

### 1 分区表各类聚合优化玄机

```sql
语句1：
select max(nbr) max_nbr
from range_part_tab
where deal_date >= TO_DATE('2015-05-01', 'YYYY-MM-DD')
and deal_date < TO_DATE('2015-06-01', 'YYYY-MM-DD');

接下来看语句2：
select max(nbr) max_nbr from range_part_tab partition(p_201505);
```

### 2  同时取最大最小值的案例

```sql
select max(object_id),min(object_id) from t;

select max, min
from (select max(object_id) max from t ) a,
(select min(object_id) min from t ) b;
```

### 3  联合索引的优化案例

```sql
--组合索引与增加检索条件
create index idx_union on t(object_type,object_id,owner);
写法1
select * from t where object_type='VIEW' and OWNER='LJB'
写法2
select /*+index(T IDX_UNION)*/
* from t T where object_type='VIEW'
and OBJECT_ID IN (20,21,22)
AND OWNER='LJB';

说明：
增加无用条件IN (20,21,22) 使得联合索引激活！
联合索引：列2依赖列1；列3依赖列2
```

