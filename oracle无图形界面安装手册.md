## centos7静默安装oracle

### 1，关闭selinux

```shell
[root@localhost ~]# vim /etc/selinux/config 
```

![1585017949913](C:\Users\lushuang\AppData\Roaming\Typora\typora-user-images\1585017949913.png)

### 2，关闭防火墙

```shell
[root@localhost ~]# systemctl stop firewalld
```



### 3，创建用户组和用户（使用root用户）

```shell
groupadd -g 501 oinstall
groupadd -g 502 dba
groupadd -g 503 oper
```

### 4，创建用户

```shell
useradd -u 502 -g oinstall -G dba,oper oracle
```

更改上面创建的oracle用户密码（这里设置成oracle）

```shell
passwd oracle

lxk
```

### 5，创建安装oracle所需要的文件夹

create directory structure

```shell
mkdir -p /ora01/app
chown oracle:oinstall /ora01/app
chmod 775 /ora01/app
```

create ORACLE_BASE directory for oracle

```shell
mkdir -p /ora01/app/oracle
chown oracle:oinstall /ora01/app/oracle
chmod 775 /ora01/app/oracle
```

create ORACLE_HOME directory for oracle

```shel
mkdir -p /ora01/app/oracle/product/11.2.0/db_1
chown oracle:oinstall -R /ora01/app/oracle
```

create oradata directory for oracle

```shell
mkdir -p /ora01/app/oraInventory
chown oracle:oinstall /ora01/app/oraInventory
chmod 775 /ora01/app/oraInventory
```

create oradata directory for oracle

```shell
mkdir -p /ora01/app/oradata
chown oracle:oinstall /ora01/app/oradata
chmod 775 /ora01/app/oradata
```

create flash_recovery_area directory for oracle

```shell
mkdir -p /ora01/app/flash_recovery_area
chown oracle:oinstall /ora01/app/flash_recovery_area
chmod 775 /ora01/app/flash_recovery_area
```

> **下面解释各个文件夹的作用**
>
> *我安装时创建的文件夹（这里要注意这些文件夹所属用户和用户组）*
>
> ![1585016308902](C:\Users\lushuang\AppData\Roaming\Typora\typora-user-images\1585016308902.png)

`database`：下载的oracle安装包（11gR2版本），解压完成后会生成database文件夹。

`flash_recovery_area`:先创建好后面配置`/data/src/database/response/dbca.rsp`时要使用该文件夹。

`oraInventory`：在静默安装过程中（即无图形界面），用来存放log以及其它一些文件。若使用图形化界面安装无需创建，会自动生成该文件夹。

`oracle`：ORACLE_BASE目录，即oracle安装基目录。

`oradata`：和`flash_recovery_area`类似,同样添加数据库实例时要用到，需要在配置dbca.rsp时使用。

### 6，修改配置文件

**1，修改/etc/sysctl.conf文件(root用户)**

```shell
vim /etc/sysctl.conf
```

修改代码如下（这里虚拟机是2G内存）：

```shell
kernel.shmmni = 4096
kernel.shmmax = 2147483647
kernel.shmall = 524288
kernel.sem = 250 32000 100 128

fs.aio-max-nr = 1048576
fs.file-max = 6815744
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194303
net.core.wmem_default = 262144
net.core.wmem_max = 1048586

```

使修改生效：

```shell
/sbin/sysctl -p
```

**2，在`/etc/security/limits.conf`添加以下代码为oracle用户设置shell限制 （root用户)**

修改内容：

```shell
oracle   soft   nproc    131072
oracle   hard   nproc    131072
oracle   soft   nofile   131072
oracle   hard   nofile   131072
oracle   soft   core     unlimited
oracle   hard   core     unlimited
oracle   soft   memlock  50000000
oracle   hard   memlock  50000000
```

**3，将服务器名写入到host文件（/etc/hosts）(root用户)**

```shell
127.0.0.1 myhost localhost localhost.localdomain
```

**4，设置环境变量(在`/etc/profile`文件末尾添加以下内容)(root用户)**

```shell
#oracle
export ORACLE_HOME=/ora01/app/oracle/product/11.2.0/db_1
export ORACLE_SID=orcl
if [ $USER = "oracle" ]; then
if [ $SHELL = "/bin/ksh" ]; then
ulimit -p 16384
ulimit -n 65536
else
ulimit -u 16384 -n 65536
fi
fi

```

使修改生效

```shell
source /etc/profile
```

**5，修改oracle用户的环境变量（oracle用户修改即可）**

.bashrc的修改

```shell
export TMP=/tmp
export ORACLE_BASE=/ora01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/11.2.0/db_1
export ORACLE_SID=orcl
export ORACLE_UNQNAME=orcl
export PATH=$ORACLE_HOME/bin:/usr/sbin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export LANG=C
export NLS_LANG=AMERICAN_AMERICA.AL32UTF8
```

使修改生效

```shell
source /home/oracle/.bashrc
```

**6，修改CentOS系统标识（据参考文档说Oracle默认不支持CentOS）**

个人感觉可能没有必要，但为了保险起见，我还是更改了系统标识（使用root用户）

修改文件`/etc/redhat-release`

将内容替换为：`redhat-7`

执行以下命令`mv /etc/redhat-release redhat-7`

**7，检查安装oracle缺少的依赖**

```shell
rpm -q --qf '%{NAME}-%{VERSION}-%{RELEASE}(%{ARCH})\n' binutils \
elfutils-libelf \
elfutils-libelf-devel \
gcc \
gcc-c++ \
glibc \
glibc-common \
glibc-devel \
glibc-headers \
ksh \
libaio \
libaio-devel \
libgcc \
libstdc++ \
libstdc++-devel \
make \
sysstat \
unixODBC \
unixODBC-devel
```

安装缺少的依赖包(可联网情况)：

```shell
yum -y install 包名
```

**8，基础准备工作做完后重启计算机，为安装oracle做准备。**

若上面所有配置均已source即以生效，可以忽略重启。直接安装。

```shell
reboot
```

### 7，oracle安装，这里选择的静默安装

采用图形化界面安装的需要安装图形化界面（对中文不友好），具体方式自行百度）

**注意安装时使用的一定是oracle用户，并非root用户**
前提已将解压好的安装包放在了/ora01/app目录下

**1，编辑数据库安装文件：/ora01/app/database/response/db_install.rsp**

```shell
vim /ora01/app/database/response/db_install.rsp
```

```shell
oracle.install.option=INSTALL_DB_SWONLY
ORACLE_HOSTNAME=oracle.server
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/ora01/app/oraInventory
SELECTED_LANGUAGES=en,zh_CN
ORACLE_HOME=/ora01/app/oracle/product/11.2.0/db_1
ORACLE_BASE=/ora01/app/oracle
oracle.install.db.InstallEdition=EE
oracle.install.db.DBA_GROUP=dba
oracle.install.db.OPER_GROUP=oinstall
oracle.install.db.config.starterdb.type=GENERAL_PURPOSE
oracle.install.db.config.starterdb.globalDBName=oral
oracle.install.db.config.starterdb.SID=oral
oracle.install.db.config.starterdb.characterSet=AL32UTF8
oracle.install.db.config.starterdb.memoryLimit=800
oracle.install.db.config.starterdb.password.ALL=oracle
DECLINE_SECURITY_UPDATES=true
```

**以上配置小白可以直接对着自己的文件更改，若想研究各参数含义参考以下文档**

> https://blog.51cto.com/loofeer/1119713

设置完成后执行以下命令

```shell
/ora01/app/database/runInstaller -silent -responseFile /ora01/app/database/response/db_install.rsp -ignorePrereq
```

-------------------漫长的等待-----------------------------

![1585017308396](C:\Users\lushuang\AppData\Roaming\Typora\typora-user-images\1585017308396.png)

根据上图的提示使用root用户再打开一个终端执行以下脚本

```shell
/ora01/app/oraInventory/orainstRoot.sh
/ora01/app/oracle/product/11.2.0/db_1/root.sh
```

到这里Oracle主程序安装完成。

**2，配置oracle监听程序（oracle用户执行）**

编辑监听配置文件：`/ora01/app/database/response/netca.rsp`

命令：

```shell
vim /ora01/app/database/response/netca.rsp
```

修改以下参数 (大部分已默认)：

```shell
INSTALL_TYPE=""custom""                               # 安装的类型
LISTENER_NUMBER=1                                     # 监听器数量
LISTENER_NAMES={"LISTENER"}                           # 监听器的名称列表
LISTENER_PROTOCOLS={"TCP;1521"}                       # 监听器使用的通讯协议列表
LISTENER_START=""LISTENER""                           # 监听器启动的名称

```

执行以下命令：

```shell
/ora01/app/oracle/product/11.2.0/db_1/bin/netca /silent /responseFile /ora01/app/database/response/netca.rsp
```

查看监听程序是否运行

```shell
netstat -tnulp | grep 1521
```

开启监听命令

```shell
/ora01/app/oracle/product/11.2.0/db_1/bin/lsnrctl start
```

关闭监听命令:

```shell
/ora01/app/oracle/product/11.2.0/db_1/bin/lsnrctl stop
```

**3，添加数据库实例：/ora01/app/database/response/dbca.rsp（oracle用户执行）**

命令：

```shell
vim /ora01/app/database/response/dbca.rsp
```

```shell
RESPONSEFILE_VERSION ="11.2.0"                              // 不要变
OPERATION_TYPE ="createDatabase"                            // 操作为创建实例  
GDBNAME ="orcl"                                             // 数据库实例名
SID ="orcl"                                                 // 实例名字
TEMPLATENAME = "General_Purpose.dbc"                        // 建库用的模板文件
SYSPASSWORD = "oracle"                                      // SYS管理员密码
SYSTEMPASSWORD = "oracle"                                   // SYSTEM管理员密码
SYSMANPASSWORD= "oracle"
DBSNMPPASSWORD= "oracle"
DATAFILEDESTINATION =/ora01/app/oradata                   // 数据文件存放目录
RECOVERYAREADESTINATION=/ora01/app/flash_recovery_area    // 恢复数据存放目录
CHARACTERSET ="AL32UTF8"                                    // 字符集
NATIONALCHARACTERSET= "AL16UTF16"                           // 字符集
TOTALMEMORY ="1638"                                         // 1638MB，物理内存2G*80%。
SOURCEDB = "myhost:1521:orcl"
SYSDBAUSERNAME = "system"
SYSDBAPASSWORD = "oracle"
```

配置完成后执行以下命令

```shell
/ora01/app/oracle/product/11.2.0/db_1/bin/dbca -silent -responseFile /ora01/app/database/response/dbca.rsp
```

**4，启动和关闭实例的程序**

修改文件`/ora01/app/oracle/product/11.2.0/db_1/bin/dbstart`和`/ora01/app/oracle/product/11.2.0/db_1/bin/dbshut`

将`ORACLE_HOME_LISTNER=$1`修改为`ORACLE_HOME_LISTNER=/ora01/app/oracle/product/11.2.0/db_1`

![1585017772119](C:\Users\lushuang\AppData\Roaming\Typora\typora-user-images\1585017772119.png)

修改文件 `/etc/oratab`将`orcl:/ora01/app/oracle/product/11.2.0/db_1:N`修改为`orcl:/ora01/app/oracle/product/11.2.0/db_1:Y`

![1585017802292](C:\Users\lushuang\AppData\Roaming\Typora\typora-user-images\1585017802292.png)

到这里，oracle就安装完成了，但是需要注意的是每次重启，都需要重新开启监听服务

执行命令：`/ora01/app/oracle/product/11.2.0/db_1/bin/lsnrctl start`

这里设置成开机自动启动（root用户执行以下命令）

```shell
chmod +x /etc/rc.d/rc.local
```

修改文件/etc/rc.d/rc.local,在文件最后追加以下内容。

```shell
su oracle -lc "/ora01/app/oracle/product/11.2.0/db_1/bin/lsnrctl start"
su oracle -lc "/ora01/app/oracle/product/11.2.0/db_1/bin/dbstart"
```

![1585017865787](C:\Users\lushuang\AppData\Roaming\Typora\typora-user-images\1585017865787.png)

**简单登录oracle**

执行以下命令

```shell
su oracle     切换oracle用户
sqlplus       进入oracle环境 
systom        用户名
******        输入设置的密码 
```





### 问题



> --通过视图查看system用户没有sysdba权限
>
> SQL> select * from V$PWFILE_USERS;
>
> 
>
> USERNAME                       SYSDB SYSOP SYSAS
> ------------------------------ ----- ----- -----
> SYS                            TRUE  TRUE  FALSE
>
> --conn /as sysdba;以sys用户登陆进去赋予sysdba权限
> SQL> grant sysdba to system;
>
> Grant succeeded.
>
> --再次查询已经有了sysdba权限，可以登陆了。
> SQL> select * from V$PWFILE_USERS;
>
>
> USERNAME                       SYSDB SYSOP SYSAS
> ------------------------------ ----- ----- -----
> SYS                            TRUE  TRUE  FALSE
> SYSTEM                    TRUE  FALSE FALSE
> ————————————————
> 版权声明：本文为CSDN博主「磨叽SP」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
> 原文链接：https://blog.csdn.net/shipeng1022/article/details/26134637





> 1、firewalld的基本使用
>
> 启动： systemctl start firewalld
>
> 查看状态： systemctl status firewalld 
>
> 停止： systemctl disable firewalld
>
> 禁用： systemctl stop firewalld
>
>  
>
> 2.systemctl是CentOS7的服务管理工具中主要的工具，它融合之前service和chkconfig的功能于一体。
>
> 启动一个服务：systemctl start firewalld.service
> 关闭一个服务：systemctl stop firewalld.service
> 重启一个服务：systemctl restart firewalld.service
> 显示一个服务的状态：systemctl status firewalld.service
> 在开机时启用一个服务：systemctl enable firewalld.service
> 在开机时禁用一个服务：systemctl disable firewalld.service
> 查看服务是否开机启动：systemctl is-enabled firewalld.service
> 查看已启动的服务列表：systemctl list-unit-files|grep enabled
> 查看启动失败的服务列表：systemctl --failed
>
> 3.配置firewalld-cmd
>
> 查看版本： firewall-cmd --version
>
> 查看帮助： firewall-cmd --help
>
> 显示状态： firewall-cmd --state
>
> 查看所有打开的端口： firewall-cmd --zone=public --list-ports
>
> 更新防火墙规则： firewall-cmd --reload
>
> 查看区域信息:  firewall-cmd --get-active-zones
>
> 查看指定接口所属区域： firewall-cmd --get-zone-of-interface=eth0
>
> 拒绝所有包：firewall-cmd --panic-on
>
> 取消拒绝状态： firewall-cmd --panic-off
>
> 查看是否拒绝： firewall-cmd --query-panic
>
>  
>
> 那怎么开启一个端口呢
>
> 添加
>
> firewall-cmd --zone=public --add-port=80/tcp --permanent    （--permanent永久生效，没有此参数重启后失效）
>
> 重新载入
>
> firewall-cmd --reload
>
> 查看
>
> firewall-cmd --zone= public --query-port=80/tcp
>
> 删除
>
> firewall-cmd --zone= public --remove-port=80/tcp --permanent