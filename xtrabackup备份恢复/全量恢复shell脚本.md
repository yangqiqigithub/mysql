```
#!/bin/bash 
#Description:恢复innobackupex全备的脚本

#所需变量的定义
innobackupex_cmd=/usr/bin/innobackupex
full_bakdir=/home/bak/zntybackup/3307/full
ince_bakdir=/home/bak/zntybackup/3307/ince
my_cnf=/data/mysql3307/ini/my.cnf
data_dir=/data/mysql3307/data
mysqld_safe_cmd=/data/mysql/bin/mysqld_safe
owner=znty
group=znty
time_time=`date +%Y%m%d%H%M`

#恢复前必须要检查的事项
echo "###郑重提醒:###"
echo "1. 确定mysql已经平滑关闭，可用kill -15 pid 关闭"
echo "2. 最好将mysql的数据目录，程序目录安全备份到一个地方，以便恢复失败的话，原有数据还存在"
echo "3. 或者只是将数据目录 mv dbdir dbdir_20180929"
echo "请务必将以上三点落实后，再进行恢复！！"

#检查innobackupex命令是否存在 
if [ ! -x ${innobackupex_cmd} ]; then
  echo "${innobackupex_cmd} 命令不存在"
  echo "现在开始安装：请耐心等待..."
  yum install http://www.percona.com/downloads/percona-release/redhat/0.1-6/percona-release-0.1-6.noarch.rpm -y > /dev/null 2>&1
  wget http://www.percona.com/downloads/percona-release/redhat/0.1-6/\
  percona-release-0.1-6.noarch.rpm > /dev/null 2>&1
  rpm -ivH percona-release-0.1-6.noarch.rpm > /dev/null 2>&1
  yum install percona-xtrabackup-24 -y > /dev/null 2>&1
  sleep 1s
  which innobackupex > /dev/null 2>&1
     if [ $? -ne 0 ]; then
     echo "${innobackupex_cmd}安装失败！"
     echo "请按照官方地址：https://www.percona.com/doc/percona-xtrabackup/LATEST/installation/yum_repo.html 手动安装后，继续恢复！"
     exit 1
     else
     echo "${innobackupex_cmd}安装成功！"
      fi
fi 
 
#检查全备目录是否存在,并打印全备的大小
total_size=`du -sh  ${full_bakdir} | awk '{print $1}'`
newest_backfile=`find ${full_bakdir}  -type d -mtime 1 | awk -F / '{print $7}' | head -1`
if [ ! -d ${full_bakdir}/${newest_backfile} ];then
	echo "最新的备份文件不存在"
	exit 1
	else
	echo "最新的备份文件是:${newest_backfile},大小是:${total_size} "
fi

#处理全备数据
echo "###郑重提醒:###"
echo "正在将全备的原目录进行备份，以便操作失误损坏全备数据..."
cp -a ${full_bakdir}/${newest_backfile} /tmp
if [ ! -d /tmp/${newest_backfile} ];then
	echo "拷贝全备文件失败,请手动检查原因！"
	exit 1
	else
	echo "已将全备目录备份至/tmp/${newest_backfile}"
fi

echo "接下来，开始处理全备数据"
echo "1. 先对全备进行一次redo-only"
${innobackupex_cmd} --apply-log --use-memory=4G --redo-only ${full_bakdir}/${newest_backfile} > ${full_bakdir}/first_handle_${time_time}.log  2>&1

if [ -z "`tail -1  ${full_bakdir}/first_handle_${time_time}.log | grep 'completed OK'`" ] ; then
	echo "对全备${newest_backfile}的第一次read-only处理失败，请手动检查原因！"
	exit 1
else
	echo "全备read-only处理成功！"
fi

echo "2. 开始回滚全备日志"
${innobackupex_cmd} --apply-log --use-memory=4G ${full_bakdir}/${newest_backfile} > ${full_bakdir}/second_handle_${time_time}.log  2>&1

if [ -z "`tail -1  ${full_bakdir}/second_handle_${time_time}.log | grep 'completed OK'`" ] ; then
	echo "对全备${newest_backfile}的第二次回顾日志处理失败，请手动检查原因！"
	exit 1
else
	echo "全备日志回滚成功！"
fi

echo "开始导入全备"

${innobackupex_cmd} --defaults-file=${my_cnf} --copy-back  ${full_bakdir}/${newest_backfile} > ${full_bakdir}/last_recovery_${time_time}.log  2>&1

if [ -z "`tail -1  ${full_bakdir}/last_recovery_${time_time}.log | grep 'completed OK'`" ] ; then
	echo "全备${newest_backfile}恢复失败，请手动检查原因！"
	exit 1
else
	echo "恭喜：全备恢复成功！！"
fi

echo "开始对数据目录授权"
chown ${owner}:${group} ${data_dir} -R
echo "授权的命令如下："
echo "chown ${owner}:${group} data_dir -R"
echo "开始启动mysql..."
${mysqld_safe_cmd} --defaults-file=${my_cnf}  --user=${owner} & > /dev/null 2>&1
if [ $? -ne 0 ]; then
  echo "mysql启动失败！"
else
  echo "mysql启动成功！"
fi 
echo "启动命令为：${mysqld_safe_cmd} --defaults-file=${my_cnf}  --user=${owner} &"
```