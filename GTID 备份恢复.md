在做备份恢复的时候，有时需要恢复出来的 MySQL 实例可以作为从库连上原来的主库继续复制，这就要求从备份恢复出来的 MySQL 实例拥有和主数据库数据一致的 gtid_executed 值。这也是通过设置 gtid_purged 实现的，下面看下 mysqldump 做备份的例子。

通过mysqldump在主库上做一个全量备份  
这里使用 --all-databases选项是因为基于 GTID 的复制会记录全部的事务, 所以要构建一个完整的dump
```
mysqldump --all-databases --single-transaction  --triggers --routines --events --host=127.0.0.1 --port=3306 --user=root -p000000 > dump.sql
```
生成的 dump.sql 文件里包含了设置 gtid_purged 的语句
```
SET @MYSQLDUMP_TEMP_LOG_BIN = @@SESSION.SQL_LOG_BIN;
SET @@SESSION.SQL_LOG_BIN= 0;
...
SET @@GLOBAL.GTID_PURGED='f75ae43f-3f5e-11e7-9b98-001c4297532a:1-14:20';
...
SET @@SESSION.SQL_LOG_BIN = @MYSQLDUMP_TEMP_LOG_BIN;
```
在从库恢复数据前需要先通过 reset master 清空 gtid_executed 变量
```
$ mysql -h127.0.0.1 --user=root -p000000 -e  'reset master'
$ mysql -h127.0.0.1 --user=root -p000000<dump.sql
```
否则执行设置 GTID_PURGED 的 SQL 时会报下面的错误：
```
ERROR 1840 (HY000) at line 24: @@GLOBAL.GTID_PURGED can only be set when @@GLOBAL.GTID_EXECUTED is empty.
```
由于恢复出 MySQL 实例已经被设置了正确的 GTID_EXECUTED ，下面以 master_auto_postion = 1 的方式 CHANGE MASTER 到原来的主节点即可开始复制。
```

mysql> CHANGE MASTER TO MASTER_HOST='192.168.2.210', MASTER_USER='repl', MASTER_PASSWORD='000000', MASTER_AUTO_POSITION = 1;

```
如果不希望备份文件中生成设置 GTID_PURGED 的 SQL，可以给 mysqldump 传入 --set-gtid-purged=OFF 关闭。