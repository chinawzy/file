mysql安装
```none
rpm -ivh https://repo.mysql.com//mysql80-community-release-el7-3.noarch.rpm
yum clean all && yum makecache
yum-config-manager --disable mysql80-community
yum-config-manager --enable mysql57-community
yum install mysql-community-server.x86_64 -y
mkdir -p /data/mysql
chown mysql.mysql /data/mysql
vim /etc/my.cnf
```
db1
```none
[root@dbs1 ~]# egrep -v '#|^$' /etc/my.cnf
[mysqld]
datadir=/data/mysql
socket=/var/lib/mysql/mysql.sock
symbolic-links=0
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
sql_mode="ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION"
character-set-server=utf8mb4
server-id=1
log-bin=mysql_bin
auto-increment-increment=2
auto-increment-offset=1
log_bin_trust_function_creators=1
expire_logs_days=10
max_binlog_size=100M
log-slave-updates=1
max_connections=600
#wait_timeout=5
[root@dbs1 ~]# systemctl start mysqld.service
[root@dbs1 ~]# grep -i password /var/log/mysqld.log
[root@dbs1 ~]# mysql -p
Enter password: 输入日志中的随机密码
mysql> set password = password('Root@123');flush privileges;
mysql> grant replication slave on *.* to 'copy'@'db2' identified by 'Copy@123';
mysql> show master status;
```
db2
```none
[root@dbs2 ~]# egrep -v '#|^$' /etc/my.cnf
[mysqld]
datadir=/data/mysql
socket=/var/lib/mysql/mysql.sock
symbolic-links=0
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
sql_mode="ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION"
character-set-server=utf8mb4
server-id=2
log-bin=mysql_bin
auto-increment-increment=2
auto-increment-offset=2
log_bin_trust_function_creators=1
expire_logs_days=10
max_binlog_size=100M
log-slave-updates=1
max_connections=600
#wait_timeout=5
[root@dbs2 ~]# systemctl start mysqld.service
[root@dbs2 ~]# grep -i password /var/log/mysqld.log
[root@dbs2 ~]# mysql -p
Enter password:
mysql> set password = password('Root@123');flush privileges;
mysql> grant replication slave on *.* to 'copy'@'db1' identified by 'Copy@123';
mysql> show master status;
mysql> change master to master_host='dbs1',master_user='copy',master_password='Copy@123',master_log_file='mysql_bin.000002',master_log_pos=855;
mysql> start slave;
mysql> show slave status\G
查看状态，单向复制即完成
```
db1
```none
在dbs1上操作
mysql> change master to master_host='db2',master_user='copy',master_password='Copy@123',master_log_file='mysql_bin.000002',master_log_pos=855;
mysql> start slave;
mysql> show slave status\G
查看无误即为完成
```
