# 规划
在一台服务器上部署3306和3307进行测试
# 部署3306和3307
### 下载mysql
```
wget https://cdn.mysql.com//Downloads/MySQL-5.7/mysql-5.7.33-linux-glibc2.12-x86_64.tar.gz
```
### 安装组件
```
yum install libaio
```
### 创建目录
```
mkdir -p /app/mysql/3306/conf
mkdir -p /app/mysql/3306/logs

mkdir -p /data/mysql/3306/data
mkdir -p /data/mysql/3306/binlog

```
### 安装mysql
```
tar zxvf mysql-5.7.33-linux-glibc2.12-x86_64.tar.gz
cd mysql-5.7.33-linux-glibc2.12-x86_64
cp -a ./* /app/mysql/3306/
```

### 添加环境变量
```
echo 'export PATH=/app/mysql/3306/bin:$PATH' >>/etc/profile
source /etc/profile
[root@m conf]# which mysql
/app/mysql/3306/bin/mysql

```
### 配置my.cnf
```
#my.cnf
[client]
default-character-set = utf8

[mysql]
default-character-set = utf8

[mysqld]
character-set-server=uft8
basedir = /app/mysql/3306/
datadir = /data/mysql/3306/data/
port = 3306
socket = /app/mysql/3306/mysql.sock
pid-file = /app/mysql/3306/mysql.pid

log-error = /app/mysql/3306/logs/error.log
server-id = 3306
slow-query-log = 1
long_query_time = 0.2
slow-query-log-file = /data/mysql/3306/slowlog/slow-queries.log

max_connections = 800
max_connect_errors = 100000

table_open_cache = 256
query_cache_size = 2M

character_set_server=utf8
init_connect='SET NAMES utf8'

innodb_file_per_table=1
innodb_buffer_pool_size = 1048M

innodb_flush_log_at_trx_commit=1 
innodb_log_buffer_size = 16M
innodb_log_file_size = 256M
innodb_log_files_in_group = 2 
innodb_max_dirty_pages_pct = 50 

log-bin = /data/mysql/3306/binlog/mysql-bin
sync_binlog=1
binlog-format=row
expire_logs_days = 10

interactive_timeout = 1800
wait_timeout = 1800

log_timestamps = system

```
### 添加mysql运行用户
```
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
```
### 部署3307
```
cp -a /app/mysql/3306 /app/mysql/3307
cp -a /data/mysql/3306 /data/mysql/3307

sed 's#3306#3307#g' /app/mysql/3307/conf/my.cnf -i

ln -s /data/mysql/3306/data /app/mysql/3306/
ln -s /data/mysql/3306/binlog /app/mysql/3306/
ln -s /data/mysql/3306/slowlog /app/mysql/3306/

ln -s /data/mysql/3307/data /app/mysql/3307/
ln -s /data/mysql/3307/binlog /app/mysql/3307/
ln -s /data/mysql/3307/slowlog /app/mysql/3307/
```
### 目录授权
```
chown mysql:mysql  /data/mysql/ -R
chown mysql:mysql  /app/mysql/ -R
```

### 初始化mysql
```
/app/mysql/3306/bin/mysqld --defaults-file=/app/mysql/3306/conf/my.cnf --initialize-insecure --explicit_defaults_for_timestamp --basedir=/app/mysql/3306 --datadir=/data/mysql/3306/data --user=mysql

/app/mysql/3307/bin/mysqld --defaults-file=/app/mysql/3307/conf/my.cnf --initialize-insecure --explicit_defaults_for_timestamp --basedir=/app/mysql/3307 --datadir=/data/mysql/3307/data --user=mysql

# --initialize-insecure 设置root为空密码
```
### 启动mysql
```
/app/mysql/3306/bin/mysqld_safe  --defaults-file=/app/mysql/3306/conf/my.cnf --user=mysql &

/app/mysql/3307/bin/mysqld_safe  --defaults-file=/app/mysql/3307/conf/my.cnf --user=mysql &
```
### 登录mysql
```
/app/mysql/3306/bin/mysql -uroot --socket=/app/mysql/3306/mysql.sock

/app/mysql/3307/bin/mysql -uroot --socket=/app/mysql/3307/mysql.sock
```
# 配置gtid主从复制
### 配置my.cnf
```

1、主：/app/mysql/3306/conf/my.cnf
[mysqld]
#GTID:
server_id=3306              #服务器id
gtid_mode=on                 #开启gtid模式
enforce_gtid_consistency=on  #强制gtid一致性，开启后对于特定create table不被支持

#binlog
log_bin=master-binlog
log-slave-updates=1    
binlog_format=row            #强烈建议，其他格式可能造成数据不一致

#relay log
skip_slave_start=1            

2、从：/app/mysql/3307/conf/my.cnf
[mysqld]
#GTID:
gtid_mode=on
enforce_gtid_consistency=on
server_id=3307

#binlog
log-bin=slave-binlog
log-slave-updates=1
binlog_format=row      #强烈建议，其他格式可能造成数据不一致

#relay log
skip_slave_start=1
```
### 重启mysql
```
pkill mysql

/app/mysql/3306/bin/mysqld_safe  --defaults-file=/app/mysql/3306/conf/my.cnf --user=mysql &

/app/mysql/3307/bin/mysqld_safe  --defaults-file=/app/mysql/3307/conf/my.cnf --user=mysql &
```
### 登录主创建复制用户
```
/app/mysql/3306/bin/mysql -uroot --socket=/app/mysql/3306/mysql.sock

grant replication slave on *.* to 'repl'@'192.168.0.%' identified by '000000';
flush privileges;
```
### 分别登录主从查看gitd是否开启
```
/app/mysql/3306/bin/mysql -uroot --socket=/app/mysql/3306/mysql.sock

/app/mysql/3307/bin/mysql -uroot --socket=/app/mysql/3307/mysql.sock
mysql> show variables like "%gtid%";
+----------------------------------+-----------+
| Variable_name                    | Value     |
+----------------------------------+-----------+
| binlog_gtid_simple_recovery      | ON        |
| enforce_gtid_consistency         | ON        |
| gtid_executed_compression_period | 1000      |
| gtid_mode                        | ON        |
| gtid_next                        | AUTOMATIC |
| gtid_owned                       |           |
| gtid_purged                      |           |
| session_track_gtids              | OFF       |
+----------------------------------+-----------+
8 rows in set (0.00 sec)
```
# 登录从配置同步
```
/app/mysql/3307/bin/mysql -uroot --socket=/app/mysql/3307/mysql.sock

配置同步
mysql>  CHANGE MASTER TO MASTER_HOST='192.168.0.13',MASTER_USER='repl',MASTER_PASSWORD='000000',MASTER_AUTO_POSITION=1;
Query OK, 0 rows affected, 2 warnings (0.02 sec)

开启同步
mysql> START SLAVE;
Query OK, 0 rows affected (0.00 sec)

查看同步状态
mysql> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.0.13
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 751
               Relay_Log_File: m-relay-bin.000002
                Relay_Log_Pos: 964
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes **
            Slave_SQL_Running: Yes **
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 751
              Relay_Log_Space: 1167
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 3306
                  Master_UUID: 5da251ea-74c1-11eb-8aac-fa163e95ab6d
             Master_Info_File: /data/mysql/3307/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 5da251ea-74c1-11eb-8aac-fa163e95ab6d:1-3
            Executed_Gtid_Set: 5da251ea-74c1-11eb-8aac-fa163e95ab6d:1-3
                Auto_Position: 1
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)
```
### 登录主查看从服务器
```
/app/mysql/3306/bin/mysql -uroot --socket=/app/mysql/3306/mysql.sock

mysql> show slave hosts;
+-----------+------+------+-----------+--------------------------------------+
| Server_id | Host | Port | Master_id | Slave_UUID                           |
+-----------+------+------+-----------+--------------------------------------+
|      3307 |      | 3307 |      3306 | 8deb070d-74c1-11eb-90e7-fa163e95ab6d |
+-----------+------+------+-----------+--------------------------------------+
1 row in set (0.00 sec)
```
### 测试同步
```
# 登录主创建库和表并插入数据
/app/mysql/3306/bin/mysql -uroot --socket=/app/mysql/3306/mysql.sock

mysql> create database master1;
Query OK, 1 row affected (0.01 sec)

mysql> use master1;
Database changed
mysql> CREATE TABLE `test1` (`id` int(11) DEFAULT NULL,`count` int(11) DEFAULT NULL);
Query OK, 0 rows affected (0.03 sec)

mysql> insert into test1 values(1,1);
Query OK, 1 row affected (0.01 sec)

mysql> exit
# 登录从查看数据
/app/mysql/3307/bin/mysql -uroot --socket=/app/mysql/3307/mysql.sock

mysql> select * from master1.test1;
+------+-------+
| id   | count |
+------+-------+
|    1 |     1 |
+------+-------+
1 row in set (0.00 sec)
```

