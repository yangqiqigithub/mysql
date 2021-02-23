```
#!/bin/sh
#on xtrabackup 2.2.9
#Name:qiqiYang

INNOBACKUPEXFULL=/usr/bin/innobackupex
MYSQL_CMD="--host=1.1.1.1 --user=xtarbackup --password=123456 --port=3306"  
MYSQL_UP="--user=xtarbackup --password=gxxx --port=3306 "
SOCKET="--socket=/data/mysql3306/logs/mysql.sock"
TMPLOG="/data/backup/xtrabackup_mysql/mysql3306/increback/innobackupex.$$.log"
MY_CNF=/data/mysql3306/ini/my.cnf
MYSQL=/data/mysql/bin/mysql
MYSQL_ADMIN=/data/mysql/bin/mysqladmin
BACKUP_DIR=/data/backup/xtrabackup_mysql/mysql3306/increback
FULLBACKUP=/data/backup/xtrabackup_mysql/mysql3306/fullback/full
INCREBACKUP_DIR=$BACKUP_DIR/incre
logfiledate=backup.`date +%Y%m%d%H%M`.txt
STARTED_TIME=`date +%s`

error()
{
    echo "$1" 1>&2
    exit 1
}

if [ ! -x $INNOBACKUPEXINCRE ]; then
  error "$INNOBACKUPEXINCRE no install/usr/bin."
fi

if [ ! -d $BACKUP_DIR ]; then
  error "backdir:$BACKUP_DIR不存在."
fi


mysql_status=`netstat -nl | awk 'NR>2{if ($4 ~ /.*:3306/) {print "Yes";exit 0}}'`

if [ "$mysql_status" != "Yes" ];then
    error "MySQL not start."
fi


if ! `echo 'exit' | $MYSQL -s $MYSQL_CMD ${SOCKET} ` ; then
 error "user passwoed is wrong!"
fi

 
echo "----------------------------"
echo
echo "$0: MySQL incre backup"
echo "start: `date +%F' '%T' '%w`"
echo

mkdir -p $INCREBACKUP_DIR

LATEST_FULL_BACKUP=`find $FULLBACKUP -mindepth 1 -maxdepth 1 -type d -printf "%P\n" | sort -nr | head -1`

	echo  "*********************************"
	echo -e "start backup...please wait..."
	echo  "*********************************"
	$INNOBACKUPEXFULL --defaults-file=$MY_CNF ${SOCKET} --incremental-basedir=$FULLBACKUP/$LATEST_FULL_BACKUP --use-memory=4G  $MYSQL_CMD --incremental $INCREBACKUP_DIR > $TMPLOG 2>&1 

	cat $TMPLOG>/$BACKUP_DIR/$logfiledate

	if [ -z "`tail -1 $TMPLOG | grep 'innobackupex: completed OK!'`" ] ; then
	 echo "$INNOBACKUPEX cmd false:"; echo
	 echo -e "---------- $INNOBACKUPEX_PATH error ----------"
	 cat $TMPLOG
	 rm -f $TMPLOG
	 exit 1
	fi

	 
	THISBACKUP=`awk -- "/Backup created in directory/ { split( \\\$0, p, \"'\" ) ; print p[2] }" $TMPLOG`
	rm -f $TMPLOG

	echo -n "success backup to:$THISBACKUP"

echo "success: `date +%F' '%T' '%w`" 
exit 0
```