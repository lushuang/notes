## mysql HA

### 环境准备

2台centos7vm，1个vip：

mysql-ha-1：192.168.56.23

mysql-ha-2：192.168.56.24

vip：192.168.56.25

### mysql主从配置

#### mysql安装

https://dev.mysql.com/downloads/mysql/,下载rpm包

https://dev.mysql.com/doc/refman/5.7/en/linux-installation-yum-repo.html,安装手册

#### 主从配置

>*注: 主从数据库都要执行相同操作

1. 主从服务器分别作以下操作：

   > 版本一致
   > 初始化表，并在后台启动mysql
   > 修改root的密码

2. 修改主服务器master:

   > #vi /etc/my.cnf
   > [mysqld]
   > log-bin=mysql-bin   //[必须]启用二进制日志
   > server-id=222      //[必须]服务器唯一ID，默认是1，一般取IP最后一段

3. 修改从服务器slave:

   > #vi /etc/my.cnf
   > [mysqld]
   > log-bin=mysql-bin   //[不是必须]启用二进制日志
   > server-id=226      //[必须]服务器唯一ID，默认是1，一般取IP最后一段

4. 重启两台服务器的mysql

   > /etc/init.d/mysql restart

5. 在主服务器上建立帐户并授权slave:

   > #/usr/local/mysql/bin/mysql -uroot -pmttang   
   > mysql>GRANT REPLICATION SLAVE ON *.* to 'mysync'@'%' identified by 'q123456'; //一般不用root帐号，&ldquo;%&rdquo;表示所有客户端都可能连，只要帐号，密码正确，此处可用具体客户端IP代替，如192.168.145.226，加强安全。

6. 登录主服务器的mysql，查询master的状态

   >mysql>show master status;
   >+------------------+----------+--------------+------------------+
   >| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
   >+------------------+----------+--------------+------------------+
   >| mysql-bin.000004 |      308 |              |                  |
   >+------------------+----------+--------------+------------------+
   >1 row in set (0.00 sec)
   >注：执行完此步骤后不要再操作主服务器MYSQL，防止主服务器状态值变化

7. 配置从服务器Slave：

   > mysql>change master to master_host='192.168.145.222',master_user='mysync',master_password='q123456',
   > 	 master_log_file='mysql-bin.000004',master_log_pos=308;   //注意不要断开，308数字前后无单引号。
   >
   > Mysql>start slave;    //启动从服务器复制功能

   

8. 检查从服务器复制功能状态：

   >
   >
   >mysql> show slave status\G
   >
   >*************************** 1. row ***************************
   >
   >		  Slave_IO_State: Waiting for master to send event
   >		  Master_Host: 192.168.2.222  //主服务器地址
   >		  Master_User: mysync   //授权帐户名，尽量避免使用root
   >		  Master_Port: 3306    //数据库端口，部分版本没有此行
   >		  Connect_Retry: 60
   >		  Master_Log_File: mysql-bin.000004
   >		  Read_Master_Log_Pos: 600     //#同步读取二进制日志的位置，大于等于Exec_Master_Log_Pos
   >		  Relay_Log_File: ddte-relay-bin.000003
   >		  Relay_Log_Pos: 251
   >		  Relay_Master_Log_File: mysql-bin.000004
   >		  Slave_IO_Running: Yes    //此状态必须YES
   >		  Slave_SQL_Running: Yes     //此状态必须YES
   >				......
   >
   >注：Slave_IO及Slave_SQL进程必须正常运行，即YES状态，否则都是错误的状态(如：其中一个NO均属错误)。
   >
   >以上操作过程，主从服务器配置完成。

9. 主从服务器测试：

   > 主服务器Mysql，建立数据库，并在这个库中建表插入一条数据：

10. 完成

    > 编写一shell脚本，用nagios监控slave的两个yes（Slave_IO及Slave_SQL进程），如发现只有一个或零个yes，就表明主从有问题了，发短信警报吧。

### keepalived配置

#### 环境准备

与 nignx 环境配置一致

#### 源码安装

1. 官网：http://www.keepalived.org/

2. 安装包下载：wget http://www.keepalived.org/software/keepalived-1.4.3.tar.gz

3. 解压：tar -zxvf keepalived-1.4.3.tar.gz -C /usr/local/src

4. 安装依赖：yum install -y openssl openssl-devel

5. 编译安装

  ```shell
  cd /usr/local/src/keepalived-1.4.3
  ./configure –prefix=/usr/local/keepalived
  make && make install
  ```

#### 配置keepalived服务

```shell
cp /usr/local/keepalived-1.4.3/keepalived/etc/init.d/keepalived /etc/init.d/
cp /usr/local/keepalived-1.4.3/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
mkdir /etc/keepalived
cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
```

- 开机启动：systemctl enable keepalived.service
- 启动：systemctl start keepalived.service
- 停止：systemctl stop keepalived.service
- 状态：systemctl status keepalived.service

#### keepalived配置

```shell
! Configuration File for keepalived
global_defs {
   router_id longxuekaha1
   vrrp_skip_check_adv_addr
   #vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_script chk_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 3
    weight -2
    fall 3
    rise 2
}

vrrp_script chk_lxk {
    script "/etc/keepalived/check_port.sh 8080"
    interval 3
    weight -2
    fall 3
    rise 2
}

vrrp_script chk_thermometry {
    script "/etc/keepalived/check_port.sh 8081"
    interval 3
    weight -2
    fall 3
    rise 2
}
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 99
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.0.188/24
    }
    track_script {
       chk_nginx
       chk_lxk 
       #chk_thermometry 
    }
    notify_master "/usr/bin/python /etc/keepalived/keepalived_notify.py master 192.168.0.52 192.168.0.188"
    notify_backup "/usr/bin/python /etc/keepalived/keepalived_notify.py backup 192.168.0.103 192.168.0.188"
}
```

#### keepalived_notify.py

```python
python script.py{脚本名} role{master|backup} ip{本keepalived服务器IP} vip{虚拟IP}
```

```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-

import smtplib
from email.mime.text import MIMEText
from email.header import Header
import sys, time, subprocess



# 第三方 SMTP 服务
mail_host="smtp.163.com"  #设置服务器
mail_user="xxx"    #用户名
mail_pass="xxx"   #口令


sender = 'xxx'    # 邮件发送者
receivers = ['xxx']  # 接收邮件，可设置为你的QQ邮箱或者其他邮箱

p = subprocess.Popen('hostname', shell=True, stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
hostname = p.stdout.readline().split('\n')[0]

message_to = ''
for i in receivers:
    message_to += i + ';'

def print_help():
    note = '''python script.py role ip vip
    '''
    print(note)
    exit(1)

time_stamp = time.strftime('%Y-%m-%d %H:%M:%S',time.localtime(time.time()))

if len(sys.argv) != 4:
    print_help()
elif sys.argv[1] == 'master':
    message_content = '%s server: %s(%s) change to Master, VIP: %s' %(time_stamp, sys.argv[2], hostname, sys.argv[3])
    subject = '%s change to Master -- keepalived notify' %(sys.argv[2])
elif sys.argv[1] == 'backup':
    message_content = '%s server: %s(%s) change to Backup, VIP: %s' %(time_stamp, sys.argv[2], hostname, sys.argv[3])
    subject = '%s change to Backup -- keepalived notify' %(sys.argv[2])
else:
    print_help()

message = MIMEText(message_content, 'plain', 'utf-8')
message['From'] = Header(sender, 'utf-8')
message['To'] =  Header(message_to, 'utf-8')

message['Subject'] = Header(subject, 'utf-8')

try:
    smtpObj = smtplib.SMTP()
    smtpObj.connect(mail_host, 25)    # 25 为 SMTP 端口号
    smtpObj.login(mail_user,mail_pass)
    smtpObj.sendmail(sender, receivers, message.as_string())
    print("邮件发送成功")
except smtplib.SMTPException as e:
    print("Error: 无法发送邮件")
    print(e)
```



#### check_nginx.sh脚本

```shell
#!/bin/bash
A=`ps -C nginx --no-header | wc -l`
if [ $A -eq 0 ];then
    #/data/installpath/nginx #尝试重新启动nginx
    sleep 2  #睡眠2秒
    if [ `ps -C nginx --no-header | wc -l` -eq 0 ];then
        killall keepalived #启动失败，将keepalived服务杀死。将vip漂移到其它备份节点
    fi
fi
```

#### check_port.sh脚本

```sh
#!/bin/bash
#keepalived 监控端口脚本
#使用方法：
#在keepalived的配置文件中
#vrrp_script check_port {#创建一个vrrp_script脚本,检查配置
#    script "/etc/keepalived/check_port.sh 6379" #配置监听的端口
#    interval 2 #检查脚本的频率,单位（秒）
#}
CHK_PORT=$1
echo $CHK_PORT
if [ "$CHK_PORT" != "" ];then

        PORT_PROCESS=`lsof -i:$CHK_PORT|wc -l`
        if [ $PORT_PROCESS -eq 0 ];then
        echo "Port $CHK_PORT Is Not Used,End."

        sleep 2
                PORT_PROCESS=`lsof -i:$CHK_PORT|wc -l`
                if [ $PORT_PROCESS -eq 0 ];then
                /etc/init.d/keepalived stop
                fi
        fi
else
        echo "Check Port Cant Be Empty!"
fi
```

### 