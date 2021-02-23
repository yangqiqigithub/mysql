# 规划
mysql5.7  
192.168.0.13 3306      
192.168.0.14 3307    
防火墙已经处理好

# 192.168.0.13-3306部署  
### 下载mysql
```
wget httpscdn.mysql.comDownloadsMySQL-5.7mysql-5.7.33-linux-glibc2.12-x86_64.tar.gz
```
### 安装组件
```
yum install libaio
```
### 创建目录
```
mkdir -p appmysql3306conf
mkdir -p appmysql3306logs

mkdir -p datamysql3306data
mkdir -p datamysql3306binlog
mkdir -p datamysql3306slowlog

ln -s datamysql3306data appmysql3306
ln -s datamysql3306binlog appmysql3306
ln -s datamysql3306slowlog appmysql3306

```
### 安装mysql
```
tar zxvf mysql-5.7.33-linux-glibc2.12-x86_64.tar.gz
cd mysql-5.7.33-linux-glibc2.12-x86_64
cp -a . appmysql3306
```

### 添加环境变量
```
echo 'export PATH=appmysql3306bin$PATH' etcprofile
source etcprofile
[root@m conf]# which mysql
appmysql3306binmysql

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
basedir = appmysql3306
datadir = datamysql3306data
port = 3306
socket = appmysql3306mysql.sock
pid-file = appmysql3306mysql.pid

log-error = appmysql3306logserror.log
server-id = 3306
slow-query-log = 1
long_query_time = 0.2
slow-query-log-file = datamysql3306slowlogslow-queries.log

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

log-bin = datamysql3306binlogmysql-bin
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
useradd -r -g mysql -s binfalse mysql
```

### 目录授权
```
chown mysqlmysql  datamysql -R
chown mysqlmysql  appmysql -R
```

### 初始化mysql
```
appmysql3306binmysqld --defaults-file=appmysql3306confmy.cnf --initialize-insecure --explicit_defaults_for_timestamp --basedir=appmysql3306 --datadir=datamysql3306data --user=mysql

# --initialize-insecure 设置root为空密码
```
### 启动mysql
```
appmysql3306binmysqld_safe  --defaults-file=appmysql3306confmy.cnf --user=mysql &

```
### 登录mysql
```
appmysql3306binmysql -uroot --socket=appmysql3306mysql.sock

```
# 192.168.0.14-3307部署 
仿照192.168.0.13-3306的部署方式


# 配置gtid主从复制
### 配置my.cnf
```
1、主：appmysql3306confmy.cnf
[mysqld]
#GTID
server_id=3306              #服务器id
gtid_mode=on                 #开启gtid模式
enforce_gtid_consistency=on  #强制gtid一致性，开启后对于特定create table不被支持

#binlog
log_bin=master-binlog
log-slave-updates=1    
binlog_format=row            #强烈建议，其他格式可能造成数据不一致

#relay log
skip_slave_start=1            

2、从：appmysql3307confmy.cnf
[mysqld]
#GTID
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
### 分别在13和14服务器上重启mysql
```
pkill mysql

appmysql3306binmysqld_safe  --defaults-file=appmysql3306confmy.cnf --user=mysql &

appmysql3307binmysqld_safe  --defaults-file=appmysql3307confmy.cnf --user=mysql &
```
==先配置192.168.0.13-3306为主 192.168.0.14-3307为从==
### 登录主3306创建复制用户
```
appmysql3306binmysql -uroot --socket=appmysql3306mysql.sock

grant replication slave on . to 'repl'@'192.168.0.%' identified by '000000';
flush privileges;
```
### 分别登录主从查看gitd是否开启
```
appmysql3306binmysql -uroot --socket=appmysql3306mysql.sock

appmysql3307binmysql -uroot --socket=appmysql3307mysql.sock
mysql show variables like %gtid%;
+----------------------------------+-----------+
 Variable_name                     Value     
+----------------------------------+-----------+
 binlog_gtid_simple_recovery       ON        
 enforce_gtid_consistency          ON        
 gtid_executed_compression_period  1000      
 gtid_mode                         ON        
 gtid_next                         AUTOMATIC 
 gtid_owned                                  
 gtid_purged                                 
 session_track_gtids               OFF       
+----------------------------------+-----------+
8 rows in set (0.00 sec)
```
# 登录从3307配置同步
```
appmysql3307binmysql -uroot --socket=appmysql3307mysql.sock

配置同步
mysql  CHANGE MASTER TO MASTER_HOST='192.168.0.13',MASTER_USER='repl',MASTER_PASSWORD='000000',MASTER_AUTO_POSITION=1;
Query OK, 0 rows affected, 2 warnings (0.02 sec)

开启同步
mysql START SLAVE;
Query OK, 0 rows affected (0.00 sec)

查看同步状态
mysql show slave status G
 1. row 
               Slave_IO_State Waiting for master to send event
                  Master_Host 192.168.0.13
                  Master_User repl
                  Master_Port 3306
                Connect_Retry 60
              Master_Log_File mysql-bin.000003
          Read_Master_Log_Pos 751
               Relay_Log_File m-relay-bin.000002
                Relay_Log_Pos 964
        Relay_Master_Log_File mysql-bin.000003
             Slave_IO_Running Yes 
            Slave_SQL_Running Yes 
              Replicate_Do_DB 
          Replicate_Ignore_DB 
           Replicate_Do_Table 
       Replicate_Ignore_Table 
      Replicate_Wild_Do_Table 
  Replicate_Wild_Ignore_Table 
                   Last_Errno 0
                   Last_Error 
                 Skip_Counter 0
          Exec_Master_Log_Pos 751
              Relay_Log_Space 1167
              Until_Condition None
               Until_Log_File 
                Until_Log_Pos 0
           Master_SSL_Allowed No
           Master_SSL_CA_File 
           Master_SSL_CA_Path 
              Master_SSL_Cert 
            Master_SSL_Cipher 
               Master_SSL_Key 
        Seconds_Behind_Master 0
Master_SSL_Verify_Server_Cert No
                Last_IO_Errno 0
                Last_IO_Error 
               Last_SQL_Errno 0
               Last_SQL_Error 
  Replicate_Ignore_Server_Ids 
             Master_Server_Id 3306
                  Master_UUID 5da251ea-74c1-11eb-8aac-fa163e95ab6d
             Master_Info_File datamysql3307datamaster.info
                    SQL_Delay 0
          SQL_Remaining_Delay NULL
      Slave_SQL_Running_State Slave has read all relay log; waiting for more updates
           Master_Retry_Count 86400
                  Master_Bind 
      Last_IO_Error_Timestamp 
     Last_SQL_Error_Timestamp 
               Master_SSL_Crl 
           Master_SSL_Crlpath 
           Retrieved_Gtid_Set 5da251ea-74c1-11eb-8aac-fa163e95ab6d1-3
            Executed_Gtid_Set 5da251ea-74c1-11eb-8aac-fa163e95ab6d1-3
                Auto_Position 1
         Replicate_Rewrite_DB 
                 Channel_Name 
           Master_TLS_Version 
1 row in set (0.00 sec)
```
### 登录主3306查看从服务器
```
appmysql3306binmysql -uroot --socket=appmysql3306mysql.sock

mysql show slave hosts;
+-----------+---------+ ------+-----------+--------------------------------------+
 Server_id  Host            Port  Master_id  Slave_UUID                           
+-----------+---------+------+-----------+--------------------------------------+
      3307  192.168.0.14  3307       3306  8deb070d-74c1-11eb-90e7-fa163e95ab6d 
+-----------+------+------+-----------+--------------------------------------+
1 row in set (0.00 sec)
```
### 测试同步
```
# 登录主3306创建库和表并插入数据
appmysql3306binmysql -uroot --socket=appmysql3306mysql.sock

mysql create database master1;
Query OK, 1 row affected (0.01 sec)

mysql use master1;
Database changed
mysql CREATE TABLE `test1` (`id` int(11) DEFAULT NULL,`count` int(11) DEFAULT NULL);
Query OK, 0 rows affected (0.03 sec)

mysql insert into test1 values(1,1);
Query OK, 1 row affected (0.01 sec)

mysql exit
# 登录从查看数据
appmysql3307binmysql -uroot --socket=appmysql3307mysql.sock

mysql select  from master1.test1;
+------+-------+
 id    count 
+------+-------+
    1      1 
+------+-------+
1 row in set (0.00 sec)
```
==再配置192.168.0.14-3307为主 192.168.0.13-3306为从==

# 登录从3306配置同步
```
appmysql3306binmysql -uroot --socket=appmysql3306mysql.sock

配置同步
mysql CHANGE MASTER TO MASTER_HOST='192.168.0.14',MASTER_PORT=3307,MASTER_USER='repl',MASTER_PASSWORD='000000',MASTER_AUTO_POSITION=1;
Query OK, 0 rows affected, 2 warnings (0.02 sec)

开启同步
mysql START SLAVE;
Query OK, 0 rows affected (0.00 sec)

查看同步状态
mysql show slave status G
 1. row 
               Slave_IO_State Waiting for master to send event
                  Master_Host 192.168.0.14
                  Master_User repl
                  Master_Port 3307
                Connect_Retry 60
              Master_Log_File mysql-bin.000004
          Read_Master_Log_Pos 234
               Relay_Log_File m-relay-bin.000002
                Relay_Log_Pos 367
        Relay_Master_Log_File mysql-bin.000004
             Slave_IO_Running Yes
            Slave_SQL_Running Yes
              Replicate_Do_DB 
          Replicate_Ignore_DB 
           Replicate_Do_Table 
       Replicate_Ignore_Table 
      Replicate_Wild_Do_Table 
  Replicate_Wild_Ignore_Table 
                   Last_Errno 0
                   Last_Error 
                 Skip_Counter 0
          Exec_Master_Log_Pos 234
              Relay_Log_Space 570
              Until_Condition None
               Until_Log_File 
                Until_Log_Pos 0
           Master_SSL_Allowed No
           Master_SSL_CA_File 
           Master_SSL_CA_Path 
              Master_SSL_Cert 
            Master_SSL_Cipher 
               Master_SSL_Key 
        Seconds_Behind_Master 0
Master_SSL_Verify_Server_Cert No
                Last_IO_Errno 0
                Last_IO_Error 
               Last_SQL_Errno 0
               Last_SQL_Error 
  Replicate_Ignore_Server_Ids 
             Master_Server_Id 3307
                  Master_UUID 8deb070d-74c1-11eb-90e7-fa163e95ab6d
             Master_Info_File datamysql3306datamaster.info
                    SQL_Delay 0
          SQL_Remaining_Delay NULL
      Slave_SQL_Running_State Slave has read all relay log; waiting for more updates
           Master_Retry_Count 86400
                  Master_Bind 
      Last_IO_Error_Timestamp 
     Last_SQL_Error_Timestamp 
               Master_SSL_Crl 
           Master_SSL_Crlpath 
           Retrieved_Gtid_Set 
            Executed_Gtid_Set 5da251ea-74c1-11eb-8aac-fa163e95ab6d1-6,
8deb070d-74c1-11eb-90e7-fa163e95ab6d1-8
                Auto_Position 1
         Replicate_Rewrite_DB 
                 Channel_Name 
           Master_TLS_Version 
1 row in set (0.00 sec)

```
### 登录主3307查看从服务器
```
appmysql3307binmysql -uroot --socket=appmysql3307mysql.sock

mysql show slave hosts;
+-----------+------+------+-----------+--------------------------------------+
 Server_id  Host         Port  Master_id  Slave_UUID                           
+-----------+------+------+-----------+--------------------------------------+
      3306  192.168.0.13  3306       3307  5da251ea-74c1-11eb-8aac-fa163e95ab6d 
+-----------+------+------+-----------+--------------------------------------+
1 row in set (0.00 sec)

```
### 测试同步
```
# 登录3307创建库和表并插入数据
appmysql3307binmysql -uroot --socket=appmysql3307mysql.sock

mysql create database slave1;
Query OK, 1 row affected (0.01 sec)

mysql use slave1;
Database changed
mysql CREATE TABLE `test1` (`id` int(11) DEFAULT NULL,`count` int(11) DEFAULT NULL);
Query OK, 0 rows affected (0.03 sec)

mysql insert into test1 values(2,2);
Query OK, 1 row affected (0.01 sec)

mysql exit
# 登录3306查看数据
appmysql3306binmysql -uroot --socket=appmysql3306mysql.sock

mysql select  from slave1.test1;
+------+-------+
 id    count 
+------+-------+
    2      2 
+------+-------+
1 row in set (0.00 sec)
```
# 配置keeplived
### 安装keeplived
下载合适的版本，在两台master都与需要安装部署
```
[root@192 ~]# wget httpwww.keepalived.orgsoftwarekeepalived-1.4.3.tar.gz
[root@192 ~]# yum -y install openssl openssl-devel
[root@192 ~]# cd keepalived-1.4.3
[root@192 keepalived-1.4.3]# .configure --prefix= && make && make install
[root@192 keepalived-1.4.3]# whereis keepalived
keepalived usrsbinkeepalived etckeepalived
检查cent7启动脚本中执行程序位置
[root@192 ~]# vim usrlibsystemdsystemkeepalived.service
[Unit]
Description=LVS and VRRP High Availability Monitor
After= network-online.target syslog.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=varrunkeepalived.pid
KillMode=process
EnvironmentFile=-etcsysconfigkeepalived
ExecStart=usrsbinkeepalived $KEEPALIVED_OPTIONS
ExecReload=binkill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```
### master-13主机的keepalived主配
```
[root@192 ~]# setenforce 0
注意关闭selinx，否则可能导致notify_down脚本无法执行
[root@192 ~]# vim etckeepalivedkeepalived.conf
! Configuration File for keepalived

global_defs {
   router_id mysql-1
}

vrrp_instance VI_1 {
    state BACKUP
    interface eno16777736
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.0.100
    }
}

virtual_server 192.168.0.100 3306 {
    delay_loop 2
    lb_algo rr
    lb_kind DR
    persistence_timeout 50
    protocol TCP

    real_server 192.168.0.13 3306 {
        weight 1
    notify_down rootmysql.sh
    TCP_CHECK{
            connect_timeout 3
            retry 3
            delay_before_retry 3
        connect_port 3306
        }
    }
}
```
### master-14主机的keepalived主配
```
[root@192 ~]# setenforce 0
注意关闭selinx，否则可能导致notify_down脚本无法执行
[root@192 ~]# vim etckeepalivedkeepalived.conf

! Configuration File for keepalived

global_defs {
   router_id mysql-2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eno16777736
    virtual_router_id 51
    priority 50
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.100
    }
}

virtual_server 192.168.1.100 3306 {
    delay_loop 2
    lb_algo rr
    lb_kind DR
    persistence_timeout 50
    protocol TCP

    real_server 192.168.0.14 3306 {
        weight 1
    notify_down rootmysql.sh
    TCP_CHECK{
            connect_timeout 3
            retry 3
            delay_before_retry 3
        connect_port 3306
        }
    }
}
```
### notify_down脚本内容
```
[root@192 ~]# cat mysql.sh
#!binbash
pkill keepalived
[root@192 ~]# chmod +x mysql.sh
```
脚本内容的操作在两台主机都需要操作
# keepalive 测试
### 两台主机均启动keepalived，并且查看vip
```
[root@192 ~]# systemctl start keepalived
[root@192 ~]# ps -ef  grep keepalived

root      24528      1  0 1735         000000 sbinkeepalived
root      24529  24528  0 1735         000000 sbinkeepalived
root      24530  24528  0 1735         000000 sbinkeepalived
root      25554   3223  0 1739 pts0    000000 grep --color=auto keepalived
[root@192 ~]# ip a

......
2 eno16777736 BROADCAST,MULTICAST,UP,LOWER_UP mtu 1500 qdisc pfifo_fast state UP qlen 1000
    linkether 000c294b6a1e brd ffffffffffff
    inet 192.168.0.1324 brd 192.168.1.255 scope global dynamic eno16777736
       valid_lft 83035sec preferred_lft 83035sec
    inet 192.168.0.1332 scope global eno16777736
       valid_lft forever preferred_lft forever
    inet6 fe8020c29fffe4b6a1e64 scope link 
       valid_lft forever preferred_lft forever
......
```
发现在master-13主机存在vip
### 宕掉master-13的mysql服务再观察vip位置
```
[root@192 ~]# systemctl stop mysqld
[root@192 ~]# ps -ef  grep keepalived

root      28118   3222  0 1738 pts0    000000 grep --color=auto keepalived
```
发现停止msyql服务后keepalived也被杀掉，说明脚本执行成功
去到master-14主机观察vip,发现vip到了14服务器上
```
.......
2 eno16777736 BROADCAST,MULTICAST,UP,LOWER_UP mtu 1500 qdisc pfifo_fast state UP qlen 1000
    linkether 000c29dbf7b8 brd ffffffffffff
    inet 192.168.1.1224 brd 192.168.1.255 scope global dynamic eno16777736
       valid_lft 82750sec preferred_lft 82750sec
    inet 192.168.0.10032 scope global eno16777736
       valid_lft forever preferred_lft forever
    inet6 fe8020c29fffedbf7b864 scope link 
       valid_lft forever preferred_lft forever
.......
```
### 重新启动master-13的msyql服务以及keepalived观察vip位置,vip又回来了
```
[root@192 ~]# systemctl start mysqld
[root@192 ~]# systemctl start keepalived

......
2 eno16777736 BROADCAST,MULTICAST,UP,LOWER_UP mtu 1500 qdisc pfifo_fast state UP qlen 1000
    linkether 000c294b6a1e brd ffffffffffff
    inet 192.168.1.1024 brd 192.168.1.255 scope global dynamic eno16777736
       valid_lft 82554sec preferred_lft 82554sec
    inet 192.168.1.10032 scope global eno16777736
       valid_lft forever preferred_lft forever
    inet6 fe8020c29fffe4b6a1e64 scope link 
       valid_lft forever preferred_lft forever
.......
```
# 总结
### 在配置双主互从过程中需要注意什么？
①keepalived+mysql双主一般来说，中小型规模是最省事的。master节点发生故障后，利用keepalived的高可用机制实现快速切换备用节点   

②在部署方案的过程中，两个节点的模式最好都为BACKUP模式，避免因为网络延迟，超过心跳检查时间，发生脑裂情况相互抢占MASTER导致写入相同数据引发的冲突  

③两个节点的auto_increment_incremenet(自增步长)和auto_increment_offset(自增起始点)设为不同值。目的为了避免master意外宕机，可能会有部分binlog未能及时复制到slave上被应用，从而导致slave新写入数据的自增值和原先的master冲突，从offset起始点开始就错开了，避免了主键id的冲突，当然，如有合适的容错机制解决冲突话，也可以不这么设置  
### 如果遇到主从延迟怎么解决？
①首先需要通过show slave statusG中 Seconds_Behind_Master观察主从之间延迟的状态 

②slave节点服务器硬件配置不能与master节点相差太大，会大大导致复制的延迟

③如果对延迟问题很敏感，可以考虑更换mariadb分支版本，或者直接上线mysql5.7最新版本，利用多线程复制的方式可以很大程度降低复制延迟
```
mysql show global variables like 'slave_paralle%';
Variable_name	        Value
slave_parallel_type     DATABASE
slave_parallel_workers   	0
```
slave_parallel_workers：默认为0，表示为单线程

slave_parallel_type：默认多线程机制为一个线程处理一个DATABASE

mysql set global slave_parallel_workers=4; #修改为四个线程操作

mysql set global slave_parallel_type='logical_clock'; #修改为并行复制

④调整master节点服务器DDL速度还有就是主库是写，对数据安全性较高，比如sync_binlog=1，innodb_flush_log_at_trx_commit= 1 之类的设置，而slave则不需要这么高的数据安全，完全可以讲sync_binlog设置为0或者关闭binlog，innodb_flushlog也可以设置为0来提高sql的执行效率。另外就是使用比主库更好的硬件设备作为slave

### keeplived抢占和非抢占模式
1，抢占式
抢占模式为当keepalived的某台机器挂了之后VIP漂移到了备节点，当主节点恢复后主动将VIP再次抢回，keepalived默认工作在抢占模式下。
主节点MASTER，备节点BACKUP

2，非抢占式
非抢占模式则是当主节挂了再次起来后不再抢回VIP。
两个节点的state都必须配置为BACKUP，两个节点都必须加上配置 nopreempt。

下面直接展示keepalived的非抢占配置。
```
主机配置如下：

vrrp_instance VI_1
{
　　state BACKUP
　　nopreempt
　　priority 100

　　advert_int 1
　　virtual_router_id 1
　　interface eth0

　　authentication
　　{
　　　　auth_type PASS
　　　　auth_pass abcd@hehe
　　}

　　virtual_ipaddress
　　{
　　　　100.92.2.110
　　}
}

 

备机配置如下：

vrrp_instance VI_1
{
　　state BACKUP
　　nopreempt
　　priority 90

　　advert_int 1
　　virtual_router_id 1
　　interface eth0

　　authentication
　　{
　　　　auth_type PASS
　　　　auth_pass abcd@hehe
　　}

　　virtual_ipaddress
　　{
　　　　100.92.2.110
　　}
}
```