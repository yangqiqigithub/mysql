### 查看连接数
##### 查看当前连接数
```
./mysqladmin -uroot -p1234.com status
Uptime: 1370150  Threads: 1 （当前连接数） Questions: 79  Slow queries: 0  Opens: 33  Flush tables: 1  Open tables: 26  Queries per second avg: 0.000

./mysql -uroot -p1234.com -e 'show status' | grep -i  Threads 
Delayed_insert_threads    0
Slow_launch_threads    0
Threads_cached    1
Threads_connected    1
Threads_created    2
Threads_running    1 ##（当前连接数）


mysql> show status like 'Threads%';
+-------------------+-------+
| Variable_name    | Value |
+-------------------+-------+
| Threads_cached    | 1    |
| Threads_connected | 1    |
| Threads_created  | 2    |
| Threads_running  | 1    |   ###当前连接数
+-------------------+-------+
4 rows in set (0.00 sec)
```
##### 查看最大连接数
```
root@xxx bin]# ./mysql -uroot -p1234.com -e 'show variables' | grep max_connections
max_connections    500

mysql> show global variables like 'max_conn%';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| max_connect_errors | 10    |
| max_connections    | 500  |## 最大连接数
+--------------------+-------+
2 rows in set (0.00 sec)
```
### 解决方式
##### 如果数据库无法登录了
一般这个时候数据库往往已经无法登录，可以重启解决，也有个不重启的方法

使用gdb工具 不用进入数据库，不用重启数据库 方法如下：
```
[root@xxx bin]# gdb -p $(cat /data/mydata/xxx.pid) -ex "set max_connections=500" -batch  
[New LWP 7667]
[New LWP 4816]
[New LWP 341]
[New LWP 338]
[New LWP 337]
[New LWP 336]
[New LWP 335]
[New LWP 331]
[New LWP 330]
[New LWP 329]
[New LWP 328]
[New LWP 327]
[New LWP 326]
[New LWP 325]
[New LWP 324]
[New LWP 323]
[New LWP 322]
[Thread debugging using libthread_db enabled]
0x00000035654df1b3 in poll () from /lib64/libc.so.6
```
查看mysql pid位置的方法
 
在配置文件 my.cnf里查找
用 ps -ef | grep mysql 查找
修改完毕后 ，尝试重新进入数据库，并查看链接数
这种方法设置后，只是暂时的，数据库重启后，会变为原来的数值，要想永久，设置完后修改配置文件my.cnf

如果可以重启数据库  
需要重启数据库  
修改 my.conf   
max_connection = 1000;  

##### 如果还可以登录数据库
进入数据库
设置新的最大连接数为200：mysql> set GLOBAL max_connections=200

显示当前运行的Query：mysql> show processlist

显示当前状态：mysql> show status

退出客户端：mysql> exit

这种方法设置后，只是暂时的，数据库重启后，会变为原来的数值，要想永久，设置完后修改配置文件my.cnf