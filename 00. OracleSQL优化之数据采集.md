Oracle SQL 优化

[TOC]

# 数据采集

## awr报告关注点

```sqlite
1. DB TIME 与 elapsed * CPUs 比较：
大于，繁忙
小于，不
2. load_profile
展示当前系统的性能参数
例如： transactions / redo size 可以看出系统事务的提交频繁程度 和 事务的大小
3. efficiency percentages
命中指标
例如：library hit过低，soft parse过低；未使用绑定变量
4. top 5 events
5. SQL statistics
6. segment statistics
知道繁忙落在哪个表的段时，可以针对此对象优化
```



## ASH关注的重点

```shell
SQL和等待事件的关联：
TOP SQL with TOP EVENTS
```



## ADDM的关注点

```
Oracle系统给出的优化建议：
从FINDING 1  FINDING 2：开始阅读
```



## AWRDD的关注点

```
两次awr报告的比较，关注点通awr报告
```



## AWRSQRPT的关注点

```
查看SQL执行计划
```

## 案例分享

```
1）并行等待 PX Deq Credit:send blkd
1.有大量的不同进程之间的数据和信息的交互导致等待，原因可能是一个比较糟糕的执行计划用于了并行执行。
2.等待是由于资源的问题，如CPU或相互连接等。例如CPU利用率达到100%，进程达到了CPU的限制，而不能足够快地发送数据。
3.由于并行查询hang住，如等待事件为"PX Deq Credit: need buffer"
去掉表的并行属性，较大的任务放在业务低峰期执行


2）gc buffer busy
热点块的争用：让不同的业务跑在不同的节点上，避免两个节点访问同一对象
可以直接查看segment by global cache buffer busy定位热点对象，对其瘦身，分区等

3）log file switch 日志切换频繁
例如：循环提交

4）IO等待
定位模块 tablespace IO stats
查看 Av Rd(ms) 此参数>7说明系统有严重的IO问题！

```

