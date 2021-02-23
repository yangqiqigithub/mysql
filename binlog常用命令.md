### 查看是否开启binlog
```
mysql> show variables like '%bin%';
+-----------------------------------------+----------------------+
| Variable_name                           | Value                |
+-----------------------------------------+----------------------+
| binlog_cache_size                       | 32768                |
| binlog_direct_non_transactional_updates | OFF                  |
| binlog_format                           | MIXED                |
| binlog_stmt_cache_size                  | 32768                |
| innodb_locks_unsafe_for_binlog          | OFF                  |
| log_bin                                 | ON ##开了                   |
| log_bin_trust_function_creators         | OFF                  |
| max_binlog_cache_size                   | 18446744073709547520 |
| max_binlog_size                         | 1073741824           |
| max_binlog_stmt_cache_size              | 18446744073709547520 |
| sql_log_bin                             | ON                   |
| sync_binlog                             | 0                    |
+-----------------------------------------+----------------------+
12 rows in set (0.00 sec)
```
###  开启binlog的方式
```
在my.cnf

[mysqld]
log-bin=mysql-bin
binlog_format=mixed
expire_logs_days = 30 #设置过期自动回收时间

#以下可选
log_bin_basename=/var/lib/mysql/mysql-bin  
log_bin_index=/var/lib/mysql/mysql-bin.index  

#log日志默认会在数据目录里
```
### 删除binlog
```
#删除5天前的binlog日志
mysql> PURGE MASTER LOGS BEFORE DATE_SUB( NOW( ),INTERVAL 5 DAY);
Query OK, 0 rows affected (0.01 sec)

#重置日志
mysql> reset master;
Query OK, 0 rows affected (0.02 sec)

#注意：手动清理的时候注意一些，如果有主从同步，还没来得及同步的日志被清理后，主动同步会造成数据 不一致
可以通过 
  show slave status\G;   #检查从服务器正在读取哪个日志，有多个从服务器，选择时间最早的一个做为目标日志。
```
### mysqlbinlog命令
##### 查看binlog文件里的内容
```
mysqlbinlog  mysql-bin.000004
#
[xxx@dbhost log]$ mysqlbinlog mysql-bin.000004
mysqlbinlog: unknown variable 'default-character-set=utf8'
出现以上错误 解决方法如下：
mysqlbinlog --no-defaults mysql-bin.000004
```
##### 将binlog转换为可用的sql
常规的将binlog日志里的所有内容转换为sql文件
```
mysqlbinlog mysql-bin.000005 > bin.sql
```
将指定库对应的binlog内容提取出来
```
# -d 指定数据库
Mysqlbinlog –d test mysql-bin.000005 > bin.sql
```
指定位置点 提取内容
```
Mysqlbinlog mysql-bin.000020 –start-position=365 –stop-position=456 > bin.sql
```
指定时间点 提取内容
```
Mysqlbinlog mysql-bin.000020 –start-datetime=’2017-01-01 17:18:00’ –stop-datetime=’2017-01-01 18:45:00’  > bin.sql
```