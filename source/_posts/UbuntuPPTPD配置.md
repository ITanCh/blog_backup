title: Ubuntu PPTPD 配置
date: 2017-02-14 13:12:48
tags: [ubuntu,技术]
---

## Server配置

### 安装pptpd  
首先需要安装pptpd，ubuntu 默认安装该工具。 

<!--more-->

### 配置网络
编辑 `/etc/pptpd.conf`，取消以下两行注释。当然也可以自己设定ip地址范围，这里使用默认的就可以。   
```
localip 192.168.0.1
remoteip 192.168.0.234-238,192.168.0.245
```

### 添加用户
编辑`/etc/ppp/chap-secrets`文件，用户名添加如下：   
```
 # Secrets for authentication using CHAP
 # client    server  secret          IP addresses
   [client name]  *   "[password]" *
```  
其中client的名字和密码自己设定即可，其它两项默认"*"就可以。

### 设置DNS
编辑`/etc/ppp/pptpd-options`，找到`ms-dns`部分，取消注释：   
```
 ms-dns 223.5.5.5
 ms-dns 223.6.6.6
```  
默认的是google的DNS服务，`8.8.8.8`。因为国内使用不便，这里我修改成阿里的DNS服务。

### 开启IP转发
编辑`/etc/sysctl.conf`文件，取消注释：
```
net.ipv4.ip_forward=1
```  
运行命令，使配置生效：   
```
sysctl -p
```

### iptables安装
安装iptables:   
```
apt-get install iptables
iptables -t nat -I POSTROUTING -j MASQUERADE #每次运行前需要运行该命令
```

### 重启pptpd
```
/etc/init.d/pptpd restart
```

##Client配置
在你需要使用代理的ubuntu设备上执行以下操作。当然，也可以用其它客户端。

### 安装pptp
一般系统已经默认安装。  
```
sudo apt-get install pptp-linux
```

### 创建一个连接
```
sudo pptpsetup --create [server name] --server [server ip] --username [client name] --password [password] --encrypt --start
```
执行完上述命令后，会在`/etc/ppp/chap-secrets`自动生成相应的用户：     
```
# added by pptpsetup for pptpd
[client name] [server name] "password" *
```   
在`/etc/ppp/peers/pptpd`中也会有生成的相应信息，这里需要注意一点，默认的是不用server 的DNS配置的，所以会出现ping ip地址可以成功，ping网址却失败的DNS解析错误，所以要加上如下一行:    
```
usepeerdns
```

在上述命令成功后，`ifconfig`命令下回出现ppp0端口。

### 路由设置
将ppp0设置为默认路由端口：   
```
ip route del default
ip route add default dev ppp0
```
这样配置基本算成功了。

### 服务开关
我自己写了如下命令方便开关和路由设置：

打开服务(需要sudo):   
```
#!/bin/bash
pon [server name]
sleep 4
ip route del default
ip route add default dev ppp0
```
关闭服务(sudo):
```
#!/bin/bash
poff pptpd
sleep 4
ip route del default
ip route add default via [your old route ip] dev eno1
```
这里需要记住原来设备的路由ip，以便删除后再回复。
