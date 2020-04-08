# nginx+keepalived高可用环境搭建

## nginx安装配置

### 一、环境准备

1. 智慧教育10-122.112.138.121 CentOS Linux release 7.7.1908 (Core)
2. 智慧教育11-122.112.136.127 CentOS Linux release 7.7.1908 (Core)
3. VIP 192.168.0.188

### 二、源码安装

1. 官网：http://nginx.org/

2. 安装包下载：wget http://nginx.org/download/nginx-1.16.1.tar.gz

3. 解压：tar -zxvf nginx-1.16.1.tar.gz -C /usr/local/src

4. 安装软件依赖：yum -y install pcre pcre-devel zlib zlib-devel openssl openssl-devel

5. 安装nginx

   ```shell
   cd /usr/local/src/ nginx-1.16.1
   ./configuration –prefix=/usr/local
   make && make install
   ```

### 三、nginx.conf配置

```sh
user  root;
worker_processes  4;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;


    # 乐学卡
    #服务器的集群  
    upstream  lxk_tomcat {  #服务器集群名字
        ip_hash;
        server  192.168.0.52:8080  weight=1;  #服务器配置   weight是权重的意思，权重越大，分配的概率越大。
        server  192.168.0.103:8080  weight=1;  #服务器配置   weight是权重的意思，权重越大，分配的概率越大。
    }

    upstream  thermometry-tomcat {  #服务器集群名字
        ip_hash;
        server  192.168.0.52:8081  weight=1;  #服务器配置   weight是权重的意思，权重越大，分配的概率越大。
        server  192.168.0.103:8081  weight=1;  #服务器配置   weight是权重的意思，权重越大，分配的概率越大。
    }

    server {
        listen 8901;
        server_name localhost;    # 指定域名

        location / {
            proxy_pass http://lxk_tomcat;
            #proxy_pass http://thermometry-tomcat;
            proxy_redirect default;
            # 处理静态文件加载问题
            proxy_set_header Host $http_host;
            proxy_set_header X-Forward-For $remote_addr;
        }

        location ~ /uploadFiles/.*\.(gif|jpg|jpeg|png|bmp|ico|js|css|ttf|woff|scss|woff2|pdf|swf|htm|apk|mp4|avi|mp3|flv|html)$ {
            root   "/data/installpath/lxk_resources";
            index  index.html index.htm;
        }

        location ~ /videoimg/.*\.(gif|jpg|jpeg|png|bmp|ico|js|css|ttf|woff|scss|woff2|pdf|swf|htm|apk|mp4|avi|mp3|flv)$ {
                root   "/data/installpath/lxk_resources";
                index  index.html index.htm;
        }
    }

    #server {
    #    listen       80;
    #    server_name  localhost;

    #    #charset koi8-r;

    #    #access_log  logs/host.access.log  main;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }

    #    #error_page  404              /404.html;

    #    # redirect server error pages to the static page /50x.html
    #    #
    #    error_page   500 502 503 504  /50x.html;
    #    location = /50x.html {
    #        root   html;
    #    }

    #    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #    #
    #    #location ~ \.php$ {
    #    #    proxy_pass   http://127.0.0.1;
    #    #}

    #    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #    #
    #    #location ~ \.php$ {
    #    #    root           html;
    #    #    fastcgi_pass   127.0.0.1:9000;
    #    #    fastcgi_index  index.php;
    #    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    #    include        fastcgi_params;
    #    #}

    #    # deny access to .htaccess files, if Apache's document root
    #    # concurs with nginx's one
    #    #
    #    #location ~ /\.ht {
    #    #    deny  all;
    #    #}
    #}


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}
```

## keepalived安装配置

### 一、环境准备

与 nignx 环境配置一致

### 二、源码安装

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

### 三、配置keepalived服务

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

### 四、keepalived配置

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

### 五、keepalived_notify.py

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
#mail_user="tiantsian"    #用户名
#mail_pass="gongxifacai2"   #口令
mail_user="lushuang0605"    #用户名
mail_pass="XZWPYUWUJTVEZWQC"   #口令


sender = 'tiantsian@163.com'    # 邮件发送者
receivers = ['lushuang0605@163.com']  # 接收邮件，可设置为你的QQ邮箱或者其他邮箱

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



### 六、check_nginx.sh脚本

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

### 七、check_port.sh脚本

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

