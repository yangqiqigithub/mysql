```
#!/bin/bash 
#created by yangqiqi 2017-08-24 

#创建备份用户
#grant select,lock tables,reload,super,file,show view on *.* to 'mysqlbackup'@'localhost' identified by 'my_password'; 
#grant execute on *.* to 'mysqlbackup'@'localhost' identified by 'my_password';  授予调用存储过程的权限
##flush privileges;

USERNAME=mysqlbackup #备份的用户名 
PASSWORD=my_password  #备份的密码
HOST=localhost #备份主机

DATE=`date +%Y-%m-%d`  #用来做备份文件名字的一部分
OLDDATE=`date +%Y-%m-%d -d '-10 days'`  #本地保存天数  

#指定命令所用的全路径
MYSQL=/app/mysql5.5/bin/mysql
MYSQLDUMP=/app/mysql5.5/bin/mysqldump
MYSQLADMIN=/app/mysql5.5/bin/mysqladmin

#创建备份的目录和文件
BACKDIR=/data/backup/db
[ -d ${BACKDIR} ] || mkdir -p ${BACKDIR}
[ -d ${BACKDIR}/${DATE} ] || mkdir ${BACKDIR}/${DATE}
[ ! -d ${BACKDIR}/${OLDDATE} ] || rm -rf ${BACKDIR}/${OLDDATE} #保存10天 多余的删除最前边的
#开始备份  for循环想要备份的数据库
MYSQLDUMP_LIST=$(${MYSQL} -uroot -p'123456'  -e "show databases"| grep -Evi "database|infor|perfor|mysql")

for DBNAME in ${MYSQLDUMP_LIST} ##使用for依次罗列需要备份的数据库名字
do
  TABLE="$(${MYSQL} -uroot -p'123456'  -e "use $DBNAME;show tables;"|sed '1d')" #依次列出需要备份的表的名字
  for tname in $TABLE
    do
        MYDIR=${BACKDIR}/${DATE}/${DBNAME} 
        #echo $MYDIR
        [ ! -d $MYDIR ] && mkdir -p $MYDIR #创建备份表的目录
        $MYSQLDUMP   -uroot -p'123456'  $DBNAME $tname |gzip >$MYDIR/${DBNAME}_${tname}_${DATE}.sql.gz
    done
 logger "${DBNAME}_${tname} has been backup successful - $DATE"
done
```