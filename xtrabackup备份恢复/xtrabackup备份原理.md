### 前言

相对于逻辑备份利用查询提取数据中的所有记录，物理备份更直接，拷贝数据库文件和日志来完成备份，因此速度会更快。

]Percona XtraBackup（简称PXB）是 Percona 公司开发的一个用于 MySQL 数据库物理热备的备份工具，支持 MySQl（Oracle）、Percona Server 和 MariaDB，并且全部开源

### 工具集
软件包安装完后一共有4个可执行文件，如下：
```
usr
├── bin
│   ├── innobackupex
│   ├── xbcrypt
│   ├── xbstream
│   └── xtrabackup
```
其中最主要的是 innobackupex 和 xtrabackup，前者是一个 perl 脚本，后者是 C/C++ 编译的二进制。

xtrabackup 是用来备份 InnoDB 表的，不能备份非 InnoDB 表，和 mysqld server 没有交互；innobackupex 脚本用来备份非 InnoDB 表，同时会调用 xtrabackup 命令来备份 InnoDB 表，还会和 mysqld server 发送命令进行交互，如加读锁（FTWRL）、获取位点（SHOW SLAVE STATUS）等。简单来说，innobackupex 在 xtrabackup 之上做了一层封装。

一般情况下，我们是希望能备份 MyISAM 表的，虽然我们可能自己不用 MyISAM 表，但是 mysql 库下的系统表是 MyISAM 的，因此备份基本都通过 innobackupex 命令进行；另外一个原因是我们可能需要保存位点信息。

### 备份流程
1.开启redo日志拷贝线程，从最新的检查点开始顺序拷贝redo日志； 

2.开启ibd文件拷贝线程，拷贝innodb表的数据 

3.ibd文件拷贝结束，通知调用FTWRL，获取一致性位点 

4.备份非innodb表(系统表)和frm文件 

5.由于此时没有新事务提交，等待redo日志拷贝完成 

6.最新的redo日志拷贝完成后，相当于此时的innodb表和非innodb表数据都是最新的 

7.获取binlog位点，此时数据库的状态是一致的。 

8.释放锁，备份结束。
https://blog.csdn.net/yu891203/article/details/106758822/
### Xtrabackup优点
（1）备份速度快，物理备份可靠   
（2）备份过程不会打断正在执行的事务（无需锁表）   
（3）能够基于压缩等功能节约磁盘空间和流量    
（4）自动备份校验    
（5）还原速度快   
（6）可以流传将备份传输到另外一台机器上   
（7）在不增加服务器负载的情况备份数据   
### 增量备份
PXB 是支持增量备份的，但是只能对 InnoDB 做增量，InnoDB 每个 page 有个 LSN 号，LSN 是全局递增的，page 被更改时会记录当前的 LSN 号，page中的 LSN 越大，说明当前page越新（最近被更新）。每次备份会记录当前备份到的LSN（xtrabackup_checkpoints 文件中），增量备份就是只拷贝LSN大于上次备份的page，比上次备份小的跳过，每个 ibd 文件最终备份出来的是增量 delta 文件。

MyISAM 是没有增量的机制的，每次增量备份都是全部拷贝的。

增量备份过程和全量备份一样，只是在 ibd 文件拷贝上有不同。

### 恢复过程
如果看恢复备份集的日志，会发现和 mysqld 启动时非常相似，其实备份集的恢复就是类似 mysqld crash后，做一次 crash recover。

恢复的目的是把备份集中的数据恢复到一个一致性位点，所谓一致就是指原数据库某一时间点各引擎数据的状态，比如 MyISAM 中的数据对应的是 15:00 时间点的，InnoDB 中的数据对应的是 15:20 的，这种状态的数据就是不一致的。PXB 备份集对应的一致点，就是备份时FTWRL的时间点，恢复出来的数据，就对应原数据库FTWRL时的状态。

因为备份时 FTWRL 后，数据库是处于只读的，非 InnoDB 数据是在持有全局读锁情况下拷贝的，所以非 InnoDB 数据本身就对应 FTWRL 时间点；InnoDB 的 ibd 文件拷贝是在 FTWRL 前做的，拷贝出来的不同 ibd 文件最后更新时间点是不一样的，这种状态的 ibd 文件是不能直接用的，但是 redo log 是从备份开始一直持续拷贝的，最后的 redo 日志点是在持有 FTWRL 后取得的，所以最终通过 redo 应用后的 ibd 数据时间点也是和 FTWRL 一致的。

所以恢复过程只涉及 InnoDB 文件的恢复，非 InnoDB 数据是不动的。备份恢复完成后，就可以把数据文件拷贝到对应的目录，然后通过mysqld来启动了。