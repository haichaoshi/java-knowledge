## RC RR隔离级别下可见性分析

### RC 可见性分析

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

### RR 可见性分析

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
