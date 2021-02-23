# 版本
percona-xtrabackup-24-2.4.9   
mysql5.7
# 安装xtrabackup
```
wget https://www.percona.com/downloads/XtraBackup/Percona-XtraBackup-2.4.9/binary/redhat/7/x86_64/percona-xtrabackup-24-2.4.9-1.el7.x86_64.rpm
yum install -y percona-xtrabackup-24-2.4.9-1.el7.x86_64.rpm 
```
# 备份
#### 全量备份
==注意，强烈推荐像这样把所有库一起备份
否则单单备份某个库，即mysql系统本身的一些库没有被备份的话之后恢复的时候，把数据目录一删，会导致恢复后无法读出数据的==
```
/usr/bin/innobackupex --defaults-file=/app/mysql/3306/conf/my.cnf --socket=/app/mysql/3306/mysql.sock --use-memory=1G --host=127.0.0.1 --user=root --password=123@qwe --port=3306 /backup/full
```
全量备份完毕后的目录展示
```
[root@m full]# tree 2021-02-22_21-21-58/
2021-02-22_21-21-58/
├── backup-my.cnf
├── ib_buffer_pool
├── ibdata1
├── test 库名
│   ├── db.opt
│   ├── test1.frm
│   └── test1.ibd
├── mysql
│   ├── columns_priv.frm
│   ├── columns_priv.MYD
│   ├── columns_priv.MYI
│   ├── db.frm
│   ├── ...
├── performance_schema
│   ├── accounts.frm
│   ├── cond_instances.frm
│   ├── ...
├── sys
│   ├── db.opt
│   ├── ...
├── xtrabackup_binlog_info 记录备份的binlog位置
├── xtrabackup_checkpoints  记录lsn位置
├── xtrabackup_info
└── xtrabackup_logfile
```
几个备份文件内容展示
```
[root@m 2021-02-22_21-21-58]# cat xtrabackup_binlog_info 
mysql-bin.000006	494	5da251ea-74c1-11eb-8aac-fa163e95ab6d:1-9,
8deb070d-74c1-11eb-90e7-fa163e95ab6d:1-8


[root@m 2021-02-22_21-21-58]# cat xtrabackup_checkpoints 
backup_type = full-backuped
from_lsn = 0  *** 注意这里，和增量备份的这个文件比较
to_lsn = 2773758 *** 注意这里，和增量备份的这个文件比较
last_lsn = 2773767
compact = 0
recover_binlog_info = 0


[root@m 2021-02-22_21-21-58]# cat xtrabackup_info 
uuid = f2135976-7510-11eb-a928-fa163e95ab6d
name = 
tool_name = innobackupex
tool_command = --defaults-file=/app/mysql/3306/conf/my.cnf --socket=/app/mysql/3306/mysql.sock --use-memory=1G --host=127.0.0.1 --user=root --password=... --port=3306 ./
tool_version = 2.4.9
ibbackup_version = 2.4.9
server_version = 5.7.33-log
start_time = 2021-02-22 21:21:58
end_time = 2021-02-22 21:22:00
lock_time = 0
binlog_pos = filename 'mysql-bin.000006', position '494', GTID of the last change '5da251ea-74c1-11eb-8aac-fa163e95ab6d:1-9,
8deb070d-74c1-11eb-90e7-fa163e95ab6d:1-8'
innodb_from_lsn = 0
innodb_to_lsn = 2773758
partial = N
incremental = N
format = file
compact = N
compressed = N
encrypted = N
```
#### 增量备份
```
/usr/bin/innobackupex --defaults-file=/app/mysql/3306/conf/my.cnf --incremental-basedir=/backup/full/2021-02-22_21-21-58 --socket=/app/mysql/3306/mysql.sock --use-memory=1G --user=root --password=123@qwe --port=3306 --incremental /backup/ince/

#--incremental-basedir=/backup/full/2021-02-22_21-21-58 指定全量备份的目录
```
```
[root@m 2021-02-22_21-33-27]# cat xtrabackup_checkpoints 
backup_type = incremental
from_lsn = 2773758 ***注意这里，和全量备份的这个文件比较
to_lsn = 2778270 ****注意这里，和全量备份的这个文件比较
last_lsn = 2778279
compact = 0
recover_binlog_info = 0
```
# 恢复
==注意 在恢复之前
强烈建议先关闭mysql  并且mv原来的数据目录 创建新的数据目录 保持是空的
首先，要清空mysql的数据目录( 保险起见，在清空前，把该目录压缩拷贝到安全的地方 )    
注意，清空数据目录，意味着mysql自己的系统库(保存着用户信息和表信息之类)也会被删掉  
注意，所以，我前面推荐不要单单备份某个库，而是全部一起备份== 

就是把每次的增备合到全备里
#### 先对全备进行一次redo-only
```
/usr/bin/innobackupex --apply-log --use-memory=4G --redo-only /backup/full/2021-02-22_21-21-58
```
#### 再分别对增倍做一次redo-only 
==(确保不是最后一次增倍，针对最后一次增备不加redo-only参数)==
```
/usr/bin/innobackupex --apply-log --use-memory=4G --redo-only /backup/full/2021-02-22_21-21-58 --incremental-dir=/backup/ince/2021-02-23_09-38-02
# 指定全备和增备目录
```
#### 最后一次的增备不需要加--redo-only
```
/usr/bin/innobackupex --apply-log --use-memory=1G  /backup/full/2021-02-23_09-33-14 --incremental-dir=/backup/ince/2021-02-23_09-39-43
# 指定全备和增备目录
```
#### 回滚全备日志
```
/usr/bin/innobackupex --apply-log --use-memory=1G /backup/full/2021-02-23_09-33-14
```
#### 导入全备
```
/usr/bin/innobackupex --defaults-file=/app/mysql/3306/conf/my.cnf --copy-back /backup/full/2021-02-23_09-33-14
```
#### 目录授权
```
 chown -R mysql:mysql mysql的数据目录
// 把mysql数据目录下的所有文件，全部改成mysql:mysql所属
```