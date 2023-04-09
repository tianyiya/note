## join流程

```text
select * from t1 straight_join t2 on (t1.a=t2.b);
```

### 被驱动表字段有索引

```text
从表 t1 中读入一行数据 R；

从数据行 R 中，取出 a 字段到表 t2 里去查找；

取出表 t2 中满足条件的行，跟 R 组成一行，作为结果集的一部分；

重复执行步骤 1 到 3，直到表 t1 的末尾循环结束。
```

### 被驱动表字段无索引

```text
把表 t1 的数据读入线程内存 join_buffer 中，由于我们这个语句中写的是 select *，因此是把整个表 t1 放入了内存；

扫描表 t2，把表 t2 中的每一行取出来，跟 join_buffer 中的数据做对比，满足 join 条件的，作为结果集的一部分返回。
```

## 优化

### 使用小表作为驱动表

在决定哪个表做驱动表的时候，应该是两个表按照各自的条件过滤，过滤完成之后，

计算参与 join 的各个字段的总数据量，数据量小的那个表，就是“小表”，应该作为驱动表

### MRR优化

查询语句在索引上做的是一个范围查询，可以得到足够多的主键 id。这样通过排序以后，再去主键索引查数据，才能体现出“顺序性”的优势

### 被驱动表上有索引

从驱动表 t1，一行行地取出 a 的值，放入join_buffer，再到被驱动表 t2 去做 join

### 被驱动表上无索引

如果一个使用 BNL 算法的 join 语句，多次扫描一个冷表，而且这个语句执行时间超过 1 秒，

就会在再次扫描冷表的时候，把冷表的数据页移到 LRU 链表头部

#### 建索引

#### 建临时表


```text
create temporary table temp_t(id int primary key, a int, b int, index(b))engine=innodb;
insert into temp_t select * from t2 where b>=1 and b<=2000;
select * from t1 join temp_t on (t1.b=temp_t.b);
```