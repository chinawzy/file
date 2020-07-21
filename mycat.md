环境
```none
CentOS Linux release 7.7.1908 (Core)
mycat 192.168.88.10
mysql-master 192.168.88.11
mysql-slave 192.168.88.12
通过mycat实现所有写操作在master，所有读操作在slave
```
mysql服务安装
```none
rpm -ivh https://repo.mysql.com//mysql80-community-release-el7-3.noarch.rpm
yum install yum-utils -y
yum-config-manager --disable mysql80-community
yum-config-manager --enable mysql57-community
yum clean all && yum makecache fast
yum install mysql-community-server -y
systemctl enable --now mysqld.service
# 重置密码,此实验目的为测试mycat,未开启主备模式
grep password /var/log/mysqld.log
mysqladmin -uroot -p'b0sHKjSfRl?s' password 'Root@123'
mysql -pRoot@123 -e "grant all on *.* to 'root'@'%' identified by 'Root@123'"
mysql -pRoot@123 -e "create database testdb"
# 分别写入ip地址
mysql -pRoot@123 -e "use testdb;create table ip(addr tinyint);insert into ip values(11);" # mysql-master
mysql -pRoot@123 -e "use testdb;create table ip(addr tinyint);insert into ip values(12);" # mysql-slave
```
mycat安装
```none
[root@mycat ~]# yum install java-1.8.0-openjdk* -y
[root@mycat ~]# echo 'export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.222.b03-1.el7.x86_64/jre/' >> /etc/profile
[root@mycat ~]# source /etc/profile
[root@mycat ~]# cd /usr/local/src/
[root@mycat /usr/local/src]# wget http://dl.mycat.org.cn/1.6.7.4/Mycat-server-1.6.7.4-release/Mycat-server-1.6.7.4-release-20200105164103-linux.tar.gz
[root@mycat /usr/local/src]# ls -lh
total 21M
-rw-r--r-- 1 root root 21M Jul 20 17:27 Mycat-server-1.6.7.4-release-20200105164103-linux.tar.gz
[root@mycat /usr/local/src]# tar -zxvf Mycat-server-1.6.7.4-release-20200105164103-linux.tar.gz -C /usr/local/
[root@mycat /usr/local/src]# echo 'export PATH=/usr/local/mycat/bin/:$PATH' >> /etc/profile
[root@mycat /usr/local/src]# source /etc/profile
[root@mycat /usr/local/src]# dos2unix /usr/local/mycat/bin/startup_nowrap.sh
```
mycat配置
```none
[root@mycat ~]# vim /usr/local/mycat/conf/server.xml # 删除原有user配置
<user name="mycatuser">
        <property name="password">mycatpasswd</property>
        <property name="schemas">testdb</property>
</user>

[root@mycat ~]# vim /usr/local/mycat/conf/schema.xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
        <schema name="testdb" checkSQLschema="false" sqlMaxLimit="100" dataNode="testdb_node"></schema>
        <dataNode name="testdb_node" dataHost="testdb_host" database="testdb" />
        <dataHost name="testdb_host" maxCon="1000" minCon="10" balance="3" writeType="0" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
                <heartbeat>select user()</heartbeat>
                <writeHost host="hostM" url="192.168.88.11:3306" user="root" password="Root@123">
                        <readHost host="hostS" url="192.168.88.12:3306" user="root" password="Root@123"/>
                </writeHost>
        </dataHost>
</mycat:schema>
[root@mycat ~]# mycat start # 启动时间较长,若日志无报错,耐心等待
[root@mycat ~]# tail -f /usr/local/mycat/logs/switch.log
MyCAT Server startup successfully. see logs in logs/mycat.log

故障处理，无法启动查看日志，报错为Startup failed: Timed out waiting for a signal from the JVM 修改如下配置
[root@mycat ~]# sed -i '/wrapper.ping.timeout/i wrapper.startup.timeout=300' /usr/local/mycat/conf/wrapper.conf
测试
mysql -umycatuser -pmycatpasswd -h192.168.88.10 -P8066 -e 'insert into testdb.ip values(100);' # 命中master
mysql -umycatuser -pmycatpasswd -h192.168.88.10 -P8066 -e 'select * from testdb.ip;' # 命中slave
mysql -umycatuser -pmycatpasswd -h192.168.88.10 -P8066 -e 'select @@hostname;' # 命中slave
```
参数解析
```
balance
当balance=0,不开启读写分离,所有读操作都发生在当前的writeHost上
当balance=1,所有读操作都随机发送到当前的writeHost对应的readHost和备用的writeHost
当balance=2,所有的读操作都随机发送到所有的writeHost,readHost上
当balance=3,所有的读操作都只发送到writeHost的readHost上
```
