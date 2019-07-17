# 如何减少SQL优化的时间

[TOC]

## 创建问题数据库脚本

```sql
/* 
假如mkdb脚本位于windows环境下的d盘的根目录下
运行方法 sqlplus "/ as sysdba" @d:\mkdb.sql
*/

create user ljb identified by ljb;
alter user ljb default tablespace users;
grant dba to ljb;

spool mkdb.log

---表空间扩展过于频繁
DROP tablespace tbs_a including contents AND datafiles;
create tablespace TBS_A datafile '/oracle/scripts/sql/TBS_A.DBF' size 1M autoextend on uniform size 64k;
CREATE TABLE ljb.t_a (id int,contents varchar2(1000)) tablespace TBS_A;
insert into ljb.t_a select rownum,LPAD('1', 1000, '*') from dual connect by level<=20000;
COMMIT;

--全局临时表被收集统计信息
drop table ljb.g_tab_del purge;
drop table ljb.g_tab_pre purge;
create global temporary table ljb.g_tab_del on commit DELETE rows as select  * from dba_objects   WHERE 1=2;
create global temporary table ljb.g_tab_pre ON commit preserve rows as select  * from dba_objects WHERE 1=2;
COMMIT;
exec dbms_stats.gather_table_stats(ownname => 'LJB', tabname => 'g_tab_del',cascade => true);
exec dbms_stats.gather_table_stats(ownname => 'LJB', tabname => 'g_tab_pre',cascade => true);

--表有过时字段
DROP   TABLE ljb.t_char_long purge;
CREATE TABLE ljb.t_char_long (NAME CHAR(10), VALUE LONG);

--表未建任何索引
DROP   TABLE ljb.t_noidx purge;
CREATE TABLE ljb.t_noidx AS SELECT * FROM dba_objects;
INSERT INTO  ljb.t_noidx SELECT * FROM ljb.t_noidx;
INSERT INTO  ljb.t_noidx SELECT * FROM ljb.t_noidx;
INSERT INTO  ljb.t_noidx SELECT * FROM ljb.t_noidx;
COMMIT;

--表建在系统表空间
DROP TABLE   ljb.t_sys purge;
CREATE TABLE ljb.t_sys tablespace system AS SELECT * FROM dba_objects WHERE rownum<=100;

--表带并行属性
DROP TABLE   ljb.t_degree purge;
CREATE TABLE ljb.t_degree parallel 8 AS SELECT * FROM dba_objects WHERE rownum<=100;

--表被压缩
DROP TABLE   ljb.t_compress purge;
CREATE TABLE ljb.t_compress compress  AS SELECT * FROM dba_objects ;

--表的日志被关闭
DROP   TABLE  ljb.t_nolog purge;
CREATE TABLE  ljb.t_nolog nologging AS SELECT * FROM dba_objects WHERE rownum<=100;

--表的列过多(比如超过100个列）
DROP TABLE ljb.t_col_max purge;
DECLARE
  l_sql VARCHAR2(32767);
BEGIN
  l_sql := 'CREATE TABLE ljb.t_col_max (';
  FOR i IN 1..100 
  LOOP
    l_sql := l_sql || 'n' || i || ' NUMBER,';
  END LOOP;
  l_sql := l_sql || 'pad VARCHAR2(1000)) PCTFREE 10';
  EXECUTE IMMEDIATE l_sql;
END;
/

--表的列过少(比如少于2个列）
DROP TABLE ljb.t_col_min1 purge;
DROP TABLE ljb.t_col_min2 purge;
CREATE TABLE ljb.t_col_min1 (id INT);
CREATE TABLE ljb.t_col_min2 (id INT,NAME varchar2(100));

--表的高水平位没释放
DROP   table ljb.t_high purge;
create table ljb.t_high as select * from dba_objects WHERE rownum<=10000;
insert into  ljb.t_high select * from ljb.t_high;
insert into  ljb.t_high select * from ljb.t_high;
insert into  ljb.t_high select * from ljb.t_high;
insert into  ljb.t_high select * from ljb.t_high;
insert into  ljb.t_high select * from ljb.t_high;
commit;
delete from ljb.t_high ;
COMMIT;

--表有触发器
DROP table ljb.t1_tri purge;
drop table ljb.t2_tri purge;

CREATE table ljb.t1_tri AS SELECT object_name, object_id, object_type FROM dba_objects WHERE rownum<=30;

CREATE table ljb.t2_tri
as
SELECT object_type, count(*) as cnt
FROM ljb.t1_tri
group by object_type;

CREATE or replace trigger ljb.tri_t2_trigger
after insert on ljb.t1_tri
for each row
begin
insert into ljb.t2_tri(object_type,cnt) values (:new.object_type,1);
end tri_t2_trigger;
/

--表分区数过多，比如超过100个
DROP TABLE   ljb.T_PART1 PURGE;
CREATE TABLE ljb.t_part1 partition BY range(dates) (partition P_MAX VALUES less than(maxvalue))
AS SELECT rownum id ,trunc(sysdate-rownum) AS DATES, ceil(dbms_random.value(0,100000)) nbr  FROM dual CONNECT BY level<=10000;
CREATE or replace procedure proc_1 AS
v_next_day DATE;
v_prev_day DATE;
v_sql_p_split_part varchar2(4000);
begin
for i in 1 .. 100 loop
       select add_months(trunc(sysdate), i) into v_next_day from dual;
       select add_months(trunc(sysdate), -i) into v_prev_day from dual;
       v_sql_p_split_part := 'alter table ljb.t_part1 split partition p_MAX at ' ||
               '(to_date(''' || to_char(v_next_day, 'yyyymmdd') ||
               ''',''yyyymmdd''))' || 'into (partition T_PART_' ||
               to_char(v_prev_day, 'yyyymm') || ' tablespace users ,partition p_MAX)';
               execute immediate v_sql_p_split_part;
end loop;
END;
/
exec proc_1;

--分区不均匀
drop   table ljb.t_part2 purge;
create table ljb.t_part2 (id INT,VALUE number)
    partition by range (id)
    (
    partition p1  values less than (1000),
    partition p2  values less than (2000),
    partition p3  values less than (3000),
    partition p4  values less than (4000),
    partition p5  values less than (5000),
    partition p6  values less than (6000),
    partition p7  values less than (7000),
    partition p8  values less than (8000),
    partition p9  values less than (9000),
    partition p10 values less than (maxvalue)
    )
    ;
INSERT INTO ljb.t_part2 SELECT rownum ,ceil(dbms_random.value(0,10000)) FROM dual CONNECT BY rownum<=10000;
INSERT INTO ljb.t_part2 SELECT rownum ,9999 FROM dual CONNECT BY rownum<=100000;
COMMIT;
exec dbms_stats.gather_table_stats('LJB', 't_part2');

--分区索引失效
---DROP INDEX ljb.idx_t_part1_id;
---DROP INDEX ljb.idx_t_part1_nbr;
CREATE INDEX ljb.idx_t_part1_id ON ljb.t_part1(id) ;
CREATE INDEX ljb.idx_t_part1_nbr ON ljb.t_part1(nbr) local;
--全局索引失效
ALTER INDEX ljb.idx_t_part1_id unusable;
--局部索引失效
alter table ljb.t_part1  move partition T_PART_201410;

--普通索引失效
DROP TABLE   ljb.t1 purge;
CREATE TABLE ljb.t1 AS SELECT * FROM dba_objects WHERE rownum<=2000;
CREATE INDEX ljb.idx_t1_obj_id ON ljb.t1(object_id);
ALTER INDEX  ljb.idx_t1_obj_id unusable;

--单表索引过多(比如超过6个）
DROP   TABLE ljb.t2 purge;
CREATE TABLE ljb.t2 AS SELECT * FROM dba_objects WHERE rownum<=2000;
CREATE INDEX ljb.idx_t2_obj_id   ON ljb.t2(object_id);
CREATE INDEX ljb.idx_t2_obj_type ON ljb.t2(object_type);
CREATE INDEX ljb.idx_t2_obj_name ON ljb.t2(object_name);
CREATE INDEX ljb.idx_t2_data_obj_id ON ljb.t2(data_object_id);
CREATE INDEX ljb.idx_t2_status ON ljb.t2(STATUS);
CREATE INDEX ljb.idx_t2_created ON ljb.t2(created);
CREATE INDEX ljb.idx_t2_last_ddl_time ON ljb.t2(last_ddl_time);

--索引被压缩
DROP TABLE   ljb.t_idx_compress purge;
CREATE TABLE ljb.t_idx_compress AS SELECT * FROM dba_objects ;
CREATE INDEX ljb.idx_id_compress ON ljb.t_idx_compress(object_id) compress;

--单表索引组合列过多
DROP   TABLE ljb.t3 purge;
CREATE TABLE ljb.t3 AS SELECT * FROM dba_objects WHERE rownum<=2000;
CREATE INDEX ljb.idx_t3_union ON ljb.t3(object_id,object_type,object_name,last_ddl_time);

--聚合因子明显不好的
drop table   ljb.colocated purge;
create table ljb.colocated ( x int, y varchar2(80) );
begin
    for i in 1 .. 10000
    loop
        insert into ljb.colocated(x,y)
        values (i, rpad(dbms_random.random,75,'*') );
    end loop;
end;
/
alter table ljb.colocated
add constraint colocated_pk
primary key(x);
begin
dbms_stats.gather_table_stats(ownname => 'LJB', tabname => 'COLOCATED',cascade => true);
END;
/

drop table   ljb.disorganized purge;
create table ljb.disorganized AS select x,y from ljb.colocated order by y;
alter table  ljb.disorganized add constraint disorganized_pk primary key (x);
begin
dbms_stats.gather_table_stats(ownname => 'LJB', tabname => 'DISORGANIZED',cascade => true);
END;
/

--索引建在系统表空间
DROP   TABLE ljb.t_idx_sys purge;
CREATE TABLE ljb.t_idx_sys tablespace system AS SELECT * FROM dba_objects WHERE rownum<=100;
CREATE INDEX ljb.idx_t_sys ON ljb.t_idx_sys(object_id) tablespace system;

--索引带并行属性
DROP   TABLE   ljb.t_idx_degree  purge;
CREATE TABLE   ljb.t_idx_degree  AS SELECT * FROM dba_objects WHERE rownum<=100;
CREATE index   ljb.idx_t_degree  ON ljb.t_idx_degree(object_id) parallel 8;


--外键未建索引
drop table   ljb.t_p cascade constraints purge;
drop table   ljb.t_c cascade constraints purge;
CREATE TABLE ljb.T_P (ID NUMBER, NAME VARCHAR2(30));
ALTER TABLE  ljb.T_P ADD CONSTRAINT  T_P_ID_PK  PRIMARY KEY (ID);
CREATE TABLE ljb.T_C (ID NUMBER, FID NUMBER, NAME VARCHAR2(30));
ALTER TABLE  ljb.T_C ADD CONSTRAINT FK_T_C FOREIGN KEY (FID) REFERENCES ljb.T_P (ID);
INSERT INTO  ljb.T_P SELECT ROWNUM, TABLE_NAME FROM ALL_TABLES;
INSERT INTO  ljb.T_C SELECT ROWNUM, MOD(ROWNUM, 1000) + 1, OBJECT_NAME  FROM ALL_OBJECTS WHERE rownum<=100;
COMMIT;
--create index ljb.idx_IND_T_C_FID on T_C(FID);


--有位图索引
DROP TABLE   ljb.t_bit  purge;
CREATE TABLE ljb.t_bit AS SELECT * FROM dba_objects WHERE rownum<=100;
CREATE INDEX ljb.idx_bit_status ON ljb.t_bit(STATUS);

--有函数索引
DROP TABLE   ljb.t_fun purge;
CREATE TABLE ljb.t_fun AS SELECT * FROM dba_objects WHERE rownum<=100;
CREATE INDEX ljb.idx_func_objname ON ljb.t_fun(upper(object_name));

--有反向键索引
DROP TABLE   ljb.t_rev purge;
--DROP INDEX   ljb.idx_rev_objid;
CREATE TABLE ljb.t_rev AS SELECT * FROM dba_objects WHERE rownum<=100;
CREATE INDEX ljb.idx_rev_objid ON ljb.t_fun(object_id) REVERSE;

--组合和单列索引存在交叉
DROP   TABLE ljb.t_cross purge;
CREATE TABLE ljb.t_cross AS SELECT * FROM dba_objects WHERE rownum<=100;
CREATE INDEX ljb.idx_obj_id_name ON ljb.t_cross(object_id,object_name);
CREATE INDEX ljb.idx_obj_id ON ljb.t_cross(object_id);


--sql长度超过100行
DROP TABLE   ljb.t_sql purge;
CREATE TABLE ljb.t_sql AS SELECT * FROM dba_objects;
SELECT * FROM ljb.t_sql WHERE 
object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND
object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND
object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND
object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND
object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND
object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND
object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 AND object_id=1 ;

--未使用绑定变量
drop   table ljb.t_unbind purge;
create table ljb.t_unbind ( x int );
create or replace procedure ljb.proc_unbind
as
begin
    for i in 1 .. 20000
    loop
        execute immediate
        'insert into ljb.t_unbind values ( '||i||')';
    end loop;
   COMMIT;
end;
/ 
exec ljb.proc_unbind;


--未使用批量提交
drop   table ljb.t_unbatch purge;
create table ljb.t_unbatch ( x int );
create or replace procedure ljb.proc_unbatch
as
begin
    for i in 1 .. 20000
    loop
     insert into ljb.t_unbatch values (i);   
     commit;
    end loop;
end;
/
exec ljb.proc_unbatch;


--失效的过程包等
drop   table ljb.t_proc purge;
CREATE table ljb.t_proc (id number,col2 number);
INSERT into  ljb.t_proc select rownum, ceil(dbms_random.value(1,10)) from dual connect by level<=10;
COMMIT;


-----有效的
Create or replace 
procedure ljb.p_insert1_t_proc (p in number ) is
begin
	insert into ljb.t_proc (id,col2) VALUES(rownum,p);
	commit;
end ;
/

-----失效的
Create or replace 
procedure ljb.p_insert2_t_proc (p in number ) is
begin
	insert into ljb.t_proc (id,col3) VALUES(rownum,p);
	commit;
end ;
/

---构造CACHE小于20的序列
DROP SEQUENCE ljb.seqtest1;
CREATE  SEQUENCE ljb.seqtest1
INCREMENT BY 1 
START WITH 1 
NOMAXvalue 
NOCYCLE 
CACHE 20; 

spool off

EXIT
```



## 获取数据库整体信息脚本

注释：此脚本输出5个文件；

1. addm最近一小时

2. ash最近半小时

3. awr最近一小时

4. awr最近7天

5. spool开头的数据库相关信息

   

```sql
/*
  运行方法 sqlplus "/ as sysdba" @d:\spooldb.sql
  created by liangjb
*/

SET markup html ON spool ON pre off entmap off

set term off
set heading on
set verify off
set feedback off

set linesize 2000
set pagesize 30000
set long 999999999
set longchunksize 999999

column index_name format a30
column table_name format a30
column num_rows format 999999999
column index_type format a24
column num_rows format 999999999
column status format a8
column clustering_factor format 999999999
column degree format a10
column blevel format 9
column distinct_keys format 9999999999
column leaf_blocks format   9999999
column last_analyzed    format a10
column column_name format a25
column column_position format 9
column temporary format a2
column partitioned format a5
column partitioning_type format a7
column partition_count format 999
column program  format a30
column spid  format a6
column pid  format 99999
column sid  format 99999
column serial# format 99999
column username  format a12
column osuser    format a12
column logon_time format  date
column event    format a32
column JOB_NAME        format a30
column PROGRAM_NAME    format a32
column STATE           format a10
column window_name           format a30
column repeat_interval       format a60
column machine format a30
column program format a30
column osuser format a15
column username format a15
column event format a50
column seconds format a10
column sqltext format a100

SET markup html off
column dbid new_value spool_dbid
column inst_num new_value spool_inst_num
select dbid from v$database where rownum = 1;
select instance_number as inst_num from v$instance where rownum = 1;
column spoolfile_name new_value spoolfile
select 'spool_'||(select name from v$database where rownum=1) ||'_'|| (select instance_number from v$instance where rownum=1)||'_'||to_char(sysdate,'yy-mm-dd_hh24.mi') as spoolfile_name from dual;
spool &&spoolfile..html

SET markup html off
set serveroutput on;
exec dbms_output.enable(9999999999);
exec dbms_output.put_line('<html>');
exec dbms_output.put_line('<style>');
exec dbms_output.put_line('th {font:bold 8pt Arial,Helvetica,Geneva,sans-serif; color:White; background:#0066CC;padding-left:4px; padding-right:4px;padding-bottom:2px}');
exec dbms_output.put_line('</style>');
exec dbms_output.put_line('<head>');
exec dbms_output.put_line('</head>');
exec dbms_output.put_line('<body>');

SET markup html on

prompt <p>版本
select * from v$version;
select * from dba_registry_history;

prompt <p>最近一次启动时间，版本，以及是否RAC
select * from
 (select name db_name from v$database),
 (select instance_name from v$instance),
 (select archiver from v$instance),
 (select snap_interval awr_interval, retention awr_retention FROM DBA_HIST_WR_CONTROL),
 (select flashback_on from v$database),
 (select parallel from v$instance),
 (select startup_time from v$instance),
 (select decode(name,null,'NOT ASM','ASM') IS_ASM from (select null name from dual union all (select name IS_ASM from v$datafile where name like '+%' and rownum = 1))),
 (select max(end_time) rman_lastcompleted from v$rman_status where status = 'COMPLETED' and object_type like 'DB FULL')
;

prompt <p>30分钟内CPU或等待最长的
select t.*, s.sid, s.serial#, s.machine, s.program, s.osuser
  from (select c.USERNAME,
               a.event,
               to_char(a.cnt) as seconds,
               a.sql_id
               --,dbms_lob.substr(b.sql_fulltext,50,1) sqltext
          from (select rownum rn, t.*
                  from (select decode(s.session_state,
                                      'WAITING',
                                      s.event,
                                      'Cpu + Wait For Cpu') Event,
                               s.sql_id,
                               s.user_id,
                               count(*) CNT
                          from v$active_session_history s
                         where sample_time > sysdate - 30 / 1440
                         group by s.user_id,
                                  decode(s.session_state,
                                         'WAITING',
                                         s.event,
                                         'Cpu + Wait For Cpu'),
                                  s.sql_id
                         order by CNT desc) t
                 where rownum < 20) a,
               v$sqlarea b,
               dba_users c
         where a.sql_id = b.sql_id
           and a.user_id = c.user_id
         order by CNT desc) t,
       v$session s
where t.sql_id = s.sql_id(+)
;

prompt <p>近期负载情况(根据AWR快照)
select s.snap_date,
       decode(s.redosize, null, '--shutdown or end--', s.currtime) "TIME",
       to_char(round(s.seconds/60,2)) "elapse(min)",
       round(t.db_time / 1000000 / 60, 2) "DB time(min)",
       s.redosize redo,
       round(s.redosize / s.seconds, 2) "redo/s",
       s.logicalreads logical,
       round(s.logicalreads / s.seconds, 2) "logical/s",
       physicalreads physical,
       round(s.physicalreads / s.seconds, 2) "phy/s",
       s.executes execs,
       round(s.executes / s.seconds, 2) "execs/s",
       s.parse,
       round(s.parse / s.seconds, 2) "parse/s",
       s.hardparse,
       round(s.hardparse / s.seconds, 2) "hardparse/s",
       s.transactions trans,
       round(s.transactions / s.seconds, 2) "trans/s"
  from (select curr_redo - last_redo redosize,
               curr_logicalreads - last_logicalreads logicalreads,
               curr_physicalreads - last_physicalreads physicalreads,
               curr_executes - last_executes executes,
               curr_parse - last_parse parse,
               curr_hardparse - last_hardparse hardparse,
               curr_transactions - last_transactions transactions,
               round(((currtime + 0) - (lasttime + 0)) * 3600 * 24, 0) seconds,
               to_char(currtime, 'yy/mm/dd') snap_date,
               to_char(currtime, 'hh24:mi') currtime,
               currsnap_id endsnap_id,
               to_char(startup_time, 'yyyy-mm-dd hh24:mi:ss') startup_time
          from (select a.redo last_redo,
                       a.logicalreads last_logicalreads,
                       a.physicalreads last_physicalreads,
                       a.executes last_executes,
                       a.parse last_parse,
                       a.hardparse last_hardparse,
                       a.transactions last_transactions,
                       lead(a.redo, 1, null) over(partition by b.startup_time order by b.end_interval_time) curr_redo,
                       lead(a.logicalreads, 1, null) over(partition by b.startup_time order by b.end_interval_time) curr_logicalreads,
                       lead(a.physicalreads, 1, null) over(partition by b.startup_time order by b.end_interval_time) curr_physicalreads,
                       lead(a.executes, 1, null) over(partition by b.startup_time order by b.end_interval_time) curr_executes,
                       lead(a.parse, 1, null) over(partition by b.startup_time order by b.end_interval_time) curr_parse,
                       lead(a.hardparse, 1, null) over(partition by b.startup_time order by b.end_interval_time) curr_hardparse,
                       lead(a.transactions, 1, null) over(partition by b.startup_time order by b.end_interval_time) curr_transactions,
                       b.end_interval_time lasttime,
                       lead(b.end_interval_time, 1, null) over(partition by b.startup_time order by b.end_interval_time) currtime,
                       lead(b.snap_id, 1, null) over(partition by b.startup_time order by b.end_interval_time) currsnap_id,
                       b.startup_time
                  from (select snap_id,
                               dbid,
                               instance_number,
                               sum(decode(stat_name, 'redo size', value, 0)) redo,
                               sum(decode(stat_name,
                                          'session logical reads',
                                          value,
                                          0)) logicalreads,
                               sum(decode(stat_name,
                                          'physical reads',
                                          value,
                                          0)) physicalreads,
                               sum(decode(stat_name, 'execute count', value, 0)) executes,
                               sum(decode(stat_name,
                                          'parse count (total)',
                                          value,
                                          0)) parse,
                               sum(decode(stat_name,
                                          'parse count (hard)',
                                          value,
                                          0)) hardparse,
                               sum(decode(stat_name,
                                          'user rollbacks',
                                          value,
                                          'user commits',
                                          value,
                                          0)) transactions
                          from dba_hist_sysstat
                         where stat_name in
                               ('redo size',
                                'session logical reads',
                                'physical reads',
                                'execute count',
                                'user rollbacks',
                                'user commits',
                                'parse count (hard)',
                                'parse count (total)')
                         group by snap_id, dbid, instance_number) a,
                       dba_hist_snapshot b
                 where a.snap_id = b.snap_id
                   and a.dbid = b.dbid
                   and a.instance_number = b.instance_number
                   and a.dbid = &&spool_dbid
                   and a.instance_number = &&spool_inst_num
                 order by end_interval_time)) s,
       (select lead(a.value, 1, null) over(partition by b.startup_time order by b.end_interval_time) - a.value db_time,
               lead(b.snap_id, 1, null) over(partition by b.startup_time order by b.end_interval_time) endsnap_id
          from dba_hist_sys_time_model a, dba_hist_snapshot b
         where a.snap_id = b.snap_id
           and a.dbid = b.dbid
           and a.instance_number = b.instance_number
           and a.stat_name = 'DB time'
           and a.dbid = &&spool_dbid
           and a.instance_number = &&spool_inst_num) t
 where s.endsnap_id = t.endsnap_id
order by s.snap_date desc ,time asc
;

prompt <p>逻辑读最多
select *
  from (select sql_id,
               s.EXECUTIONS,
               s.LAST_LOAD_TIME,
               s.FIRST_LOAD_TIME,
               s.DISK_READS,
               s.BUFFER_GETS
          from v$sql s
         where s.buffer_gets > 300
         order by buffer_gets desc)
where rownum <= 10
;

prompt <p>物理读最多
select *
  from (select sql_id,
       s.EXECUTIONS,
       s.LAST_LOAD_TIME,
       s.FIRST_LOAD_TIME,
       s.DISK_READS,
       s.BUFFER_GETS,
       s.PARSE_CALLS
  from v$sql s
where s.disk_reads > 300
order by disk_reads desc)
where rownum<=10
;

prompt <p>执行次数最多
select *
  from (select sql_id,
               s.EXECUTIONS,
               s.LAST_LOAD_TIME,
               s.FIRST_LOAD_TIME,
               s.DISK_READS,
               s.BUFFER_GETS,
               s.PARSE_CALLS
          from v$sql s
         order by s.EXECUTIONS desc)
where rownum <= 10
;

prompt <p>解析次数最多
select *
  from (select sql_id,
               s.EXECUTIONS,
               s.LAST_LOAD_TIME,
               s.FIRST_LOAD_TIME,
               s.DISK_READS,
               s.BUFFER_GETS,
               s.PARSE_CALLS
          from v$sql s
         order by s.PARSE_CALLS desc)
where rownum <= 10
;

prompt <p>磁盘排序最多
select sess.username, sql.address, sort1.blocks
  from v$session sess, v$sqlarea sql, v$sort_usage sort1
where sess.serial# = sort1.session_num
   and sort1.sqladdr = sql.address
   and sort1.sqlhash = sql.hash_value
   and sort1.blocks > 200
order by sort1.blocks desc
;

prompt <p>提交次数超过10000的session
select t1.sid, t1.value, t2.name
  from v$sesstat t1, v$statname t2
 where t2.name like '%user commits%'
   and t1.STATISTIC# = t2.STATISTIC#
   and value >= 10000
 order by value desc
;

prompt <p>长度超过100的SQL
SELECT SQL_ID, COUNT(*) line_count
  FROM V$SQLTEXT
 GROUP BY SQL_ID
HAVING COUNT(*) >= 100
 ORDER BY COUNT(*) DESC
;

prompt <p>查询共享内存占有率
select count(*),round(sum(sharable_mem)/1024/1024,2)
  from v$db_object_cache a
;

prompt <p>表有带并行度
select t.owner, t.table_name, degree
  from dba_tables t
where  trim(t.degree) <>'1'
   and trim(t.degree)<>'0'
   and owner not in ('SYSTEM','SYSMAN','SYS','CTXSYS','MDSYS','OLAPSYS','WMSYS','EXFSYS','LBACSYS','WKSYS','XDB','ORDSYS','DBSNMP','OUTLN','TSMSYS')
   and owner not like 'FLOWS%'
   and owner not like 'WK%'
;

prompt <p>索引有带并行度
select t.owner, t.table_name, index_name, degree, status
  from dba_indexes t
where  trim(t.degree) <>'1'
   and trim(t.degree)<>'0'
   and owner not in ('SYSTEM','SYSMAN','SYS','CTXSYS','MDSYS','OLAPSYS','WMSYS','EXFSYS','LBACSYS','WKSYS','XDB','ORDSYS','DBSNMP','OUTLN','TSMSYS')
   and owner not like 'FLOWS%'
   and owner not like 'WK%'
;

prompt <p>失效索引
select t.index_name,
       t.table_name,
       blevel,
       t.num_rows,
       t.leaf_blocks,
       t.distinct_keys
  from dba_indexes t
  where status = 'UNUSABLE'
  and  table_owner not in ('SYSTEM','SYSMAN','SYS','CTXSYS','MDSYS','OLAPSYS','WMSYS','EXFSYS','LBACSYS','WKSYS','XDB')
  and  table_owner not like 'FLOWS%'
;
select t2.owner,
       t1.blevel,
       t1.leaf_blocks,
       t1.INDEX_NAME,
       t2.table_name,
       t1.PARTITION_NAME,
       t1.STATUS
  from dba_ind_partitions t1, dba_indexes t2
where t1.index_name = t2.index_name
   and t1.STATUS = 'UNUSABLE'
;

prompt <p>失效对象
select t.owner,
       t.object_type,
       t.object_name
  from dba_objects t
 where STATUS='INVALID'
order by 1, 2
;

prompt <p>位图索引和函数索引、反向键索引
select t.owner,
       t.table_name,
       t.index_name,
       t.index_type,
       t.status,
       t.blevel,
       t.leaf_blocks
  from dba_indexes t
 where index_type in ('BITMAP', 'FUNCTION-BASED NORMAL', 'NORMAL/REV')
   and owner not in ('SYSTEM','SYSMAN','SYS','CTXSYS','MDSYS','OLAPSYS','WMSYS','EXFSYS','LBACSYS','WKSYS','XDB','ORDSYS','DBSNMP','OUTLN','TSMSYS')
   and owner not like 'FLOWS%'
   and owner not like 'WK%'
;

prompt <p>组合索引组合列超过4个的
select table_owner,table_name, index_name, count(*)
  from dba_ind_columns
where  table_owner not in ('SYSTEM','SYSMAN','SYS','CTXSYS','MDSYS','OLAPSYS','WMSYS','EXFSYS','LBACSYS','WKSYS','XDB','ORDSYS','DBSNMP','OUTLN','TSMSYS')
   and table_owner not like 'FLOWS%'
   and table_owner not like 'WK%'
 group by table_owner,table_name, index_name
having count(*) >= 4
 order by count(*) desc
;

prompt <p>索引个数字超过5个的
select owner,table_name, count(*) cnt
  from dba_indexes
where  owner not in ('SYSTEM','SYSMAN','SYS','CTXSYS','MDSYS','OLAPSYS','WMSYS','EXFSYS','LBACSYS','WKSYS','XDB','ORDSYS','DBSNMP','OUTLN','TSMSYS')
   and owner not like 'FLOWS%'
   and owner not like 'WK%'
 group by owner,table_name
having count(*) >= 5
order by cnt desc
;

prompt <p>哪些大表从未建过索引。
select segment_name,
       bytes / 1024 / 1024 / 1024 "GB",
       blocks,
       tablespace_name
  from dba_segments
 where segment_type = 'TABLE'
   and segment_name not in (select table_name from dba_indexes)
   and bytes / 1024 / 1024 / 1024 >= 0.5
 order by GB desc
;
select segment_name, sum(bytes) / 1024 / 1024 / 1024 "GB", sum(blocks)
  from dba_segments
 where segment_type = 'TABLE PARTITION'
   and segment_name not in (select table_name from dba_indexes)
 group by segment_name
having sum(bytes) / 1024 / 1024 / 1024 >= 0.5
 order by GB desc
;

prompt <p>哪些表的组合索引与单列索引存在交叉的情况。

select table_name, trunc(count(distinct(column_name)) / count(*),2) cross_idx_rate
  from dba_ind_columns
 where table_owner not in ('SYSTEM','SYSMAN','SYS','CTXSYS','MDSYS','OLAPSYS','WMSYS','EXFSYS','LBACSYS','WKSYS','XDB','ORDSYS','DBSNMP','OUTLN','TSMSYS')
   and table_owner not like 'FLOWS%'
   and table_owner not like 'WK%'
 group by table_name
having count(distinct(column_name)) / count(*) < 1 
order by cross_idx_rate desc
;

prompt <p>哪些对象建在系统表空间上。
select * from (
select owner, segment_name, tablespace_name, count(*) num
  from dba_segments
 where tablespace_name in('SYSTEM','SYSAUX')
group by owner, segment_name, tablespace_name)
where  owner not in ('SYSTEM','SYSMAN','SYS','CTXSYS','MDSYS','OLAPSYS','WMSYS','EXFSYS','LBACSYS','WKSYS','XDB','ORDSYS','DBSNMP','OUTLN','TSMSYS')
   and owner not like 'FLOWS%'
   and owner not like 'WK%'
;

prompt <p>检查统计信息是否被收集
select t.job_name,t.program_name,t.state,t.enabled
  from dba_scheduler_jobs t
where job_name = 'GATHER_STATS_JOB'
;
select client_name,status
  from dba_autotask_client
;
select window_next_time,autotask_status
  from DBA_AUTOTASK_WINDOW_CLIENTS
;

prompt <p>检查哪些未被收集或者很久没收集
select owner, count(*)
  from dba_tab_statistics t
where (t.last_analyzed is null or t.last_analyzed < sysdate - 100)
   and table_name not like 'BIN$%'
group by owner
order by owner
;

prompt <p>被收集统计信息的临时表
select owner, table_name, t.last_analyzed, t.num_rows, t.blocks
  from dba_tables t
where t.temporary = 'Y'
   and last_analyzed is not null
;

prompt <p>日志切换频率分析
select *
  from (select thread#, sequence#, to_char(first_time, 'MM/DD/RR HH24:MI:SS')
          from v$log_history
         order by first_time desc)
 where rownum <= 50
;

prompt <p>最近10天中每天日志切换的量
SELECT SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH:MI:SS'),1,5) Day,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'00',1,0)) H00,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'01',1,0)) H01,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'02',1,0)) H02,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'03',1,0)) H03,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'04',1,0)) H04,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'05',1,0)) H05,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'06',1,0)) H06,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'07',1,0)) H07,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'08',1,0)) H08,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'09',1,0)) H09,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'10',1,0)) H10,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'11',1,0)) H11,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'12',1,0)) H12,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'13',1,0)) H13,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'14',1,0)) H14,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'15',1,0)) H15,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'16',1,0)) H16,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'17',1,0)) H17,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'18',1,0)) H18,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'19',1,0)) H19,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'20',1,0)) H20,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'21',1,0)) H21,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'22',1,0)) H22 ,
       SUM(DECODE(SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH24:MI:SS'),10,2),'23',1,0)) H23,
       COUNT(*) TOTAL
FROM v$log_history  a
   where first_time>=to_char(sysdate-11)
GROUP BY SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH:MI:SS'),1,5)
ORDER BY SUBSTR(TO_CHAR(first_time, 'MM/DD/RR HH:MI:SS'),1,5) DESC
;

prompt <p>日志组大小
select group#,bytes,status
  from v$log
;

prompt <p>查看recovery_file_dest使用率
 select substr(name, 1, 30) name,
        space_limit as quota,
        space_used as used,
        space_reclaimable as reclaimable,
        number_of_files as files
   from v$recovery_file_dest
;
select * from V$FLASH_RECOVERY_AREA_USAGE
;

prompt <p>检查序列小于20的情况
select sequence_owner,
       count(*) CNT,
       sum(case when t.cache_size <= 20 then 1 else 0 end ) CNT_LESS_20,
       sum(case when t.cache_size > 20 then 1 else 0 end ) CNT_MORE_20
  from dba_sequences t
 group by sequence_owner
;

prompt <p>表空间使用情况
set markup html off
prompt <p>
declare
  type NUMBER_ARRAY is table of number(15) index by varchar2(30);
  ts_free_mb NUMBER_ARRAY;
  cursor c1 is
    select tablespace_name, sum(free_mb) + sum(expired_mb) free_mb
      from (SELECT tablespace_name,
                   round(sum(nvl(bytes, 0)) / 1024 / 1024, 2) free_mb,
                   0 expired_mb
              FROM dba_free_space
             GROUP BY tablespace_name
            union all
            select tablespace_name,
                   0 free_mb,
                   round(sum(nvl(bytes, 0)) / 1024 / 1024, 2) expired_mb
              from dba_undo_extents d
             where tablespace_name =
                   (select value
                      from v$parameter
                     where name = 'undo_tablespace')
               and status = 'EXPIRED'
             group by tablespace_name)
     group by tablespace_name;
  cursor c2 is
    SELECT /*+ rule */
     tablespace_name, round(sum(bytes) / 1024 / 1024, 2) total_mb
      FROM dba_data_files
     where tablespace_name not in
           (select tablespace_name
              from dba_data_files
             where upper(AUTOEXTENSIBLE) = 'YES')
     GROUP BY tablespace_name;
  ts_name  varchar2(30);
  ts_total number(15);
  ts_used  number(15);
  ts_free  number(15);
  ts_rate  varchar2(5);
begin
  for rec1 in c1 loop
    ts_free_mb(rec1.tablespace_name) := rec1.free_mb;
  end loop;
  dbms_output.put_line('<table border="1" width="90%" align="center" summary="Script output">');
  dbms_output.put_line('<tr><th>ts_name</th><th>ts_total</th><th>ts_used</th><th>ts_free</th><th>ts_rate</th></tr>');
  for rec2 in c2 loop
    ts_name  := null;
    ts_total := null;
    ts_used  := null;
    ts_free  := null;
    ts_rate  := null;
    ts_name  := rec2.tablespace_name;
    ts_total := rec2.total_mb;
    ts_free  := nvl(ts_free_mb(ts_name), 0);
    ts_used  := nvl(ts_total - ts_free, 0);
    ts_rate  := to_char(round((ts_total - ts_free) / ts_total * 100, 2),
                        'fm990.99');
    dbms_output.put_line('<tr><td>' || ts_name || '</td><td>' || ts_total ||
                         '</td><td>' || ts_used || '</td><td>' || ts_free ||
                         '</td><td>' || ts_rate || '</td></tr>');
  end loop;
  dbms_output.put_line('</table>');
end;
/
prompt <p>
set markup html on

prompt <p>整个数据库有多大
select owner, round(sum(bytes) / 1024 / 1024 / 1024, 2) "GB"
  from dba_segments
 group by owner
 order by 2 desc
;

prompt <p>对象大小TOP10
select *
  from (select owner,
               segment_name,
               segment_type,
               round(sum(bytes) / 1024 / 1024) object_size
          from DBA_segments
         group by owner, segment_name, segment_type
         order by object_size desc)
where rownum <= 10
;

prompt <p>回收站情况(大小及数量）
select *
  from (select SUM(BYTES) / 1024 / 1024 / 1024 as recyb_size
          from DBA_SEGMENTS
         WHERE SEGMENT_NAME LIKE 'BIN$%') a,
       (select count(*) as recyb_cnt from dba_recyclebin)
;

prompt <p>查谁占用了undo表空间

SELECT r.name "roll_segment_name", rssize/1024/1024/1024 "RSSize(G)",
  s.sid,
  s.serial#,
  s.username,
  s.status,
  s.sql_hash_value,
  s.SQL_ADDRESS,
  s.MACHINE,
  s.MODULE,
  substr(s.program, 1, 78) program,
  r.usn,
  hwmsize/1024/1024/1024, shrinks ,xacts
FROM sys.v_$session s,sys.v_$transaction t,sys.v_$rollname r, v$rollstat rs
WHERE t.addr = s.taddr and t.xidusn = r.usn and r.usn=rs.USN
Order by rssize desc
;

prompt <p>查谁占用了temp表空间

select sql.sql_id,
       t.Blocks * 16 / 1024 / 1024,
       s.USERNAME,
       s.SCHEMANAME,
       t.tablespace,
       t.segtype,
       t.extents,
       s.PROGRAM,
       s.OSUSER,
       s.TERMINAL,
       s.sid,
       s.SERIAL#
  from v$sort_usage t, v$session s , v$sql sql
where t.SESSION_ADDR = s.SADDR
  and t.SQLADDR=sql.ADDRESS
  and t.SQLHASH=sql.HASH_VALUE
;

prompt <p>观察回滚段，临时段及普通段否是自动扩展
select t2.contents, t1.*
  from (select file_name, tablespace_name, bytes, maxbytes, autoextensible
          from dba_temp_files
        union all
        select file_name, tablespace_name, bytes, maxbytes, autoextensible
          from dba_data_files) t1,
       dba_tablespaces t2
 where t1.tablespace_name = t2.tablespace_name
;

prompt <p>表大小超过10GB未建分区的
select owner,
       segment_name,
       segment_type,
       round(sum(bytes) / 1024 / 1024 / 1024,2) object_size
  from dba_segments
where segment_type = 'TABLE'
  and bytes > 10*1024*1024*1024
group by owner, segment_name, segment_type
order by object_size desc
;

prompt <p>分区最多的前10个对象
select *
  from (select table_owner, table_name, count(*) cnt
          from dba_tab_partitions
         group by table_owner, table_name
         order by cnt desc)
where rownum <= 10
;

prompt <p>分区不均匀的表
select *
  from (select table_owner,
               table_name,
               max(num_rows) max_num_rows,
               trunc(avg(num_rows), 0) avg_num_rows,
               sum(num_rows) sum_num_rows,
               case
                 when sum(num_rows) = 0 then
                  0
                 else
                  trunc(max(num_rows) / trunc(avg(num_rows), 0), 2)
               end rate,
			   count(*) part_count
          from dba_tab_partitions
         group by table_owner, table_name)
 where rate > 5;

prompt <p>列数量超过100个或小于2的表
select *
  from (select owner, table_name, count(*) col_count
          from dba_tab_cols
         group by owner, table_name)
 where col_count > 100
    or col_count <= 2
 and  owner not in ('SYSTEM','SYSMAN','SYS','CTXSYS','MDSYS','OLAPSYS','WMSYS','EXFSYS','LBACSYS','WKSYS','XDB')
 and  owner not like 'FLOWS%'
;

prompt <p>表属性是nologging的
select owner, table_name, tablespace_name, logging
  from dba_tables
 where logging = 'NO'
 and  owner not in ('SYSTEM','SYSMAN','SYS','CTXSYS','MDSYS','OLAPSYS','WMSYS','EXFSYS','LBACSYS','WKSYS','XDB')
 and  owner not like 'FLOWS%'
 ;

prompt <p>表属性含COMPRESSION的
select owner, table_name, tablespace_name, COMPRESSION
  from dba_tables
 where COMPRESSION = 'ENABLED'
 and  owner not in ('SYSTEM','SYSMAN','SYS','CTXSYS','MDSYS','OLAPSYS','WMSYS','EXFSYS','LBACSYS','WKSYS','XDB')
 and  owner not like 'FLOWS%'
;

prompt <p>索引属性含COMPRESSION的
select owner, index_name, table_name, COMPRESSION
  from dba_indexes
 where COMPRESSION = 'ENABLED'
 and  owner not in ('SYSTEM','SYSMAN','SYS','CTXSYS','MDSYS','OLAPSYS','WMSYS','EXFSYS','LBACSYS','WKSYS','XDB')
 and  owner not like 'FLOWS%'
;

prompt <p>触发器
select OWNER, TRIGGER_NAME, TABLE_NAME, STATUS
  from dba_triggers
 where owner not in ('SYSTEM','SYSMAN','SYS','CTXSYS','MDSYS','OLAPSYS','WMSYS','EXFSYS','LBACSYS','WKSYS','XDB')
 and  owner not like 'FLOWS%';
 
prompt <p>将外键未建索引的情况列出
select *
  from (select pk.owner PK_OWNER,
               pk.constraint_name PK_NAME,
               pk.table_name PK_TABLE_NAME,
               fk.owner FK_OWNER,
               fk.constraint_name FK_NAME,
               fk.table_name FK_TABLE_NAME,
               fk.delete_rule FK_DELETE_RULE,
               ind_col.INDEX_NAME FK_INDEX_NAME,
               ind.index_type FK_INDEX_TYPE,
               con_col.COLUMN_NAME FK_INDEX_COLUMN_NAME,
               con_col.POSITION FK_INDEX_COLUMN_POSITION,
               decode(ind_col.INDEX_NAME,
                      NULL,
                      NULL,
                      NVL(x_ind.status, 'VALID')) IS_IND_VALID,
               decode(ind_col.INDEX_NAME,
                      NULL,
                      NULL,
                      NVL(x_ind_part.status, 'VALID')) IS_IND_PART_VALID
          from (select * from dba_constraints where constraint_type = 'R') fk,
               (select * from dba_constraints where constraint_type = 'P') pk,
               dba_cons_columns con_col,
               dba_ind_columns ind_col,
               dba_indexes ind,
               (select index_owner, index_name, status
                  from dba_ind_partitions
                 where status <> 'VALID') x_ind_part,
               (select owner as index_owner, index_name, status
                  from dba_indexes
                 where status <> 'VALID') x_ind
         where fk.r_constraint_name = pk.constraint_name
           and pk.owner = fk.owner
           and fk.owner = con_col.owner
           and fk.table_name = con_col.table_name
           and fk.constraint_name = con_col.CONSTRAINT_NAME
           and con_col.owner = ind_col.TABLE_OWNER(+)
           and con_col.TABLE_NAME = ind_col.TABLE_NAME(+)
           and con_col.COLUMN_NAME = ind_col.COLUMN_NAME(+)
           and ind_col.INDEX_OWNER = ind.owner(+)
           and ind_col.INDEX_NAME = ind.index_name(+)
           and ind_col.INDEX_OWNER = x_ind.index_owner(+)
           and ind_col.INDEX_NAME = x_ind.index_name(+)
           and ind_col.INDEX_OWNER = x_ind_part.index_owner(+)
           and ind_col.INDEX_NAME = x_ind_part.index_name(+))
 where FK_INDEX_NAME is null
 order by FK_OWNER ASC
;

/* 性能不好
prompt <p>热点块(汇总）
SELECT *+ rule *
         e.owner, e.segment_name, e.segment_type, sum(b.tch) tch
          FROM dba_extents e,
               (SELECT *
                  FROM (SELECT addr, ts#, file#, dbarfil, dbablk, tch
                          FROM x$bh
                         ORDER BY tch DESC)
                 WHERE ROWNUM <= 10) b
         WHERE e.relative_fno = b.dbarfil
           AND e.block_id <= b.dbablk
           AND e.block_id + e.blocks > b.dbablk
		   group by e.owner, e.segment_name, e.segment_type
order by tch desc
;
prompt <p>热点块(展开，未汇总）
SELECT *+ rule *
        distinct e.owner, e.segment_name, e.segment_type, dbablk,b.tch
          FROM dba_extents e,
               (SELECT *
                  FROM (SELECT addr, ts#, file#, dbarfil, dbablk, tch
                          FROM x$bh
                         ORDER BY tch DESC)
                 WHERE ROWNUM <= 10) b
         WHERE e.relative_fno = b.dbarfil
           AND e.block_id <= b.dbablk
           AND e.block_id + e.blocks > b.dbablk
order by tch desc
;
*/

prompt <p>附录：查看session_cached_cursors的参数设置情况，如果使用率为100%则增大这个参数值
SELECT 'session_cached_cursors' PARAMETER,  
         LPAD(VALUE, 5) VALUE,  
         DECODE(VALUE, 0, ' n/a', TO_CHAR(100 * USED / VALUE, '990') || '%') USAGE  
   FROM (SELECT MAX(S.VALUE) USED  
            FROM V$STATNAME N, V$SESSTAT S  
           WHERE N.NAME = 'session cursor cache count'  
             AND S.STATISTIC# = N.STATISTIC#),  
         (SELECT VALUE FROM V$PARAMETER WHERE NAME = 'session_cached_cursors')  
  UNION ALL  
SELECT 'open_cursors',  
         LPAD(VALUE, 5),  
         TO_CHAR(100 * USED / VALUE, '990') || '%'  
   FROM (SELECT MAX(SUM(S.VALUE)) USED  
            FROM V$STATNAME N, V$SESSTAT S  
           WHERE N.NAME IN  
                 ('opened cursors current', 'session cursor cache count')  
             AND S.STATISTIC# = N.STATISTIC#  
           GROUP BY S.SID),  
         (SELECT VALUE FROM V$PARAMETER WHERE NAME = 'open_cursors');  


prompt <p>附录：供参考的Oracle所有参数
show parameter

set markup html off
exec dbms_output.put_line('</body>');
exec dbms_output.put_line('</html>');

spool off


/* 获取awr、addm、ash */
--以下不使用html标签
SET markup html off spool ON pre off entmap off

set trim on
set trimspool on
set heading off

--查询dbid、instance_number
column dbid new_value awr_dbid
column instance_number new_value awr_inst_num
select dbid from v$database;
select instance_number from v$instance;

--半小时内的ash报告
column ashbegintime new_value ashbegin_str
column ashendtime new_value ashend_str
select to_char(sysdate-3/144,'yyyymmddhh24miss') as ashbegintime, to_char(sysdate,'yyyymmddhh24miss') as ashendtime from dual;

column ashfile_name new_value ashfile
select 'ashrpt_' || to_char(&&awr_inst_num) || '_' || to_char(&&ashbegin_str) || '_' || to_char(&&ashend_str) ashfile_name from dual;
spool &&ashfile..html
select * from table(dbms_workload_repository.ash_report_html(to_char(&&awr_dbid),to_char(&&awr_inst_num),to_date(to_char(&&ashbegin_str),'yyyymmddhh24miss'),to_date(to_char(&&ashend_str),'yyyymmddhh24miss')));
spool off;

--按需创建awr断点
column begin_snap new_value awr_begin_snap
column end_snap new_value awr_end_snap
select max(snap_id) begin_snap
  from dba_hist_snapshot
 where snap_id < (select max(snap_id) from dba_hist_snapshot);
select max(snap_id) end_snap from dba_hist_snapshot;
declare
  snap_maxtime date;
  snap_mintime date;
begin
  select max(end_interval_time) + 0
    into snap_maxtime
    from dba_hist_snapshot
   where snap_id = to_number(&&awr_end_snap);
  select max(end_interval_time) + 0
    into snap_mintime
    from dba_hist_snapshot
   where snap_id = to_number(&&awr_begin_snap);
  if sysdate - snap_maxtime > 10/1445 then
    dbms_workload_repository.create_snapshot();
  end if;
end;
/

--最新两次snap_id间的awr报告
column begin_snap new_value awr_begin_snap
column end_snap new_value awr_end_snap
select max(snap_id) begin_snap
  from dba_hist_snapshot
 where snap_id < (select max(snap_id) from dba_hist_snapshot);
select max(snap_id) end_snap from dba_hist_snapshot;
column awrfile_name new_value awrfile
select 'awrrpt_' || to_char(&&awr_inst_num) || '_' || to_char(&&awr_begin_snap) || '_' || to_char(&&awr_end_snap) awrfile_name from dual;

spool &&awrfile..html
select output from table(dbms_workload_repository.awr_report_html(&&awr_dbid,&&awr_inst_num,&&awr_begin_snap,&&awr_end_snap));
spool off;


--可获取的最长awr报告(一周以来的所有分析)
column begin_snap new_value awr_begin_snap
column end_snap new_value awr_end_snap
select a.begin_snap, a.end_snap
  from (select startup_time, min(snap_id) begin_snap, max(snap_id) end_snap
          from dba_hist_snapshot
         group by startup_time) a,
       (select max(startup_time) startup_time from dba_hist_snapshot) b
 where a.startup_time = b.startup_time
   and rownum = 1;
column awrfile_name new_value awrfile
select 'awrrpt_' || to_char(&&awr_inst_num) || '_' || to_char(&&awr_begin_snap) || '_' || to_char(&&awr_end_snap) ||'_all' awrfile_name from dual;

spool &&awrfile..html
select output from table(dbms_workload_repository.awr_report_html(&&awr_dbid,&&awr_inst_num,&&awr_begin_snap,&&awr_end_snap));
spool off;


--最新addm报告
column addmfile_name new_value addmfile
select 'addmrpt_' || to_char(&&awr_inst_num) || '_' || to_char(&&awr_begin_snap) || '_' || to_char(&&awr_end_snap) addmfile_name from dual;
set serveroutput on
spool &&addmfile..txt
declare
  id          number;
  name		  varchar2(200) := '';
  descr       varchar2(500) := '';
  addmrpt     clob;
  v_ErrorCode number;
BEGIN
  name := '&&addmfile';
  begin
    dbms_advisor.create_task('ADDM', id, name, descr, null);
    dbms_advisor.set_task_parameter(name, 'START_SNAPSHOT', &&awr_begin_snap);
    dbms_advisor.set_task_parameter(name, 'END_SNAPSHOT', &&awr_end_snap);
    dbms_advisor.set_task_parameter(name, 'INSTANCE', &&awr_inst_num);
    dbms_advisor.set_task_parameter(name, 'DB_ID', &&awr_dbid);
    dbms_advisor.execute_task(name);
  exception
    when others then
      null;
  end;
  select dbms_advisor.get_task_report(name, 'TEXT', 'TYPICAL')
    into addmrpt
    from sys.dual;
  dbms_output.enable(20000000000);
  for i in 1 .. (DBMS_LOB.GETLENGTH(addmrpt) / 2000 + 1) loop
    dbms_output.put_line(substr(addmrpt, 1900 * (i - 1) + 1, 1900));
  end loop;
  dbms_output.put_line('');
  begin
    dbms_advisor.delete_task(name);
  exception
    when others then
      null;
  end;
end;
/
spool off;

exit;
```

