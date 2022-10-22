## RC RR隔离级别

### RC 可见性操作

开启两个session，并设置两个session隔离级别为RC

```
set session transaction isolation level READ COMMITTED;
```
|session 1| session 2|
|---------|----------|
| ```mysql> BEGIN;``` <br> ```Query OK, 0 rows affected (0.01 sec)```| ```mysql> BEGIN;``` <br> ```Query OK, 0 rows affected (0.01 sec)```|
|```mysql> UPDATE `tag` SET `name`='test' WHERE id = 1;``` <br> ```Query OK, 1 row affected (0.01 sec)```||
||```mysql> SELECT name FROM `tag` WHERE id =1;``` <br> ```name == ?????```|
|```mysql> COMMIT;``` <br> ```Query OK, 0 rows affected (0.01 sec)```||
||```mysql> SELECT name FROM `tag` WHERE id =1;``` ```name == test```|
||```mysql> COMMIT;``` ```Query OK, 0 rows affected (0.01 sec)```|

如上，在RC隔离级别下。当一个事务提交修改，其他事务能读到最新的修改。

### RR 可见性操作

开启两个session，并设置两个session隔离级别为RR

```
set session transaction isolation level REPEATABLE READ;
```

|session 1| session 2|
|---------|----------|
| ```mysql> BEGIN;``` <br> ```Query OK, 0 rows affected (0.01 sec)```| ```mysql> BEGIN;``` <br> ```Query OK, 0 rows affected (0.01 sec)```|
|```mysql> UPDATE `tag` SET `name`='test' WHERE id = 1;``` <br> ```Query OK, 1 row affected (0.01 sec)```||
||```mysql> SELECT name FROM `tag` WHERE id =1;``` <br> ```name == aaa```|
|```mysql> COMMIT;``` <br> ```Query OK, 0 rows affected (0.01 sec)```||
||```mysql> SELECT name FROM `tag` WHERE id =1;``` ```name == aaa```|
||```mysql> COMMIT;``` ```Query OK, 0 rows affected (0.01 sec)```|

如上，在RR隔离级别下。当一个事务提交修改，其他事务只能读到事务开始时的数据。

## MVCC

以上，在RR RC隔离级别下，不同事务读取到的数据不同。而InnDb通过MVCC(Multi-Version Concurrency Control)机制来实现。开启MVCC时，会生成Undo日志和ReadView，在这两种数据的配合下，来实现数据的隔离。

### Undo日志
在多版本的实现下，必定要存多版本的数据。undo日志就是用来存储多版本数据的快照。  

如下undo日志的结构:

*  row id：innodb隐藏的主键id
*  trx id：事务id，当有事务提交时，会拷贝完整的一行数据。并设置trx id的值为当前事务的id
*  roll pointer：回滚指针，指向拷贝源数据的地址，用于回滚操作。
*  field：表中具体定义字段数据

![undo](./img/undo.png)


### ReadView
有了Undo记录的历史数据，就要设置不同事务对那些版本数据是可见的，哪些数据是不可见的。InnDb用ReadView来记录事务开始时的事务顺序快照，并通过ReadView来判断数据的可见性。
如下ReadView结构:

 * m\_low\_limit\_id：当前系统中活跃的读写事务中最小的事务id
 * m\_ids：当前系统中活跃的读写事务id列表
 * m\_up\_limit_id：生成ReadView时，系统中应该分配给下一个事务的id值
 * m\_creator\_trx\_id：生成该ReadView的事务的事务id

![readview](./img/readview.png)

 ReadView判断逻辑如下  
 
 * Undo.trx_id < ReadView.m\_low\_limit\_id，表明生成该版本的事务在生成 ReadView 前已经提交，所以该版本可以被当前事务访问  
 * Undo.trx\_id == ReadView.m\_creator\_trx\_id，表明当前事务产生的数据，数据是可见的
 * Undo.trx\_id >= ReadView.m\_up\_limit_id，表示在生成ReadView后才产
生的数据，所以该版本数据不可见。  
 * 当Undo中trx\_id属性值，处在m\_low\_limit\_id和m\_up\_limit_id中间时，如果在m\_ids中，表明事务还在活跃，数据不可见。当不在m\_ids中时，表明事务已经提交完毕，数据是可见的

## RR RC可见性分析

设session 1的trx\_id为2。设session 2的trx\_id为3。在以上两个不同事务下，会产生如下的Undo日志  

![rr_undo](./img/rr_undo.png)

### RR 分析

![rr_readview](./img/rr_readview.png)  

RR级别下，在第一次读取时会产生如上的ReadView，遇到第一条Undo日志时，trx\_id为2，在m_ids里，表明事务还在活跃，此版本的数据不可见。找下一个版本的数据，判断数据可见。查到值为aaa。第二次查询时，依旧使用事务开始生成的ReadView，得到相同的值。
### RC 分析

![rr_readview](./img/rc_readview.png)  

RC级别下，和RR判断逻辑一致。只能读到aaa值。第二读时，会再生成新的ReadView，由于session 1事务已经提交。m_ids里的事物id去除。所以第二次读的时候，能读取到Undo日志的最新数据test。

综上所述，InnDb在RR RC下开启MVCC机制，使用Undo日志和ReadView实现数据的隔离。并且ReadView生成时机的不同，造成RR RC不同的数据隔离表现。