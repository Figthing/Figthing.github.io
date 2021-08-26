title: Ambari 2.7.3 离线安装手册
author: Figthing
tags:
  - bigdata
  - ambari
categories:
  - bigdata
  - ambari
date: 2021-07-06 17:12:00
---
##  Ambari 2.7.3 离线安装手册

## 准备工作

### 版本介绍

Ambari 2.7.3仅支持HDP-3.1.0，HDP-3.0.1，HDP-3.0.0使用以下URL确定对每个产品版本的支持https://supportmatrix.hortonworks.com/，以及下载报告

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/bigdata/ambari/1.png" style="zoom: 67%;" />

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/bigdata/ambari/2.png" style="zoom: 67%;" />

<!-- more -->

### 工具包下载

Ambari-2.7.3.0：

http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.7.3.0/ambari-2.7.3.0-centos7.tar.gz

HDP-3.1.0：

http://public-repo-1.hortonworks.com/HDP/centos7/3.x/updates/3.1.0.0/HDP-3.1.0.0-centos7-rpm.tar.gz

HDP-UTILS-1.1.0.22：

http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.22/repos/centos7/HDP-UTILS-1.1.0.22-centos7.tar.gz

JDK：1.8版本：

操作系统:centos7任意版本，系统为英文，64位，内存最好每台都10G以上。

mysql-connector-java-5.1.47.jar

https://repo1.maven.org/maven2/mysql/mysql-connector-java/5.1.47/mysql-connector-java-5.1.47.jar

Mysql5.7版本：

https://downloads.mysql.com/archives/get/p/23/file/mysql-5.7.17-linux-glibc2.5-x86_64.tar.gz

Linux ZIP:

http://mirror.centos.org/centos/7/os/x86_64/Packages/zip-3.0-11.el7.x86_64.rpm

Linux Unzip:

 http://mirror.centos.org/centos/7/os/x86_64/Packages/unzip-6.0-21.el7.x86_64.rpm

Libtirpc-devel：

https://buildlogs.centos.org/c7.1810.00.x86_64/libtirpc/20181031001901/0.2.4-0.15.el7.x86_64/libtirpc-0.2.4-0.15.el7.x86_64.rpm

Percona-XtraDB-Cluster-Shared：

http://www.percona.com/redir/downloads/Percona-XtraDB-Cluster/5.5.37-25.10/RPM/rhel6/x86_64/Percona-XtraDB-Cluster-shared-55-5.5.37-25.10.756.el6.x86_64.rpm

## Linux系统软件安装配置
### 服务器规划

| IP            | 操作系统        | 节点角色 |
| ------------- | --------------- | -------- |
| 192.168.1.121 | CentOS 7.6.1810 | master   |
| 192.168.1.122 | CentOS 7.6.1810 | work     |
| 192.168.1.123 | CentOS 7.6.1810 | work     |

### 系统配置

#### 服务器防火墙关闭

关闭掉linux防火墙（所有机器）

```shell
[root@localhost ~]# systemctl stop firewalld.service
[root@localhost ~]# systemctl disable firewalld.service
```


安装完成后，可以重新启动iptables。如果您环境中的安全协议阻止禁用iptables，则可以启用iptables，如果所有必需端口都已打开且可用，Ambari会在Ambari Server安装过程中检查iptables是否正在运行。如果iptables正在运行，则会显示警告，提醒检查所需端口是否已打开且可用。群集安装向导中的“主机确认”步骤还会为运行iptables的每个主机发出警告。

#### 修改主机名及机器隐射

**修改主机名**

```shell
1)192.168.1.121
[root@localhost ~]# hostnamectl set-hostname master

2)192.168.1.122
[root@localhost ~]# hostnamectl set-hostname slave1

3)192.168.1.123
[root@localhost ~]# hostnamectl set-hostname slave2
```

**修改/etc/hosts文件（所有机器）**

这里主要是为了可以实现通过名称来查找相应的服务器

```shell
[root@localhost ~]# cat << EOF > /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.1.121 master
192.168.1.122 slave1
192.168.1.123 slave2
EOF
```

**修改主机为英文**

```shell
[root@localhost ~]# cat << EOF > /etc/locale.conf
LANG="en_US.UTF-8"
EOF
```

**重启电脑**

```shell
[root@localhost ~]# reboot
```

#### 服务器文件句柄设置

**永久设置（所有机器）**

```shell
[root@master ~]# vi /etc/security/limits.conf 

# End of file
* soft nofile 65536
* hard nofile 65536
* soft nproc 131072
* hard nproc 131072
```

**关闭当前的shell窗口，重新打开ulimit -a查看是否设置成功**

```shell
[root@master ~]# ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 31116
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 65536
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 131072
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

#### 禁用SELinux和PackageKit将检查umask值

**禁用selinux（所有机器）**

```shell
[root@master ~]# vi /etc/selinux/config 

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

主要是把SELINUX改为disabled

**umask值（所有机器）**

```shell
[root@master ~]# echo umask 0022 >> /etc/profile
[root@master ~]# source /etc/profile
```

#### 服务器免登录

**配置`所有节点`无密码登录到其他节点**

```shell
[root@master ~]# ssh-keygen -t rsa		## 一路回车
[root@master ~]# chmod 700 ~/.ssh
[root@master ~]# ssh-copy-id slave1
[root@master ~]# ssh-copy-id slave2
[root@master ~]# ssh-copy-id master
```

**测试所有机器是否SSH免登录互通**

```shell
[root@master ~]# ssh slave1 date; ssh slave2 date; ssh master date;
Wed Jun 30 02:56:40 CST 2021
Wed Jun 30 02:56:40 CST 2021
Wed Jun 30 02:56:40 CST 2021
```

**将创建的密钥拷贝出来**

```shell
[root@master ~]# mkdir /home/tools				## 所有节点
[root@master ~]# cp /root/.ssh/id_rsa /home/tools/
[root@master ~]# ls /home/tools/
id_rsa
```

> 注意：后面ambari安装的时候需要上传这个密钥，所以需要先拷贝出来。

#### 挂载本地Yum源配置

```shell
1）将CentOS7的ISO文件挂载到本地（所有机器）
[root@localhost ~]# mkdir /mnt/cdrom
[root@localhost ~]# mount /dev/cdrom /mnt/cdrom 

2）创建yum本地源，并将系统自带的迁移到tmp目录中（所有机器）
[root@localhost ~]# mv /etc/yum.repos.d/* /tmp/
[root@localhost ~]# touch /etc/yum.repos.d/local.repo
[root@localhost ~]# cat << EOF > /etc/yum.repos.d/local.repo

[LocalRepo]
name=LocalRepository
baseurl=file:///mnt/cdrom
enable=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
EOF

3）重置yum源
[root@localhost ~]# yum clean all
[root@localhost ~]# yum makecache
[root@localhost ~]# yum repolist
```

### 系统安装软件

#### 服务器时间同步

**安装ntp服务（所有机器）**

```shell
[root@master cdrom]# yum -y install ntp
```

**设置master为主服务器，开启nptd服务**

```shell
[root@master ~]# vi /etc/ntp.conf 
```

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/bigdata/ambari/3.png" style="zoom: 67%;" />

```shell
# 红色框内是需要修改的部分：
restrict 192.168.0.0 mask 255.255.255.0

# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst

# 新增
server 127.127.1.0
fudge 127.127.1.0 stratum 10
```

**重启服务**

```shell
[root@master ~]# systemctl restart ntpd.service
```

**开机自启动**

```shell
[root@master ~]# systemctl enable ntpd.service
```

**子节点设置同步（子节点）**

主服务器开启ntp服务器以后，子节点就不需要开启了，因为当server与client之间的时间误差过大时（可能是1000秒），处于对修改时间可能对系统和应用带来不可预知的问题，NTP将停止时间同步！所以如果发现NTP启动之后时间并不进行同步时，应该考虑到可能是时间差过大引起的，此时需要先手动进行时间同步！所以直接使用定时手动同步的方式就可以了。

添加任务计划实时同步master服务器时间：

```shell
[root@slave1 ~]# crontab -e
0-59/10 * * * * /usr/sbin/ntpdate master
[root@slave1 ~]# crontab -l
0-59/10 * * * * /usr/sbin/ntpdate master
```

**设置Master时间为当前时间**

```shell
[root@master ~]# date -s '2021-06-30 13:06'
Wed Jun 30 13:06:00 CST 2021
```

#### 删除默认JDK

**删除openJDK（所有节点）**

一些开发版的centos会自带jdk，我们一般用自己的jdk，把自带的删除。先看看有没有安装java -version

```shell
[root@master ~]# java -version
openjdk version "1.8.0_101"
OpenJDK Runtime Environment (build 1.8.0_101-b13)
OpenJDK 64-Bit Server VM (build 25.101-b13, mixed mode)
```

**查找他们的安装位置（所有节点）**

```shell
[root@master ~]# rpm -qa | grep java
java-1.8.0-openjdk-headless-1.8.0.101-3.b13.el7_2.x86_64
tzdata-java-2016f-1.el7.noarch
java-1.8.0-openjdk-1.8.0.101-3.b13.el7_2.x86_64
javapackages-tools-3.4.1-11.el7.noarch
java-1.7.0-openjdk-headless-1.7.0.111-2.6.7.2.el7_2.x86_64
java-1.7.0-openjdk-1.7.0.111-2.6.7.2.el7_2.x86_64
python-javapackages-3.4.1-11.el7.noarch
```

**删除全部，noarch文件可以不用删除（所有节点）**

```shell
[root@master ~]# rpm -e --nodeps java-1.8.0-openjdk-headless-1.8.0.101-3.b13.el7_2.x86_64
[root@master ~]# rpm -e --nodeps java-1.8.0-openjdk-1.8.0.101-3.b13.el7_2.x86_64
[root@master ~]# rpm -e --nodeps java-1.7.0-openjdk-headless-1.7.0.111-2.6.7.2.el7_2.x86_64
[root@master ~]# rpm -e --nodeps java-1.7.0-openjdk-1.7.0.111-2.6.7.2.el7_2.x86_64
```

**检查有没有删除**

```shell
[root@master ~]# java -version
-bash: /usr/bin/java: 没有那个文件或目录
```

注：如果还没有删除，则用`yum -y remove`去删除他们

#### 安装JDK

**上传jdk到Master的/home/tools/目录下**

```shell
[root@master tools]# tar -zxvf jdk-8u291-linux-x64.tar.gz
[root@master tools]# rm -rf jdk-8u291-linux-x64.tar.gz

## 把JDK复制到其他机器
[root@master tools]# scp -r /home/tools/jdk1.8.0_291 root@slave1:/home/tools/jdk1.8.0_291
[root@master tools]# scp -r /home/tools/jdk1.8.0_291 root@slave2:/home/tools/jdk1.8.0_291
```

**进入配置JAVA环境（所以机器）**

```shell
vi /etc/profile

JAVA_HOME=/home/tools/jdk1.8.0_291
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
```

**使环境变量生效（所以机器）**

```shell
[root@master tools]# source /etc/profile
```

**测试JDK**

```shell
[root@master mysql]# java -version
java version "1.8.0_291"
Java(TM) SE Runtime Environment (build 1.8.0_291-b10)
Java HotSpot(TM) 64-Bit Server VM (build 25.291-b10, mixed mode)
```

#### 安装Httpd

**Master安装httpd**

```shell
[root@master tools]# yum -y install httpd
[root@master tools]# service httpd restart
[root@master tools]# chkconfig httpd on
```

#### 安装Mysql

**上传mysql安装包到/home/tools中（Master）**

```shell
# 解压mysql-5.7.17-linux-glibc2.5-x86_64.tar.gz
[root@master tools]# tar -zxvf mysql-5.7.17-linux-glibc2.5-x86_64.tar.gz

# 迁移Mysql
[root@master tools]# mv mysql-5.7.17-linux-glibc2.5-x86_64 /usr/local/mysql
```

**创建mysql用户组及用户**

```shell
[root@master tools]# groupadd mysql
[root@master tools]# useradd -r -g mysql mysql
```

**修改Mysql文件夹用户组及用户**

```shell
[root@master tools]# cd /usr/local/
[root@master local]# chown -R mysql:mysql mysql
```

**创建Mysql数据目录**

```shell
[root@master local]# mkdir -p /usr/local/mysql/data
[root@master local]# chown -R mysql:mysql data
```

**配置Mysql**

```shell
[root@master mysql]# cd /usr/local/mysql/
[root@master mysql]# cp ./support-files/my-default.cnf /etc/my.cnf
[root@master mysql]# vi /etc/my.cnf

[mysqld]

# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M

# Remove leading # to turn on a very important data integrity option: logging
# changes to the binary log between backups.
# log_bin

# These are commonly set, remove the # and set as required.
basedir = /usr/local/mysql
datadir = /usr/local/mysql/data
port = 3306
# server_id = .....
socket = /tmp/mysql.sock
character-set-server = utf8
log_error = error.log
# socket = .....

# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M

#sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
sql_mode=STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTIO
skip_ssl
```

**初始化Mysql**

```shell
[root@master mysql]# yum install perl			# mysql依赖
[root@master mysql]# yum install autoconf		# mysql依赖
[root@master mysql]# ./bin/mysqld --initialize-insecure --user=mysql
```

**设置Mysql开机启动**

```shell
[root@master mysql]# cp /usr/local/mysql/support-files/mysql.server /etc/rc.d/init.d/mysqld
[root@master mysql]# chmod +x /etc/rc.d/init.d/mysqld 
[root@master mysql]# chkconfig --add mysqld

# 启动Mysql
[root@master mysql]# service mysqld start
```

**终端使用mysql命令**

```shell
[root@master mysql]# vi ~/.bash_profile 

# 加入
export PATH=$PATH:/usr/local/mysql/bin

# 加载
[root@master mysql]# source ~/.bash_profile 
```

#### 安装ZIP和UNZIP

**上传zip和unzip安装包到/home/tools中（所有节点）**

```shell
[root@master tools]# rpm -ivh zip-3.0-11.el7.x86_64.rpm
[root@master tools]# rpm -ivh unzip-6.0-21.el7.x86_64.rpm
```

#### 安装libtirpc

**上传libtirpc-devel安装包到/home/tools中（所有节点）**

```shell
[root@master tools]# yum install -y libtirpc
[root@master tools]# rpm -ivh libtirpc-devel-0.2.4-0.15.el7.x86_64.rpm
```

#### 安装Kerberos

**在Master中安装软件**

```shell
[root@master tools]# yum install -y krb5-server krb5-workstation
```

**编辑文件/etc/krb5.conf**

```shell
[root@master tools]# vi /etc/krb5.conf

[libdefaults]
  renew_lifetime = 7d
  forwardable = true
  default_realm = BIGDATA.COM			# 这个名字必须和realms名字相同
  ticket_lifetime = 24h
  dns_lookup_realm = false
  dns_lookup_kdc = false
  default_ccache_name = /tmp/krb5cc_%{uid}
  #default_tgs_enctypes = aes des3-cbc-sha1 rc4 des-cbc-md5
  #default_tkt_enctypes = aes des3-cbc-sha1 rc4 des-cbc-md5

[logging]
  default = FILE:/var/log/krb5kdc.log
  admin_server = FILE:/var/log/kadmind.log
  kdc = FILE:/var/log/krb5kdc.log

[realms]
  BIGDATA.COM = {
    admin_server = master				# 主机名
    kdc = master						# 主机名
  }
```

**编辑文件/var/kerberos/krb5kdc/kdc.conf**

```shell
[root@master tools]# vi /var/kerberos/krb5kdc/kdc.conf

[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 BIGDATA.COM = {							# 名字和krb5.conf中的realms名字相同
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
 }
```

**编辑文件/var/kerberos/krb5kdc/kadm5.acl**

```shell
[root@master tools]# vi /var/kerberos/krb5kdc/kadm5.acl
*/admin@BIGDATA.COM     *
```

**初始化数据库**

```shell
[root@master ~]# kdb5_util create -s -r BIGDATA.COM
Loading random data
Initializing database '/var/kerberos/krb5kdc/principal' for realm 'BIGDATA.COM',
master key name 'K/M@BIGDATA.COM'
You will be prompted for the database Master Password.
It is important that you NOT FORGET this password.
Enter KDC database master key: 								# 数据库密码，可以回车
Re-enter KDC database master key to verify: 				# 数据库密码，可以回车

[root@master ~]# ll /var/kerberos/krb5kdc/				# 生成的文件
total 24
-rw-------. 1 root root   22 Jul  7 00:22 kadm5.acl
-rw-------. 1 root root  451 Jul  7 00:22 kdc.conf
-rw-------. 1 root root 8192 Jul  7 00:23 principal
-rw-------. 1 root root 8192 Jul  7 00:23 principal.kadm5
-rw-------. 1 root root    0 Jul  7 00:23 principal.kadm5.lock
-rw-------. 1 root root    0 Jul  7 00:23 principal.ok
```

**初始化KDC超级管理员**

```shell
[root@master ~]# kadmin.local
Authenticating as principal root/admin@BIGDATA.COM with password.
kadmin.local:  addprinc admin/admin				# 新增admin/admin用户
WARNING: no policy specified for admin/admin@BIGDATA.COM; defaulting to no policy
Enter password for principal "admin/admin@BIGDATA.COM": 					# 输入超级管理员密码
Re-enter password for principal "admin/admin@BIGDATA.COM": 					# 输入超级管理员密码
Principal "admin/admin@BIGDATA.COM" created.

```

**启动kerberos**

```shell
[root@master ~]# systemctl start krb5kdc
[root@master ~]# systemctl start kadmin
[root@master ~]# systemctl enable krb5kdc			# 开机启动
[root@master ~]# systemctl enable kadmin			# 开机启动
```

## 安装Ambari

### 制作本地Ambari源

**将ambari、hdp、hdp-utils放到/var/www/html/hdp目录下，解压（Master）**

```shell
[root@master tools]# mkdir -p /var/www/html/hdp
[root@master tools]# cd /var/www/html/hdp
[root@master hdp]# ll
total 10844988
-rw-r--r--. 1 root root 1947685893 Jun 30 16:55 ambari-2.7.3.0-centos7.tar.gz
-rw-r--r--. 1 root root 9066967592 Jun 30 17:41 HDP-3.1.0.0-centos7-rpm.tar.gz
-rw-r--r--. 1 root root   90606616 Jun 30 17:35 HDP-UTILS-1.1.0.22-centos7.tar.gz

# 解压
[root@master hdp]# tar -zxvf ambari-2.7.3.0-centos7.tar.gz
[root@master hdp]# tar -zxvf HDP-3.1.0.0-centos7-rpm.tar.gz
[root@master hdp]# mkdir HDP-UTILS-1.1.0.22
[root@master hdp]# tar -zxvf HDP-UTILS-1.1.0.22-centos7.tar.gz -C /var/www/html/hdp/HDP-UTILS-1.1.0.22/

# 删除所有的tar.gz包
[root@master hdp]# rm -rf ambari-2.7.3.0-centos7.tar.gz HDP-3.1.0.0-centos7-rpm.tar.gz HDP-UTILS-1.1.0.22-centos7.tar.gz

[root@master hdp]# ll
total 0
drwxr-xr-x. 3 root root  21 Jun 30 17:46 ambari
drwxr-xr-x. 3 1001 users 21 Dec 11  2018 HDP
drwxr-xr-x. 3 root root  23 Jul  1 09:43 HDP-UTILS-1.1.0.22
```

现在可以通过访问http://192.168.1.121/hdp/查看是否能成功访问

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/bigdata/ambari/5.png" style="zoom: 67%;" />

**安装本地源制作相关工具（Master）**

```shell
[root@master hdp]# yum install yum-utils createrepo yum-plugin-priorities -y
[root@master hdp]# createrepo  ./
Spawning worker 0 with 29 pkgs
Spawning worker 1 with 29 pkgs
Spawning worker 2 with 29 pkgs
Spawning worker 3 with 29 pkgs
Spawning worker 4 with 29 pkgs
Spawning worker 5 with 29 pkgs
Spawning worker 6 with 28 pkgs
Spawning worker 7 with 28 pkgs
Workers Finished
Saving Primary metadata
Saving file lists metadata
Saving other metadata
Generating sqlite DBs
Sqlite DBs complete
```

**设置Ambari的yum源（Master）**

```shell
[root@master hdp]# vi ambari/centos7/2.7.3.0-139/ambari.repo 

#VERSION_NUMBER=2.7.3.0-139
[ambari-2.7.3.0]
#json.url = http://public-repo-1.hortonworks.com/HDP/hdp_urlinfo.json
name=ambari Version - ambari-2.7.3.0
#baseurl=http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.7.3.0
baseurl=http://192.168.1.121/hdp/ambari/centos7/2.7.3.0-139
gpgcheck=1
#gpgkey=http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.7.3.0/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
gpgkey=http://192.168.1.121/hdp/ambari/centos7/2.7.3.0-139/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1
```

**设置HDP的yum源（Master）**

```shell
[root@master hdp]# vi HDP/centos7/3.1.0.0-78/hdp.repo 

#VERSION_NUMBER=3.1.0.0-78
[HDP-3.1.0.0]
name=HDP Version - HDP-3.1.0.0
#baseurl=http://public-repo-1.hortonworks.com/HDP/centos7/3.x/updates/3.1.0.0
baseurl=http://192.168.1.121/hdp/HDP/centos7/3.1.0.0-78
gpgcheck=1
gpgkey=http://192.168.1.121/hdp/HDP/centos7/3.1.0.0-78/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1


[HDP-UTILS-1.1.0.22]
name=HDP-UTILS Version - HDP-UTILS-1.1.0.22
#baseurl=http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.22/repos/centos7
baseurl=http://192.168.1.121/hdp/HDP-UTILS-1.1.0.22/HDP-UTILS/centos7/1.1.0.22
gpgcheck=1
#gpgkey=http://public-repo-1.hortonworks.com/HDP/centos7/3.x/updates/3.1.0.0/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
gpgkey=http://192.168.1.121/hdp/HDP-UTILS-1.1.0.22/HDP-UTILS/centos7/1.1.0.22/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1
```

**把设置好的yum文件，复制到/etc/yum.repos.d/（Master）**

```shell
[root@master hdp]# cp ambari/centos7/2.7.3.0-139/ambari.repo /etc/yum.repos.d/
[root@master hdp]# cp HDP/centos7/3.1.0.0-78/hdp.repo /etc/yum.repos.d/
```

**分发yum文件到slave1和slave2中**

```shell
[root@master hdp]# scp ambari/centos7/2.7.3.0-139/ambari.repo root@slave1:/etc/yum.repos.d/
ambari.repo                                                                                                                                                                    100%  529   925.5KB/s   00:00    
[root@master hdp]# scp ambari/centos7/2.7.3.0-139/ambari.repo root@slave2:/etc/yum.repos.d/
ambari.repo       

[root@master hdp]# scp HDP/centos7/3.1.0.0-78/hdp.repo root@slave1:/etc/yum.repos.d/
hdp.repo                                                                                                                                                                       100%  801     1.3MB/s   00:00    
[root@master hdp]# scp HDP/centos7/3.1.0.0-78/hdp.repo root@slave2:/etc/yum.repos.d/
hdp.repo    
```

**把设置好的yum重置（所有节点）**

```shell
[root@master hdp]# yum clean all
[root@master hdp]# yum makecache
[root@master hdp]# yum repolist
```

### 设置Mysql

```shell
[root@master mysql]# mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.7.17 MySQL Community Server (GPL)

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

## 设置登录密码
mysql> set password for 'root'@'localhost'=password('000000');
Query OK, 0 rows affected (0.00 sec)

## 添加远程登录用户
mysql> grant all privileges on *.* to 'root'@'%' identified by '000000';
Query OK, 0 rows affected (0.00 sec)
```

> root远程连接赋所有权限，grant all privileges on *.* to 'root'@'%' with grant option;



### 安装配置Ambari

**安装Ambari-Server（Master）**

```shell
[root@master hdp]# yum install ambari-server
```

**创建Ambari-Server数据表和用户**

```shell
[root@master hdp]# mysql -u root -p

## 创建表
mysql> CREATE DATABASE ambari DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
Query OK, 1 row affected (0.00 sec)

mysql> use ambari;

## 创建ambari用户
mysql> CREATE USER 'ambari'@'%' IDENTIFIED BY 'ambari';
Query OK, 1 row affected (0.00 sec)

mysql> CREATE USER 'ambari'@'localhost' IDENTIFIED BY 'ambari';
Query OK, 1 row affected (0.00 sec)

mysql> CREATE USER 'ambari'@'master' IDENTIFIED BY 'ambari';
Query OK, 1 row affected (0.00 sec)

## 授权
mysql> GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'%';  
Query OK, 0 rows affected (0.00 sec)

mysql> GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'localhost';  
Query OK, 0 rows affected (0.00 sec)

mysql> GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'master';
Query OK, 0 rows affected (0.00 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)

## 将SQL脚本加载到ambari数据库中
mysql> source /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql 
```

**建立mysql与ambari-server的连接**

上传mysql-connector-java-5.1.47.jar到/home/tools中

```shell
[root@master ~]# mkdir /usr/share/java
[root@master ~]# cp /home/tools/mysql-connector-java-5.1.47.jar /usr/share/java/mysql-connector-java.jar
[root@master ~]# cp /usr/share/java/mysql-connector-java.jar /var/lib/ambari-server/resources/mysql-connector-java.jar
```

**初始化配置Ambari并启动**

```shell
[root@master ~]# ambari-server setup
Using python  /usr/bin/python
Setup ambari-server
Checking SELinux...
SELinux status is 'enabled'
SELinux mode is 'permissive'
WARNING: SELinux is set to 'permissive' mode and temporarily disabled.		# 检查防火墙是否关闭
OK to continue [y/n] (y)? y
Customize user account for ambari-server daemon [y/n] (n)? y				# 提示是否自定义设置
Enter user account for ambari-server daemon (root):							# 如果直接回车就是默认选择root用户
Adjusting ambari-server permissions and ownership...
Checking firewall status...
Checking JDK...
[1] Oracle JDK 1.8 + Java Cryptography Extension (JCE) Policy Files 8
[2] Custom JDK
==============================================================================
Enter choice (1): 2															# 设置JDK
WARNING: JDK must be installed on all hosts and JAVA_HOME must be valid on all hosts.
WARNING: JCE Policy files are required for configuring Kerberos security. If you plan to use Kerberos,please make sure JCE Unlimited Strength Jurisdiction Policy Files are valid on all hosts.
Path to JAVA_HOME: /home/tools/jdk1.8.0_291									# JAVA_HOME地址
Validating JDK on Ambari Server...done.
Check JDK version for Ambari Server...
JDK version found: 8
Minimum JDK version is 8 for Ambari. Skipping to setup different JDK for Ambari Server.
Checking GPL software agreement...
GPL License for LZO: https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html
Enable Ambari Server to download and install GPL Licensed LZO packages [y/n] (n)? 
Completing setup...
Configuring database...
Enter advanced database configuration [y/n] (n)? y							# 是否自定义配置数据库
Configuring database...
==============================================================================
Choose one of the following options:
[1] - PostgreSQL (Embedded)
[2] - Oracle
[3] - MySQL / MariaDB
[4] - PostgreSQL
[5] - Microsoft SQL Server (Tech Preview)
[6] - SQL Anywhere
[7] - BDB
==============================================================================
Enter choice (1): 3															# Mysql
Hostname (localhost): 														# Mysql地址
Port (3306): 																# Mysql端口
Database name (ambari): 													# 数据库名
Username (ambari): 															# 用户名
Enter Database Password (bigdata): 											# 密码
Re-enter password: 															# 再次输入密码
Configuring ambari database...
Should ambari use existing default jdbc /usr/share/java/mysql-connector-java.jar [y/n] (y)? 
Configuring remote database connection properties...
WARNING: Before starting Ambari Server, you must run the following DDL directly from the database shell to create the schema: /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql
Proceed with configuring remote database connection properties [y/n] (y)? 
Extracting system views...
ambari-admin-2.7.3.0.139.jar
....
Ambari repo file contains latest json url http://public-repo-1.hortonworks.com/HDP/hdp_urlinfo.json, updating stacks repoinfos with it...
Adjusting ambari-server permissions and ownership...
Ambari Server 'setup' completed successfully.
```

**启动日志查看**

```shell
[root@master ~]# ambari-server start
Using python  /usr/bin/python
Starting ambari-server
Ambari Server running with administrator privileges.
Organizing resource files at /var/lib/ambari-server/resources...
Ambari database consistency check started...
Server PID at: /var/run/ambari-server/ambari-server.pid
Server out at: /var/log/ambari-server/ambari-server.out
Server log at: /var/log/ambari-server/ambari-server.log
Waiting for server start.................
Server started listening on 8080

DB configs consistency check: no errors and warnings were found.
Ambari Server 'start' completed successfully.

# 可通过192.168.1.121:8080查看ambari界面

# 通过tail查看日志
[root@master ~]# tail -f /var/log/ambari-server/ambari-server.log
```

**安装Ambari-Agent并设置自启动（所有节点）**

```shell
[root@master ~]# yum -y install ambari-agent
[root@master ~]# chkconfig --add ambari-agent
```

### 可视化配置Ambari

**访问Ambari web页面**

> 默认端口8080，Username：admin；Password：admin；http://192.168.1.121:8080

**开始集群安装**

点击启用安装向导，只有在没有安装过HDP集群才有这个界面

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/bigdata/ambari/6.png" style="zoom: 67%;" />

输入集群名称

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/bigdata/ambari/7.png" style="zoom: 67%;" />

设置HDP版本和安装源

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/bigdata/ambari/8.png" style="zoom: 67%;" />

设置集群域的名称和上传密钥

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/bigdata/ambari/9.png" style="zoom: 67%;" />

集群注册

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/bigdata/ambari/10.png" style="zoom: 67%;" />

安装基础插件HDFS、YARN + MapReduce2、Zookeeper、Ambari Metrics、SmartSense

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/bigdata/ambari/11.png" style="zoom: 67%;" />

选择节点安装服务

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/bigdata/ambari/12.png" style="zoom: 67%;" />

集群节点选择配置《DataNode，NodeManager，Client》

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/bigdata/ambari/13.png" style="zoom: 67%;" />

设置密码

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/bigdata/ambari/14.png" style="zoom: 67%;" />

组件地址配置

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/bigdata/ambari/15.png" style="zoom: 67%;" />

组件用户名配置

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/bigdata/ambari/16.png" style="zoom: 67%;" />

CPU、内存资源配置

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/bigdata/ambari/17.png" style="zoom: 67%;" />

查看安装信息，点击DEPLOY进行安装

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/bigdata/ambari/18.png" style="zoom: 67%;" />

### Ambari安装问题

#### HDP的本地yum源未设置成功

```shell
RuntimeError: Failed to execute command '/usr/bin/yum -y install hdp-select', exited with code '1', message: '



 One of the configured repositories failed (Unknown),

 and yum doesn't have enough cached data to continue. At this point the only

 safe thing yum can do is fail. There are a few ways to work "fix" this:



     1. Contact the upstream for the repository and get them to fix the problem.
```

**解决（所有节点）**

```shell
# 进入yum.repos.d
[root@master ~]# cd /etc/yum.repos.d/

# 查看，多了一个ambari-hdp-1.repo文件
[root@master yum.repos.d]# ll
total 16
-rw-r--r--. 1 root root 171 Jul  1 17:50 ambari-hdp-1.repo
-rw-r--r--. 1 root root 529 Jul  1 17:29 ambari.repo
-rw-r--r--. 1 root root 801 Jul  1 12:53 hdp.repo
-rw-r--r--. 1 root root 132 Jun 30 01:30 local.repo

# 查看ambari-hdp-1.repo内容，发现本地repo没有设置进去
[root@master yum.repos.d]# cat ambari-hdp-1.repo
[HDP-3.1-repo-1]
name=HDP-3.1-repo-1
baseurl=

path=/
enabled=1
gpgcheck=0
[HDP-UTILS-1.1.0.22-repo-1]
name=HDP-UTILS-1.1.0.22-repo-1
baseurl=

path=/
enabled=1
gpgcheck=0

# 删除所有节点的ambari-hdp-1.repo文件
[root@master yum.repos.d]# rm -rf /etc/yum.repos.d/ambari-hdp-1.repo
```

重新点击`Select Version`，重新将`HDP-3.1`和`HDP-UTILS-1.1.0.22`设置

<img src="./images/8.png" style="zoom:50%;" />



### Ambari组件安装问题

#### Yarn

1.  Error running Client；org.apache.hadoop.security.AccessControlException: Permission denied: user=ambari-qa, access=EXECUTE, inode="/user":hdfs:hdfs:drwxrwx---

   > 解决：
   >
   > [root@slave2 ~]# su - hdfs
   >
   > [hdfs@slave2 ~]$ hdfs dfs -setfacl -m user:ambari-qa:rwx /user



#### Hive

1. resource_management.core.exceptions.Fail: Check db_connection_check was unsuccessful. Exit code: 1. Message: The MySQL JDBC driver has not been set. Please ensure that you have executed 'ambari-server setup --jdbc-db=mysql --jdbc-driver=/usr/share/jdbc_driver'.

   > 解决：ambari-server setup --jdbc-db=mysql --jdbc-driver=/usr/share/java/mysql-connector-java.jar



#### Ranger

1. Specified key was too long; max key length is 767 bytes

   > 解决：Mysql 5.6数据库需要设置`set global innodb_file_format=BARRACUDA;`和`set global innodb_large_prefix=1;`

2. If Ranger Hive Plugin is disabled. hive.security.authorization.manager needs to be set to org.apache.hadoop.hive.ql.security.authorization.plugin.sqlstd.SQLStdHiveAuthorizerFactory

   > 解决：在Hive配置中将hive.security.authorization.manager设置为 org.apache.hadoop.hive.ql.security.authorization.plugin.sqlstd.SQLStdHiveAuthorizerFactory



#### Hbase

1. Warning Ambari Metrics hbase_master_heapsize 896 Value is less than the recommended default of 1024. HBase Master Heap Size. In embedded mode, total heap size is sum of master and regionserver heap sizes.

   > 解决：在Ambari Metrics中将HBase Master Maximum Memory调整为1024



#### Knox

1. dfs.permissions.enabled needs to be set to true if Ranger HDFS Plugin is enabled.

   > 解决：在HDFS中将dfs.permissions.enabled设置为true



#### Atlas

1. Atlas is configured to use the HBase installed in this cluster. If you would like Atlas to use another HBase instance, please configure this property and HBASE_CONF_DIR variable in atlas-env appropriately.

   > 解决：将atlas.graph.storage.hostname设置为对应服务器的IP或根据上实例中的master,slave1,slave2



#### Linux

1. Linux内核报错，Message from syslogd@slave2 at Jul  1 19:16:26 ... kernel:NMI watchdog: BUG: soft lockup - CPU#7 stuck for 36s! [sshd:29945]

   > 解决：
   >
   > echo 30 > /proc/sys/kernel/watchdog_thresh
   >
   > echo "kernel.watchdog_thresh=30" >> /etc/sysctl.conf
   >
   > sysctl -w kernel.watchdog_thresh=30
   >
   > sysctl -q vm.swappiness