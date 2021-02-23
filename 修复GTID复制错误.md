在基于 GTID 的复制拓扑中，要想修复从库的 SQL 线程错误，过去的 SQL_SLAVE_SKIP_COUNTER 方式不再适用。需要通过设置 gtid_next 或 gtid_purged 来完成，当然前提是已经确保主从数据一致，仅仅需要跳过复制错误让复制继续下去。    
在从库上执行以下SQL：  
```
mysql stop slave;
Query OK, 0 rows affected (0.00 sec)

mysql set gtid_next='f75ae43f-3f5e-11e7-9b98-001c4297532a20';
Query OK, 0 rows affected (0.00 sec)

mysql begin;
Query OK, 0 rows affected (0.00 sec)

mysql commit;
Query OK, 0 rows affected (0.00 sec)

mysql set gtid_next='AUTOMATIC';
Query OK, 0 rows affected (0.00 sec)

mysql start slave;
Query OK, 0 rows affected (0.02 sec)
```
其中 gtid_next 就是跳过某个执行事务，设置 gtid_next 的方法一次只能跳过一个事务，要批量的跳过事务可以通过设置 gtid_purged 完成