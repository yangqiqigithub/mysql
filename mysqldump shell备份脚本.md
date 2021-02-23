### 创建备份用户
```
grant SELECT,RELOAD,SHOW DATABASES,SHOW VIEW,show events,LOCK TABLES,EVENT,REPLICATION CLIENT,process,execute on . to 'mybackuser'@'10.64.34.%' identified by 'xxxx';  
```
### 脚本内容
```
#！binbash
USERNAME=mysqluser
PASSWORD=xxxxx
HOST=1.1.1.1

DATE=`date +%Y%m%d`  #用来做备份文件名字的一部分
OLDDATE=`date +%Y%m%d -d '-30 days'`  #本地保存天数  
DATE1=`date +%Y%m`

#指定命令所用的全路径
MYSQL=usrbinmysql
MYSQLDUMP=usrbinmysqldump
MYSQLADMIN=usrbinmysqladmin

#创建备份的目录和文件
BACKDIR=optaliyundbbackupmysqldbbqj_sh_mysqldb
[ -d ${BACKDIR} ]  mkdir -p ${BACKDIR}
[ -d ${BACKDIR}${DATE1}${DATE} ]  mkdir -p ${BACKDIR}${DATE1}${DATE}
[ ! -d ${BACKDIR}${DATE1}${OLDDATE} ]  rm -rf ${BACKDIR}${DATE1}${OLDDATE} 
#开始备份  for循环想要备份的数据库
MYSQLDUMP_LIST=$(${MYSQL} -u${USERNAME} -p${PASSWORD} -h${HOST}  -e show databases grep -Evi databaseinforperformysql)
echo $MYSQLDUMP_LIST

for DBNAME in ${MYSQLDUMP_LIST} ##使用for依次罗列需要备份的数据库名字
do
    $MYSQLDUMP   -u${USERNAME} -p${PASSWORD} -h${HOST}  --single-transaction -B --set-gtid-purged=off  --triggers  --routines --events   $DBNAME   gzip  ${BACKDIR}${DATE1}${DATE}${DBNAME}-${DATE}.sql.gz

 logger ${DBNAME} has been backup successful - $DATE
done
```
备份完毕后 就长这个样子：
```
201808
├── 2018-08-30
│   ├── bqj_activity-2018-08-30.sql.gz
│   ├── bqj_auth_service-2018-08-30.sql.gz
```
### mysqldump参数
-F   
同参数--flush-logs，在dump之前刷新日志，即生成一个新的二进制日志。一次dump多个库时，每个库都会刷新一次

--master-data=2     
会在备份文件里记录当前备份文件备份到了那个位置，例如
```
-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000006', MASTER_LOG_POS=234;
```
