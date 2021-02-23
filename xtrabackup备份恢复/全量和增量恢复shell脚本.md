```
#!/bin/sh
#on xtrabackup 2.2.9
#Name:JamesXiaoXu

INNOBACKUPEXFULL=/usr/bin/innobackupex
MYSQL_CMD="--host=localhost --user=back_user --password=123456 --port=3306"  
MYSQL_UP=" --user=back_user --password=123456 --port=3306 "
SOCKET="--socket=/data/mysql/logs/mysql.sock"
TMPLOG="/backup/fullbackup/innobackupex.$$.log"
MY_CNF=/data/mysql/conf/my.cnf
MYSQL=/data/mysql/bin/mysql
MYSQL_ADMIN=/data/mysql/bin/mysqladmin
FULLBACKUP_DIR=/backup/fullbackup/full
INCREBACKUP_DIR=/backup/increbackup/incre
logfiledate=backup.`date +%Y%m%d%H%M`.txt
STARTED_TIME=`date +%s`

error()
{
    echo "$1" 1>&2
    exit 1
}
#######检查目录和命令是否存在 不存在就退出了
if [ ! -x $INNOBACKUPEXFULL ]; then
  error "$INNOBACKUPEXFULL no install/usr/bin."
fi

if [ ! -d $FULLBACKUP_DIR ]; then
  error "backdir:$FULLBACKUP_DIR不存在."
fi

if [ ! -d $INCREBACKUP_DIR ]; then
  error "backdir:$INCREBACKUP_DIR不存在."
fi

LATEST_FULL_BACKUP=`find $FULLBACKUP_DIR -mindepth 1 -maxdepth 1 -type d -printf "%P\n" | sort -nr | head -1`
RECOVERY_TIME=${LATEST_FULL_BACKUP:0:10}
echo "最新的全备是 $LATEST_FULL_BACKUP，脚本默认恢复的是最新日期的全备+增备 如有其它需求请看修改说明"

echo "开始redo-only全备"
$INNOBACKUPEXFULL --apply-log --use-memory=4G --redo-only $FULLBACKUP_DIR/$LATEST_FULL_BACKUP  >> $TMPLOG 2>&1 
if [ $? -ne 0 ]; then
    echo "$LATEST_FULL_BACKUP redo-only is fail!!"
	exit 1
else
    echo "$LATEST_FULL_BACKUP redo-only is success!!"
fi

echo "开始redo-only增备"
incre_files_list=`find ${INCREBACKUP_DIR}  -mindepth 1 -maxdepth 1 -type d -printf "%P\n" | grep ${RECOVERY_TIME}`
last_incre_backfile=`find ${INCREBACKUP_DIR}  -mindepth 1 -maxdepth 1 -type d -printf "%P\n" | grep ${RECOVERY_TIME}| sort -nr | head -1`
 
for i in ${incre_files_list}
do 
	if [ "$i" == "${last_incre_backfile}" ];then
		$INNOBACKUPEXFULL --apply-log --use-memory=4G $FULLBACKUP_DIR/$LATEST_FULL_BACKUP --incremental-dir=$INCREBACKUP_DIR/$i  >> $TMPLOG 2>&1 
		if [ $? -ne 0 ]; then
			echo "$i last noredo-only is fail!!"
			exit 1
		else
			echo "$i last noredo-only is success!!"
		fi
		
	else
		$INNOBACKUPEXFULL --apply-log --use-memory=4G --redo-only $FULLBACKUP_DIR/$LATEST_FULL_BACKUP --incremental-dir=$INCREBACKUP_DIR/$i  >> $TMPLOG 2>&1 
		if [ $? -ne 0 ]; then
			echo "$i  redo-only is fail!!"
			exit 1
		else
			echo "$i redo-only is success!!"
		fi
	fi


done

echo "开始回滚全备$LATEST_FULL_BACKUP日志 "
$INNOBACKUPEXFULL --apply-log --use-memory=4G $FULLBACKUP_DIR/$LATEST_FULL_BACKUP
if [ $? -ne 0 ]; then
			echo "$LATEST_FULL_BACKUP 回滚成功！！"
		else
			echo "$LATEST_FULL_BACKUP 回滚失败！！"
			exit 1
		fi
$INNOBACKUPEXFULL --defaults-file=${MY_CNF} --copy-back $FULLBACKUP_DIR/$LATEST_FULL_BACKUP
if [ $? -ne 0 ]; then
			echo "$LATEST_FULL_BACKUP 导入成功！！"
		else
			echo "$LATEST_FULL_BACKUP 导入失败！！"
			exit 1
		fi
echo "success: `date +%F' '%T' '%w`" 
exit 0
```