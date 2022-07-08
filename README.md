# mysql_repl_repair
mysql_repl_repair是用于修复mysql主从复制错误的python工具，该工具可以修复由于主从数据不一致导致的1062(duplicate key), 1032(key not found)错误。这里有2个文件 mysql_repl_repair.py 与 mysql_repl_repair2.py, 这两个文件都可以解决复制问题，但他们的用法不一样，你可以根据你的实际实际情况来选择使用方式

他们的主要区别在：

mysql_repl_repair.py 必须slave本地执行，靠读取relay log中的数据来构造修复复制所需sql

mysql_repl_repair2.py可在所有网络通的机器上执行，但依赖python-mysql-replication工具(该工具可以模拟从库获取master上的binlog)，通过python-mysql-replication得到的binlog后即可构造修复复制所需的sql

对比来说：

mysql_repl_repair.py 不够便利（只能再从库执行），但安全（不会对主库造成影响， 只需对用户本地授权），无依赖，不支持json geo类型

mysql_repl_repair2.py 便利（中心化管理），但不够安全（需要主、从库对脚本所在机器授权，获取主库binlog对主库有额外开销），依赖python-mysql-replication(需要先下载)，支持json，geo类型，但解析mysql5.6之后的时间(支持微妙)字段时有bug，见 https://github.com/noplay/python-mysql-replication/issues/231 mysql_repl_repair.py没有这个问题

目前网易内部的使用方法：监控服务定期监控mysql主从复制状态，如遇1062,1032 则执行mysql_repl_repair.py进行自动修复

 原理
======
1. 当从库sql apply线程遇到1062错误(唯一键冲突)时，说明主库上的操作在从库上遇到唯一冲突，这个操作可能是insert或者update，那么需要按照 唯一约束键们(可能多个唯一约束) 来删除从库上的数据，如果是insert导致的1062错误，那么解析WRITE_ROWS_EVENT来构造delete语句, 而如果是update导致的1062错误，则需要从这个UPDATE_ROWS_EVENT获取记录的后镜像(update后的数据)，通过后镜像来构造delete语句。最终构造的sql是
```sql
delete from table where (pk_col = xxx ) or (uk1_col1 = xxx and uk1_col2=yyy)
```

如果事务中存在多条insert或者update语句， 那么将构造多条delete语句，而事务中有delete的话，将忽略

2. 当从库sql apply线程遇到1032错误时，说明slave sql线程在执行update或者delete时找不到对应需要变更的数据，那么需要先写入这条数据才行，因为binlog为row模式时变更语句（对应DELETE_ROWS_EVENT或UPDATE_ROWS_EVENT）中包含变更前数据，因此可以构造出这条数据。最终构造的sql是
```sql
replace into table set a=xxx,b=xxx,c=xxx
```
如果事务中包含多条delete/update语句，那么最终需要执行多次 replace操作，而事务中有insert的话，将忽略

 限制
======
* 支持5.1 ~ 5.7，
* 目前只支持ROW格式binlog且为FULL row image格式
* mysql_repl_repair.py不支持修复json，空间数据类型的表造成的复制异常，但mysql_repl_repair2.py支持，mysql_repl_repair2.py解析mysql5.6之后的时间(支持微妙)字段时有bug，已提交issue给python-mysql-replication项目

#### 改进
======
1. 脚本使用pymsql作为mysql 连接库,不使用mysqldb 这个老库
2. 修复了若干个BUG,使用脚本更加稳定

USAGE
===========
```shell
python mysql_repl_repair.py -h

Usage: 
python mysql_repl_repair.py [options]

this script is used to repair mysql replication errors(1062, 1032)

example:
python mysql_repl_repair.py -u mysql -p mysql -S /tmp/mysql.sock  -d -v
python mysql_repl_repair.py -u mysql -p mysql -S /tmp/mysql3306.sock,/tmp/mysql3307.sock -l /tmp


Options:
  -h, --help            show this help message and exit
  -u USER, --user=USER  username for login mysql
  -p PASSWORD, --password=PASSWORD
                        Password to use when connecting to server
  -l LOGDIR, --logdir=LOGDIR
                        log will output to screen by default,if run with
                        daemon mode, default logdir is /tmp, logfile is
                        $logdir/mysql_repl_repair.$port.log
  -S SOCKETS, --socket=SOCKETS
                        mysql sockets for connecting to server, you can input
                        multi socket to repair multi mysql instance, each
                        socket separate by ','
  -d, --daemon          run as a daemon
  -t TIME, --time=TIME  unit is second, default is 0 mean run forever
  -v, --verbose         debug log mode
```

```
python mysql_repl_repair2.py -h
Usage: 
python mysql_repl_repair2.py [options]

this script is used to repair mysql replication errors(1062, 1032)

example:
python mysql_repl_repair2.py -i 192.168.1.1:3306  -u mysql -p mysql -v
python mysql_repl_repair2.py -i 192.168.1.1:3306,192.168.1.2:3306 -u mysql -p mysql -d -l tmp


Options:
  -h, --help            show this help message and exit
  -u USER, --user=USER  username for login mysql instance and its master
  -p PASSWORD, --password=PASSWORD
                        Password to use when connecting to mysql instance and
                        its master
  -l LOGDIR, --logdir=LOGDIR
                        log will output to screen by default,if run with
                        daemon mode, default logdir is /tmp, logfile is
                        $logdir/mysql_repl_repair.$port.log
  -i INSTANCES, --instances=INSTANCES
                        mysql instances which need repair, separate by ','. it
                        will repair all instances store in config file if this
                        option not set
  -d, --daemon          run as a daemon
  -t TIME, --time=TIME  unit is second, default is 0 mean run forever
  -v, --verbose         debug log mode
  ```
 
  
  示例
================
1，授权

a.如果使用的是mysql_repl_repair.py，则授权用户本地权限,如果需要在多实例上同时执行，则每个实例都需要赋权
```sql
grant all on *.* to mysql@'localhost' identified by 'mysql';
```
b.如果使用的是mysql_repl_repair2.py，需要在主从都授权用户执行机器权限,比如 我再 192.168.1.5 上执行脚本修复 192.168.1.7:3306 192.168.1.8:3306复制错误，那么我需要在192.168.1.7:3306 192.168.1.8:3306及其主库上授权
```
grant all on *.* to mysql@'192.168.1.5' identified by 'mysql';
```

2，执行mysql_repl_repair.py脚本，注意：系统用户需要有读relay log的权限，没有的话用sudo
```shell
非debug日志模式：
sudo python mysql_repl_repair.py -u mysql -p mysql --socket=/tmp/mysql3306.sock
[INFO] [2017-09-14 12:49:31,792] [3306] ****************************************************************
[INFO] [2017-09-14 12:49:31,792] [3306]                          PROCESS START
[INFO] [2017-09-14 12:49:31,792] [3306] ****************************************************************
[INFO] [2017-09-14 12:50:21,851] [3306] SLAVE IS OK !, SKIP...
[INFO] [2017-09-14 12:50:23,853] [3306] ****************************************************************
[INFO] [2017-09-14 12:50:23,853] [3306]           REPL ERROR FOUND !!! STRAT REPAIR ERROR...
[INFO] [2017-09-14 12:50:23,853] [3306] ****************************************************************
[INFO] [2017-09-14 12:50:23,853] [3306] RELAYLOG FILE : /mysql_data/mysqld-relay-bin.000075 
[INFO] [2017-09-14 12:50:23,853] [3306] START POSITION : 94820032 . STOP POSITION : 94820392 
[INFO] [2017-09-14 12:50:23,853] [3306] ERROR MESSAGE : Could not execute Update_rows event on table dmy2.mytest; Can't find record in 'mytest', Error_code: 1032; handler error HA_ERR_KEY_NOT_FOUND; the event's master log mysql-bin.000009, end_log_pos 94820228
[INFO] [2017-09-14 12:50:23,853] [3306] start parse relay log to fix this error...
[INFO] [2017-09-14 12:50:23,856] [3306] try to run this sql to resolve repl error, sql: replace into `dmy2`.`mytest` set `a` = 101,`c` = 121,`b` = 10,`e` = x'746467736467',`d` = 51,`g` = x'6433797a736166',`f` = x'3433357479',`i` = '10:10:10.11',`h` = 10.22200,`k` = from_unixtime(1504613928.710),`j` = '1999-1-1',`m` = '1990',`l` = 555,`o` = 3,`n` = 1,`y` = 10.1999998093,`x` = 101.12
[INFO] [2017-09-14 12:50:23,962] [3306] slave repl error fixed success!
[INFO] [2017-09-14 12:50:25,964] [3306] SLAVE IS OK !, SKIP...
[INFO] [2017-09-14 12:50:27,967] [3306] SLAVE IS OK !, SKIP...
^CBye.Bye

开启debug日志：
sudo python mysql_repl_repair.py -u mysql -p mysql --socket=/tmp/mysql3306.sock -v
[INFO] [2017-09-14 12:51:37,191] [3306] ****************************************************************
[INFO] [2017-09-14 12:51:37,191] [3306]                          PROCESS START
[INFO] [2017-09-14 12:51:37,191] [3306] ****************************************************************
[DEBUG] [2017-09-14 12:51:37,191] [3306] get file lock on /tmp/mysql_repl_repair3306.lck success
[DEBUG] [2017-09-14 12:51:37,191] [3306] start run sql: select @@datadir datadir
[DEBUG] [2017-09-14 12:51:37,192] [3306] sql result: {'datadir': '/mysql_data/'}
[DEBUG] [2017-09-14 12:51:37,192] [3306] start run sql: show slave status
[DEBUG] [2017-09-14 12:51:37,192] [3306] sql result: {'Replicate_Wild_Do_Table': '', 'Retrieved_Gtid_Set': '', 'Master_SSL_CA_Path': '', 'Last_Error': "Could not execute Write_rows event on table dmy2.mytest; Duplicate entry '10-11' for key 'uk_bc', Error_code: 1062; handler error HA_ERR_FOUND_DUPP_KEY; the event's master log mysql-bin.000009, end_log_pos 94836423", 'Until_Log_File': '', 'SQL_Delay': 0L, 'Seconds_Behind_Master': None, 'Master_User': 'replicaUser', 'Master_Port': 3306L, 'Master_Retry_Count': 86400L, 'Until_Log_Pos': 0L, 'Master_Log_File': 'mysql-bin.000009', 'Read_Master_Log_Pos': 94837081L, 'Replicate_Do_DB': '', 'Master_SSL_Verify_Server_Cert': 'No', 'Exec_Master_Log_Pos': 94836143L, 'Replicate_Ignore_Server_Ids': '', 'Replicate_Ignore_Table': '', 'Master_Server_Id': 4787L, 'Relay_Log_Space': 94837584L, 'Last_SQL_Error': "Could not execute Write_rows event on table dmy2.mytest; Duplicate entry '10-11' for key 'uk_bc', Error_code: 1062; handler error HA_ERR_FOUND_DUPP_KEY; the event's master log mysql-bin.000009, end_log_pos 94836423", 'SQL_Remaining_Delay': None, 'Relay_Master_Log_File': 'mysql-bin.000009', 'Master_SSL_Allowed': 'No', 'Master_SSL_CA_File': '', 'Slave_IO_State': 'Waiting for master to send event', 'Last_SQL_Error_Timestamp': '170914 12:51:34', 'Relay_Log_File': 'mysqld-relay-bin.000076', 'Replicate_Ignore_DB': '', 'Last_IO_Error': '', 'Until_Condition': 'None', 'Slave_SQL_Running_State': '', 'Replicate_Do_Table': '', 'Last_Errno': 1062L, 'Master_Host': '192.168.1.1000', 'Master_Info_File': '/mysql_data/master.info', 'Master_SSL_Key': '', 'Executed_Gtid_Set': '', 'Master_Bind': '', 'Skip_Counter': 0L, 'Slave_SQL_Running': 'No', 'Relay_Log_Pos': 15960L, 'Master_SSL_Cert': '', 'Last_IO_Errno': 0L, 'Slave_IO_Running': 'Yes', 'Connect_Retry': 60L, 'Last_SQL_Errno': 1062L, 'Last_IO_Error_Timestamp': '', 'Replicate_Wild_Ignore_Table': '', 'Master_UUID': 'b4dfc344-975c-11e6-addd-fa163e7f8534', 'Auto_Position': 0L, 'Master_SSL_Crl': '', 'Master_SSL_Cipher': '', 'Master_SSL_Crlpath': ''}
[INFO] [2017-09-14 12:51:37,192] [3306] ****************************************************************
[INFO] [2017-09-14 12:51:37,192] [3306]           REPL ERROR FOUND !!! STRAT REPAIR ERROR...
[INFO] [2017-09-14 12:51:37,193] [3306] ****************************************************************
[INFO] [2017-09-14 12:51:37,193] [3306] RELAYLOG FILE : /mysql_data/mysqld-relay-bin.000076 
[INFO] [2017-09-14 12:51:37,193] [3306] START POSITION : 15960 . STOP POSITION : 16240 
[INFO] [2017-09-14 12:51:37,193] [3306] ERROR MESSAGE : Could not execute Write_rows event on table dmy2.mytest; Duplicate entry '10-11' for key 'uk_bc', Error_code: 1062; handler error HA_ERR_FOUND_DUPP_KEY; the event's master log mysql-bin.000009, end_log_pos 94836423
[INFO] [2017-09-14 12:51:37,193] [3306] start parse relay log to fix this error...
[DEBUG] [2017-09-14 12:51:37,195] [3306] read column for dmy2.mytest, column: a, data type: int, read bytes 4, column value: 100
[DEBUG] [2017-09-14 12:51:37,195] [3306] read column for dmy2.mytest, column: b, data type: smallint, read bytes 2, column value: 10
[DEBUG] [2017-09-14 12:51:37,195] [3306] read column for dmy2.mytest, column: c, data type: int, read bytes 4, column value: 11
[DEBUG] [2017-09-14 12:51:37,195] [3306] read column for dmy2.mytest, column: d, data type: mediumint, read bytes 3, column value: 10
[DEBUG] [2017-09-14 12:51:37,195] [3306] read column for dmy2.mytest, column: y, data type: float, read bytes 4, column value: 10.1999998093
[DEBUG] [2017-09-14 12:51:37,195] [3306] read column for dmy2.mytest, column: x, data type: double, read bytes 8, column value: 10.1
[DEBUG] [2017-09-14 12:51:37,196] [3306] read column for dmy2.mytest, column: e, data type: varchar, read bytes 8, column value: x'746467736467'
[DEBUG] [2017-09-14 12:51:37,196] [3306] read column for dmy2.mytest, column: f, data type: char, read bytes 6, column value: x'3433357479'
[DEBUG] [2017-09-14 12:51:37,196] [3306] read column for dmy2.mytest, column: g, data type: text, read bytes 9, column value: x'6433797a736166'
[DEBUG] [2017-09-14 12:51:37,196] [3306] read column for dmy2.mytest, column: h, data type: decimal, read bytes 6, column value: 10.22200
[DEBUG] [2017-09-14 12:51:37,196] [3306] read column for dmy2.mytest, column: i, data type: time, read bytes 4, column value: '10:10:10.11'
[DEBUG] [2017-09-14 12:51:37,197] [3306] read column for dmy2.mytest, column: j, data type: date, read bytes 3, column value: '1999-1-1'
[DEBUG] [2017-09-14 12:51:37,197] [3306] read column for dmy2.mytest, column: k, data type: timestamp, read bytes 6, column value: from_unixtime(1504235903.086)
[DEBUG] [2017-09-14 12:51:37,197] [3306] read column for dmy2.mytest, column: l, data type: bigint, read bytes 8, column value: 555
[DEBUG] [2017-09-14 12:51:37,197] [3306] read column for dmy2.mytest, column: m, data type: year, read bytes 1, column value: '1990'
[DEBUG] [2017-09-14 12:51:37,198] [3306] read column for dmy2.mytest, column: n, data type: enum, read bytes 1, column value: 1
[DEBUG] [2017-09-14 12:51:37,198] [3306] read column for dmy2.mytest, column: o, data type: set, read bytes 1, column value: 3
[DEBUG] [2017-09-14 12:51:37,198] [3306] filename: /mysql_data/mysqld-relay-bin.000076,start_pos: 16236,rowdata: {'table_name': 'mytest', 'table_schema': 'dmy2', 'data': {'a': 100, 'c': 11, 'b': 10, 'e': "x'746467736467'", 'd': 10, 'g': "x'6433797a736166'", 'f': "x'3433357479'", 'i': "'10:10:10.11'", 'h': Decimal('10.22200'), 'k': 'from_unixtime(1504235903.086)', 'j': "'1999-1-1'", 'm': "'1990'", 'l': 555, 'o': 3, 'n': 1, 'y': 10.199999809265137, 'x': 10.1}, 'event_type': 30, 'data2': {}}
[INFO] [2017-09-14 12:51:37,198] [3306] try to run this sql to resolve repl error, sql: delete from `dmy2`.`mytest` where  ( `a` = 100 ) or ( `b` = 10 and `c` = 11 ) 
[DEBUG] [2017-09-14 12:51:37,199] [3306] start run sql: delete from `dmy2`.`mytest` where  ( `a` = 100 ) or ( `b` = 10 and `c` = 11 ) 
[DEBUG] [2017-09-14 12:51:37,200] [3306] sql result: None
[DEBUG] [2017-09-14 12:51:37,200] [3306] start run sql: stop slave;
[DEBUG] [2017-09-14 12:51:37,202] [3306] sql result: None
[DEBUG] [2017-09-14 12:51:37,202] [3306] start run sql: start slave
[DEBUG] [2017-09-14 12:51:37,205] [3306] sql result: None
[DEBUG] [2017-09-14 12:51:37,306] [3306] start run sql: show slave status
[DEBUG] [2017-09-14 12:51:37,306] [3306] sql result: {'Replicate_Wild_Do_Table': '', 'Retrieved_Gtid_Set': '', 'Master_SSL_CA_Path': '', 'Last_Error': '', 'Until_Log_File': '', 'SQL_Delay': 0L, 'Seconds_Behind_Master': 0L, 'Master_User': 'replicaUser', 'Master_Port': 3306L, 'Master_Retry_Count': 86400L, 'Until_Log_Pos': 0L, 'Master_Log_File': 'mysql-bin.000009', 'Read_Master_Log_Pos': 94837081L, 'Replicate_Do_DB': '', 'Master_SSL_Verify_Server_Cert': 'No', 'Exec_Master_Log_Pos': 94837081L, 'Replicate_Ignore_Server_Ids': '', 'Replicate_Ignore_Table': '', 'Master_Server_Id': 4787L, 'Relay_Log_Space': 17237L, 'Last_SQL_Error': '', 'SQL_Remaining_Delay': None, 'Relay_Master_Log_File': 'mysql-bin.000009', 'Master_SSL_Allowed': 'No', 'Master_SSL_CA_File': '', 'Slave_IO_State': 'Waiting for master to send event', 'Last_SQL_Error_Timestamp': '', 'Relay_Log_File': 'mysqld-relay-bin.000077', 'Replicate_Ignore_DB': '', 'Last_IO_Error': '', 'Until_Condition': 'None', 'Slave_SQL_Running_State': 'Slave has read all relay log; waiting for the slave I/O thread to update it', 'Replicate_Do_Table': '', 'Last_Errno': 0L, 'Master_Host': '192.168.1.1000', 'Master_Info_File': '/mysql_data/master.info', 'Master_SSL_Key': '', 'Executed_Gtid_Set': '', 'Master_Bind': '', 'Skip_Counter': 0L, 'Slave_SQL_Running': 'Yes', 'Relay_Log_Pos': 285L, 'Master_SSL_Cert': '', 'Last_IO_Errno': 0L, 'Slave_IO_Running': 'Yes', 'Connect_Retry': 60L, 'Last_SQL_Errno': 0L, 'Last_IO_Error_Timestamp': '', 'Replicate_Wild_Ignore_Table': '', 'Master_UUID': 'b4dfc344-975c-11e6-addd-fa163e7f8534', 'Auto_Position': 0L, 'Master_SSL_Crl': '', 'Master_SSL_Cipher': '', 'Master_SSL_Crlpath': ''}
[INFO] [2017-09-14 12:51:37,307] [3306] slave repl error fixed success!
```
3，执行mysql_repl_repair2.py
```
python mysql_repl_repair2.py -u mysql -p mysql -i 192.168.1.111:3306 -v
[DEBUG] [2017-09-26 21:05:08,073] [192.168.1.111.3306] get file lock on /tmp/mysql_repl_repair.192.168.1.111.3306.lck success
[DEBUG] [2017-09-26 21:05:08,073] [192.168.1.111.3306] start run sql: show slave status
[DEBUG] [2017-09-26 21:05:08,074] [192.168.1.111.3306] sql result: {'Replicate_Wild_Do_Table': '', 'Retrieved_Gtid_Set': '', 'Master_SSL_CA_Path': '', 'Last_Error': "Could not execute Update_rows event on table dmy2.mytest; Can't find record in 'mytest', Error_code: 1032; handler error HA_ERR_KEY_NOT_FOUND; the event's master log mysql-bin.000011, end_log_pos 20190669", 'Until_Log_File': '', 'SQL_Delay': 0L, 'Seconds_Behind_Master': None, 'Master_User': 'replicaUser', 'Master_Port': 3306L, 'Master_Retry_Count': 86400L, 'Until_Log_Pos': 0L, 'Master_Log_File': 'mysql-bin.000011', 'Read_Master_Log_Pos': 20191536L, 'Replicate_Do_DB': '', 'Master_SSL_Verify_Server_Cert': 'No', 'Exec_Master_Log_Pos': 20190309L, 'Replicate_Ignore_Server_Ids': '', 'Replicate_Ignore_Table': '', 'Master_Server_Id': 4787L, 'Relay_Log_Space': 70376L, 'Last_SQL_Error': "Could not execute Update_rows event on table dmy2.mytest; Can't find record in 'mytest', Error_code: 1032; handler error HA_ERR_KEY_NOT_FOUND; the event's master log mysql-bin.000011, end_log_pos 20190669", 'SQL_Remaining_Delay': None, 'Relay_Master_Log_File': 'mysql-bin.000011', 'Master_SSL_Allowed': 'No', 'Master_SSL_CA_File': '', 'Slave_IO_State': 'Waiting for master to send event', 'Last_SQL_Error_Timestamp': '170926 21:05:05', 'Relay_Log_File': 'mysqld-relay-bin.000129', 'Replicate_Ignore_DB': '', 'Last_IO_Error': '', 'Until_Condition': 'None', 'Slave_SQL_Running_State': '', 'Replicate_Do_Table': '', 'Last_Errno': 1032L, 'Master_Host': '192.168.1.112', 'Master_Info_File': '/ebs/mysql_data/master.info', 'Master_SSL_Key': '', 'Executed_Gtid_Set': '', 'Master_Bind': '', 'Skip_Counter': 0L, 'Slave_SQL_Running': 'No', 'Relay_Log_Pos': 24320L, 'Master_SSL_Cert': '', 'Last_IO_Errno': 0L, 'Slave_IO_Running': 'Yes', 'Connect_Retry': 60L, 'Last_SQL_Errno': 1032L, 'Last_IO_Error_Timestamp': '', 'Replicate_Wild_Ignore_Table': '', 'Master_UUID': 'b4dfc344-975c-11e6-addd-fa163e7f8534', 'Auto_Position': 0L, 'Master_SSL_Crl': '', 'Master_SSL_Cipher': '', 'Master_SSL_Crlpath': ''}
[INFO] [2017-09-26 21:05:08,075] [192.168.1.111.3306] ****************************************************************
[INFO] [2017-09-26 21:05:08,075] [192.168.1.111.3306]           REPL ERROR FOUND !!! STRAT REPAIR ERROR...
[INFO] [2017-09-26 21:05:08,075] [192.168.1.111.3306] ****************************************************************
[INFO] [2017-09-26 21:05:08,075] [192.168.1.111.3306] MASTER HOST: 192.168.1.112, MASTER PORT: 3306
[INFO] [2017-09-26 21:05:08,075] [192.168.1.111.3306] MASTERLOG FILE : mysql-bin.000011 
[INFO] [2017-09-26 21:05:08,075] [192.168.1.111.3306] START POSITION : 20190309, STOP POSITION : 20190669 
[INFO] [2017-09-26 21:05:08,075] [192.168.1.111.3306] ERROR MESSAGE : Could not execute Update_rows event on table dmy2.mytest; Can't find record in 'mytest', Error_code: 1032; handler error HA_ERR_KEY_NOT_FOUND; the event's master log mysql-bin.000011, end_log_pos 20190669
[INFO] [2017-09-26 21:05:08,075] [192.168.1.111.3306] start to dump master binlog to fix this error...
[DEBUG] [2017-09-26 21:05:08,082] [192.168.1.111.3306] event data: {'data': {u'a': 9, u'c': 101, u'b': 1, u'e': u'aasfsd', u'd': 1021, u'g': u'd3yzsaf', u'f': u'435ty', u'i': datetime.timedelta(0, 36610, 11), u'h': Decimal('10.22200'), u'k': datetime.datetime(2017, 9, 26, 21, 3, 14, 165), u'j': datetime.date(1999, 1, 1), u'm': 1990, u'l': 555, u'o': set([u'a', u'b']), u'n': u'a', u'y': 10.010000228881836, u'x': 10.3}, 'table_schema': u'dmy2', 'table_name': u'mytest', 'event_type': 'update', 'data2': {u'a': 9, u'c': 101, u'b': 1, u'e': u'aasfsd', u'd': 102, u'g': u'd3yzsaf', u'f': u'435ty', u'i': datetime.timedelta(0, 36610, 11), u'h': Decimal('10.22200'), u'k': datetime.datetime(2017, 9, 26, 21, 5, 5, 257), u'j': datetime.date(1999, 1, 1), u'm': 1990, u'l': 555, u'o': set([u'a', u'b']), u'n': u'a', u'y': 10.010000228881836, u'x': 10.3}}
[INFO] [2017-09-26 21:05:08,083] [192.168.1.111.3306] try to run this sql to resolve repl error, sql: replace into `dmy2`.`mytest` set `a` = 9,`c` = 101,`b` = 1,`e` = 'aasfsd',`d` = 1021,`g` = 'd3yzsaf',`f` = '435ty',`i` = '10:10:10.000011',`h` = 10.22200,`k` = '2017-09-26 21:03:14.000165',`j` = '1999-01-01',`m` = 1990,`l` = 555,`o` = "a,b",`n` = 'a',`y` = 10.0100002289,`x` = 10.3
[DEBUG] [2017-09-26 21:05:08,083] [192.168.1.111.3306] start run sql: replace into `dmy2`.`mytest` set `a` = 9,`c` = 101,`b` = 1,`e` = 'aasfsd',`d` = 1021,`g` = 'd3yzsaf',`f` = '435ty',`i` = '10:10:10.000011',`h` = 10.22200,`k` = '2017-09-26 21:03:14.000165',`j` = '1999-01-01',`m` = 1990,`l` = 555,`o` = "a,b",`n` = 'a',`y` = 10.0100002289,`x` = 10.3
[DEBUG] [2017-09-26 21:05:08,086] [192.168.1.111.3306] sql result: None
[DEBUG] [2017-09-26 21:05:08,086] [192.168.1.111.3306] start run sql: stop slave;
[DEBUG] [2017-09-26 21:05:08,091] [192.168.1.111.3306] sql result: None
[DEBUG] [2017-09-26 21:05:08,091] [192.168.1.111.3306] start run sql: start slave
[DEBUG] [2017-09-26 21:05:08,093] [192.168.1.111.3306] sql result: None
[DEBUG] [2017-09-26 21:05:08,193] [192.168.1.111.3306] start run sql: show slave status
[DEBUG] [2017-09-26 21:05:08,194] [192.168.1.111.3306] sql result: {'Replicate_Wild_Do_Table': '', 'Retrieved_Gtid_Set': '', 'Master_SSL_CA_Path': '', 'Last_Error': '', 'Until_Log_File': '', 'SQL_Delay': 0L, 'Seconds_Behind_Master': 0L, 'Master_User': 'replicaUser', 'Master_Port': 3306L, 'Master_Retry_Count': 86400L, 'Until_Log_Pos': 0L, 'Master_Log_File': 'mysql-bin.000011', 'Read_Master_Log_Pos': 20191536L, 'Replicate_Do_DB': '', 'Master_SSL_Verify_Server_Cert': 'No', 'Exec_Master_Log_Pos': 20191536L, 'Replicate_Ignore_Server_Ids': '', 'Replicate_Ignore_Table': '', 'Master_Server_Id': 4787L, 'Relay_Log_Space': 25886L, 'Last_SQL_Error': '', 'SQL_Remaining_Delay': None, 'Relay_Master_Log_File': 'mysql-bin.000011', 'Master_SSL_Allowed': 'No', 'Master_SSL_CA_File': '', 'Slave_IO_State': 'Waiting for master to send event', 'Last_SQL_Error_Timestamp': '', 'Relay_Log_File': 'mysqld-relay-bin.000130', 'Replicate_Ignore_DB': '', 'Last_IO_Error': '', 'Until_Condition': 'None', 'Slave_SQL_Running_State': 'Slave has read all relay log; waiting for the slave I/O thread to update it', 'Replicate_Do_Table': '', 'Last_Errno': 0L, 'Master_Host': '192.168.1.112', 'Master_Info_File': '/ebs/mysql_data/master.info', 'Master_SSL_Key': '', 'Executed_Gtid_Set': '', 'Master_Bind': '', 'Skip_Counter': 0L, 'Slave_SQL_Running': 'Yes', 'Relay_Log_Pos': 285L, 'Master_SSL_Cert': '', 'Last_IO_Errno': 0L, 'Slave_IO_Running': 'Yes', 'Connect_Retry': 60L, 'Last_SQL_Errno': 0L, 'Last_IO_Error_Timestamp': '', 'Replicate_Wild_Ignore_Table': '', 'Master_UUID': 'b4dfc344-975c-11e6-addd-fa163e7f8534', 'Auto_Position': 0L, 'Master_SSL_Crl': '', 'Master_SSL_Cipher': '', 'Master_SSL_Crlpath': ''}
[INFO] [2017-09-26 21:05:08,194] [192.168.1.111.3306] slave repl error fixed success!
^CBye.Bye
```
4，结束

如果不是以daemon方式运行，那么只需要Ctrl+C即可结束，如果是以daemon方式，直接kill进程即可

Q&A
======
1.mysql支持slave_exec_mode=IDEMPOTENT，来跳过复制错误，为啥还要找个工具?

slave_exec_mode为IDEMPOTENT时，从库slave的表现是遇到insert出错时replace，遇到update、delete出错时时跳过，本工具的做法insert与delete与之效果一样，差别在于update, 从库跳过update的话，数据相当于丢失，本工具会先插入update前的数据，复制修复成功后数据不会丢失

2.这个工具支持远程执行吗？

支持，请选择使用 mysql_repl_repair2.py

3.这个工具支持mysql5.7的GTID吗？

支持

感谢
======
代码大部分解析binlog的函数代码都是参考https://github.com/noplay/python-mysql-replication
在此非常感谢python-mysql-replication的作者们的付出

作者
=====
杜明友、赵天元,许绍惠

问题反馈方式
================
* 提issue
