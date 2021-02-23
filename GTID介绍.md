# GTID的概念
```
1、全局事务标识：global transaction identifiers。
2、GTID是一个事务一一对应，并且全局唯一ID。
3、一个GTID在一个服务器上只执行一次，避免重复执行导致数据混乱或者主从不一致。
4、GTID用来代替传统复制方法，不再使用MASTER_LOG_FILE+MASTER_LOG_POS开启复制。而是使用MASTER_AUTO_POSTION=1的方式开始复制。
5、MySQL-5.6.5开始支持的，MySQL-5.6.10后开始完善。
6、在传统的slave端，binlog是不用开启的，但是在GTID中slave端的binlog是必须开启的，目的是记录执行过的GTID（强制）。
```
# GTID的组成
```
GTID = source_id:transaction_id
source_id，用于鉴别原服务器，即mysql服务器唯一的的server_uuid，由于GTID会传递到slave，所以也可以理解为源ID。
transaction_id，为当前服务器上已提交事务的一个序列号，通常从1开始自增长的序列，一个数值对应一个事务。        
示例：          
3E11FA47-71CA-11E1-9E33-C80AA9429562:23
前面的一串为服务器的server_uuid，即3E11FA47-71CA-11E1-9E33-C80AA9429562，后面的23为transaction_id
```
# GTID的优势
```
1、更简单的实现failover，不用以前那样在需要找log_file和log_pos。
2、更简单的搭建主从复制。
3、比传统的复制更加安全。
4、GTID是连续的没有空洞的，保证数据的一致性，零丢失。
```
# GTID的工作原理
```
1、当一个事务在主库端执行并提交时，产生GTID，一同记录到binlog日志中。
2、binlog传输到slave,并存储到slave的relaylog后，读取这个GTID的这个值设置gtid_next变量，即告诉Slave，下一个要执行的GTID值。
3、sql线程从relay log中获取GTID，然后对比slave端的binlog是否有该GTID。
4、如果有记录，说明该GTID的事务已经执行，slave会忽略。
5、如果没有记录，slave就会执行该GTID事务，并记录该GTID到自身的binlog，
   在读取执行事务前会先检查其他session持有该GTID，确保不被重复执行。
6、在解析过程中会判断是否有主键，如果没有就用二级索引，如果没有就用全部扫描
```