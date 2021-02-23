# 环境
percona-xtrabackup-24-2.4.9   
mysql5.7

当前数据库的数据如下：
```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test1              |
+--------------------+
5 rows in set (0.00 sec)

mysql> select * from test1.t1;
+------+-------+
| id   | count |
+------+-------+
|    1 |     1 |
+------+-------+
1 row in set (0.00 sec)
```
# 备份
### 做一次全备
```
[root@m backup]# /usr/bin/innobackupex --defaults-file=/app/mysql/3306/conf/my.cnf --socket=/app/mysql/3306/mysql.sock --use-memory=1G --host=127.0.0.1 --user=root --password=123@qwe --port=3306 /backup/full/
210223 10:46:22 innobackupex: Starting the backup operation

IMPORTANT: Please check that the backup run completes successfully.
           At the end of a successful backup run innobackupex
           prints "completed OK!".
...
210223 10:46:23 [01] Copying ./sys/sys_config.ibd to /backup/full/2021-02-23_10-46-22/sys/sys_config.ibd
210223 10:46:23 [01]        ...done
...
210223 10:46:25 [01]        ...done
210223 10:46:25 Finished backing up non-InnoDB tables and files
210223 10:46:25 [00] Writing /backup/full/2021-02-23_10-46-22/xtrabackup_binlog_info
210223 10:46:25 [00]        ...done
210223 10:46:25 Executing FLUSH NO_WRITE_TO_BINLOG ENGINE LOGS...
xtrabackup: The latest check point (for incremental): '2826806'
xtrabackup: Stopping log copying thread.
.210223 10:46:25 >> log scanned up to (2826815)

210223 10:46:25 Executing UNLOCK TABLES
210223 10:46:25 All tables unlocked
210223 10:46:25 [00] Copying ib_buffer_pool to /backup/full/2021-02-23_10-46-22/ib_buffer_pool
210223 10:46:25 [00]        ...done
210223 10:46:25 Backup created in directory '/backup/full/2021-02-23_10-46-22/'
MySQL binlog position: filename 'mysql-bin.000007', position '1763', GTID of the last change '5da251ea-74c1-11eb-8aac-fa163e95ab6d:1-21,
722e4aaa-7579-11eb-b5d9-fa163e95ab6d:1-8,
8deb070d-74c1-11eb-90e7-fa163e95ab6d:1-8'
210223 10:46:25 [00] Writing /backup/full/2021-02-23_10-46-22/backup-my.cnf
210223 10:46:25 [00]        ...done
210223 10:46:25 [00] Writing /backup/full/2021-02-23_10-46-22/xtrabackup_info
210223 10:46:25 [00]        ...done
xtrabackup: Transaction log of lsn (2826806) to (2826815) was copied.
210223 10:46:25 completed OK!
```
备份完毕后的目录内容如下：  
```
[root@m 2021-02-23_10-46-22]# pwd
/backup/full/2021-02-23_10-46-22
[root@m 2021-02-23_10-46-22]# ls
backup-my.cnf   mysql               test1                   xtrabackup_info
ib_buffer_pool  performance_schema  xtrabackup_binlog_info  xtrabackup_logfile
ibdata1         sys                 xtrabackup_checkpoints
```
### 添加数据库数据
```
mysql> create database test2;
Query OK, 1 row affected (0.00 sec)

mysql> use test2
Database changed
mysql> CREATE TABLE `t2` (`id` int(11) DEFAULT NULL,`count` int(11) DEFAULT NULL);
Query OK, 0 rows affected (0.03 sec)

mysql> insert into t2 values(2,2);
Query OK, 1 row affected (0.01 sec)

mysql> select * from test2.t2;
+------+-------+
| id   | count |
+------+-------+
|    2 |     2 |
+------+-------+
1 row in set (0.00 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test1              |
| test2              |
+--------------------+
6 rows in set (0.00 sec)
```
### 做一次增备
```
[root@m backup]# /usr/bin/innobackupex --defaults-file=/app/mysql/3306/conf/my.cnf --incremental-basedir=/backup/full/2021-02-23_10-46-22 --socket=/app/mysql/3306/mysql.sock --use-memory=1G --user=root --password=123@qwe --port=3306 --incremental /backup/ince
210223 10:54:54 innobackupex: Starting the backup operation

IMPORTANT: Please check that the backup run completes successfully.
           At the end of a successful backup run innobackupex
           prints "completed OK!".
...
210223 10:54:54 >> log scanned up to (2838307)
xtrabackup: Generating a list of tablespaces
InnoDB: Allocated tablespace ID 21 for sys/sys_config, old maximum was 0
xtrabackup: using the full scan for incremental backup
210223 10:54:54 [01] Copying ./ibdata1 to /backup/ince/2021-02-23_10-54-54/ibdata1.delta
210223 10:54:54 [01]        ...done
210223 10:54:54 [01] Copying ./sys/sys_config.ibd to /backup/ince/2021-02-23_10-54-54/sys/sys_config.ibd.delta
210223 10:54:54 [01]        ...done
210223 10:54:54 [01] Copying ./test1/t1.ibd to /backup/ince/2021-02-23_10-54-54/test1/t1.ibd.delta
210223 10:54:54 [01]        ...done
210223 10:54:54 [01] Copying ./test2/t2.ibd to /backup/ince/2021-02-23_10-54-54/test2/t2.ibd.delta
...
210223 10:54:56 Executing UNLOCK TABLES
210223 10:54:56 All tables unlocked
210223 10:54:56 [00] Copying ib_buffer_pool to /backup/ince/2021-02-23_10-54-54/ib_buffer_pool
210223 10:54:56 [00]        ...done
210223 10:54:56 Backup created in directory '/backup/ince/2021-02-23_10-54-54/'
MySQL binlog position: filename 'mysql-bin.000007', position '3371', GTID of the last change '5da251ea-74c1-11eb-8aac-fa163e95ab6d:1-21,
722e4aaa-7579-11eb-b5d9-fa163e95ab6d:1-16,
8deb070d-74c1-11eb-90e7-fa163e95ab6d:1-8'
210223 10:54:56 [00] Writing /backup/ince/2021-02-23_10-54-54/backup-my.cnf
210223 10:54:56 [00]        ...done
210223 10:54:56 [00] Writing /backup/ince/2021-02-23_10-54-54/xtrabackup_info
210223 10:54:56 [00]        ...done
xtrabackup: Transaction log of lsn (2838298) to (2838307) was copied.
210223 10:54:56 completed OK!
[root@m backup]# 
```
备份完的目录内容如下：
```
[root@m 2021-02-23_10-54-54]# pwd
/backup/ince/2021-02-23_10-54-54
[root@m 2021-02-23_10-54-54]# ls
backup-my.cnf   ibdata1.meta        sys    xtrabackup_binlog_info  xtrabackup_logfile
ib_buffer_pool  mysql               test1  xtrabackup_checkpoints
ibdata1.delta   performance_schema  test2  xtrabackup_info
```
### 继续添加数据内容
```
mysql> create database test3;
Query OK, 1 row affected (0.01 sec)

mysql> use test3;
Database changed
mysql> CREATE TABLE `t3` (`id` int(11) DEFAULT NULL,`count` int(11) DEFAULT NULL);
Query OK, 0 rows affected (0.05 sec)

mysql> insert into t3 values(3,3);
Query OK, 1 row affected (0.01 sec)

mysql> select * from test3.t3;
+------+-------+
| id   | count |
+------+-------+
|    3 |     3 |
+------+-------+
1 row in set (0.00 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test1              |
| test2              |
| test3              |
+--------------------+
7 rows in set (0.00 sec)
```
### 再做一次增备
注意这次增备是在上次增备的基础上做的，所以
--incremental-basedir=/backup/ince/2021-02-23_10-54-54 指定的是上次增备的目录
```
[root@m 2021-02-23_10-54-54]# /usr/bin/innobackupex --defaults-file=/app/mysql/3306/conf/my.cnf --incremental-basedir=/backup/ince/2021-02-23_10-54-54 --socket=/app/mysql/3306/mysql.sock --use-memory=1G --user=root --password=123@qwe --port=3306 --incremental /backup/ince/
210223 11:00:55 innobackupex: Starting the backup operation


IMPORTANT: Please check that the backup run completes successfully.
           At the end of a successful backup run innobackupex
           prints "completed OK!".
...
210223 11:00:56 >> log scanned up to (2842681)
xtrabackup: Generating a list of tablespaces
InnoDB: Allocated tablespace ID 21 for sys/sys_config, old maximum was 0
xtrabackup: using the full scan for incremental backup
210223 11:00:56 [01] Copying ./ibdata1 to /backup/ince/2021-02-23_11-00-55/ibdata1.delta
210223 11:00:56 [01]        ...done
210223 11:00:56 [01] Copying ./sys/sys_config.ibd to /backup/ince/2021-02-23_11-00-55/sys/sys_config.ibd.delta
...
210223 11:00:59 Executing UNLOCK TABLES
210223 11:00:59 All tables unlocked
210223 11:00:59 [00] Copying ib_buffer_pool to /backup/ince/2021-02-23_11-00-55/ib_buffer_pool
210223 11:00:59 [00]        ...done
210223 11:00:59 Backup created in directory '/backup/ince/2021-02-23_11-00-55/'
MySQL binlog position: filename 'mysql-bin.000007', position '4008', GTID of the last change '5da251ea-74c1-11eb-8aac-fa163e95ab6d:1-21,
722e4aaa-7579-11eb-b5d9-fa163e95ab6d:1-19,
8deb070d-74c1-11eb-90e7-fa163e95ab6d:1-8'
210223 11:00:59 [00] Writing /backup/ince/2021-02-23_11-00-55/backup-my.cnf
210223 11:00:59 [00]        ...done
210223 11:00:59 [00] Writing /backup/ince/2021-02-23_11-00-55/xtrabackup_info
210223 11:00:59 [00]        ...done
xtrabackup: Transaction log of lsn (2842672) to (2842681) was copied.
210223 11:00:59 completed OK!
```
备份完毕后目录内容如下：
```
[root@m 2021-02-23_11-06-25]# pwd
/backup/ince/2021-02-23_11-06-25
[root@m 2021-02-23_11-06-25]# ls
backup-my.cnf   ibdata1.meta        sys    test3                   xtrabackup_info
ib_buffer_pool  mysql               test1  xtrabackup_binlog_info  xtrabackup_logfile
ibdata1.delta   performance_schema  test2  xtrabackup_checkpoints
```
# 恢复
### 先对全备进行一次redo-only
```
[root@m backup]# /usr/bin/innobackupex --apply-log --use-memory=4G --redo-only /backup/full/2021-02-23_10-46-22/
210223 11:08:32 innobackupex: Starting the apply-log operation

IMPORTANT: Please check that the apply-log run completes successfully.
           At the end of a successful apply-log run innobackupex
           prints "completed OK!".

/usr/bin/innobackupex version 2.4.9 based on MySQL server 5.7.13 Linux (x86_64) (revision id: a467167cdd4)
xtrabackup: cd to /backup/full/2021-02-23_10-46-22/
xtrabackup: This target seems to be not prepared yet.
InnoDB: Number of pools: 1
xtrabackup: xtrabackup_logfile detected: size=8388608, start_lsn=(2826806)
xtrabackup: using the following InnoDB configuration for recovery:
xtrabackup:   innodb_data_home_dir = .
xtrabackup:   innodb_data_file_path = ibdata1:12M:autoextend
xtrabackup:   innodb_log_group_home_dir = .
xtrabackup:   innodb_log_files_in_group = 1
xtrabackup:   innodb_log_file_size = 8388608
xtrabackup: using the following InnoDB configuration for recovery:
xtrabackup:   innodb_data_home_dir = .
xtrabackup:   innodb_data_file_path = ibdata1:12M:autoextend
xtrabackup:   innodb_log_group_home_dir = .
xtrabackup:   innodb_log_files_in_group = 1
xtrabackup:   innodb_log_file_size = 8388608
xtrabackup: Starting InnoDB instance for recovery.
xtrabackup: Using 4294967296 bytes for buffer pool (set by --use-memory parameter)
InnoDB: PUNCH HOLE support available
InnoDB: Mutexes and rw_locks use GCC atomic builtins
InnoDB: Uses event mutexes
InnoDB: GCC builtin __atomic_thread_fence() is used for memory barrier
InnoDB: Compressed tables use zlib 1.2.7
InnoDB: Number of pools: 1
InnoDB: Using CPU crc32 instructions
InnoDB: Initializing buffer pool, total size = 4G, instances = 1, chunk size = 128M
InnoDB: Completed initialization of buffer pool
InnoDB: page_cleaner coordinator priority: -20
InnoDB: Highest supported file format is Barracuda.
InnoDB: Log scan progressed past the checkpoint lsn 2826806
InnoDB: Doing recovery: scanned up to log sequence number 2826815 (0%)
InnoDB: Doing recovery: scanned up to log sequence number 2826815 (0%)
InnoDB: Database was not shutdown normally!
InnoDB: Starting crash recovery.
InnoDB: xtrabackup: Last MySQL binlog file position 1763, file name mysql-bin.000007
InnoDB: xtrabackup: Last MySQL binlog file position 1763, file name mysql-bin.000007

xtrabackup: starting shutdown with innodb_fast_shutdown = 1
InnoDB: Starting shutdown...
InnoDB: Shutdown completed; log sequence number 2826824
InnoDB: Number of pools: 1
210223 11:08:33 completed OK!
```
### 再分别对增倍做一次redo-only 
==(确保不是最后一次增倍，针对最后一次增备不加redo-only参数)==
```
[root@m 2021-02-23_10-54-54]# /usr/bin/innobackupex --apply-log --use-memory=4G --redo-only /backup/full/2021-02-23_10-46-22/ --incremental-dir=/backup/ince/2021-02-23_10-54-54
210223 11:10:26 innobackupex: Starting the apply-log operation

IMPORTANT: Please check that the apply-log run completes successfully.
           At the end of a successful apply-log run innobackupex
           prints "completed OK!".

/usr/bin/innobackupex version 2.4.9 based on MySQL server 5.7.13 Linux (x86_64) (revision id: a467167cdd4)
incremental backup from 2826806 is enabled.
xtrabackup: cd to /backup/full/2021-02-23_10-46-22/
xtrabackup: This target seems to be already prepared with --apply-log-only.
InnoDB: Number of pools: 1
xtrabackup: xtrabackup_logfile detected: size=8388608, start_lsn=(2838298)
xtrabackup: using the following InnoDB configuration for recovery:
xtrabackup:   innodb_data_home_dir = .
xtrabackup:   innodb_data_file_path = ibdata1:12M:autoextend
xtrabackup:   innodb_log_group_home_dir = /backup/ince/2021-02-23_10-54-54/
xtrabackup:   innodb_log_files_in_group = 1
xtrabackup:   innodb_log_file_size = 8388608
xtrabackup: Generating a list of tablespaces
InnoDB: Allocated tablespace ID 21 for sys/sys_config, old maximum was 0
xtrabackup: page size for /backup/ince/2021-02-23_10-54-54//ibdata1.delta is 16384 bytes
Applying /backup/ince/2021-02-23_10-54-54//ibdata1.delta to ./ibdata1...
xtrabackup: page size for /backup/ince/2021-02-23_10-54-54//sys/sys_config.ibd.delta is 16384 bytes
...
configuration for recovery:
xtrabackup:   innodb_data_home_dir = .
xtrabackup:   innodb_data_file_path = ibdata1:12M:autoextend
xtrabackup:   innodb_log_group_home_dir = /backup/ince/2021-02-23_10-54-54/
xtrabackup:   innodb_log_files_in_group = 1
xtrabackup:   innodb_log_file_size = 8388608
xtrabackup: Starting InnoDB instance for recovery.
xtrabackup: Using 4294967296 bytes for buffer pool (set by --use-memory parameter)
InnoDB: PUNCH HOLE support available
InnoDB: Mutexes and rw_locks use GCC atomic builtins
InnoDB: Uses event mutexes
InnoDB: GCC builtin __atomic_thread_fence() is used for memory barrier
InnoDB: Compressed tables use zlib 1.2.7
InnoDB: Number of pools: 1
InnoDB: Using CPU crc32 instructions
InnoDB: Initializing buffer pool, total size = 4G, instances = 1, chunk size = 128M
InnoDB: Completed initialization of buffer pool
InnoDB: page_cleaner coordinator priority: -20
InnoDB: Highest supported file format is Barracuda.
InnoDB: Log scan progressed past the checkpoint lsn 2838298
InnoDB: Doing recovery: scanned up to log sequence number 2838307 (0%)
InnoDB: Doing recovery: scanned up to log sequence number 2838307 (0%)
InnoDB: Database was not shutdown normally!
InnoDB: Starting crash recovery.
InnoDB: xtrabackup: Last MySQL binlog file position 3371, file name mysql-bin.000007
InnoDB: xtrabackup: Last MySQL binlog file position 3371, file name mysql-bin.000007

xtrabackup: starting shutdown with innodb_fast_shutdown = 1
InnoDB: Starting shutdown...
InnoDB: Shutdown completed; log sequence number 2838316
InnoDB: Number of pools: 1
210223 11:10:28 [01] Copying /backup/ince/2021-02-23_10-54-54/sys/sys_config.frm to ./sys/sys_config.frm
210223 11:10:28 [01]        ...done
210223 11:10:28 [01] Copying /backup/ince/2021-02-23_10-54-54/sys/x@0024wait_classes_global_by_avg_latency.frm to ./sys/x@0024wait_classes_global_by_avg_latency.frm
210223 11:10:28 [01]        ...done
210223 11:10:28 [01] Copying /backup/ince/2021-02-23_10-54-54/sys/x@0024user_summary_by_statement_type.frm to ./sys/x@0024user_summary_by_statement_type.frm
210223 11:10:29 [01]        ...done
210223 11:10:29 [01] Copying /backup/ince/2021-02-23_10-54-54/sys/processlist.frm to ./sys/processlist.frm
210223 11:10:29 [01]        ...done
210223 11:10:29 [01] Copying /backup/ince/2021-02-23_10-54-54/sys/x@0024waits_by_host_by_latency.frm to ./sys/x@0024waits_by_host_by_latency.frm
210223 11:10:29 [01]        ...done
210223 11:10:29 [01] Copying /backup/ince/2021-02-23_10-54-54/sys/x@0024schema_index_statistics.frm to ./sys/x@0024schema_index_statistics.frm
210223 11:10:29 [01]        ...done
210223 11:10:29 [01] Copying /backup/ince/2021-02-23_10-54-54/sys/x@0024innodb_lock_waits.frm to ./sys/x@0024innodb_lock_waits.frm
210223 11:10:29 [01]        ...done
210223 11:10:29 [01] Copying /backup/ince/2021-02-23_10-54-54/sys/x@0024session.frm to ./sys/x@0024session.frm
210223 11:10:29 [01]        ...done
210223 11:10:29 [01] Copying /backup/ince/2021-02-23_10-54-54/sys/schema_tables_with_full_table_scans.frm to ./sys/schema_tables_with_full_table_scans.frm
210223 11:10:29 [01]        ...done
210223 11:10:29 [01] Copying /backup/ince/2021-02-23_10-54-54/sys/x@0024io_by_thread_by_latency.frm to ./sys/x@0024io_by_thread_by_latency.frm
210223 11:10:29 [01]        ...done
...
210223 11:10:29 [01]        ...done
210223 11:10:29 [00] Copying /backup/ince/2021-02-23_10-54-54//xtrabackup_binlog_info to ./xtrabackup_binlog_info
210223 11:10:29 [00]        ...done
210223 11:10:29 [00] Copying /backup/ince/2021-02-23_10-54-54//xtrabackup_info to ./xtrabackup_info
210223 11:10:29 [00]        ...done
210223 11:10:29 completed OK!

```
### 最后一次的增备不需要加--redo-only
```
[root@m 2021-02-23_11-06-25]# /usr/bin/innobackupex --apply-log --use-memory=1G  /backup/full/2021-02-23_10-46-22/  --incremental-dir=/backup/ince/2021-02-23_11-06-25
210223 11:14:51 innobackupex: Starting the apply-log operation

IMPORTANT: Please check that the apply-log run completes successfully.
           At the end of a successful apply-log run innobackupex
           prints "completed OK!".

/usr/bin/innobackupex version 2.4.9 based on MySQL server 5.7.13 Linux (x86_64) (revision id: a467167cdd4)
incremental backup from 2838298 is enabled.
xtrabackup: cd to /backup/full/2021-02-23_10-46-22/
xtrabackup: This target seems to be already prepared with --apply-log-only.
InnoDB: Number of pools: 1
xtrabackup: xtrabackup_logfile detected: size=8388608, start_lsn=(2842672)
xtrabackup: using the following InnoDB configuration for recovery:
xtrabackup:   innodb_data_home_dir = .
xtrabackup:   innodb_data_file_path = ibdata1:12M:autoextend
xtrabackup:   innodb_log_group_home_dir = /backup/ince/2021-02-23_11-06-25/
xtrabackup:   innodb_log_files_in_group = 1
xtrabackup:   innodb_log_file_size = 8388608
xtrabackup: Generating a list of tablespaces
InnoDB: Allocated tablespace ID 21 for sys/sys_config, old maximum was 0
xtrabackup: page size for /backup/ince/2021-02-23_11-06-25//ibdata1.delta is 16384 bytes
Applying /backup/ince/2021-02-23_11-06-25//ibdata1.delta to ./ibdata1...
xtrabackup: page size for /backup/ince/2021-02-23_11-06-25//sys/sys_config.ibd.delta is 16384 bytes
Applying /backup/ince/2021-02-23_11-06-25//sys/sys_config.ibd.delta to ./sys/sys_config.ibd...
xtrabackup: page size for /backup/ince/2021-02-23_11-06-25//test3/t3.ibd.delta is 16384 bytes
...
Applying /backup/ince/2021-02-23_11-06-25//mysql/innodb_index_stats.ibd.delta to ./mysql/innodb_index_stats.ibd...
xtrabackup: using the following InnoDB configuration for recovery:
xtrabackup:   innodb_data_home_dir = .
xtrabackup:   innodb_data_file_path = ibdata1:12M:autoextend
xtrabackup:   innodb_log_group_home_dir = /backup/ince/2021-02-23_11-06-25/
xtrabackup:   innodb_log_files_in_group = 1
xtrabackup:   innodb_log_file_size = 8388608
xtrabackup: Starting InnoDB instance for recovery.
xtrabackup: Using 1073741824 bytes for buffer pool (set by --use-memory parameter)
InnoDB: PUNCH HOLE support available
InnoDB: Mutexes and rw_locks use GCC atomic builtins
InnoDB: Uses event mutexes
InnoDB: GCC builtin __atomic_thread_fence() is used for memory barrier
InnoDB: Compressed tables use zlib 1.2.7
InnoDB: Number of pools: 1
InnoDB: Using CPU crc32 instructions
InnoDB: Initializing buffer pool, total size = 1G, instances = 1, chunk size = 128M
InnoDB: Completed initialization of buffer pool
InnoDB: page_cleaner coordinator priority: -20
InnoDB: Highest supported file format is Barracuda.
InnoDB: Log scan progressed past the checkpoint lsn 2842672
InnoDB: Doing recovery: scanned up to log sequence number 2842681 (0%)
InnoDB: Doing recovery: scanned up to log sequence number 2842681 (0%)
InnoDB: Database was not shutdown normally!
InnoDB: Starting crash recovery.
InnoDB: xtrabackup: Last MySQL binlog file position 4008, file name mysql-bin.000007
InnoDB: Creating shared tablespace for temporary tables
InnoDB: Setting file './ibtmp1' size to 12 MB. Physically writing the file full; Please wait ...
InnoDB: File './ibtmp1' size is now 12 MB.
InnoDB: 96 redo rollback segment(s) found. 1 redo rollback segment(s) are active.
InnoDB: 32 non-redo rollback segment(s) are active.
InnoDB: Waiting for purge to start
InnoDB: 5.7.13 started; log sequence number 2842681
InnoDB: xtrabackup: Last MySQL binlog file position 4008, file name mysql-bin.000007

xtrabackup: starting shutdown with innodb_fast_shutdown = 1
InnoDB: FTS optimize thread exiting.
InnoDB: Starting shutdown...
InnoDB: Shutdown completed; log sequence number 2842700
InnoDB: Number of pools: 1
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/sys_config.frm to ./sys/sys_config.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/x@0024wait_classes_global_by_avg_latency.frm to ./sys/x@0024wait_classes_global_by_avg_latency.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/x@0024user_summary_by_statement_type.frm to ./sys/x@0024user_summary_by_statement_type.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/processlist.frm to ./sys/processlist.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/x@0024waits_by_host_by_latency.frm to ./sys/x@0024waits_by_host_by_latency.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/x@0024schema_index_statistics.frm to ./sys/x@0024schema_index_statistics.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/x@0024innodb_lock_waits.frm to ./sys/x@0024innodb_lock_waits.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/x@0024session.frm to ./sys/x@0024session.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/schema_tables_with_full_table_scans.frm to ./sys/schema_tables_with_full_table_scans.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/x@0024io_by_thread_by_latency.frm to ./sys/x@0024io_by_thread_by_latency.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/statements_with_full_table_scans.frm to ./sys/statements_with_full_table_scans.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/io_global_by_wait_by_latency.frm to ./sys/io_global_by_wait_by_latency.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/x@0024ps_digest_95th_percentile_by_avg_us.frm to ./sys/x@0024ps_digest_95th_percentile_by_avg_us.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/x@0024processlist.frm to ./sys/x@0024processlist.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/x@0024innodb_buffer_stats_by_schema.frm to ./sys/x@0024innodb_buffer_stats_by_schema.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/user_summary_by_statement_type.frm to ./sys/user_summary_by_statement_type.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/host_summary_by_file_io.frm to ./sys/host_summary_by_file_io.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/waits_global_by_latency.frm to ./sys/waits_global_by_latency.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/schema_auto_increment_columns.frm to ./sys/schema_auto_increment_columns.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/x@0024host_summary_by_file_io.frm to ./sys/x@0024host_summary_by_file_io.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/x@0024schema_tables_with_full_table_scans.frm to ./sys/x@0024schema_tables_with_full_table_scans.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/x@0024io_global_by_wait_by_latency.frm to ./sys/x@0024io_global_by_wait_by_latency.frm
210223 11:14:53 [01]        ...done
210223 11:14:53 [01] Copying /backup/ince/2021-02-23_11-06-25/sys/version.frm to ./sys/version.frm
...
210223 11:14:54 [00] Copying /backup/ince/2021-02-23_11-06-25//xtrabackup_info to ./xtrabackup_info
210223 11:14:54 [00]        ...done
xtrabackup: using the following InnoDB configuration for recovery:
xtrabackup:   innodb_data_home_dir = .
xtrabackup:   innodb_data_file_path = ibdata1:12M:autoextend
xtrabackup:   innodb_log_group_home_dir = .
xtrabackup:   innodb_log_files_in_group = 2
xtrabackup:   innodb_log_file_size = 268435456
InnoDB: PUNCH HOLE support available
InnoDB: Mutexes and rw_locks use GCC atomic builtins
InnoDB: Uses event mutexes
InnoDB: GCC builtin __atomic_thread_fence() is used for memory barrier
InnoDB: Compressed tables use zlib 1.2.7
InnoDB: Number of pools: 1
InnoDB: Using CPU crc32 instructions
InnoDB: Initializing buffer pool, total size = 1G, instances = 1, chunk size = 128M
InnoDB: Completed initialization of buffer pool
InnoDB: page_cleaner coordinator priority: -20
InnoDB: Setting log file ./ib_logfile101 size to 256 MB
InnoDB: Progress in MB:
 100 200
InnoDB: Setting log file ./ib_logfile1 size to 256 MB
InnoDB: Progress in MB:
 100 200
InnoDB: Renaming log file ./ib_logfile101 to ./ib_logfile0
InnoDB: New log files created, LSN=2842700
InnoDB: Highest supported file format is Barracuda.
InnoDB: Log scan progressed past the checkpoint lsn 2843148
InnoDB: Doing recovery: scanned up to log sequence number 2843157 (0%)
InnoDB: Doing recovery: scanned up to log sequence number 2843157 (0%)
InnoDB: Database was not shutdown normally!
InnoDB: Starting crash recovery.
InnoDB: xtrabackup: Last MySQL binlog file position 4008, file name mysql-bin.000007
InnoDB: Removed temporary tablespace data file: "ibtmp1"
InnoDB: Creating shared tablespace for temporary tables
InnoDB: Setting file './ibtmp1' size to 12 MB. Physically writing the file full; Please wait ...
InnoDB: File './ibtmp1' size is now 12 MB.
InnoDB: 96 redo rollback segment(s) found. 1 redo rollback segment(s) are active.
InnoDB: 32 non-redo rollback segment(s) are active.
InnoDB: 5.7.13 started; log sequence number 2843157
xtrabackup: starting shutdown with innodb_fast_shutdown = 1
InnoDB: FTS optimize thread exiting.
InnoDB: Starting shutdown...
InnoDB: Shutdown completed; log sequence number 2843176
210223 11:14:58 completed OK!
```
### 回滚全备日志
```
[root@m 2021-02-23_11-06-25]# /usr/bin/innobackupex --apply-log --use-memory=1G /backup/full/2021-02-23_10-46-22
210223 11:16:40 innobackupex: Starting the apply-log operation

IMPORTANT: Please check that the apply-log run completes successfully.
           At the end of a successful apply-log run innobackupex
           prints "completed OK!".

/usr/bin/innobackupex version 2.4.9 based on MySQL server 5.7.13 Linux (x86_64) (revision id: a467167cdd4)
xtrabackup: cd to /backup/full/2021-02-23_10-46-22/
xtrabackup: This target seems to be already prepared.
InnoDB: Number of pools: 1
xtrabackup: notice: xtrabackup_logfile was already used to '--prepare'.
xtrabackup: using the following InnoDB configuration for recovery:
xtrabackup:   innodb_data_home_dir = .
xtrabackup:   innodb_data_file_path = ibdata1:12M:autoextend
xtrabackup:   innodb_log_group_home_dir = .
xtrabackup:   innodb_log_files_in_group = 2
xtrabackup:   innodb_log_file_size = 268435456
xtrabackup: using the following InnoDB configuration for recovery:
xtrabackup:   innodb_data_home_dir = .
xtrabackup:   innodb_data_file_path = ibdata1:12M:autoextend
xtrabackup:   innodb_log_group_home_dir = .
xtrabackup:   innodb_log_files_in_group = 2
xtrabackup:   innodb_log_file_size = 268435456
xtrabackup: Starting InnoDB instance for recovery.
xtrabackup: Using 1073741824 bytes for buffer pool (set by --use-memory parameter)
InnoDB: PUNCH HOLE support available
InnoDB: Mutexes and rw_locks use GCC atomic builtins
InnoDB: Uses event mutexes
InnoDB: GCC builtin __atomic_thread_fence() is used for memory barrier
InnoDB: Compressed tables use zlib 1.2.7
InnoDB: Number of pools: 1
InnoDB: Using CPU crc32 instructions
InnoDB: Initializing buffer pool, total size = 1G, instances = 1, chunk size = 128M
InnoDB: Completed initialization of buffer pool
InnoDB: page_cleaner coordinator priority: -20
InnoDB: Highest supported file format is Barracuda.
InnoDB: Removed temporary tablespace data file: "ibtmp1"
InnoDB: Creating shared tablespace for temporary tables
InnoDB: Setting file './ibtmp1' size to 12 MB. Physically writing the file full; Please wait ...
InnoDB: File './ibtmp1' size is now 12 MB.
InnoDB: 96 redo rollback segment(s) found. 1 redo rollback segment(s) are active.
InnoDB: 32 non-redo rollback segment(s) are active.
InnoDB: 5.7.13 started; log sequence number 2843176
InnoDB: xtrabackup: Last MySQL binlog file position 4008, file name mysql-bin.000007

xtrabackup: starting shutdown with innodb_fast_shutdown = 1
InnoDB: FTS optimize thread exiting.
InnoDB: Starting shutdown...
InnoDB: Shutdown completed; log sequence number 2843195
InnoDB: Number of pools: 1
xtrabackup: using the following InnoDB configuration for recovery:
xtrabackup:   innodb_data_home_dir = .
xtrabackup:   innodb_data_file_path = ibdata1:12M:autoextend
xtrabackup:   innodb_log_group_home_dir = .
xtrabackup:   innodb_log_files_in_group = 2
xtrabackup:   innodb_log_file_size = 268435456
InnoDB: PUNCH HOLE support available
InnoDB: Mutexes and rw_locks use GCC atomic builtins
InnoDB: Uses event mutexes
InnoDB: GCC builtin __atomic_thread_fence() is used for memory barrier
InnoDB: Compressed tables use zlib 1.2.7
InnoDB: Number of pools: 1
InnoDB: Using CPU crc32 instructions
InnoDB: Initializing buffer pool, total size = 1G, instances = 1, chunk size = 128M
InnoDB: Completed initialization of buffer pool
InnoDB: page_cleaner coordinator priority: -20
InnoDB: Highest supported file format is Barracuda.
InnoDB: Removed temporary tablespace data file: "ibtmp1"
InnoDB: Creating shared tablespace for temporary tables
InnoDB: Setting file './ibtmp1' size to 12 MB. Physically writing the file full; Please wait ...
InnoDB: File './ibtmp1' size is now 12 MB.
InnoDB: 96 redo rollback segment(s) found. 1 redo rollback segment(s) are active.
InnoDB: 32 non-redo rollback segment(s) are active.
InnoDB: 5.7.13 started; log sequence number 2843195
xtrabackup: starting shutdown with innodb_fast_shutdown = 1
InnoDB: FTS optimize thread exiting.
InnoDB: Starting shutdown...
InnoDB: Shutdown completed; log sequence number 2843214
210223 11:16:43 completed OK!
```
# 停止mysql并备份清空数据目录
```
[root@m 3306]# pkill mysql 
[root@m 3306]# cd /data/mysql/3306/
[root@m 3306]# ls
binlog  data  slowlog
[root@m 3306]# mv data data_bak
[root@m 3306]# mkdir data
[root@m 3306]# chown mysql:mysql data -R
```
### 导入全备
```
[root@m 3306]# /usr/bin/innobackupex --defaults-file=/app/mysql/3306/conf/my.cnf --copy-back /backup/full/2021-02-23_10-46-22
210223 11:19:29 innobackupex: Starting the copy-back operation

IMPORTANT: Please check that the copy-back run completes successfully.
           At the end of a successful copy-back run innobackupex
           prints "completed OK!".

/usr/bin/innobackupex version 2.4.9 based on MySQL server 5.7.13 Linux (x86_64) (revision id: a467167cdd4)
210223 11:19:29 [01] Copying ib_logfile0 to /data/mysql/3306/data/ib_logfile0
210223 11:19:29 [01]        ...done
210223 11:19:29 [01] Copying ib_logfile1 to /data/mysql/3306/data/ib_logfile1
210223 11:19:30 [01]        ...done
210223 11:19:31 [01] Copying ibdata1 to /data/mysql/3306/data/ibdata1
210223 11:19:31 [01]        ...done
210223 11:19:31 [01] Copying ./xtrabackup_binlog_pos_innodb to /data/mysql/3306/data/xtrabackup_binlog_pos_innodb
...
210223 11:19:32 [01] Copying ./performance_schema/accounts.frm to /data/mysql/3306/data/performance_schema/accounts.frm
210223 11:19:32 [01]        ...done
210223 11:19:32 [01] Copying ./performance_schema/socket_summary_by_instance.frm to /data/mysql/3306/data/performance_schema/socket_summary_by_instance.frm
210223 11:19:32 [01]        ...done
210223 11:19:32 [01] Copying ./performance_schema/table_handles.frm to /data/mysql/3306/data/performance_schema/table_handles.frm
210223 11:19:32 [01]        ...done
210223 11:19:32 [01] Copying ./performance_schema/events_stages_summary_by_account_by_event_name.frm to /data/mysql/3306/data/performance_schema/events_stages_summary_by_account_by_event_name.frm
210223 11:19:32 [01]        ...done
210223 11:19:32 [01] Copying ./ib_buffer_pool to /data/mysql/3306/data/ib_buffer_pool
210223 11:19:32 [01]        ...done
210223 11:19:33 completed OK!
```
### 目录授权
```
[root@m 3306]# chown mysql:mysql /data/mysql/ -R
[root@m 3306]# chown mysql:mysql /app/mysql/ -R
```
### 启动mysql
```
/app/mysql/3306/bin/mysqld_safe  --defaults-file=/app/mysql/3306/conf/my.cnf --user=mysql &
```
### 登录数据库验证数据
```mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test1              |
| test2              |
| test3              |
+--------------------+
7 rows in set (0.00 sec)

mysql> select *  from test1.t1;
+------+-------+
| id   | count |
+------+-------+
|    1 |     1 |
+------+-------+
1 row in set (0.01 sec)

mysql> select *  from test2.t2;
+------+-------+
| id   | count |
+------+-------+
|    2 |     2 |
+------+-------+
1 row in set (0.00 sec)

mysql> select *  from test3.t3;
+------+-------+
| id   | count |
+------+-------+
|    3 |     3 |
+------+-------+
1 row in set (0.00 sec)
```
到此测试完毕，数据都恢复了