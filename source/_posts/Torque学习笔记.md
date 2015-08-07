title: Torque学习笔记
date: 2015-08-06 11:59:57
tags: [Torque,技术]
---
1.更新系统软件，防止相关软件版本过低  

```
sudo apt-get update  
sudo apt-get upgrade
```


2.从官网下载Torque  

3.解压文件

```
tar -xzvf torque.tar.gz
```

4.运行configure，可加prefix指定Torque命令安装位置，也可不加。利用configure可以检测依赖软件，按照提示安装这些软件。

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

+ 添加共享库到Torque的配置文件中（不确定是否必须）  

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
./torque.setup root
{% endcodeblock %}

+ 添加子节点，np为节点cpu核个数

{% codeblock %}
vi /var/spool/torque/server_priv/nodes  
添加如下内容：  
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

{% codeblock %}
 vi /var/spool/torque/mom_priv/config  
添加：  
$pbsserver 主节点名称  
$logevent 255  
{% endcodeblock %}

+ 启动：

{% codeblock %}
pbs_mom
{% endcodeblock %}

10.注意！因为配置文件中使用了*主节点名称*和*子节点名称*，所以需要修改

{% codeblock %}
/etc/hosts  
添加这些名称和对应的ip
{% endcodeblock %}

11.测试：

{% codeblock %}
echo "sleep 300" | qsub  
qstat
{% endcodeblock %}
