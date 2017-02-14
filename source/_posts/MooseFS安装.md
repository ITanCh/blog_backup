title: MooseFS安装
date: 2015-09-22 16:21:51
tags: [MooseFS,技术,ubuntu]
---
> MooseFS is a Fault tolerant, Highly available, Highly performing, Scaling-out, Network distributed file system. It spreads data over several physical commodity servers, which are visible to the user as one resource. For standard file operations MooseFS acts like any other Unix-like file system:
		1.A hierarchical structure (directory tree)
    2.Stores POSIX file attributes (permissions, last access and modification times)
    3.Supports special files (block and character devices, pipes and sockets)
    4.Symbolic links (file names pointing to target files, not necessarily on MooseFS) and hard links (different names of files which refer to the same data on MooseFS)
    5.Access to the file system can be limited based on IP address and/or password

### 基本信息  
实验环境：两个Ubuntu14.04计算机。  
MooseFS版本：2.0.76。  

MooseFS[官网1](http://www.moosefs.org/)，[官网2](http://moosefs.com/index.html)。据观察，官网2内容较新。

[参考1](http://moosefs.com/download/ubuntudebian.html)，[参考2](http://moosefs.com/Content/Downloads/MooseFS-2-0-60-User-Manual.pdf)。
<!--more-->

### 安装master(在选定的master节点上操作)
添加key:  
{% codeblock lang:sh %}
$ wget -O - http://ppa.moosefs.com/moosefs.key | sudo apt-key add -
{% endcodeblock %}  

在文件`/etc/apt/sources.list.d/moosefs.list`中添加如下软件源：  
{% codeblock lang:sh %}
deb http://ppa.moosefs.com/stable/apt/ubuntu/trusty trusty main
{% endcodeblock %}

更新软件源，安装master：
{% codeblock lang:sh %}
$ sudo apt-get update
$ sudo apt-get install moosefs-master
{% endcodeblock %}

拷贝`/etc/mfs`下的如下文件，可以按需修改其中的参数，也可以使用默认值：
{% codeblock lang:sh %}
$ cp mfsmaster.cfg.dist mfsmaster.cfg
$ cp mfsexports.cfg.dist mfsexports.cfg
{% endcodeblock %}

启动：  
{% codeblock lang:sh %}
$ sudo mfsmaster start
{% endcodeblock %}  
可以修改`/etc/default/moosefs-ce-master`中的`MFSMASTER_ENABLE`为`true`，使得该服务开机启动。

### 安装其它工具  
{% codeblock lang:sh %}
$ sudo apt-get install moosefs-cgi
$ sudo apt-get install moosefs-cgiserv
$ sudo apt-get install moosefs-cli
{% endcodeblock %}  

启动mfscgiserv，可以在浏览器观察mfs运行情况。如果本机为master，可以访问[http://127.0.0.1:9425](http://127.0.0.1:9425)。
{% codeblock lang:sh %}
$ sudo mfscgiserv start
{% endcodeblock %}

### 安装chunkserver(在选定的chunk节点上操作)  
首先像安装master一样添加软件源，然后安装：
{% codeblock lang:sh %}
$ sudo apt-get install moosefs-chunkserver
{% endcodeblock %}  

同样，修改`/etc/mfs`文件下的如下文件：  
{% codeblock lang:sh %}
$ sudo cp mfschunkserver.cfg.dist mfschunkserver.cfg
$ sudo cp mfshdd.cfg.dist mfshdd.cfg
{% endcodeblock %}
其中，`mfschunkserver.cfg`文件的参数`MASTER_HOST = mfsmaster`可以指定master的hostname，这里需要在`/etc/hosts`中写明，并且如果master也作为chunkserver时，hosts中mfsmaster的ip不能使用127.0.0.1。

建立文件,文件名字可以写自己喜欢的，这里取名mfschunk：
{% codeblock lang:sh %}
$ mkdir -p /mnt/mfschunk
$ chown -R mfs:mfs /mnt/mfschunk
{% endcodeblock %}

在`mfshdd.cfg`文件中添加一行`/mnt/mfschunk`。启动mfschunkserver:
{% codeblock lang:sh %}
$ sudo mfschunkserver start
{% endcodeblock %}

### 安装客户端  
安装客户端，添加挂载点，进行挂载。mfsmaster在这里为master的hostname。
{% codeblock lang:sh %}
$ sudo apt-get install moosefs-client
$ sudo mkdir -p /mnt/mfs
$ sudo mfsmount mfs -H mfsmaster
{% endcodeblock %}

查看：
{% codeblock lang:sh %}
$ df -h
mfsmaster:9421  1.3T   70G  1.2T    6% /mnt/mfs
{% endcodeblock %}
