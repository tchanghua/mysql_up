1. 集群架构要求
    1主，2从，1MHA
    首先实现一主两从的同步复制功能（采用半同步复制机制及并行复制）
    先查询主从库数据
    主库添加数据后，再次查询主从库数据
    查看log，是否显示半同步
    然后采用MHA实现主机出故障，从库能自动切换功能
    关闭主服务器，查看MHA服务器是否正常切换
    在切换后的主节点机器上查看状态展示效果
    在切换后的主节点机器数据库添加数据，分别查询主从库数据一致
    
2. 环境软件版本及机器介绍
    centos-7.6
    mysql-5.7.31    

3. 环境安装过程
3.1 在各个服务器分别安装mysql
安装步骤可以参考：mysql-5.7.31安装

3.2 master配置
编辑配置文件my.cnf，开启binlog，指定需要同步数据库和不同步的数据库
# 开启binlog
log_bin=mysql-bin
# 主库的server-id
server-id=1  
# 每次提交事务都刷盘
sync-binlog=1
# 指定哪些库不同步,不设置全部同步
binlog-ignore-db=information_schema
binlog-ignore-db=mysql
binlog-ignore-db=performance_schema
binlog-ignore-db=sys
# 指定同步哪些库
binlog-do-db=test

binlog刷盘策略

sync_binlog=0 的时候，表示每次提交事务都只 write，不 fsync；

sync_binlog=1 的时候，表示每次提交事务都会执行 fsync；

sync_binlog=N(N>1) 的时候，表示每次提交事务都 write，但累积 N 个事务后才 fsync。

write，指的就是指把日志写入到文件系统的 page cache，并没有把数据持久化到磁盘，所以速度比较快。

fsync，才是将数据持久化到磁盘的操作。一般情况下，我们认为 fsync 才占磁盘的 IOPS。

查看是否开启binlog
mysql> show variables like '%log_bin%';
+---------------------------------+-------+
| Variable_name                   | Value |
+---------------------------------+-------+
| log_bin                         | OFF   |
| log_bin_basename                |       |
| log_bin_index                   |       |
| log_bin_trust_function_creators | OFF   |
| log_bin_use_v1_row_events       | OFF   |
| sql_log_bin                     | ON    |
+---------------------------------+-------+

重启服务
service mysql restart
1
可执行上面命令的前提是先执行下面的操作

cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysql
1
查看binlog相关配置
mysql> show variables like '%log_bin%';
+---------------------------------+---------------------------------------+
| Variable_name                   | Value                                 |
+---------------------------------+---------------------------------------+
| log_bin                         | ON                                    |
| log_bin_basename                | /usr/local/mysql/data/mysql-bin       |
| log_bin_index                   | /usr/local/mysql/data/mysql-bin.index |
| log_bin_trust_function_creators | OFF                                   |
| log_bin_use_v1_row_events       | OFF                                   |
| sql_log_bin                     | ON                                    |
+---------------------------------+---------------------------------------+

授权同步数据的从数据库的用户
grant replication slave on *.* to 'root'@'%' identified by 'root';
# 刷新权限
flush privileges;

3.3 slave配置及同步数据
编辑配置文件my.cnf
# server id
server-id=2
# 中继日志名称
relay_log=mysql-relay-bin
# 设置为只读权限
read_only=1
重启服务
service mysql restart
查看同步状态
mysql> show slave status;
Empty set (0.00 sec)
同上面的显示才表示还没有开始同步数据

查看主节点同步位置
mysql> show master status;
+------------------+----------+--------------+-------------------------------------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB                                | Executed_Gtid_Set |
+------------------+----------+--------------+-------------------------------------------------+-------------------+
| mysql-bin.000003 |     154 | test         | information_schema,mysql,performance_schema,sys |                   |
+------------------+----------+--------------+-------------------------------------------------+-------------------+
1 row in set (0.00 sec)

初始化同步信息
change master to master_host='47.94.80.41',master_port=3306,master_user='root',master_password='root',master_log_file='mysql-bin.000003',master_log_pos=154;
master_host 主库ip
master_port 主库端口
master_user主库用户
master_password 主库用户对应的密码
master_log_file 主库binlog日志文件名称
master_log_pos 主库binlog日志的开始同步的位移点position
开始同步：start slave;
查看同步状态：show slave status;

3.5 半同步复制配置
3.5.1 主库配置
    查看主库是否安装相应的插件，若没有安装相应的插件。
3.5.2 从库配置
    与主库配置基本相同，修改相应的配置。
 
 3.6 MHA安装配置
3.6.1 MHA 工作原理
MHA软件由两部分组成，Manager工具包和Node工具包，manager就是我们架构中说的monitor，node相当于监控节点，后续每台mysql机器上都需要装。

Manager工具包介绍：
masterha_check_ssh              检查MHA的SSH配置状况
masterha_check_repl             检查MySQL复制状况
masterha_manger                 启动MHA
masterha_check_status           检测当前MHA运行状态
masterha_master_monitor         检测master是否宕机
masterha_master_switch          控制故障转移（自动或者手动）
masterha_conf_host              添加或删除配置的server信息

Node工具包介绍（这些工具通常由MHA Manager的脚本触发，无需人为操作）
save_binary_logs                保存和复制master的二进制日志
apply_diff_relay_logs           识别差异的中继日志事件并将其差异的事件应用于其他的slave
filter_mysqlbinlog              去除不必要的ROLLBACK事件（MHA已不再使用这个工具）
purge_relay_logs                清除中继日志（不会阻塞SQL线程）

3.6.2 每个服务器下载安装node和manager
3.6.3 安装完node和manager后，各服务器之间配置ssh互认（免登陆认证）
