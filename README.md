```
系统：  ubuntu20
mysql 版本： mysql8.0
mha版本 mha4mysql-manager_0.58-0_all.deb   mha4mysql-node_0.58-0_all.deb
mha官方网址： https://github.com/yoshinorim/
  
  
  PS： 使用root 部署的。
```
#mysql主从配置
```
  1. 主机设计
  172.16.0.101   master  ，VIP（172.16.0.107）
  172.16.0.100   slave (backup master)
  172.16.0.102   slave and  mha manager
  
  2. 修改hosts文件（3台机器都执行），新增以下内容
  vim   etc/hosts
  172.16.0.101 mha1
  172.16.0.100 mha2
  172.16.0.102 mha3
  3. 修改机器名称 （3台分别执行）
  sudo hostnamectl   set-hostname   mha1
  sudo hostnamectl   set-hostname   mha2
  sudo hostnamectl   set-hostname   mha3
  4. 配置节点互信，即ssh免密登录 （3台机器全部执行）
  ssh-keygen -t rsa   一路回车即可
  ssh-copy-id 172.16.0.101
  ssh-copy-id 172.16.0.100
  ssh-copy-id 172.16.0.102
  
  验证： 3台机器都做，（每执行一次ssh,操作完成后记得exit）
  ssh root@172.16.0.101
  ssh root@172.16.0.102
  ssh root@172.16.0.103
  ssh root@mha1
  ssh root@mha2
  ssh root@mha3
  
  5. mysql  主从配置
  a. master节点 172.16.0.101： vim /etc/mysql/mysql.conf.d/mysqld.cnf  最后新增：
  server-id=1                   #  每个节点不能相同
  #log_bin=master-bin  # 不写路径默认在目录下
  #relay-log=master-relay-bin  # 不写路径默认在目录下
  log_bin=slave-bin  # 不写路径默认在目录下
  relay-log=slave-relay-bin  # 不写路径默认在目录下
  skip-name-resolve              #  建议加上 非必须项
  relay_log_purge = 0            #  关闭自动清理中继日志
  log_slave_updates = 1          #  从库通过binlog更新的数据写进从库二进制日志中，必加，否则切换后可能丢失数据
  binlog_format=row
  gtid-mode=on                        # gtid开关
  enforce-gtid-consistency=true       # 强制GTID一致
  
  b. slave 节点（ backup master节点 mha2） 172.16.0.100  vim /etc/mysql/mysql.conf.d/mysqld.cnf  最后新增：
  server-id=2                  #  每个节点不能相同
  log_bin=slave-bin  # 不写路径默认在目录下
  relay-log=slave-relay-bin  # 不写路径默认在目录下
  skip-name-resolve              #  建议加上 非必须项
  relay_log_purge = 0            #  关闭自动清理中继日志
  log_slave_updates = 1          #  从库通过binlog更新的数据写进从库二进制日志中，必加，否则切换后可能丢失数据
  binlog_format=row
  gtid-mode=on                        # gtid开关
  enforce-gtid-consistency=true       # 强制GTID一致
  read_only = 1
  
  c. slave 节点 172.16.0.102  vim /etc/mysql/mysql.conf.d/mysqld.cnf  最后新增：
  
  server-id=3                  #  每个节点不能相同
  log_bin=slave-bin  # 不写路径默认在目录下
  relay-log=slave-relay-bin  # 不写路径默认在目录下
  skip-name-resolve              #  建议加上 非必须项
  relay_log_purge = 0            #  关闭自动清理中继日志
  log_slave_updates = 1          #  从库通过binlog更新的数据写进从库二进制日志中，必加，否则切换后可能丢失数据
  binlog_format=row
  gtid-mode=on                        # gtid开关
  enforce-gtid-consistency=true       # 强制GTID一致
  read_only = 1
  
  执行完成后 ，systemctl restart  mysql.service  ；systemctl status  mysql.service
  
  6. 为主从复制创建账号
  master:
  mysql -u root -p  输入root密码
  CREATE USER 'replication'@'%' IDENTIFIED WITH mysql_native_password BY 'replication';
  GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
  FLUSH PRIVILEGES;
  show  master  status;
  exit;
```
  ![image](https://github.com/sky1230/mysql-mha/assets/8722731/322667c6-59b0-428a-8288-47241895125f)
```
  其他2个节点：
  mysql -u root -p
STOP SLAVE;
CHANGE MASTER TO MASTER_HOST='主服务器IP地址', MASTER_USER='replication', MASTER_PASSWORD='replication', MASTER_LOG_FILE='主服务器的二进制日志文件名', MASTER_LOG_POS=主服务器的二进制日志位置;
START SLAVE;
exit;

以上执行完成，在3台机器都执行，systemctl  restart  mysql 
ps: MASTER_LOG_FILE  ，MASTER_LOG_POS 就是在master节点执行的 show master  status;  file 和Position 的值。


在主从服务器上执行:
sudo systemctl start mysql

CREATE DATABASE test;
use  test;

CREATE TABLE myinfo(
  id INT NOT NULL AUTO_INCREMENT,
  name VARCHAR(255),
  age INT,
  PRIMARY KEY (id)
);

INSERT INTO   myinfo(name, age) VALUES
  ('John Doe', 20),
  ('Jane Doe', 21);


查询结果： 
select  * from myinfo;
在主库上执行 

 INSERT INTO myinfo(name, age) VALUES('master',1);

delete from myinfo  where  id=3

update myinfoset  age=100  where id=2
主要主库发生删除和修改，就在从库上执行： 
select  * from  myinfo;

如果在主库上执行增删改，从库数据保持一致表示，mysql 主从配置完成。
```

#安装MHA

```
# 安装MHA

https://github.com/yoshinorim/mha4mysql-manager/wiki

## 所有节点都需要安装node

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/64e9ef9b-f09a-4e88-a3a5-9a1914824b4c/Untitled.png)

apt-get install libmysqlclient21 libdbi-perl libdbd-mysql-perl

dpkf -i   安装

dpkg -L  查看安装信息

dpkg -s  验证是否安装成功

## 安装MHA Manager  deb软件包

```
## Install dependent Perl modules
# apt-get install libdbd-mysql-perl
# apt-get install libconfig-tiny-perl
# apt-get install liblog-dispatch-perl
# apt-get install libparallel-forkmanager-perl
```

`dpkg -i mha4mysql-manager_X.Y_all.deb`

以上，完成了 MHA 安装。接下来要进行配置和检查已经切换vip

PS :  安装依赖包，不断出现缺少依赖，就执行 apt-get install  -f 

# 配置MHA

## a 在master节点即****主库上添加mha管理账号（主库会自动同步到从库）   — 如果从库mysql表中，没有复制账号 就表示有问题。****

```jsx
use  msyql;
CREATE USER 'mha'@'172.16.0.%' IDENTIFIED WITH mysql_native_password  BY 'mha';
GRANT ALL PRIVILEGES ON *.* TO 'mha'@'172.16.0.%';

任何权限的更新，都需要执行：
FLUSH PRIVILEGES;

create user 'mha'@'localhost' identified  WITH mysql_native_password  by 'mha';
GRANT ALL PRIVILEGES ON *.* TO 'mha'@'localhost';
FLUSH PRIVILEGES;

use  msyql;
CREATE USER 'repl666'@'172.16.0.%' IDENTIFIED WITH mysql_native_password  BY 'repl666';
GRANT ALL PRIVILEGES ON *.* TO 'repl666'@'172.16.0.%';

任何权限的更新，都需要执行：
FLUSH PRIVILEGES;

create user 'repl666'@'localhost' identified  WITH mysql_native_password  by 'repl666';
GRANT ALL PRIVILEGES ON *.* TO 'repl666'@'localhost';
FLUSH PRIVILEGES;

```

# 特别注意： 复制账号和 mha manager 管理账号 mha 都必须加上 具体代码段和  localhost权限，否则导致后续很多很多麻烦。

验证，创建的账号是否同步到其他节点

`select host, user, authentication_string, plugin from user;
执行完上面的命令后会显示一个表格 @
查看表格中 root 用户的 host，默认应该显示的 localhost，只支持本地访问，不允许远程访问。`

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/180b485f-a70f-4eee-bfca-204f4fe488e2/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c154a0b7-fc05-47f1-bc0a-c195564f4134/Untitled.png)

## b 在mha manager 节点上配置

参数配置参考：https://raw.githubusercontent.com/wiki/yoshinorim/mha4mysql-manager/Parameters.md

```jsx
mkdir -p /etc/mha #创建必须目录
mkdir -p /var/log/mha/app1 #创建日志，目录可以管理多套主从复制

vim /etc/mha/app1.cnf
```

目前mha 密码 mha

repl666  repl666

```jsx
[server default]
manager_workdir=/var/log/masterha/app1
manager_log=/var/log/masterha/app1/manager.log
master_binlog_dir=/var/lib/mysql,/var/log/mysql
ping_interval=1
repl_password=repl666
repl_user=repl666
secondary_check_script=/usr/bin/masterha_secondary_check -s mha2 -s mha3  --user=root --master_host=mha1 --master_ip=172.16.0.101 --master_port=3306
shutdown_script=""
ssh_user=root
user=mha
password=mha
#master_ip_failover_script=/usr/bin/master_ip_failover
[server1]
hostname=172.16.0.101
ssh_port=22

[server2]
candidate_master=1
check_repl_delay=0
hostname=172.16.0.100
port=3306

[server3]
hostname=172.16.0.102
port=3306
```

# 使用mha测试ssh

```jsx
masterha_check_ssh --conf=/etc/mha/app1.cnf
```

root@mha3:~# masterha_check_ssh   --conf=/etc/masterha/app2.cnf
Thu Aug 17 08:39:50 2023 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Thu Aug 17 08:39:50 2023 - [info] Reading application default configuration from /etc/masterha/app2.cnf..
Thu Aug 17 08:39:50 2023 - [info] Reading server configuration from /etc/masterha/app2.cnf..
Thu Aug 17 08:39:50 2023 - [info] Starting SSH connection tests..
Thu Aug 17 08:39:52 2023 - [debug]
Thu Aug 17 08:39:50 2023 - [debug]  Connecting via SSH from [root@172.16.0.101](mailto:root@172.16.0.101)(172.16.0.101:22) to [root@172.16.0.100](mailto:root@172.16.0.100)(172.16.0.100:22)..
Thu Aug 17 08:39:51 2023 - [debug]   ok.
Thu Aug 17 08:39:51 2023 - [debug]  Connecting via SSH from [root@172.16.0.101](mailto:root@172.16.0.101)(172.16.0.101:22) to [root@172.16.0.102](mailto:root@172.16.0.102)(172.16.0.102:22)..
Thu Aug 17 08:39:52 2023 - [debug]   ok.
Thu Aug 17 08:39:53 2023 - [debug]
Thu Aug 17 08:39:51 2023 - [debug]  Connecting via SSH from [root@172.16.0.102](mailto:root@172.16.0.102)(172.16.0.102:22) to [root@172.16.0.101](mailto:root@172.16.0.101)(172.16.0.101:22)..
Thu Aug 17 08:39:52 2023 - [debug]   ok.
Thu Aug 17 08:39:52 2023 - [debug]  Connecting via SSH from [root@172.16.0.102](mailto:root@172.16.0.102)(172.16.0.102:22) to [root@172.16.0.100](mailto:root@172.16.0.100)(172.16.0.100:22)..
Thu Aug 17 08:39:53 2023 - [debug]   ok.
Thu Aug 17 08:39:53 2023 - [debug]
Thu Aug 17 08:39:50 2023 - [debug]  Connecting via SSH from [root@172.16.0.100](mailto:root@172.16.0.100)(172.16.0.100:22) to [root@172.16.0.101](mailto:root@172.16.0.101)(172.16.0.101:22)..
Thu Aug 17 08:39:51 2023 - [debug]   ok.
Thu Aug 17 08:39:51 2023 - [debug]  Connecting via SSH from [root@172.16.0.100](mailto:root@172.16.0.100)(172.16.0.100:22) to [root@172.16.0.102](mailto:root@172.16.0.102)(172.16.0.102:22)..
Thu Aug 17 08:39:53 2023 - [debug]   ok.
Thu Aug 17 08:39:53 2023 - [info] All SSH connection tests passed successfully.
Use of uninitialized value in exit at /usr/bin/masterha_check_ssh line 44.

如果报错，就是ssh-keygen  ，ssh-copy-id  有问题。

解决方案：

例如master节点  (3个节点全部执行)

**`[ssh-keygen -f ~/.ssh/known_hosts -R](https://www.techrepublic.com/article/how-to-remove-or-update-a-single-entry-from-the-ssh-known-hosts-file/) 172.16.0.100`**

ssh-keygen -f  ~/.ssh/knon_hosts -R 172.16.0.102 

ssh-keygen -f  ~/.ssh/knon_hosts -R 172.16.0.101

ssh-keygen -f  ~/.ssh/knon_hosts -R 172.16.0.102 

执行完成后，在3个节点都执行：

ssh-copy-id root@172.16.0.101

ssh-copy-id root@172.16.0.100

ssh-copy-id root@172.16.0.102

# 检查主从复制

```css
masterha_check_repl --conf=/etc/mha/app1.cnf
最后提示MySQL Replication Health is OK.则主从正常masterha_check_ssh --conf=/etc/mha/app1.cnf

网络上资料很多都是copy ,很多都是无法使用的。

mysql  mha  masterha_check_repl 执行后报错  [error][/usr/share/perl5/MHA/MasterMonitor.pm, ln427] Error happened on checking configurations. Redundant argument in sprintf at /usr/share/perl5/MHA/NodeUtil.pm line 195.
 [error][/usr/share/perl5/MHA/MasterMonitor.pm, ln525] Error happened on monitoring servers.

vim  /usr/share/perl5/MHA/NodeUtil.pm
#use warnings FATAL => ‘all’   注释改行 

最好是reboot 机器  skip-grant-tables
 运行 masterha_check_repl --conf 就有很多内容了：
```

```jsx
root@mha3:/etc/masterha# masterha_check_repl --conf=/etc/masterha/app2.cnf
Thu Aug 17 07:41:04 2023 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Thu Aug 17 07:41:04 2023 - [info] Reading application default configuration from /etc/masterha/app2.cnf..
Thu Aug 17 07:41:04 2023 - [info] Reading server configuration from /etc/masterha/app2.cnf..
Thu Aug 17 07:41:04 2023 - [info] MHA::MasterMonitor version 0.58.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
Thu Aug 17 07:41:05 2023 - [info] GTID failover mode = 1
Thu Aug 17 07:41:05 2023 - [info] Dead Servers:
Thu Aug 17 07:41:05 2023 - [info] Alive Servers:
Thu Aug 17 07:41:05 2023 - [info]   172.16.0.101(172.16.0.101:3306)
Thu Aug 17 07:41:05 2023 - [info]   172.16.0.100(172.16.0.100:3306)
Thu Aug 17 07:41:05 2023 - [info]   172.16.0.102(172.16.0.102:3306)
Thu Aug 17 07:41:05 2023 - [info] Alive Slaves:
Thu Aug 17 07:41:05 2023 - [info]   172.16.0.101(172.16.0.101:3306)  Version=8.0.34-0ubuntu0.20.04.1 (oldest major version between slaves) log-bin:enabled
Thu Aug 17 07:41:05 2023 - [info]     GTID ON
Thu Aug 17 07:41:05 2023 - [info]     Replicating from 172.16.0.101(172.16.0.101:3306)
Thu Aug 17 07:41:05 2023 - [info]     Primary candidate for the new Master (candidate_master is set)
Thu Aug 17 07:41:05 2023 - [info]   172.16.0.100(172.16.0.100:3306)  Version=8.0.34-0ubuntu0.20.04.1 (oldest major version between slaves) log-bin:enabled
Thu Aug 17 07:41:05 2023 - [info]     GTID ON
Thu Aug 17 07:41:05 2023 - [info]     Replicating from 172.16.0.101(172.16.0.101:3306)
Thu Aug 17 07:41:05 2023 - [info]     Primary candidate for the new Master (candidate_master is set)
Thu Aug 17 07:41:05 2023 - [info]   172.16.0.102(172.16.0.102:3306)  Version=8.0.34-0ubuntu0.20.04.1 (oldest major version between slaves) log-bin:enabled
Thu Aug 17 07:41:05 2023 - [info]     GTID ON
Thu Aug 17 07:41:05 2023 - [info]     Replicating from 172.16.0.101(172.16.0.101:3306)
Thu Aug 17 07:41:05 2023 - [info] Current Alive Master: 172.16.0.101(172.16.0.101:3306)
Thu Aug 17 07:41:05 2023 - [info] Checking slave configurations..
Thu Aug 17 07:41:05 2023 - [info]  read_only=1 is not set on slave 172.16.0.101(172.16.0.101:3306).
Thu Aug 17 07:41:05 2023 - [info]  read_only=1 is not set on slave 172.16.0.100(172.16.0.100:3306).
Thu Aug 17 07:41:05 2023 - [info] Checking replication filtering settings..
Thu Aug 17 07:41:05 2023 - [info]  binlog_do_db= , binlog_ignore_db=
Thu Aug 17 07:41:05 2023 - [info]  Replication filtering check ok.
Thu Aug 17 07:41:05 2023 - [info] GTID (with auto-pos) is supported. Skipping all SSH and Node package checking.
Thu Aug 17 07:41:05 2023 - [info] Checking SSH publickey authentication settings on the current master..
Thu Aug 17 07:41:06 2023 - [info] HealthCheck: SSH to 172.16.0.101 is reachable.
Thu Aug 17 07:41:06 2023 - [info]
172.16.0.101(172.16.0.101:3306) (current master)
 +--172.16.0.101(172.16.0.101:3306)
 +--172.16.0.100(172.16.0.100:3306)
 +--172.16.0.102(172.16.0.102:3306)

Thu Aug 17 07:41:06 2023 - [info] Checking replication health on 172.16.0.101..
Thu Aug 17 07:41:06 2023 - [error][/usr/share/perl5/MHA/Server.pm, ln490] Slave IO thread is not running on 172.16.0.101(172.16.0.101:3306)
Thu Aug 17 07:41:06 2023 - [error][/usr/share/perl5/MHA/ServerManager.pm, ln1526]  failed!
Thu Aug 17 07:41:06 2023 - [error][/usr/share/perl5/MHA/MasterMonitor.pm, ln427] Error happened on checking configurations.  at /usr/share/perl5/MHA/MasterMonitor.pm line 420.
Thu Aug 17 07:41:06 2023 - [error][/usr/share/perl5/MHA/MasterMonitor.pm, ln525] Error happened on monitoring servers.
Thu Aug 17 07:41:06 2023 - [info] Got exit code 1 (Not master dead).

MySQL Replication Health is NOT OK!
```

解决方案： 本质上还是权限不足。 按照上面的 修改下  mha manager管理账号 mha 和复制账号 repl666

```jsx

ALTER  USER 'mha'@'172.16.0.%' IDENTIFIED WITH mysql_native_password BY 'mha';

FLUSH PRIVILEGES;

ALTER  USER 'mha'@'localhost' IDENTIFIED WITH mysql_native_password BY 'mha';

FLUSH PRIVILEGES;

ALTER  USER 'repl666'@'172.16.0.%' IDENTIFIED WITH mysql_native_password BY 'repl666';

FLUSH PRIVILEGES;

ALTER  USER 'repl666'@'localhost' IDENTIFIED WITH mysql_native_password BY 'repl666';

FLUSH PRIVILEGES;

GRANT ALL PRIVILEGES ON *.* TO 'mha'@'172.16.0.%';

```

修改后：

```jsx
root@mha3:~# masterha_check_repl  --conf=/etc/masterha/app1.cnf
Fri Aug 18 06:30:28 2023 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Fri Aug 18 06:30:28 2023 - [info] Reading application default configuration from /etc/masterha/app1.cnf..
Fri Aug 18 06:30:28 2023 - [info] Reading server configuration from /etc/masterha/app1.cnf..
Fri Aug 18 06:30:28 2023 - [info] MHA::MasterMonitor version 0.58.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
Fri Aug 18 06:30:30 2023 - [info] GTID failover mode = 1
Fri Aug 18 06:30:30 2023 - [info] Dead Servers:
Fri Aug 18 06:30:30 2023 - [info] Alive Servers:
Fri Aug 18 06:30:30 2023 - [info]   172.16.0.101(172.16.0.101:3306)
Fri Aug 18 06:30:30 2023 - [info]   172.16.0.100(172.16.0.100:3306)
Fri Aug 18 06:30:30 2023 - [info]   172.16.0.102(172.16.0.102:3306)
Fri Aug 18 06:30:30 2023 - [info] Alive Slaves:
Fri Aug 18 06:30:30 2023 - [info]   172.16.0.100(172.16.0.100:3306)  Version=8.0.34-0ubuntu0.20.04.1 (oldest major version between slaves) log-bin:enabled
Fri Aug 18 06:30:30 2023 - [info]     GTID ON
Fri Aug 18 06:30:30 2023 - [info]     Replicating from 172.16.0.101(172.16.0.101:3306)
Fri Aug 18 06:30:30 2023 - [info]     Primary candidate for the new Master (candidate_master is set)
Fri Aug 18 06:30:30 2023 - [info]   172.16.0.102(172.16.0.102:3306)  Version=8.0.34-0ubuntu0.20.04.1 (oldest major version between slaves) log-bin:enabled
Fri Aug 18 06:30:30 2023 - [info]     GTID ON
Fri Aug 18 06:30:30 2023 - [info]     Replicating from 172.16.0.101(172.16.0.101:3306)
Fri Aug 18 06:30:30 2023 - [info] Current Alive Master: 172.16.0.101(172.16.0.101:3306)
Fri Aug 18 06:30:30 2023 - [info] Checking slave configurations..
Fri Aug 18 06:30:30 2023 - [info]  read_only=1 is not set on slave 172.16.0.100(172.16.0.100:3306).
Fri Aug 18 06:30:30 2023 - [info] Checking replication filtering settings..
Fri Aug 18 06:30:30 2023 - [info]  binlog_do_db= , binlog_ignore_db=
Fri Aug 18 06:30:30 2023 - [info]  Replication filtering check ok.
Fri Aug 18 06:30:30 2023 - [info] GTID (with auto-pos) is supported. Skipping all SSH and Node package checking.
Fri Aug 18 06:30:30 2023 - [info] Checking SSH publickey authentication settings on the current master..
Fri Aug 18 06:30:30 2023 - [info] HealthCheck: SSH to 172.16.0.101 is reachable.
Fri Aug 18 06:30:30 2023 - [info]
172.16.0.101(172.16.0.101:3306) (current master)
 +--172.16.0.100(172.16.0.100:3306)
 +--172.16.0.102(172.16.0.102:3306)

Fri Aug 18 06:30:30 2023 - [info] Checking replication health on 172.16.0.100..
Fri Aug 18 06:30:30 2023 - [info]  ok.
Fri Aug 18 06:30:30 2023 - [info] Checking replication health on 172.16.0.102..
Fri Aug 18 06:30:30 2023 - [info]  ok.
Fri Aug 18 06:30:30 2023 - [warning] master_ip_failover_script is not defined.
Fri Aug 18 06:30:30 2023 - [warning] shutdown_script is not defined.
Fri Aug 18 06:30:30 2023 - [info] Got exit code 0 (Not master dead).

MySQL Replication Health is OK.
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/316169c0-2c48-4ae3-a7d1-3734f008969b/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/be3c45af-6de1-4d74-9965-346532693226/Untitled.png)

# 检查主库状态

# masterha_check_status  --conf=/etc/masterha/app1.cnf
app1 is stopped(2:NOT_RUNNING).

正常的 因为没启动啊！

启动命令：

```bash
nohup masterha_manager --conf=/etc/masterha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /var/log/mha/app1/manager.log 2>&1 &

```

root@mha3:~# nohup masterha_manager --conf=/etc/masterha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /var/log/mha/app1/manager.log 2>&1 &
[1] 25736 

这个是进程id。   kill -9 25736  

root@mha3:~# masterha_check_status   --conf=/etc/masterha/app1.cnf
app1 (pid:25736) is running(0:PING_OK), master:172.16.0.101

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/edb75d22-7123-4125-85c2-b9ab3b4b8a3a/Untitled.png)

# VIP 切换

添加vip       apt-get install net-tools

ifconfig ens33:1 172.16.0.107

ip a 查看网卡是 ens33   似乎部署好mysql 主从， master节点就多了一个IP

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/fd9108f8-bf25-4256-b859-685551d28000/Untitled.png)

master_ip_failover :

```jsx
#!/usr/bin/env perl
use strict;
use warnings FATAL => 'all';
use Getopt::Long;
my (
    $command,          $ssh_user,        $orig_master_host, $orig_master_ip,
    $orig_master_port, $new_master_host, $new_master_ip,    $new_master_port
);
my $vip = '172.16.0.107/24';
my $key = '1'; #创建的ens33：#  #号说几就写几 这里是1
my $ssh_start_vip = "/sbin/ifconfig ens33:$key $vip";
my $ssh_stop_vip = "/sbin/ifconfig ens33:$key down";

GetOptions(
    'command=s'          => \$command,
    'ssh_user=s'         => \$ssh_user,
    'orig_master_host=s' => \$orig_master_host,
    'orig_master_ip=s'   => \$orig_master_ip,
    'orig_master_port=i' => \$orig_master_port,
    'new_master_host=s'  => \$new_master_host,
    'new_master_ip=s'    => \$new_master_ip,
    'new_master_port=i'  => \$new_master_port,
);

exit &main();

sub main {

    print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";

    if ( $command eq "stop" || $command eq "stopssh" ) {

        my $exit_code = 1;
        eval {
            print "Disabling the VIP on old master: $orig_master_host \n";
            &stop_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn "Got Error: $@\n";
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "start" ) {

        my $exit_code = 10;
        eval {
            print "Enabling the VIP - $vip on the new master - $new_master_host \n";
            &start_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn $@;
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "status" ) {
        print "Checking the Status of the script.. OK \n";
        exit 0;
    }
    else {
        &usage();
        exit 1;
    }
}

sub start_vip() {
    `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
}
sub stop_vip() {
     return 0  unless  ($ssh_user);
    `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
}

sub usage {
    print
    "Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
}

#脚本部分修改内容如下：
#my $vip = '10.0.0.55/24';  #VIP地址（该地址要选择在现网中未被使用的）
#my $key = '1'; 
#my $ssh_start_vip = "/sbin/ifconfig eth0:$key $vip"; #启动VIP时的网卡名
#my $ssh_stop_vip = "/sbin/ifconfig eth0:$key down";   #关闭VIP时的网卡名
```

配置文件加上vip 漂移，再次检查成功：

```bash
root@mha3:~# masterha_check_repl   --conf=/etc/masterha/app1.cnf
Fri Aug 18 06:48:22 2023 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Fri Aug 18 06:48:22 2023 - [info] Reading application default configuration from /etc/masterha/app1.cnf..
Fri Aug 18 06:48:22 2023 - [info] Reading server configuration from /etc/masterha/app1.cnf..
Fri Aug 18 06:48:22 2023 - [info] MHA::MasterMonitor version 0.58.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
Fri Aug 18 06:48:23 2023 - [info] GTID failover mode = 1
Fri Aug 18 06:48:23 2023 - [info] Dead Servers:
Fri Aug 18 06:48:23 2023 - [info] Alive Servers:
Fri Aug 18 06:48:23 2023 - [info]   172.16.0.101(172.16.0.101:3306)
Fri Aug 18 06:48:23 2023 - [info]   172.16.0.100(172.16.0.100:3306)
Fri Aug 18 06:48:23 2023 - [info]   172.16.0.102(172.16.0.102:3306)
Fri Aug 18 06:48:23 2023 - [info] Alive Slaves:
Fri Aug 18 06:48:23 2023 - [info]   172.16.0.100(172.16.0.100:3306)  Version=8.0.34-0ubuntu0.20.04.1 (oldest major version between slaves) log-bin:enabled
Fri Aug 18 06:48:23 2023 - [info]     GTID ON
Fri Aug 18 06:48:23 2023 - [info]     Replicating from 172.16.0.101(172.16.0.101:3306)
Fri Aug 18 06:48:23 2023 - [info]     Primary candidate for the new Master (candidate_master is set)
Fri Aug 18 06:48:23 2023 - [info]   172.16.0.102(172.16.0.102:3306)  Version=8.0.34-0ubuntu0.20.04.1 (oldest major version between slaves) log-bin:enabled
Fri Aug 18 06:48:23 2023 - [info]     GTID ON
Fri Aug 18 06:48:23 2023 - [info]     Replicating from 172.16.0.101(172.16.0.101:3306)
Fri Aug 18 06:48:23 2023 - [info] Current Alive Master: 172.16.0.101(172.16.0.101:3306)
Fri Aug 18 06:48:23 2023 - [info] Checking slave configurations..
Fri Aug 18 06:48:23 2023 - [info]  read_only=1 is not set on slave 172.16.0.100(172.16.0.100:3306).
Fri Aug 18 06:48:23 2023 - [info] Checking replication filtering settings..
Fri Aug 18 06:48:23 2023 - [info]  binlog_do_db= , binlog_ignore_db=
Fri Aug 18 06:48:23 2023 - [info]  Replication filtering check ok.
Fri Aug 18 06:48:23 2023 - [info] GTID (with auto-pos) is supported. Skipping all SSH and Node package checking.
Fri Aug 18 06:48:23 2023 - [info] Checking SSH publickey authentication settings on the current master..
Fri Aug 18 06:48:24 2023 - [info] HealthCheck: SSH to 172.16.0.101 is reachable.
Fri Aug 18 06:48:24 2023 - [info]
172.16.0.101(172.16.0.101:3306) (current master)
 +--172.16.0.100(172.16.0.100:3306)
 +--172.16.0.102(172.16.0.102:3306)

Fri Aug 18 06:48:24 2023 - [info] Checking replication health on 172.16.0.100..
Fri Aug 18 06:48:24 2023 - [info]  ok.
Fri Aug 18 06:48:24 2023 - [info] Checking replication health on 172.16.0.102..
Fri Aug 18 06:48:24 2023 - [info]  ok.
Fri Aug 18 06:48:24 2023 - [info] Checking master_ip_failover_script status:
Fri Aug 18 06:48:24 2023 - [info]   /usr/bin/master_ip_failover --command=status --ssh_user=root --orig_master_host=172.16.0.101 --orig_master_ip=172.16.0.101 --orig_master_port=3306

IN SCRIPT TEST====/sbin/ifconfig ens33:1 down==/sbin/ifconfig ens33:1 172.16.0.107/24===

Checking the Status of the script.. OK
Fri Aug 18 06:48:24 2023 - [info]  OK.
Fri Aug 18 06:48:24 2023 - [warning] shutdown_script is not defined.
Fri Aug 18 06:48:24 2023 - [info] Got exit code 0 (Not master dead).

MySQL Replication Health is OK.
```

按照上面的步骤，先启动了mha，后修改了配置文件。所以需要重启

masterha_stop --conf=/etc/masterha/app1.cnf

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/bdd4e139-2677-409d-aa3b-5b3232f115fd/Untitled.png)

关闭 master 节点上的 测试会不会自动切换vip;

不小心没给mha2安装net-tools工具，导致VIP无法创建。将mysql主从修复 后，继续

```bash
stop slave;
Query OK, 0 rows affected, 2 warnings (0.00 sec)

mysql> reset  slave;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> reset  master;
Query OK, 0 rows affected (0.02 sec)

mysql>  CHANGE MASTER TO MASTER_HOST='172.16.0.101', MASTER_USER='repl666', MASTER_PASSWORD='repl666', MASTER_LOG_FILE='slave-bin.000007', MASTER_LOG_POS=197;
ERROR 1776 (HY000): Parameters SOURCE_LOG_FILE, SOURCE_LOG_POS, RELAY_LOG_FILE and RELAY_LOG_POS cannot be set when SOURCE_AUTO_POSITION is active.
mysql> stop  slave;
Query OK, 0 rows affected, 2 warnings (0.00 sec)

mysql> CHANGE MASTER TO  master_auto_position=0;
Query OK, 0 rows affected, 2 warnings (0.03 sec)

mysql>  CHANGE MASTER TO MASTER_HOST='172.16.0.101', MASTER_USER='repl666', MASTER_PASSWORD='repl666', MASTER_LOG_FILE='slave-bin.000007', MASTER_LOG_POS=197;
Query OK, 0 rows affected, 8 warnings (0.01 sec)

mysql> start slave;
Query OK, 0 rows affected, 1 warning (0.01 sec)
```

同时也导致了  配置文件发生变化

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ee8929fb-eb4c-4895-910e-ac69957b671a/Untitled.png)

修正后：启动mysql mha  记得   master  也要添加上  ifconfig ens33:1 172.16.0.107

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a8ac7e23-a98f-4538-b314-9b0faf7fbec0/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/42576f01-d278-4967-9a70-4e5aa8d5a249/Untitled.png)

关闭掉master 节点 systemctl  stop mysql

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/664e5cb4-96fd-496f-a038-33c85d2dc3ce/Untitled.png)

VIP已经飘逸到 mha2了。

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/488989e0-8b00-4ac6-b830-e200468d95d0/Untitled.png)

```bash
IN SCRIPT TEST====/sbin/ifconfig ens33:1 down==/sbin/ifconfig ens33:1 172.16.0.107/24===

Enabling the VIP - 172.16.0.107/24 on the new master - 172.16.0.100
Fri Aug 18 10:17:06 2023 - [info]  OK.
Fri Aug 18 10:17:06 2023 - [info] ** Finished master recovery successfully.
Fri Aug 18 10:17:06 2023 - [info] * Phase 3: Master Recovery Phase completed.
Fri Aug 18 10:17:06 2023 - [info]
Fri Aug 18 10:17:06 2023 - [info] * Phase 4: Slaves Recovery Phase..
Fri Aug 18 10:17:06 2023 - [info]
Fri Aug 18 10:17:06 2023 - [info]
Fri Aug 18 10:17:06 2023 - [info] * Phase 4.1: Starting Slaves in parallel..
Fri Aug 18 10:17:06 2023 - [info]
Fri Aug 18 10:17:06 2023 - [info] -- Slave recovery on host 172.16.0.102(172.16.0.102:3306) started, pid: 2604. Check tmp log /var/log/masterha/app1/172.16.0.102_3306_20230818101704.log if it takes time..
Fri Aug 18 10:17:07 2023 - [info]
Fri Aug 18 10:17:07 2023 - [info] Log messages from 172.16.0.102 ...
Fri Aug 18 10:17:07 2023 - [info]
Fri Aug 18 10:17:06 2023 - [info]  Resetting slave 172.16.0.102(172.16.0.102:3306) and starting replication from the new master 172.16.0.100(172.16.0.100:3306)..
Fri Aug 18 10:17:06 2023 - [info]  Executed CHANGE MASTER.
Fri Aug 18 10:17:06 2023 - [info]  Slave started.
Fri Aug 18 10:17:06 2023 - [info]  gtid_wait(f2f54a47-328b-11ee-99f3-000c292b6dec:1) completed on 172.16.0.102(172.16.0.102:3306). Executed 0 events.
Fri Aug 18 10:17:07 2023 - [info] End of log messages from 172.16.0.102.
Fri Aug 18 10:17:07 2023 - [info] -- Slave on host 172.16.0.102(172.16.0.102:3306) started.
Fri Aug 18 10:17:07 2023 - [info] All new slave servers recovered successfully.
Fri Aug 18 10:17:07 2023 - [info]
Fri Aug 18 10:17:07 2023 - [info] * Phase 5: New master cleanup phase..
Fri Aug 18 10:17:07 2023 - [info]
Fri Aug 18 10:17:07 2023 - [info] Resetting slave info on the new master..
Fri Aug 18 10:17:07 2023 - [info]  172.16.0.100: Resetting slave info succeeded.
Fri Aug 18 10:17:07 2023 - [info] Master failover to 172.16.0.100(172.16.0.100:3306) completed successfully.
Fri Aug 18 10:17:07 2023 - [info]

----- Failover Report -----

app1: MySQL Master failover 172.16.0.101(172.16.0.101:3306) to 172.16.0.100(172.16.0.100:3306) succeeded

Master 172.16.0.101(172.16.0.101:3306) is down!

Check MHA Manager logs at mha3:/var/log/masterha/app1/manager.log for details.

Started automated(non-interactive) failover.
Invalidated master IP address on 172.16.0.101(172.16.0.101:3306)
Selected 172.16.0.100(172.16.0.100:3306) as a new master.
172.16.0.100(172.16.0.100:3306): OK: Applying all logs succeeded.
172.16.0.100(172.16.0.100:3306): OK: Activated master IP address.
172.16.0.102(172.16.0.102:3306): OK: Slave started, replicating from 172.16.0.100(172.16.0.100:3306)
172.16.0.100(172.16.0.100:3306): Resetting slave info succeeded.
Master failover to 172.16.0.100(172.16.0.100:3306) completed successfully.
```

测试数据同步：

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4026041f-41f9-4d5d-a5fb-7451443a9bf6/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/86a24229-4ae0-4d8b-b120-b16e88b57290/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/28fb8019-5fa3-4678-9069-c6a58a85e198/Untitled.png)

总结： VIP自动转移成功，数据主从同步正常，日志也正常。 但是呢  mha死掉了

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/b3c61d1d-0d98-4c52-a7d8-f212c4d40325/Untitled.png)

发现在执行了一次自动故障转移后，MHAManager进程停止了。官网上对这种情况的解释如下：

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/b39aca77-e2e5-4a98-bb93-73d47ca94cf5/Untitled.png)

# 邮件告警

```bash

apt install cpanminus
apt install -y make
cpanm Email::Simple
apt-get install gcc
cpanm Email::Address::XS
cpanm   install Email::Sender::Simple
cpanm    install Email::Sender::Transport::SMTP::TLS

```

/usr/bin/master_send_report 执行下  看看能不能发送，如果不行 一般都有提示的。缺少东西就提示了

mha 配置文件如何下

```bash
cat /etc/masterha/app1.cnf

[server default]
manager_workdir=/var/log/masterha/app1
manager_log=/var/log/masterha/app1/manager.log
master_binlog_dir=/var/lib/mysql,/var/log/mysql
ping_interval=1
master_ip_failover_script=/usr/bin/master_ip_failover
report_script=/usr/bin/master_send_report                  # 就是这一行
repl_password=repl666
repl_user=repl666
secondary_check_script=/usr/bin/masterha_secondary_check -s mha2 -s mha3  --user=root --master_host=mha1 --master_ip=172.16.0.101 --master_port=3306
shutdown_script=""
ssh_user=root
user=mha
password=mha

[server1]
hostname=172.16.0.101
ssh_port=22

[server2]
candidate_master=1
check_repl_delay=0
hostname=172.16.0.100
port=3306

[server3]
hostname=172.16.0.102
port=3306
```

邮件告警脚本 master_send_report

```bash
cat /usr/bin/master_send_report
```

```bash

#!/usr/bin/env perl

use strict;
use warnings FATAL => 'all';
use Email::Simple;
use Email::Sender::Simple qw(sendmail);
use Email::Sender::Transport::SMTP::TLS;
use Getopt::Long;

#new_master_host and new_slave_hosts are set only when recovering master succeeded
my ( $dead_master_host, $new_master_host, $new_slave_hosts, $subject, $body );
my $smtp='smtp.qq.com';                         # QQ的第三方服务器
my $mail_from='971130229@qq.com';       # 发件人
my $mail_user='971130229@qq.com';       # 登录邮箱
my $mail_pass='vuyrjgdokavubdgh';       # 授权令牌
my $mail_to='zhouliuyang@ata.net.cn';   # 收件人

GetOptions(
  'orig_master_host=s' => \$dead_master_host,
  'new_master_host=s'  => \$new_master_host,
  'new_slave_hosts=s'  => \$new_slave_hosts,
  'subject=s'          => \$subject,
  'body=s'             => \$body,
);

mailToContacts($smtp,$mail_from,$mail_user,$mail_pass,$mail_to,$subject,$body);

sub mailToContacts {
        my ( $smtp, $mail_from, $user, $passwd, $mail_to, $subject, $msg ) = @_;
                open my $DEBUG, "> /var/log/mha/app1/mail.log"
        or die "Can't open the debug      file:$!\n";
        my $transport = Email::Sender::Transport::SMTP::TLS->new(
                host     => 'smtp.qq.com',
                port     => 25,
                username => '971130229@qq.com', # 登录邮箱
                password => 'vuyrjgdokavubdgh',         # 授权令牌
                );

        my $message = Email::Simple->create(
    header => [
        From           => '971130229@qq.com',                      # 发件人
        To             => 'zhouliuyang@ata.net.cn',                        # 收件人
        Subject        => 'MHA-manager(10.0.0.100) ERROR'  # 邮件的主题
        ],
        body           =>$body,                                                    # 这里邮件正文，将来会自动读取MHA的日志信息，如果是发送测试邮件的话，正文为空
);
sendmail( $message, {transport => $transport} );
    return 1;
}
# Do whatever you want here
exit 0;
```

具体登录qq邮箱，设置服务。需要发送短信

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a7c6e4ae-40cd-44a4-a899-c94d3c4f55f5/Untitled.png)

vuyrjgdokavubdgh

参考： https://wx.mail.qq.com/list/readtemplate?name=app_intro.html#/agreement/authorizationCode

```bash
在第三方客户端/服务怎么设置
登录时，请在第三方客户端的密码输入框里面填入授权码进行验证。（不是填入QQ的密码）

IMAP/SMTP 设置方法
用户名/帐户： 你的QQ邮箱完整的地址

密码： 生成的授权码

电子邮件地址： 你的QQ邮箱的完整邮件地址

接收邮件服务器： imap.qq.com，使用SSL，端口号993

发送邮件服务器： smtp.qq.com，使用SSL，端口号465或587

POP3/SMTP 设置方法
用户名/帐户： 你的QQ邮箱完整的地址

密码： 生成的授权码

电子邮件地址： 你的QQ邮箱的完整邮件地址

接收邮件服务器： pop.qq.com，使用SSL，端口号995

发送邮件服务器： smtp.qq.com，使用SSL，端口号465或587
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/bb4d5f1f-4006-4aed-89d7-02e863a7ac44/Untitled.png)

# 关于自动重启  masterha_manager

 `apt-get install supervisor -y`

命令检查 **`supervisor`** 的版本  `supervisord -v`

修改配置文件**`[/etc/supervisor/supervisord.conf](http://supervisord.org/configuration.html)`**，最后新增模块

```bash
[program:masterha_manager]
command=nohup masterha_manager --conf=/etc/masterha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /var/log/mha/app1/manager.log 2>&1 &
autostart=true
autorestart=true
```

```
修改配置文件后：
sudo supervisorctl reread
sudo supervisorctl update
```

记得要启动奥：

```
sudo supervisorctl start masterha_manager
sudo supervisorctl stop masterha_manager
sudo supervisorctl restart masterha_manager
```

/////////////////////////////////////////////////////////////////////////////////////////////////////////// 

# 错误处理：

```jsx
root@mha3:~# masterha_check_repl --conf=/etc/masterha/app1.cnf
Thu Aug 17 04:31:07 2023 - [error][/usr/share/perl5/MHA/MasterMonitor.pm, ln255] Configuration file /etc/masterha/app1.cnf not found!
Thu Aug 17 04:31:07 2023 - [error][/usr/share/perl5/MHA/MasterMonitor.pm, ln386] Error happend on checking configurations.  at /usr/bin/masterha_check_repl line 48.
Thu Aug 17 04:31:07 2023 - [error][/usr/share/perl5/MHA/MasterMonitor.pm, ln482] Error happened on monitoring servers.
Thu Aug 17 04:31:07 2023 - [info] Got exit code 1 (Not master dead).

MySQL Replication Health is NOT OK!
```

解决方案：
/etc/masterha/aap1.cnf中的datadir路径应该是mysql中bin-log的位置

MHA软件由两部分组成：Manager工具包和Node工具包，具体说明如下：

MHA Manager：

1. masterha_check_ssh：检查MHA的SSH配置状况
2. masterha_check_repl：检查MySQL的复制状况
3. masterha_manager：启动MHA
4. masterha_check_status：检测当前MHA运行状态
5. masterha_master_monitor：检测master是否宕机
6. masterha_master_switch：控制故障转移（自动或手动）
7. masterha_conf_host：添加或删除配置的server信息
8. masterha_stop：关闭MHA

MHA Node:

save_binary_logs：保存或复制master的二进制日志

apply_diff_relay_logs：识别差异的relay log并将差异的event应用到其它slave中

filter_mysqlbinlog：去除不必要的ROLLBACK事件（MHA已不再使用这个工具）

purge_relay_logs：消除中继日志（不会堵塞SQL线程）

另有如下几个脚本需自定义：

1. master_ip_failover：管理VIP
2. master_ip_online_change：
3. masterha_secondary_check：当MHA manager检测到master不可用时，通过masterha_secondary_check脚本来进一步确认，减低误切的风险。
4. send_report：当发生故障切换时，可通过send_report脚本发送告警信息。

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7f8f5fd1-38c0-4673-9677-cb63242e47b8/Untitled.png)

https://blog.csdn.net/boyuser/article/details/109428162

# 总是报错无法解决，网上绝大多数资料都是  ok  或者not ok。 详细报错什么都没提

```jsx
root@mha3:~# masterha_check_repl   -conf=/etc/masterha/app1.cnf
Fri Aug 18 02:08:42 2023 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Fri Aug 18 02:08:42 2023 - [info] Reading application default configuration from /etc/masterha/app1.cnf..
Fri Aug 18 02:08:42 2023 - [info] Reading server configuration from /etc/masterha/app1.cnf..
Fri Aug 18 02:08:42 2023 - [info] MHA::MasterMonitor version 0.58.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
Fri Aug 18 02:08:43 2023 - [warning] SQL Thread is stopped(no error) on 172.16.0.101(172.16.0.101:3306)
Fri Aug 18 02:08:43 2023 - [info] GTID failover mode = 1
Fri Aug 18 02:08:43 2023 - [info] Dead Servers:
Fri Aug 18 02:08:43 2023 - [info]   172.16.0.100(172.16.0.100:3306)
Fri Aug 18 02:08:43 2023 - [info] Alive Servers:
Fri Aug 18 02:08:43 2023 - [info]   172.16.0.101(172.16.0.101:3306)
Fri Aug 18 02:08:43 2023 - [info]   172.16.0.102(172.16.0.102:3306)
Fri Aug 18 02:08:43 2023 - [info] Alive Slaves:
Fri Aug 18 02:08:43 2023 - [info]   172.16.0.101(172.16.0.101:3306)  Version=8.0.34-0ubuntu0.20.04.1 (oldest major version between slaves) log-bin:enabled
Fri Aug 18 02:08:43 2023 - [info]     GTID ON
Fri Aug 18 02:08:43 2023 - [info]     Replicating from 172.16.0.101(172.16.0.101:3306)
Fri Aug 18 02:08:43 2023 - [info]   172.16.0.102(172.16.0.102:3306)  Version=8.0.34-0ubuntu0.20.04.1 (oldest major version between slaves) log-bin:enabled
Fri Aug 18 02:08:43 2023 - [info]     GTID ON
Fri Aug 18 02:08:43 2023 - [info]     Replicating from 172.16.0.101(172.16.0.101:3306)
Fri Aug 18 02:08:43 2023 - [info] Current Alive Master: 172.16.0.101(172.16.0.101:3306)
Fri Aug 18 02:08:43 2023 - [info] Checking slave configurations..
Fri Aug 18 02:08:43 2023 - [info]  read_only=1 is not set on slave 172.16.0.101(172.16.0.101:3306).
Fri Aug 18 02:08:43 2023 - [info] Checking replication filtering settings..
Fri Aug 18 02:08:43 2023 - [info]  binlog_do_db= , binlog_ignore_db=
Fri Aug 18 02:08:43 2023 - [info]  Replication filtering check ok.
Fri Aug 18 02:08:43 2023 - [info] GTID (with auto-pos) is supported. Skipping all SSH and Node package checking.
Fri Aug 18 02:08:43 2023 - [error][/usr/share/perl5/MHA/ServerManager.pm, ln492]  Server 172.16.0.100(172.16.0.100:3306) is dead, but must be alive! Check server settings.
Fri Aug 18 02:08:43 2023 - [error][/usr/share/perl5/MHA/MasterMonitor.pm, ln427] Error happened on checking configurations.  at /usr/share/perl5/MHA/MasterMonitor.pm line 402.
Fri Aug 18 02:08:43 2023 - [error][/usr/share/perl5/MHA/MasterMonitor.pm, ln525] Error happened on monitoring servers.
Fri Aug 18 02:08:43 2023 - [info] Got exit code 1 (Not master dead).

MySQL Replication Health is NOT OK!
```

不管了，再次耽误很久了，往下试试

```jsx
nohup   masterha_manager  --conf=/etc/masterha/app1.cnf   --ignore_last_failover  < /dev/null> /var/log/masterha/app1/manager.log  2>&1 &
[1] 15649

cat  /var/log/masterha/app1/manager.log
Fri Aug 18 02:14:58 2023 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Fri Aug 18 02:14:58 2023 - [info] Reading application default configuration from /etc/masterha/app1.cnf..
Fri Aug 18 02:14:58 2023 - [info] Reading server configuration from /etc/masterha/app1.cnf..
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
(172.16.0.101:3306)
Fri Aug 18 02:15:00 2023 - [info] GTID failover mode = 1
Fri Aug 18 02:15:00 2023 - [info] Dead Servers:
Fri Aug 18 02:15:00 2023 - [info] Alive Servers:
Fri Aug 18 02:15:00 2023 - [info]   172.16.0.101(172.16.0.101:3306)
Fri Aug 18 02:15:00 2023 - [info]   172.16.0.100(172.16.0.100:3306)
Fri Aug 18 02:15:00 2023 - [info]   172.16.0.102(172.16.0.102:3306)
Fri Aug 18 02:15:00 2023 - [info] Alive Slaves:
Fri Aug 18 02:15:00 2023 - [info]   172.16.0.101(172.16.0.101:3306)  Version=8.0.34-0ubuntu0.20.04.1 (oldest major version between slaves) log-bin:enabled
Fri Aug 18 02:15:00 2023 - [info]     GTID ON
Fri Aug 18 02:15:00 2023 - [info]     Replicating from 172.16.0.101(172.16.0.101:3306)
Fri Aug 18 02:15:00 2023 - [info]   172.16.0.100(172.16.0.100:3306)  Version=8.0.34-0ubuntu0.20.04.1 (oldest major version between slaves) log-bin:enabled
Fri Aug 18 02:15:00 2023 - [info]     GTID ON
Fri Aug 18 02:15:00 2023 - [info]     Replicating from 172.16.0.101(172.16.0.101:3306)
Fri Aug 18 02:15:00 2023 - [info]     Primary candidate for the new Master (candidate_master is set)
Fri Aug 18 02:15:00 2023 - [info]   172.16.0.102(172.16.0.102:3306)  Version=8.0.34-0ubuntu0.20.04.1 (oldest major version between slaves) log-bin:enabled
Fri Aug 18 02:15:00 2023 - [info]     GTID ON
Fri Aug 18 02:15:00 2023 - [info]     Replicating from 172.16.0.101(172.16.0.101:3306)
Fri Aug 18 02:15:00 2023 - [info] Current Alive Master: 172.16.0.101(172.16.0.101:3306)
Fri Aug 18 02:15:00 2023 - [info] Checking slave configurations..
Fri Aug 18 02:15:00 2023 - [info]  read_only=1 is not set on slave 172.16.0.101(172.16.0.101:3306).
Fri Aug 18 02:15:00 2023 - [info]  read_only=1 is not set on slave 172.16.0.100(172.16.0.100:3306).
Fri Aug 18 02:15:00 2023 - [info] Checking replication filtering settings..
Fri Aug 18 02:15:00 2023 - [info]  binlog_do_db= , binlog_ignore_db=
Fri Aug 18 02:15:00 2023 - [info]  Replication filtering check ok.
Fri Aug 18 02:15:00 2023 - [info] GTID (with auto-pos) is supported. Skipping all SSH and Node package checking.
Fri Aug 18 02:15:00 2023 - [info] Checking SSH publickey authentication settings on the current master..
Fri Aug 18 02:15:00 2023 - [info] HealthCheck: SSH to 172.16.0.101 is reachable.
Fri Aug 18 02:15:00 2023 - [info]
172.16.0.101(172.16.0.101:3306) (current master)
 +--172.16.0.101(172.16.0.101:3306)
 +--172.16.0.100(172.16.0.100:3306)
 +--172.16.0.102(172.16.0.102:3306)

Fri Aug 18 02:15:00 2023 - [warning] master_ip_failover_script is not defined.
Fri Aug 18 02:15:00 2023 - [warning] shutdown_script is not defined.
Fri Aug 18 02:15:00 2023 - [info] Set master ping interval 1 seconds.
Fri Aug 18 02:15:00 2023 - [info] Set secondary check script: /usr/bin/masterha_secondary_check -s mha2 -s mha3  --user=root --master_host=mha1 --master_ip=172.16.0.101 --master_port=3306
Fri Aug 18 02:15:00 2023 - [info] Starting ping health check on 172.16.0.101(172.16.0.101:3306)..
Fri Aug 18 02:15:00 2023 - [info] Ping(SELECT) succeeded, waiting until MySQL doesn't respond..

masterha_check_status --conf=/etc/masterha/app1.cnf
app1 (pid:15649) is running(0:PING_OK), master:172.16.0.101
```

MHA  启动了

### **MySQL8 卸载**

1. 查看MySQL依赖 ： **`dpkg --list|grep mysql`**
2. 卸载： **`sudo apt-get remove mysql-common`**
3. 卸载： **`sudo apt-get autoremove --purge mysql-server-8.0`**(这里版本对应即可)
4. 清除残留数据: **`dpkg -l|grep ^rc|awk '{print$2}'|sudo xargs dpkg -P`**
5. 再次查看MySQL的剩余依赖项: **`dpkg --list|grep mysql`**(这里一般就没有输出了，如果有执行下一步)
6. 继续删除剩余依赖项，如：**`sudo apt-get autoremove --purge mysql-apt-config`**

特别注意   mha启动后，没设置 master_ip_failover 。停止主库，不会导致 删除

[server1]
hostname=192.168.100.131
ssh_port=22

masterha_check_repl --conf=/etc/mha/app1.cnf

MySQL Replication Health is NOT OK!

配置文件不会发生改变。

vip 转移了也不会对配置文件进行修改。 只有在命令**masterha_manager --conf=/etc/app1.cnf --remove_dead_master_conf**才会删除配置文件中的东西

```bash
nohup   masterha_manager  --conf=/etc/masterha/app1.cnf  **--remove_dead_master_conf**   --ignore_last_failover  < /dev/null> /var/log/masterha/app1/manager.log  2>&1 &
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/710b4b1a-5f77-4140-bdb7-f0e8c1e55892/Untitled.png)

原来的master重新上线：

第一修改master 的配置文件 将  id修改不同的   ，stop slave ; reset master 等等

如果总是不行，重启下master .  mha配置文件增加  删除的东西。

如果总是不行，重启下master .  mha配置文件增加  删除的东西。

如果总是不行，重启下master .  mha配置文件增加  删除的东西。

如果总是不行，重启下master .  mha配置文件增加  删除的东西。

如果总是不行，重启下master .  mha配置文件增加  删除的东西。

[server1]
candidate_master=1
check_repl_delay=0
hostname=192.168.100.131
port=3306

注意：

[在 MySQL 8.0 中，**`GRANT REPLICATION SLAVE ON *.* TO 'user'@'host';`** 命令用于授予用户复制从服务器的权限。这意味着该用户可以连接到主服务器并从中获取二进制日志数据1](https://dev.mysql.com/doc/refman/8.0/en/replication-howto-repuser.html)。

[而 **`GRANT ALL PRIVILEGES ON *.* TO 'user'@'host';`** 命令用于授予用户所有可用的全局权限](https://www.bing.com/search?q=Bing+AI&showconv=1&FORM=hpcodx#)[2](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html)。这包括 **`REPLICATION SLAVE`** 权限，但也包括许多其他权限，如 **`SELECT`**、**`INSERT`**、**`UPDATE`** 等。

**`[GRANT ALL ON *.* TO 'user'@'host';`** 命令与 **`GRANT ALL PRIVILEGES ON *.* TO 'user'@'host';`** 命令相同，都用于授予用户所有可用的全局权限2](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html)。

因此，如果您只想授予用户复制从服务器的权限，应使用 **`GRANT REPLICATION SLAVE ON *.* TO 'user'@'host';`** 命令。如果您想授予用户所有可用的全局权限，可以使用 **`GRANT ALL PRIVILEGES ON *.* TO 'user'@'host';`** 或 **`GRANT ALL ON *.* TO 'user'@'host';`** 命令。

desc   mysql.user;    查看表结构

# 发现其他 机器无法ping通 mysql 3306 但网络好 端口在3306

解决方案： 将 mysql 配置文件  **`bind-address`**设置为**`0.0.0.0`**。 或者注释掉

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7909f17e-deff-4a47-9f98-d72012d3dfbe/Untitled.png)

总结就是 权限导致了很多问题：

第一：

vim    /usr/share/perl5/MHA/NodeUtil.pm  必须注销掉   

vim  /usr/share/perl5/MHA/NodeUtil.pm
#use warnings FATAL => ‘all’   注释改行 

第二 mha  replication 权限 . 如果主节点 创建了账号，但是从节点没有，也需要手动增加。但是等主从同步做好了，在主节点创建的账号会自动同步到从节点

```bash
CREATE USER 'replication'@'%' IDENTIFIED WITH mysql_native_password  BY 'replication';
GRANT ALL PRIVILEGES ON *.* TO 'replication'@'%';
FLUSH PRIVILEGES;

create user 'replication'@'localhost' identified  WITH mysql_native_password  by 'replication';
GRANT ALL PRIVILEGES ON *.* TO 'replication'@'localhost';
FLUSH PRIVILEGES;
```

第三：  telnet  必须 通  3306.每台都要试过

如果发现 telnet 不同，但是 ufw 没开，netstat -tunlp |grep  3306 有，表示配置文件中的没弄好

解决： vim   /etc/mysql/mysql.conf/mysqld.cnf  (每台都要做)

**`bind-address`**设置为**`0.0.0.0`**

第四关于  root 权限：   root系统用户的密码最好和 mysql  -u  root -p 一致。 而且同步权限要有

‘root’@’%’  or ‘root’@’IP 段’。 如果master 节点没有 root@’localhost’ 不必添加

ALTER USER 'root'@'%' IDENTIFIED BY 'Ata@1234';

ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Ata@1234';

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3aa51776-e2b2-4f0d-9e5a-b3664036642a/Untitled.png)

第五： 发现app1.cnf 配置文件中，找不到 mha_Secondary-xxx

实际上是 /usr/bin//masterha_secondary_check

第六：  配置文件检查通过了，但是mha启动后， 不会切换，查看日志发现 backup master ssh不通。 解决办法：  将 配置文件mha用户 变成root用户

第七   master 挂掉，mha 切换到backupmaster  ,但是 slave 和backmaster  master 主从报错。

```css
mysql   mha   主从: SHOW SLAVE STATUS\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: 10.101.1.195
                  Master_User: replication
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: slave-bin.000007
          Read_Master_Log_Pos: 237
               Relay_Log_File: slave-relay-bin.000003
                Relay_Log_Pos: 373
        Relay_Master_Log_File: slave-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 1062
                   Last_Error: Coordinator stopped because there were error(s) in the worker(s). The most recent failure being: Worker 1 failed executing transaction '9869b57f-367a-11ee-ad4e-005056912b1d:2' at source log slave-bin.000001, end_log_pos 744. See error log and/or performance_schema.replication_applier_status_by_worker table for more details about this failure or others, if any.
```

重置 slave ; stop  slave;  reset  slave; reset  master ;change  master  to  xxx  start slave; 也无法解决。怎么办？

考虑： 将backup master  也执行stop  xxx  start  xxx

或者将 backup  master  和slave 重新构建主从

如果您想重新构建主从，您可以按照以下步骤操作：

在主服务器上：

1. 运行 `RESET MASTER;` 命令。
2. 运行 `FLUSH TABLES WITH READ LOCK;` 命令。
3. 运行 `SHOW MASTER STATUS;` 命令并记下结果中的值。

在不关闭客户端连接（因为这会释放读锁）的情况下，使用以下命令获取主服务器的转储：

`mysqldump -u root -p --all-databases > /a/path/mysqldump.sql`
 Copy

现在，即使转储尚未结束，您也可以释放锁。要执行此操作，请在MySQL客户端中执行以下命令：

`UNLOCK TABLES;`
 Copy

现在使用scp或您喜欢的工具将转储文件复制到从服务器。

在从服务器上：

1. 打开与mysql的连接并键入：`STOP SLAVE;`
2. 使用此控制台命令加载主服务器的数据转储：`mysql -uroot -p < mysqldump.sql`
3. 同步从服务器和主服务器日志：`RESET SLAVE;`
4. 运行 `CHANGE MASTER TO MASTER_HOST='10.101.1.195', MASTER_USER='your_master_user', MASTER_PASSWORD='your_master_password', MASTER_LOG_FILE='your_master_log_file', MASTER_LOG_POS=your_master_log_pos;` 其中上述字段的值是您之前复制的值。
5. 最后，键入：`START SLAVE;`

要检查一切是否再次正常工作，键入 `SHOW SLAVE STATUS;` 后，您应该看到：

`Slave_IO_Running: Yes
Slave_SQL_Running: Yes`
 Copy

就这样！希望这些信息对您有所帮助。 😊
```
