[TOC]

# 非B Tree索引与SQL优化

## 位图索引

简单说，位图索引存放的就是比特值

```SQL
位图索引
键值重复率高的字段比较适合使用位图索引。
count、and、or、in这些特定的操作更适合位图索引。

DML操作比较多的表不适合使用位图索引。


实验：
创建实验表：
create table t
(name_id,
gender not null,
location not null,
age_group not null,
data
)
as
select rownum,
decode(ceil(dbms_random.value(0,2)),
1,'M',
2,'F')gender,
ceil(dbms_random.value(1,50)) location,
decode(ceil(dbms_random.value(0,3)),
1,'child',
2,'young',
3,'middle_age',
4,'old'),
rpad('*',400,'*')
from dual
connect by rownum<=100000;
收集统计信息：
exec dbms_stats.gather_table_stats(ownname => 'LJB',tabname => 'T',estimate_percent => 10,method_opt=> 'for all indexed columns',cascade=>TRUE) ;

创建bitmap index
create bitmap index gender_idx on t(gender);
create bitmap index location_idx on t(location);
create bitmap index age_group_idx on t(age_group);

select * from t where gender='M' and location in (1,10,30) and age_group='child';
--查询效率提高10倍！

--位图索引对count提升也很明显
创建实验表：
create table t as select * from dba_objects;
insert into t select * from t;
insert into t select * from t;
insert into t select * from t;
insert into t select * from t;
insert into t select * from t;
insert into t select * from t;
update t set object_id=rownum;
commit;

set autotrace on
set linesize 1000
select count(*) from t;
查看普通索引的效率：
create index idx_t_obj on t(object_id);
alter table T modify object_id not null;
select count(*) from t;
再查看位图索引的效率：
create bitmap index idx_bitm_t_status on t(status);


```

## 函数索引

函数索引是将函数计算得到的结果存储在行的列中，而不是存储列数据本身。

可以把函数索引看做有个虚拟列上的索引。

```SQL
--函数索引使用部分记录建索引
实验：
1 创建实验表
create table t (id int ,status varchar2(2));
insert into t select rownum ,'Y' from dual connect by rownum<=1000000;
insert into t select 1 ,'N' from dual;

2 创建普通索引
create index id_normal on t(status);

3 分析索引，查看执行计划
analyze table t compute statistics for table for all indexes for all indexed columns;
set autotrace traceonly
select * from t where status='N';
-----------------------------------------------------------------------------------------
| Id  | Operation                   | Name      | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |           |     1 |    10 |     4   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID| T         |     1 |    10 |     4   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN          | ID_NORMAL |     1 |       |     3   (0)| 00:00:01 |
-----------------------------------------------------------------------------------------
4 查看普通索引结构
select name,btree_space,lf_rows,height from index_stats; 
NAME                           BTREE_SPACE    LF_ROWS     HEIGHT
------------------------------ ----------- ---------- ----------
ID_NORMAL                         22589052    1000001          3


5 创建函数索引
drop index id_normal;
create index id_status on  t (Case when status= 'N' then 'N' end);
analyze table t compute statistics for table for all indexes for all indexed columns;

6 使用函数索引的执行计划
select * from t where (case when status='N' then 'N' end)='N';
-----------------------------------------------------------------------------------------
| Id  | Operation                   | Name      | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |           |     1 |    10 |     2   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID| T         |     1 |    10 |     2   (0)| 00:00:01 |
|*  2 |   INDEX RANGE SCAN          | ID_STATUS |     1 |       |     1   (0)| 00:00:01 |
-----------------------------------------------------------------------------------------

7 函数索引的结构
select name,btree_space,lf_rows,height from index_stats;
NAME                           BTREE_SPACE    LF_ROWS     HEIGHT
------------------------------ ----------- ---------- ----------
ID_STATUS                             7996          1          1


--函数索引减少函数递归调用
实验：
create table t1 (first_name varchar2(200),last_name varchar2(200),id number);
create table t2 as select * from dba_objects where rownum<=1000;
insert into t1 (first_name,last_name,id) select object_name,object_type,rownum from dba_objects where rownum<=1000;
commit;


create or replace function get_obj_name(p_id t2.object_id%type) return t2.object_name%type DETERMINISTIC is
v_name t2.object_name%type;
begin
select object_name
into v_name
from t2
where object_id=p_id;
return v_name;
end;
/

create index idx_func_id on t1(get_obj_name(id)); --创建函数索引，减少递归调用
select *   from t1 where get_obj_name(id)='TEST'  ;
```



## 反向键索引

反向键索引就是普通的B*Tree索引，只是键中的字节会“反转”。





## 全文索引

oracle实现全文检索，通过oracle的词法分析器来实现

```sql
create table test as select * from dba_objects;
update test set object_name='高兴' where rownum<=2;

--创建全文索引
sqlplus "/ as sysdba"
grant ctxapp to ljb;
alter user ctxsys  account unlock;
alter user ctxsys identified  by ctxsys;
connect ctxsys/ctxsys;
grant execute on ctx_ddl to ljb;
connect ljb/ljb

Begin
ctx_ddl.drop_preference('club_lexer');
ctx_ddl.drop_preference('mywordlist');
ctx_ddl.create_preference('club_lexer','CHINESE_LEXER');
ctx_ddl.create_preference('mywordlist', 'BASIC_WORDLIST');
ctx_ddl.set_attribute('mywordlist','PREFIX_INDEX','TRUE');
ctx_ddl.set_attribute('mywordlist','PREFIX_MIN_LENGTH',1);
ctx_ddl.set_attribute('mywordlist','PREFIX_MAX_LENGTH', 5);
ctx_ddl.set_attribute('mywordlist','SUBSTRING_INDEX', 'YES');
end;
/

create index  id_cont_test on TEST (object_name) indextype is ctxsys.context
parameters (
'DATASTORE CTXSYS.DIRECT_DATASTORE FILTER
CTXSYS.NULL_FILTER LEXER club_lexer WORDLIST mywordlist');

--使用全文检索查询
exec ctx_ddl.sync_index('id_cont_TEST', '20M');
select * from test where contains(OBJECT_NAME,'高兴')>0;
```

## 相关案例

### 1 位图索引在重复率低的列慎建

```sql
create bitmap index idx_bit_object_id on t(object_id); --重复率低
create bitmap index idx_bit_status on t(status);       --重复率高

select segment_name,blocks,bytes/1024/1024 "SIZE(M)"
from user_segments
where segment_name in( 'IDX_BIT_OBJECT_ID','IDX_BIT_STATUS');
--两个索引的大小相差100倍
```



### 2 位图索引易出现死锁

### 3 函数索引之30553错误

```SQL
create table test as select * from user_objects ;
create or replace function f_minus1(i int)
return int
is
begin
return(i-1);
end;
/
---建完函数后我们试着建立函数索引，发现建立失败
create index idx_ljb_test on test (f_minus1(object_id));
将会出现如下错误：
create or replace function f_minus1(i int)
return int DETERMINISTIC
is
begin
return(i-1);
end;
/
--现在发现加上DETERMINISTIC关键字后的自定义函数可以建立函数索引了！
create index idx_test on test (f_minus1(object_id));
```

### 4 函数变更对函数索引的影响

```sql
--在自定义函数代码更新时，对应的函数索引需要重建。否则查询结果会出现错误结果！
在重建索引后查询没有记录，这才是正确的结果：
SQL> drop index idx_pkg_f_y;
索引已删除。
SQL> create index idx_pkg_f_y on t ( pkg_f.f(y));
索引已创建。
```

### 5 反向键索引不适用于范围查询

### 6 全文索引之数据更新

```sql
update test set object_name='高兴' where object_id>=3 and object_id<=5;
commit;
--发现由于又修改了3条记录，查询结果本应该由2条变更为5条记录，但是发现再查，依然是2条!
select count(*) from test where contains(OBJECT_NAME,'高兴')>0;
---继续执行同步命令后
exec ctx_ddl.sync_index('id_cont_test', '20M');
---再次查询后，终于发现这下是5条记录了。
SQL> select count(*) from test where contains(OBJECT_NAME,'高兴')>0;
```



### 