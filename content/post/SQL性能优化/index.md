---
title: "SQL性能优化"
description: Markdown语法大全

image:
math: false
license: false
hidden: false
comments: true
toc: true
style:
  color: red
keywords:
redingTime: true
---

# SQL性能优化
## 对于宽表大字段排序可能导致RowID排序的问题
### 问题原因
```sql
SELECT *
FROM article
WHERE city = 'Beijing'
ORDER BY age
LIMIT 100;
```
上述sql就算`city`和`age`都命中索引，如果select * 引入了大字段 `varchar(5000)`，会导致MySQL的Sort Buffer空间溢出，单行数据的总长度超过了`max_length_for_sort_data`，优化器会判定内存缓存容量过小，无法容纳完整的宽行数据进行排序，会选择TwoPass算法导致二次回表（只排ID + Age）只将主键ID和排序键Age载入缓冲区，排序只有一串有序的ID，数据的不完整导致二次读取，在物理层面这些ID对应的数据行往往是离散分布在不同咯的扇区，随机读取会让磁盘IOPS瞬间饱和
### 解决方案
延迟join
```sql
SELECT a.*
FROM Article a
         JOIN (SELECT id FROM Article WHERE city = 'BJ' ORDER BY age LIMIT 100) b ON a.id = b.id;
```
由于子查询只涉及ID和排序键，直接在索引树上就能完成扫描和排序，拿到id数据再通过JOIN操作回表获取完整数据，此时参与回表的数据量已经呗严格限制在分页范围内，且不再涉及排序开销，通过最小化回表数据集，将IO降到最低  