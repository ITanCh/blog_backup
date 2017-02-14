title: MESOS and MARATHON安装
date: 2015-11-16 09:40:40
tags: [MESOS,技术,ubuntu]
---
### 基本环境  
系统环境：ubuntu14.04
mesos: 0.25.0  

### 从源代码安装  
根据官网[Getting Started](http://mesos.apache.org/documentation/latest/getting-started/)介绍，尝试从源代码编译安装，但未成功。在`make check`命令执行时出现一个错误，经过分析终于发现原因在于测试中会通过字符串比较验证结果，由于我的系统是中文系统，测试中预想的结果是英文的，所以就导致错误。最后，需要能够访问code.google.com才能进行`make install`。  

从源代码安装时会安装一些依赖库，同时会安装openjdk。
<!--more-->
### apt-get  
根据[mesosphere](https://open.mesosphere.com/getting-started/install/)介绍，可以安装以下步骤进行安装：

1. 添加库  
{% codeblock lang:sh %}
$ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv E56151BF
$ DISTRO=$(lsb_release -is | tr '[:upper:]' '[:lower:]')
$ CODENAME=$(lsb_release -cs)
$ echo "deb http://repos.mesosphere.com/${DISTRO} ${CODENAME} main" | \
  sudo tee /etc/apt/sources.list.d/mesosphere.list
$ sudo apt-get -y update
{% endcodeblock %}

2. 安装  
{% codeblock lang:sh %}
$ sudo apt-get -y install mesos
{% endcodeblock %}

3. 启动master  
{% codeblock lang:sh %}
$ sudo mesos-master --ip=127.0.0.1 --work_dir=/var/lib/mesos
{% endcodeblock %}

4. 启动slave  
{% codeblock lang:sh %}
$ mesos-slave --master=127.0.0.1:5050
{% endcodeblock %}

5. 从浏览器观察结果`http://127.0.0.1:5050`。

### MARATHON安装  
MARATHON依赖于Java 8。参考(mesosphere)[https://mesosphere.com/downloads/]的内容，安装过程如下：  

1. 添加必要的库  
{% codeblock lang:sh %}
$ sudo apt-get update -y
$ sudo DEBIAN_FRONTEND=noninteractive apt-get install -y python-software-properties software-properties-common
$ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv E56151BF
$ DISTRO=$(lsb_release -is | tr '[:upper:]' '[:lower:]')
$ CODENAME=$(lsb_release -cs)
$ sudo echo "deb http://repos.mesosphere.io/${DISTRO} ${CODENAME} main" | tee /etc/apt/sources.list.d/mesosphere.list
{% endcodeblock %}

2. 安装Java 8  
{% codeblock lang:sh %}
$ sudo add-apt-repository ppa:webupd8team/java
$ sudo apt-get update -y
$ sudo apt-get install -y oracle-java8-installer oracle-java8-set-default
$ sudo apt-get -y install marathon
{% endcodeblock %}

3. 运行MARATHON
{% codeblock lang:sh %}
$ marathon --master zk://127.0.0.1:2181,127.0.0.1:2181/mesos --zk zk://127.0.0.1:2181,127.0.0.1:2181/marathon
{% endcodeblock %}

4. 从浏览器观察结果`http://127.0.0.1:8080`

### MESOS的一些命令
To (start | stop | restart) mesos-master:
{% codeblock lang:sh %}
sudo service mesos-master (start | stop | restart)
{% endcodeblock %}  

To (start | stop | restart) mesos-slave:
{% codeblock lang:sh %}
sudo service mesos-slave (start | stop | restart)
{% endcodeblock %}

If you want to disable these service from launching automatically on a reboot:
{% codeblock lang:sh %}
echo manual > /etc/init/mesos-master.override
echo manual > /etc/init/mesos-slave.override
echo manual > /etc/init/zookeeper.override
{% endcodeblock %}
