环境
```none
cat /etc/system-release
CentOS Linux release 7.7.1908 (Core)
```
安装软件
```none
rpm -ivh https://repo.mysql.com//mysql80-community-release-el7-3.noarch.rpm
yum install yum-utils -y
yum-config-manager --disable mysql80-community
yum-config-manager --enable mysql57-community
yum clean all && yum makecache fast
yum install mysql-community-server -y
systemctl enable --now mysqld.service
```
安装完成后必须修改密码
```none
grep password /var/log/mysqld.log
mysqladmin -uroot -p'b0sHKjSfRl?s' password 'Wzy@123.com'
```

多实例
```none
mkdir -p /mysqldir/{3307,3308}/data/

cat > /mysqldir/3307/my.cnf <<EOF
[mysqld]
datadir=/mysqldir/3307/data/
socket=/mysqldir/3307/mysql.sock
log_error=/mysqldir/3307/mysql.log
port=3307
user=mysql
daemonize=true
symbolic-links=0
explicit_defaults_for_timestamp=true
EOF
cat > /mysqldir/3308/my.cnf <<EOF
[mysqld]
datadir=/mysqldir/3308/data/
socket=/mysqldir/3308/mysql.sock
log_error=/mysqldir/3308/mysql.log
port=3308
user=mysql
daemonize=true
symbolic-links=0
explicit_defaults_for_timestamp=true
EOF

chown -R mysql.mysql /mysqldir/
初始化并创建空密码的超级用户
mysqld --initialize-insecure  --user=mysql --datadir=/mysqldir/3307/data/
mysqld --initialize-insecure  --user=mysql --datadir=/mysqldir/3308/data/
mysqld --defaults-file=/mysqldir/3307/my.cnf
mysqld --defaults-file=/mysqldir/3308/my.cnf

修改密码
mysqladmin -uroot -h127.0.0.1 -P3307 password 'Wzy@123.com'
mysqladmin -uroot -h127.0.0.1 -P3308 password 'Wzy@123.com'
使用新密码登陆验证
mysql -h127.0.0.1 -uroot -P3307 -p'Wzy@123.com'
mysql -h127.0.0.1 -uroot -P3308 -p'Wzy@123.com'
```
mysql5.7重置密码
```none
# grep skip  /etc/my.cnf         
skip_grant_tabless
# systemctl restart mysqld.service
mysql> update mysql.user set authentication_string=password('s3usZT0YaKe7/CkZ+35LOC6K3gGM') where user='root' and host='localhost';  
Query OK, 0 rows affected, 1 warning (0.00 sec)
Rows matched: 1  Changed: 0  Warnings: 1

mysql> 
mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

# grep skip  /etc/my.cnf         
#skip_grant_tables
# systemctl restart mysqld.service
```


