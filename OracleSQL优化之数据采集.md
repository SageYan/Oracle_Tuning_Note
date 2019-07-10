Oracle SQL 优化

[TOC]

# 数据采集

* awr报告关注点

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

  

* ASH关注的重点

  ```shell
  SQL和等待事件的关联：
  TOP SQL with TOP EVENTS
  ```

  

* ADDM的关注点

  ```
  Oracle系统给出的优化建议：
  从FINDING 1 ：FINDING 2：开始阅读
  ```

  