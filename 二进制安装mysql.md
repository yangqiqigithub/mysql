# 下载mysql
```
wget https://cdn.mysql.com//Downloads/MySQL-5.7/mysql-5.7.33-linux-glibc2.12-x86_64.tar.gz
```
# 安装组件
```
yum install libaio
```
# 创建目录
```
mkdir -p /app/mysql/3306/conf
mkdir -p /app/mysql/3306/logs

mkdir -p /data/mysql/3306/data
mkdir -p /data/mysql/3306/binlog

ln -s /data/mysql/3306/data /app/mysql/3306/
ln -s /data/mysql/3306/binlog/ /app/mysql/3306/
```
# 安装mysql
```
tar zxvf mysql-5.7.33-linux-glibc2.12-x86_64.tar.gz
cd mysql-5.7.33-linux-glibc2.12-x86_64
cp -a ./* /app/mysql/3306/
```
# 添加环境变量
```
echo 'export PATH=/app/mysql/3306/bin:$PATH' >>/etc/profile
source /etc/profile
[root@m conf]# which mysql
/app/mysql/3306/bin/mysql

```
# 配置my.cnf
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
server-id=1
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
# 添加mysql运行用户
```
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
```
# 目录授权
```
chown mysql:mysql  /data/mysql/ -R
chown mysql:mysql  /app/mysql/ -R
```
# 初始化mysql
```
/app/mysql/3306/bin/mysqld --defaults-file=/app/mysql/3306/conf/my.cnf --initialize-insecure --explicit_defaults_for_timestamp --basedir=/app/mysql/3306 --datadir=/data/mysql/3306/data --user=mysql

# --initialize-insecure 设置root为空密码
```
# 启动mysql
```
/app/mysql/3306/bin/mysqld_safe  --defaults-file=/app/mysql/3306/conf/my.cnf --user=mysql &
```
# 设置root密码
```
/app/mysql/3306/bin/mysqladmin -u root password 123@qwe --socket=/app/mysql/3306/mysql.sock
```
# 登录mysql
```
/app/mysql/3306/bin/mysql -uroot -p --socket=/app/mysql/3306/mysql.sock
```
# 创建用户
```
mysql> grant all on *.* to 'admin'@'%' identified by '123' with grant option;
Query OK, 0 rows affected, 1 warning (0.01 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.01 sec)

mysql> SELECT DISTINCT CONCAT('User: ''',user,'''@''',host,''';') AS query FROM mysql.user;
+------------------------------------+
| query                              |
+------------------------------------+
| User: 'admin'@'%';                 |
| User: 'mysql.session'@'localhost'; |
| User: 'mysql.sys'@'localhost';     |
| User: 'root'@'localhost';          |
+------------------------------------+
4 rows in set (0.00 sec)
** 为了安全，根据自身情况决定是否删除root用户和mysql自带用户
```
