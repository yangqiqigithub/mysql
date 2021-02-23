```

#!/bin/bash
#Description:测试阿里云线上mysql生产库的备份恢复通用可用性脚本

DATE=`date +%Y%m%d`  #当天 20180915
TODAY=`date +%Y%m%d` #20180917 当天
YESTERDAY=`date -d '1 days ago' +%Y%m%d` #20180916 前一天
MONTH=`date +"%Y%m"` #当月 201809


mysql_backupdir=/home/bak/aliyundbbackup/mysqldb #本地存放mysql备份数据的总目录
singledb_backdir=qukuailianfabric_sh_mysqldb #备份数据库的备份目录的名字

#本地测试备份的mysql数据库信息
mysql_cmd=/data/mysql/bin/mysql
mysql_user=root
mysql_socet=/data/mysql3310/logs/mysql.sock


#判断最新的备份目录是否存在
total_size1=`du -sh  ${mysql_backupdir}/${singledb_backdir}/${MONTH}/${TODAY} | awk '{print $1}'`
if [ ! -d ${mysql_backupdir}/${singledb_backdir}/${MONTH}/${TODAY} ];then
echo "最新的备份文件不存在"
exit 1
else
echo "最新的备份文件是:${TODAY},大小是:${total_size1} "
fi

#解压所有备份文件
echo "开始解压所有sql文件"
cd ${mysql_backupdir}/${singledb_backdir}/${MONTH}/${TODAY}
gunzip -f  ${mysql_backupdir}/${singledb_backdir}/${MONTH}/${TODAY}/*  > /dev/null 2>&1
total_size2=`du -sh  ${mysql_backupdir}/${singledb_backdir}/${MONTH}/${TODAY} | awk '{print $1}'`
echo "解压完毕,解压后的总大小为:${total_size2}"

#开始导入备份文件
echo "开始将sql文件导入mysql3310"
sql_list=`ls -la ${mysql_backupdir}/${singledb_backdir}/${MONTH}/${TODAY} | egrep  "*.sql" | awk '{print $9}'`
for sql_obj in ${sql_list}
do
        ${mysql_cmd} -u${mysql_user}  --socket=${mysql_socet} < ${mysql_backupdir}/${singledb_backdir}/${MONTH}/${TODAY}/${sql_obj}
        if [ $? -ne 0 ]; then
                        echo "${sql_obj}恢复失败 请手动检查原因！"
                        exit 1
                else
                        echo "${sql_obj} 恢复成功！"
                fi
done

```