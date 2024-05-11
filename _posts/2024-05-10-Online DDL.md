---
layout: post
title:  关于Online DDL
categories: [mysql]
tags: [mysql]
---



# Online DDL的发展史

在早期的MySQL版本中,DDL操作通常需要对数据表加锁，操作过程中DML操作都会被阻塞，影响正常业务。MySQL5.6和MariaDB 10.0开始支持Online DDL，可以在支持DDL操作的同时，不影响DML的正常执行，线上直接执行DDL操作对用户基本无感知（部分操作会对性能有影响）。

- MySQL 5.6之前
MySQL的DDL操作会按照原来的表复制一份，并做相应的修改：
1. 按照表A的定义新建一个表B
2. 对表A加锁
3. 在表B上执行DDL指定的操作
4. 将表A的数据拷贝到B
5. 释放A的写锁
6. 删除表A
7. 将表B重命名为A


- MySQL 5.6
官方开始支持更多的ALTER TABLE类型操作来避免数据拷贝，同时支持了在线上DDL的过程中不阻塞DML操作，真正意义上实现了Online DDL。然而并不是所有的DDL操作都支持在线操作。

- MySQL 5.7
在5.6的基础上又增加了新的特性，比如：增加了重命名索引支持，支持数值类型长度的增大和减少，支持了VARCHAR类型的在线增大等。基本逻辑和限制条件相比5.6并没有大的变化。

- MySQL 8.0对DDL的实现重新进行了设计
DDL操作支持了原子特性。另外，Online DDL的ALGORITHM参数增加了一个新的选项：INSTANT，只需要修改数据字典中的元数据，无需拷贝数据也无需重建表，同样也无需加排他MDL锁，原表数据也不受影响。整个DDL过程几乎瞬间完成的，也不会阻塞DML。

# Online DDL的算法

## Copy算法（之前DDL的算法）

1. 按照原表定义创建一个新的临时表
2. 对原表加写锁（禁止DML，允许select）
3. 对临时表进行DDL
4. 将原表数据copy到临时表
5. 释放原表的写锁
6. 将原表删除，并将临时表重命名为原表

![2024-5-1111:11:39-1715397098637.png](https://gitee.com/grsswh/drawing-bed/raw/master/image/2024-5-1111:11:39-1715397098637.png)

## Inplace算法

![2024-5-1111:12:50-1715397170155.png](https://gitee.com/grsswh/drawing-bed/raw/master/image/2024-5-1111:12:50-1715397170155.png)



在原表上进行更改，不需要生成临时表，不需要进行数据copy的过程。根据是否变更行记录格式，分为两类：
- 1. rebuild：需要重建表（重新组织聚簇索引）。比如optimize table、添加索引、添加/删除列、修改列NULL/NOT NULL属性等；
- 2. no-rebuild：不需要重建表，只需要修改表的元数据，比如删除索引、修改列名、修改列默认值、修改列自增值等。

> 对于rebuild方式实现Online 是通过缓存DDL期间的DML，待DDL完成之后，将DML应用到表上来实现的。





## 说明

1. 在copy数据到新表期间，在原表上是加的MDL读锁（允许DML，禁止DDL）
2. 在应用增量期间对原表加MDL写锁（禁止DML和DDL）
3. 根据表A重建出来的数据是放在tmp_file里的，这个临时文件是InnoDB在内部创建出来的，整个DDL过程都在InnoDB内部完成。对于server层来说，没有把数据挪动到临时表，是一个原地操作，这就是'inplace'名称的来源。

### MySQL中，表级别的锁有2种

一种是我们通常说的表锁，由InnoDB引擎实现，如lock tables...
read/write,表锁影响较大，不常用。另一种表级别的锁是MDL（metadata lock），由server层实现，MDL我们不显示使用，是在访问数据库时由数据库自动加的，对表记录增删改查时，加MDL读锁；对表结构进行变更时，加MDL写锁。MDL锁，读读不互斥，读写、写写互斥。

### 哪些常用操作“锁表”

创建二级索引、删除索引、重命名索引、改变索引类型--不“锁表”。
添加字段、删除字段、重命名字段、调整字段顺序、设置字段默认值、删除字段默认值、修改auto-increment值、调整字段允许NULL、调整字段不允许Null ---不“锁表”。
扩展Varchar字段大小---不锁表。
更改字段数据类型，如varchar改成text---锁表。

# Online DDL过程中的锁

默认情况下，MySQL就是支持Online的DDL操作的，在online DDL语句执行过程中，MySQL会尽量少使用锁的限制，我们不需要特殊的操作来启用它。
MySQL在选择的时候，尽量少使用锁，但不排除它会使用锁。而如果我们担心它使用了锁而导致我们不能读也不能写，显然这不是我们想要的结果，我们希望：如果选择锁，就不要执行，直接退出执行；如果没有选择锁就执行。

可以在执行我们的Online DDL语句时，使用Algorithm和lock关键字，这两个关键字在我们DDL语句的最后面，用逗号隔开即可。
``` sql
alter table tabl_name add column col_name col_type, algorithm=inplace,lock=none;
```

## algorithm的选项

- inplace：替换：直接在原表上面执行DDL的操作。
- copy：复制：使用一种临时表的方式，克隆出一个临时表，在临时表上执行DDL，然后再把数据导入到临时表中，在重命名等。这期间需要多出一倍的磁盘空间来支撑这样的操作。执行期间，表不允许DML的操作。
- default：默认方式，由MySQL自己选择，优先使用inplace的方式。

## lock选项

- share：共享锁，执行DDL的表可以读，但是不可以写。
- None：没有任何限制，执行DDL的表可读可写。
- exclusive：排它锁，执行DDL的表不可以读也不可以写。
- default：默认值，由MySQL来决定是否锁表。不建议使用，如果你确定你的DDL语句不会锁表，你可以不指定lock或者指定它的值为default，否则建议指定它的锁类型。

执行DDL操作时，algorithm选项可以不指定，这时候MySQL按照instant、inplace、copy的顺序自动选择合适的模式。也可以指定algorithm=default，也是同样的效果。如果指定algorithm选项，但不支持的话，会直接报错。


> 在执行OnlineDDL之前，要在非业务高峰期去执行，并要确认待执行的表上面没有未提交的事务、锁等信息。可以通过如下的SQL语句查看是否有事务和锁等信息。

``` sql
select * from information_schema.innodb_locks; //查询是否有锁
select * from information_schema.innodb_trx; //查询事务信息
select * from information_schema.innodb_lock_waits;//查询锁等待信息
select * from information_schema.processlist;//数据库连接信息
```

# MySQL 5.7的在线DDL功能特点

- 支持添加辅助索引：可以在运行中的表上添加辅助索引，而不会对整个表进行锁定。
- 支持修改列定义：可以在线修改列的数据类型、长度等定义。
- 修改字符集合排序规则：可以在线修改表的字符集和排序规则设置。
- 支持重命名列：可以在不影响正在进行的读写操作的情况下，对表中的列进行重命名。

# 实现原理和优化

1. 创建临时表：通过创建临时表来存储将要进行的DDL操作所需要的新结构。这样旧表仍可用于读写操作。
2. 数据复制和同步：将旧表的数据逐步复制到临时表中，并保持旧表数据和临时表数据的同步，这一过程保证了数据在DDL操作期间的完整性和一致性。
3. 变更捕获和重放：通过使用日志和重做日志等机制，捕获在执行DDL操作期间发生的数据变更，并将其重放到临时表中。这确保了DDL操作完成后数据的一致性。
4. 最终切换：当DDL操作完成时，数据库引擎将在适当的时机切换到临时表，使其成为新的表结构，并且对新表进行后续的读写操作。

# 使用限制和注意事项

= 并非所有DDL操作都支持在线执行，某些操作仍然需要锁定整个表。

- 在进行DDL操作期间，可能会占用较多的系统资源，因此在高负载时应谨慎使用。
- 进行在线DDL操作时，需要对操作进行充分的评估和测试，以确保数据的完整性和一致性。



# Online DDL的执行过程

![2024-5-1113:15:50-1715404549579.png](https://gitee.com/grsswh/drawing-bed/raw/master/image/2024-5-1113:15:50-1715404549579.png)



1. 初始化

服务器会根据存储引擎的能力，操作的语句和用户指定的algorithm和lock选项来决定允许多大程度的并发。在这个阶段会创建一个可升级元素的共享锁（SU）来保护表定义。



2. 执行

这个阶段会准备并执行DDL语句，根据阶段1评估的结果来决定是否将元数据锁升级为排它锁（X），如果需要升级为排它锁，则只在DDL的准备阶段短暂的添加排它锁。



3. 提交表定义

元数据锁会升级为排它锁来更新表的定义。独占排他锁的时间非常短。

> 元数据锁（MDL，metadata lock）主要用于DDL和DML操作之间的并发访问控制，保护表结构（表定义）的一致，保证读写的正确性。MDL不需要显式的使用，在访问时会自动加上。



Online DDL过程必须等待已经持有元数据锁的并发事务提交或者回滚才能继续执行。

> 当Online DDL操作正在等待元数据锁时，该元数据锁会处于挂起状态，后续的所有事务都会被阻塞。



# 评估Online DDL操作的性能



Online DDL操作的性能取决于是否发生了表的重建。在对大表执行DDL操作之前，为了避免影响正常的业务操作，最好是先评估一下DDL语句的性能再选择如何操作。



1. 复制表结构，创建一个新的表
2. 在新创建的表中插入少量数据。
3. 在新表上执行DDL操作
4. 检查执行操作后返回的rows affected 是否为0。 如果该值非0，则意味着需要拷贝表数据，此时对DDL的线上操作需要慎重考虑，周密计划。







# 各版本支持的 Online DDL语句



![2024-5-1113:49:16-1715406556009.png](https://gitee.com/grsswh/drawing-bed/raw/master/image/2024-5-1113:49:16-1715406556009.png)



## 各版本Online DDL支持情况

![2024-5-1113:57:36-1715407056279.png](https://gitee.com/grsswh/drawing-bed/raw/master/image/2024-5-1113:57:36-1715407056279.png)





![2024-5-1113:59:40-1715407180203.png](https://gitee.com/grsswh/drawing-bed/raw/master/image/2024-5-1113:59:40-1715407180203.png)



![2024-5-1114:00:05-1715407204973.png](https://gitee.com/grsswh/drawing-bed/raw/master/image/2024-5-1114:00:05-1715407204973.png)



![2024-5-1114:00:34-1715407233855.png](https://gitee.com/grsswh/drawing-bed/raw/master/image/2024-5-1114:00:34-1715407233855.png)







# DDL的执行模式

1. instant DDL 是MySQL8.0 引入的新功能，当前支持的范围较小，包括：

- 修改二级索引类型
- 新增列
- 修改列默认值
- 修改列ENUM值
- 重命名表

2. 在执行DDL操作时，MySQL内部对algorithm的选择策略：
如果用户显示指定了algorithm，那么使用用户指定的选项。
如果用户未指定，那么如果该操作支持inplace在优先选择inplace，否则选择copy。

- 目前不支持inplace的操作主要有：
删除主键
修改列数据类型
修改表字符集

3. 我们常说的Online  DDL，其实是从DML操作的角度描述的，如果DDL操作不阻塞DML操作，那个这个DDL就是Online的。目前8.0默认非Online的DDL有：
- 新增全文索引
- 新增空间索引
- 删除主键
- 修改列数据类型
- 指定表字符集
- 修改表字符集

# Q&A

## Online DDL会不会锁表？

很多MySQL用户经常在表无法正常的进行DML时就觉得是表锁了，这种说法其实是过于宽泛，实际上能够影响DML操作的锁至少包括以下几种（InnoDB）：
- MDL锁
- 表锁
- 行锁
- GAP锁
其中除了MDL锁是在Server层加的之外，其他三种都是在InnoDB层加的。所有操作都是需要先拿Server层的MDL锁，然后再去拿InnoDB层的某个需要的锁。

一个DDL的基本过程：

- 在开始进行DDL时，需要拿到对应表的MDL X锁，然后进行一系列的准备工作；
- 将MDL X锁降级为MDL S锁，进行真正的DDL操作。
- 再次将MDL S锁升级为MDL X锁，完成DDL操作，释放DML锁。

> 真正执行DDL操作期间，确实是不会“锁表”的，但如果在第一阶段拿MDL X锁时无法正常获取，那就可能真的会“锁表”。

``` sql
# session 1
select sleep(500) from mytest.t1;

# session 2 
optimize table mytest.t1;

# session 3
select * from mytest.t1;

```
session 1模拟了一个慢查询，然后session 2 可是进行DDL操作，无法拿到MDL X锁，处于等待中。此时session 3 需要执行一个查询，发现无法执行。实际上，在session 1 结束前，表t1的所有操作都无法进行了，也可以说表t1“锁表”了。MySQL 5.7/8.0可以在开启performance_schema的情况下直接查询metadata_locks表。阿里云RDS新增了L_S.MDL_INFO表，提供DML的查询。

现在回答问题

**Online DDL并不是绝对安全，更不是可以随意执行的。线上操作还是需要在业务低峰期谨慎操作。**

## 支持inplace算法的DDL一定是Online的？

inplace和Online是两个不同维度的事情。copy和inplace指的是DDL内部的执行逻辑。
copy是在server层的操作，inplace是在InnoDB层的操作。
而用户更加关心Online与否，通常只和一个问题有关：是否允许并发DML。

- copy算法执行的DDL肯定不是Online的。
- inplace算法执行的DDL不一定是Online的。

## inplace DDL需不需要额外的数据空间

MySQL内部对于DDL的algorithm有两种选择：inplace和copy（8.0新增了instant，但使用范围较小）。
copy：创建一张临时表，然后将原表的数据拷贝到临时表中，最后再用临时表替换原表。对于上面的步骤，由于需要将原表的数据拷贝到临时表中，所以肯定需要消耗额外的数据空间。

- 对于支持inplace算法的DDL，是不是不需要额外的数据空间？

需要。inplace描述的是表，而不是数据文件。只要不创建临时表，那么就都是inplace的。
实际上，很多inplace DDL都会重建表（会创建临时数据文件），所以都会需要额外的数据空间。

需要重建表的操作：
- 增加主键
- 重建主键
- 新增列（8.0支持instant DDL，不需要）
- 删除列
- 调整列顺序
- 删除列默认值
- 增加列默认值
- 修改表的row_format
- optimize表

