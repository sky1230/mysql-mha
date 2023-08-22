# mysql-mha
```
系统：  ubuntu20
mysql 版本： mysql8.0
mha版本 mha4mysql-manager_0.58-0_all.deb   mha4mysql-node_0.58-0_all.deb
mha官方网址： https://github.com/yoshinorim/
  
  
  PS： 使用root 部署的。

#mysql主从配置
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
