title: Torque安装笔记(多节点)
date: 2015-08-06 11:59:57
tags: [Torque,技术,安装]
---
1.更新系统软件，防止相关软件版本过低  

```
sudo apt-get update  
sudo apt-get upgrade
```


2.从官网下载Torque，在这里为Torque 5.1.1。  

3.解压文件

```
tar -xzvf torque.tar.gz
```

4.运行configure，可加prefix指定Torque命令安装位置，也可不加，不加参数命令默认装在/usr/local/bin和/usr/local/sbin下。利用configure可以检测依赖软件，按照提示安装这些软件。

```
./configure --prefix=/usr/local/torque
```
<!--more-->

5.编译安装

```
make  
make install  
```

6.生成子节点安装包，mom和clients为需要拷贝的文件。需要安装ssh，ubuntu默认安装openssh-client，所以需要手动再安装openssh-server，让其他计算机登陆。  

```
make packages
scp tpackages name@ip:dir  #非必要
scp torque-package-mom-linux-x86_64.sh name@ip:dir  
scp torque-package-clients-linux-x86_64.sh name@ip:dir
```

7.在子节点上安装，mom和clients。

```
sudo ./torque-package-mom-linux-x86_64.sh --install
sudo ./torque-package-clients-linux-x86_64.sh --install
```

8.配置主节  

+ 将/usr/local/torque/bin和/usr/local/torque/sbin添加进环境变量，这里我将其添加入.bashrc文件。  

{% codeblock %}
export PATH=$PATH:/usr/local/torque/bin:/usr/local/torque/sbin  
{% endcodeblock %}  

+ 添加共享库到Torque的配置文件中（如果出现有什么库找不到，则必须加上）  

{% codeblock %}
echo '/usr/local/lib' > /etc/ld.so.conf.d/torque.conf
ldconfig  
{% endcodeblock %}

+ 添加主节点名字：  

{% codeblock %}
echo 主节点名字 > /var/spool/torque/server_name  
{% endcodeblock %}  

+ 初始化serverdb文件，在使用pbs_sever之前必须完成该步骤：

{% codeblock %}
./torque.setup 用户名
{% endcodeblock %}

+ 添加子节点，np为节点cpu核个数  

vi /var/spool/torque/server_priv/nodes  
子节点的名字为计算机名，而非用户名。每次添加节点需要重启pbs_sever。添加如下内容（也可以先在子节点启动mom，在主节点启动server，然后用命令`qmgr -c 'create node ubuntu np=4'`进行添加节点）：
{% codeblock %}
子节点1名字 np=4  
子节点2名字 np=4
{% endcodeblock %}

+ 启动

{% codeblock %}
pbs_server
pbs_sched  
pbs_mom
{% endcodeblock %}

9.配置子节点，同样需要添加环境变量。  

+ 修改config
使用make packages 命令生成mom安装于子节点的，无需进行这一步。
{% codeblock %}
 vi /var/spool/torque/mom_priv/config  
添加：  
pbsserver 主节点名称  
logevent 255  
{% endcodeblock %}

+ 启动：

{% codeblock %}
pbs_mom
{% endcodeblock %}

10.注意！因为配置文件中使用了*主节点名称*和*子节点名称*，所以主、从节点都需要修改

{% codeblock %}
/etc/hosts  
添加这些名称和对应的ip
{% endcodeblock %}

11.检查运行情况  
保证在server上运行`trqauthd`、`pbs_server`、`pbs_sched`这三个程序。  
保证在子节点上运行`pbs_mom`。

12.常用命令  
`pbsnodes`: 子节点信息。  
`qterm`: 结束pbs_server。
`qsub`: 提交job。
`qstat`: job运行信息。

13.测试：
注意！在提交Job时，需要主节点和子节点为相同的用户名，同时保证操作的文件目录也相同。
{% codeblock %}
echo "sleep 300" | qsub  
qstat
{% endcodeblock %}
