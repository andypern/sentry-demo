1.Get the sandbox and import that into virtualbox.
MapR-Sandbox-For-Apache-Drill-0.6.0-r2-4.0.1.ova

Login and perform some prep-steps.
Administrators-MacBook-Pro-3:mapr11142014 sthota$ ssh root@localhost -p 2222
mapr

1a.Create unix users user1 and user2.
[root@maprdemo ~]# useradd -u 7001 -g 2002 user1
[root@maprdemo ~]# passwd user1
[root@maprdemo ~]# useradd -u 7002 -g 2002 user2
[root@maprdemo ~]# passwd user2

2.Apply the patch
On Mac:
cd /Users/sthota/mapr11142014/user1 --Change dir to your store
Administrators-MacBook-Pro-3:user1 sthota$ scp -P 2222 * root@localhost:/home/user1/

Login to virtualbox from terminal.
cd /home/user1
http://yum.qa.lab/v4.0.1-patch-EBF/mapr-patch-4.0.1.27334.GA-28668.x86_64.rpm
yum localinstall mapr-patch-4.0.1.27334.GA-28668.x86_64.rpm

3.Change the repository location to point to new location in /etc/yum.repos.d/mapr_ecosystem.repo, remove the existing url and add the following.
baseurl = http://archive.mapr.com/releases/ecosystem-4.x/redhat/
#yum clean all

4.Bring up mysql database and create hive user and give grants.
service mysqld status
service mysqld start
mysql -u root
SET PASSWORD FOR 'root'@'localhost' = PASSWORD('mapr');
SET PASSWORD FOR 'root'@'127.0.0.1' = PASSWORD('mapr');
exit;
mysql -u root -p
#Provide password mapr

create database hive;
CREATE USER 'hive'@'localhost' IDENTIFIED BY 'hive';
CREATE USER 'hive'@'%' IDENTIFIED BY 'hive';
GRANT ALL PRIVILEGES ON hive.* TO 'hive'@'%' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON hive.* TO 'hive'@'localhost' WITH GRANT OPTION;
flush privileges;
exit;

5.Update hive-0.13 to Hive 0.13.0-mapr-1410 and configure it to use mysql as metastore.
hive --version
Stop hiveMeta and hiveServer2 from MCI.
To login to MCI: https://localhost:8443/
Username/passwd: root/mapr
(OR)
[root@maprdemo user1]# maprcli node services -name hs2 -action stop -filter csvc==hs2
[root@maprdemo user1]# maprcli node services -name hivemeta -action stop -filter csvc==hivemeta
[root@maprdemo user1]# maprcli node list -columns hn,svc


# Perform all the steps as root. For info look at this link http://doc.mapr.com/display/MapR/Hive. The following installs hive, hiveserver2, hivemetastore
yum remove mapr-hive
[root@maprdemo user1]# rm -Rf /opt/mapr/hive/
yum install mapr-hivemetastore mapr-hiveserver2 
[root@maprdemo user1]# hive --version
Hive 0.13.0-mapr-1410

6.Configure hive to have metastore to use mysql

6a.[root@maprdemo conf]# ln -s /opt/mapr/lib/mysql-connector-java-5.1.25-bin.jar /opt/mapr/hive/hive-0.13/lib/mysql-connector-java-5.1.25-bin.jar
[root@maprdemo user1]# ls -l /opt/mapr/hive/hive-0.13/lib/mysql-connector-java-5.1.25-bin.jar

6b.export METASTORE_PORT=9083

6c.  cp hive-site.xml.mysql /opt/mapr/hive/hive-0.13/conf/hive-site.xml

6d. Using MCI please restart "HiveMeta" service.

6e.  verify the tables in hive database logging into mysql database in another window.

[root@maprdemo ~]# mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 223
Server version: 5.1.73 Source distribution

mysql> use hive;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+---------------------------+
| Tables_in_hive            |
+---------------------------+
| BUCKETING_COLS            |
| CDS                       |
| VERSION                   |
+---------------------------+
27 rows in set (0.00 sec)

7.Install sentry, perform configuration
#Docs at http://doc.mapr.com/display/MapR/Sentry, http://doc.mapr.com/display/MapR/Configuring+Sentry
7a.[root@maprdemo server]# yum check-update
7b.[root@maprdemo server]# yum install mapr-sentry
7c. cp sentry-site.xml /opt/mapr/sentry/sentry-1.4.0/conf/sentry-site.xml
7d. cp global-policy.ini /opt/mapr/sentry/sentry-1.4.0/conf/global-policy.ini

Install  impala:
# Its requirement is Hbase-0.98, remove hbase-0.94 and install hbase-0.98
[root@maprdemo user1]# yum remove mapr-hbase 
[root@maprdemo user1]# rm -Rf /opt/mapr/hbase/

[root@maprdemo ~]# yum clean all
[root@maprdemo ~]# yum install mapr-hbase-master
[root@maprdemo ~]# yum install mapr-hbase-regionserver

[root@maprdemo ~]# yum install mapr-impala
[root@maprdemo user1]# cp env.sh.nosentry /opt/mapr/impala/impala-1.4.1/conf/env.sh 
[root@maprdemo user1]# yum install mapr-impala-statestore
[root@maprdemo user1]# yum install mapr-impala-catalog
[root@maprdemo user1]# yum install mapr-impala-server 
[root@maprdemo user1]# /opt/mapr/server/configure.sh -R

7h. Restart cluster and verify hive and impala etc.
service mapr-warden stop
service mapr-warden start
[root@maprdemo user1]# maprcli node list -columns hn,svc

[root@maprdemo ~]# hive --service beeline
Beeline version 0.13.0-mapr-1409 by Apache Hive
beeline> !connect jdbc:hive2://127.0.0.1:10000 mapr org.apache.hive.jdbc.HiveDriver
scan complete in 9ms
Connecting to jdbc:hive2://127.0.0.1:10000
Connected to: Apache Hive (version 0.13.0-mapr-1409)
Driver: Hive JDBC (version 0.13.0-mapr-1409)
Transaction isolation: TRANSACTION_REPEATABLE_READ
0: jdbc:hive2://127.0.0.1:10000> create database db1;
No rows affected (0.619 seconds)
0: jdbc:hive2://127.0.0.1:10000> create database db2;
No rows affected (0.038 seconds)
0: jdbc:hive2://127.0.0.1:10000> create database db3;
No rows affected (0.035 seconds)
0: jdbc:hive2://127.0.0.1:10000> show databases;
+----------------+
| database_name  |
+----------------+
| db1            |
| db2            |
| db3            |
| default        |
+----------------+
4 rows selected (0.036 seconds)
0: jdbc:hive2://127.0.0.1:10000> !quit

[root@maprdemo user1]# impala-shell -u mapr
Starting Impala Shell without Kerberos authentication
Connected to maprdemo:21000
Server version: impalad version 1.4.1 RELEASE (build 5c8eee824e46bfeadde72a5552e43e6559b2dc6d)
Welcome to the Impala shell. Press TAB twice to see a list of available commands.

Copyright (c) 2012 Cloudera, Inc. All rights reserved.

(Shell build version: Impala Shell v1.4.1 (5c8eee8) built on Wed Nov 19 10:21:25 PST 2014)
[maprdemo:21000] > show databases;
Query: show databases
+------------------+
| name             |
+------------------+
| _impala_builtins |
| default          |
+------------------+
Returned 2 row(s) in 0.31s
[maprdemo:21000] > invalidate metadata ;
Query: invalidate metadata

Returned 0 row(s) in 1.29s
[maprdemo:21000] > show databases;
Query: show databases
+------------------+
| name             |
+------------------+
| _impala_builtins |
| db1              |
| db2              |
| db3              |
| default          |
+------------------+
Returned 5 row(s) in 0.01s

8.Verify with hive and impala with ruleset in a file using another document.
