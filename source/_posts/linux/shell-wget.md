title: WGET命令
tags:
  - linux
  - shell
  - wget
categories:
  - linux
author: Figthing
copyright: true
date: 2018-01-07 21:53:00
updated: 2018-01-08 13:18:54
---
### 简介

wget命令用来从指定的URL下载文件。wget非常稳定，它在带宽很窄的情况下和不稳定网络中有很强的适应性，如果是由于网络的原因下载失败，wget会不断的尝试，直到整个文件下载完毕。如果是服务器打断下载过程，它会再次联到服务器上从停止的地方继续下载。这对从那些限定了链接时间的服务器上下载大文件非常有用。


### 参数说明

``` bash
-a<日志文件>：在指定的日志文件中记录资料的执行过程；
-A<后缀名>：指定要下载文件的后缀名，多个后缀名之间使用逗号进行分隔；
-b：进行后台的方式运行wget；
-B<连接地址>：设置参考的连接地址的基地地址；
-c：继续执行上次终端的任务；
-C<标志>：设置服务器数据块功能标志on为激活，off为关闭，默认值为on；
-d：调试模式运行指令；
-D<域名列表>：设置顺着的域名列表，域名之间用“，”分隔；
-e<指令>：作为文件“.wgetrc”中的一部分执行指定的指令；
-h：显示指令帮助信息；
-i<文件>：从指定文件获取要下载的URL地址；
-l<目录列表>：设置顺着的目录列表，多个目录用“，”分隔；
-L：仅顺着关联的连接；
-r：递归下载方式；
-nc：文件存在时，下载文件不覆盖原有文件；
-nv：下载时只显示更新和出错信息，不显示指令的详细执行过程；
-q：不显示指令执行过程；
-nh：不查询主机名称；
-v：显示详细执行过程；
-V：显示版本信息；
--passive-ftp：使用被动模式PASV连接FTP服务器；
--follow-ftp：从HTML文件中下载FTP连接文件。
```

<!--more-->

## 示例

### 下载单个文件

> 以下的例子是从网络下载一个文件并保存在当前目录，在下载的过程中会显示进度条，包含（下载完成百分比，已经下载的字节，当前下载速度，剩余下载时间）。

``` shell
$ wget http://www.xxx.net/xxx.zip
```


### 下载并以不同的文件名保存

> wget默认会以最后一个符合/的后面的字符来命令，对于动态链接的下载通常文件名会不正确。

``` shell
$ wget -O wordpress.zip http://www.xxx.net/xxx.aspx?id=1080
```


### wget限速下载

> 当你执行wget的时候，它默认会占用全部可能的宽带下载。但是当你准备下载一个大文件，而你还需要下载其它文件时就有必要限速了。

``` shell
$ wget --limit-rate=300k http://www.xxx.net/testfile.zip
```


### 使用wget断点续传

> 使用wget -c重新启动下载中断的文件，对于我们下载大文件时突然由于网络等原因中断非常有帮助，我们可以继续接着下载而不是重新下载一个文件。需要继续中断的下载时可以使用-c参数。

``` shell
$ wget -c http://www.xxx.net/testfile.zip
```

### 使用wget后台下载

``` shell
$ wget -b http://www.xxx.net/testfile.zip
```


### 伪装代理名称下载

``` shell
$ wget --user-agent="Mozilla/5.0 (Windows; U; Windows NT 6.1; en-US) AppleWebKit/534.16 (KHTML, like Gecko) Chrome/10.0.648.204 Safari/534.16" http://www.xxx.net/testfile.zip
```

### 测试下载链接
> 当你打算进行定时下载，你应该在预定时间测试下载链接是否有效。我们可以增加--spider参数进行检查。

``` shell
$ wget --spider URL
```

### 增加重试次数
> 如果网络有问题或下载一个大文件也有可能失败。wget默认重试20次连接下载文件。如果需要，你可以使用--tries增加重试次数。

``` shell
$ wget --tries=40 URL
```

### 下载多个文件
> 首先，保存一份下载链接文件：
cat > filelist.txt
url1
url2
url3
url4
接着使用这个文件和参数-i下载。

``` shell
$ wget -i filelist.txt
```

### 镜像网站
> 下载整个网站到本地。
--miror开户镜像下载。
-p下载所有为了html页面显示正常的文件。
--convert-links下载后，转换成本地的链接。
-P ./LOCAL保存所有文件和目录到本地指定目录。

``` shell
$ wget --mirror -p --convert-links -P ./LOCAL URL
```

### 过滤指定格式下载
> 下载一个网站，但你不希望下载图片，可以使用这条命令。

``` shell
$ wget --reject=gif ur
```

### 把下载信息存入日志文件
> 不希望下载信息直接显示在终端而是在一个日志文件，可以使用。

``` shell
$ wget -o download.log URL
```

### 限制总下载文件大小
> 当你想要下载的文件超过5M而退出下载，你可以使用。注意：这个参数对单个文件下载不起作用，只能递归下载时才有效。

``` shell
$ wget -Q5m -i filelist.txt
```

### 下载指定格式文件
> 可以在以下情况使用该功能：
下载一个网站的所有图片。
下载一个网站的所有视频。
下载一个网站的所有PDF文件。

``` shell
$ wget -r -A.pdf url
```

### FTP下载

``` shell
$ wget ftp-url
$ wget --ftp-user=USERNAME --ftp-password=PASSWORD url
```

### Https下载

``` shell
$ wget -r -np -nd --accept=gz --no-check-certificate https://www.xxx.com/dir/ --http-user=username --http-password=password
```