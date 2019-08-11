# Tablespace的管理

## 存储结构：

```SQL
  逻辑结构          物理结构
 database
    |
tablespace   ---<  datafile
    |                 |
 segment              |
    |                 |
 extent               |
    |                 ^
oracle block  ---<  OS block

```



## tablespace的空间管理：



```SQL
DMT : dictionary management tablespace
LMT : local management tablespace

select tablespace_name,extent_management from dba_tablespaces;
alter system dump datafile 7 block min 1 block max 127;
--可以看到LMT使用文件头和位图块对表空间的数据文件进行本地管理！

实验：查看操作系统块的大小
[root@lianxi ~]# du -ah 1;
4.0K    1
      查看db block
show parameter db_block_size;



实验：查看文件块的使用
create tablespace te1 datafile '/u01/app/oracle/oradata/dbtest/te1.dbf' size 88K;
select file_id,file_name from dba_data_files;
!ls -lk /u01/app/oracle/oradata/lianxi/data01.dbf   --有8K的偏移量作为文件头
alter system dump datafile 7 block min 1 block max 11; --dump在诊断目录下
找到相关的/u01/app/oracle/diag/rdbms/lianxi/lianxi/trace  跟踪文件查看
最小的数据文件大小是88K

```

