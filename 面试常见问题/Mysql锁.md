Mysql锁



1.表锁



常见 ddl索引创建会触发表锁或显示表锁LOCK TABLES



一般会问生产中是否遇到死锁表的情况？

当一个事务正在对某个表进行写操作（如 `INSERT`、`UPDATE`、`DELETE`），而另一个事务想要对该表进行写操作时，就会产生表级锁等待的情况。如果两个事务的操作发生了冲突，MySQL 将会阻塞其中一个事务，直到另一个事务释放了表级锁

一般就是由于这种情况导致



2.行锁

对表中的行进行加锁，使得其他事务无法修改或删除该行。常见的行锁有共享锁（Shared Lock）和排他锁（Exclusive Lock）。

SELECT * FROM table_name WHERE id = 1 FOR UPDATE; -- 排他锁，锁定id=1的行 SELECT * FROM table_name WHERE id = 1 LOCK IN SHARE MODE; -- 共享锁，锁定id=1的行

3.间隙锁

触发条件 范围查询 SELECT * FROM table_name WHERE id BETWEEN 10 AND 20 FOR UPDATE;

​		唯一索引插入 INSERT INTO table_name (id, name) VALUES (15, 'John');

MySQL会对id为15的记录以及id为14和16之间的间隙进行加锁

为了保证数据的一致性和唯一性会对相邻间隙加锁

间隙锁属于悲观锁范畴







mysql数据结构

Oracle、MySQL 和 Doris都为b-tree数据结构

数据都在叶子节点



mysql事务日志

innodb_flush_log_at_trx_commit默认为1，不可以根据单表设置，属于全局配置

默认为1 性能最差，最多丢一条持久化数据

0 性能最好，但是会丢失内存内事务日志

2每秒策略，介于1-0之间的性能





常见问题？

索引级别

索引如何优化

事务隔离级别

如何进行分表

死锁如何解决

非自增的uuid造成的页面分裂问题



