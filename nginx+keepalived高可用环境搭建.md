# nginx+keepalived高可用环境搭建

## nginx安装配置

### 一、环境准备

> 2台centos7 vm
>
> nginx-ha-01：192.168.56.7
>
> nginx-ha-02：192.168.56.8

### 二、源码安装

* 官网：http://nginx.org/
* 安装包下载：wget http://nginx.org/download/nginx-1.14.0.tar.gz
* 解压：tar -zxvf nginx-1.14.0.tar.gz -C /usr/local/src
* 安装软件依赖：yum -y install pcre pcre-devel zlib zlib-devel openssl openssl-devel
* 安装nginx：

```shell
cd /usr/local/src/ nginx-1.14.0
./configuration –prefix=/usr/local
make && make install
```

### 三、nginx.conf配置

```sh
user  root;
worker_processes  1;

error_log  logs/error.log;
error_log  logs/error.log  notice;
error_log  logs/error.log  info;

pid        logs/nginx.pid;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  120;

    #gzip  on;

    upstream i.cloud.cnpc {
        server 11.2.60.33:8080;
    }

    upstream lb_cloudMarket {
        server 11.2.60.18:8080;
    }

    server {

        listen 80 ;
        server_name i.cloud.cnpc;

        location = /portal {
			proxy_set_header Host $host;
			proxy_set_header  X-Real-IP $remote_addr;
			proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_pass http://lb_cloudMarket/cloudMarket/;
        }

        location = /bss/ {
			proxy_set_header Host $host;
			proxy_set_header  X-Real-IP $remote_addr;
			proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_pass http://lb_cloudMarket/cloudMarket/bss/login.html;
        }

        location = /oss/ {
			proxy_set_header Host $host;
			proxy_set_header  X-Real-IP $remote_addr;
			proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_pass http://lb_cloudMarket/cloudMarket/oss/login.html;
        }

        location = /mss/ {
			proxy_set_header Host $host;
			proxy_set_header  X-Real-IP $remote_addr;
			proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_pass http://lb_cloudMarket/cloudMarket/mss/login.html;
        }

		location /cloudMarket/ {
			proxy_set_header Host $host;
			proxy_set_header  X-Real-IP $remote_addr;
			proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_pass http://lb_cloudMarket/cloudMarket/;
		}

		#location ~ .*\.(js|css)?$ {
		#
		#}

		location / {
			proxy_set_header Host $host;
			proxy_set_header  X-Real-IP $remote_addr;
					proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_pass http://lb_cloudMarket/cloudMarket/;
		}
    }
}
```

## keepalived安装配置

### 一、环境准备

> 与 nignx 环境配置一致

### 二、源码安装

* 官网：http://www.keepalived.org/

* 安装包下载：wget http://www.keepalived.org/software/keepalived-1.4.3.tar.gz

* 解压：tar -zxvf keepalived-1.4.3.tar.gz -C /usr/local/src

* 安装依赖：yum install -y openssl openssl-devel

* 编译安装

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

> 开机启动：systemctl enable keepalived.service
> 启动：systemctl start keepalived.service
> 停止：systemctl stop keepalived.service
> 状态：systemctl status keepalived.service

### 四、keepalived配置

```shell

! Configuration File for keepalived

global_defs {
   notification_email {
	lushuang@cnpc.com.cn
   }
   notification_email_from cloudadmin@cnpc.com.cn
   smtp_server mail.cnpc.com.cn
   smtp_connect_timeout 30
   router_id cloudnginxha1
   vrrp_skip_check_adv_addr
   vrrp_strict
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

vrrp_script chk_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 3
    weight -2
    fall 3
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
	11.11.176.112
    }
    track_script {
       chk_nginx
       chk_haproxy
}

notify_master "/usr/bin/python /etc/keepalived/keepalived_notify.py master 11.11.176.110 11.11.176.112"
notify_backup "/usr/bin/python /etc/keepalived/keepalived_notify.py backup 11.11.176.111 11.11.176.112"
}
```

***

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
    }
    #notify_master "/usr/bin/python /etc/keepalived/keepalived_notify.py master 192.168.0.52 192.168.0.188"
    #notify_backup "/usr/bin/python /etc/keepalived/keepalived_notify.py backup 192.168.0.103 192.168.0.188"
}


! Configuration File for keepalived
global_defs {
   router_id longxuekaha2
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

vrrp_instance VI_1 {
    state SLAVE
    interface eth0
    virtual_router_id 99
    priority 90
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
    }
    #notify_master "/usr/bin/python /etc/keepalived/keepalived_notify.py master 192.168.0.52 192.168.0.188"
    #notify_backup "/usr/bin/python /etc/keepalived/keepalived_notify.py backup 192.168.0.103 192.168.0.188"
}
```



### 五、发邮件调用

```python
python script.py{脚本名} role{master|backup} ip{本keepalived服务器IP} vip{虚拟IP}
```

### 六、check_nginx.sh脚本

```shell
#!/bin/bash

counter=$(ps -C nginx --no-heading|wc -l)
if [ "${counter}" = "0" ]; then
    /usr/local/nginx/sbin/nginx
    sleep 2
    counter=$(ps -C nginx --no-heading|wc -l)
    #if [ "${counter}" = "0" ]; then
    #    /etc/init.d/keepalived stop
    #fi
fi
check_nginx.sh执行权限
chmod +x check_nginx.sh

check_haproxy.sh 配置如下：

#!/bin/bash

counter=$(ps -C haproxy --no-heading|wc -l)
if [ "${counter}" = "0" ]; then
    /usr/local/haproxy/sbin/haproxy -f /usr/local/haproxy/haproxy.cfg
    sleep 2
    counter=$(ps -C haproxy --no-heading|wc -l)
    #if [ "${counter}" = "0" ]; then
    #    /etc/init.d/keepalived stop
    #fi
Fi
```

```shell
#!/bin/bash
A=`ps -C nginx --no-header | wc -l`
if [ $A -eq 0 ];then
    /data/installpath/nginx #尝试重新启动nginx
    sleep 2  #睡眠2秒
    if [ `ps -C nginx --no-header | wc -l` -eq 0 ];then
        killall keepalived #启动失败，将keepalived服务杀死。将vip漂移到其它备份节点
    fi
fi
```





