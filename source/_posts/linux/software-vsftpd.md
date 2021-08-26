title: vsftpd安装
author: Figthing
tags:
  - linux
  - software
  - ''
  - vsftpd
categories:
  - linux
date: 2018-01-12 20:39:00
---
#### 准备工作

- linux系统centos7
- vsftpd软件
- yum安装

#### 执行步骤

##### 安装vsftpd服务器

 ``` shell
 $ yum install vsftpd
 ```

##### 安装一个加密工具

 ```shell
 $ yum install libdb-utils.x86_64
 ```

##### 修改配置VSFTP

 ```shell
 $ vi /etc/vsftpd/vsftpd.conf
 ```

##### 配置文件参数说明

 > 
 anonymous_enable=NO #设定不允许匿名访问
 local_enable=YES #设定本地用户可以访问。注：如使用虚拟宿主用户，在该项目设定为NO的情况下所有虚拟用户将无法访问。
 chroot_list_enable=YES #使用户不能离开主目录
 ascii_upload_enable=YES #允许使用ASCII模式上传
 ascii_download_enable=YES #设定支持ASCII模式的上传和下载功能。
 pam_service_name=vsftpd #PAM认证文件名。PAM将根据/etc/pam.d/vsftpd进行认证
 guest_enable=YES #设定启用虚拟用户功能。
 guest_username=ftp #指定虚拟用户的宿主用户。-RHEL/CentOS中已经有内置的ftp用户了
 user_config_dir=/etc/vsftpd/vuser_conf #设定虚拟用户个人vsftp的RHEL/CentOS FTP服务文件存放路  径。
 listen=YES	# 只监听ipv4的地址
 xferlog_file=/var/log/xferlog	# 日志文件的路径
 listen_port=1315		#FTP端口
 pasv_enable=YES			#开启被动模式
 pasv_min_port=10060
 pasv_max_port=10070

##### 创建chroot list，将ftp用户加入其中

 ```shell
 $ touch /etc/vsftpd/chroot_list
 $ echo ftp >> /etc/vsftpd/chroot_list
 ```

<!-- more -->


##### 安装Berkeley DB工具

 ```shell
 $ yum install db4 db4-utils
 ```

##### 创建用户密码文本

 ```shell
 $ touch /etc/vsftpd/vuser_passwd.txt 
 ```

##### 生成虚拟用户认证的db文件

 ```shell
 $ db_load -T -t hash -f /etc/vsftpd/vuser_passwd.txt /etc/vsftpd/vuser_passwd.db
 $ chmod 600 /etc/vsftpd/vuser_passwd.db
 ```

##### 编辑认证文件

 ```shell
 $ vi /etc/pam.d/vsftpd
 ```

 > 
 把前面的注释去掉，然后加上以下几条
 
 > 系统为32位：
 auth required pam_userdb.so db=/etc/vsftpd/vuser_passwd account 
 required pam_userdb.so db=/etc/vsftpd/vuser_passwd

 > 系统为64位：
auth required /lib64/security/pam_userdb.so 
db=/etc/vsftpd/vuser_passwd account required 
/lib64/security/pam_userdb.so db=/etc/vsftpd/vuser_passwd

##### 修改VSFTPD端口

 执行`vi /etc/services`，将其中的 `ftp 21/tcp` 改为 `ftp 1315/tcp` , `ftp21/udp`改为 `ftp 1315/udp`

##### 重启动vsftp服务

 ```shell
 $ service vsftpd restart
 ```








