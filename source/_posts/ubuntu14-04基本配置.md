title: ubuntu14.04基本配置
date: 2015-08-06 11:59:57
tags: [ubuntu,技术]
---

## 更新：

```
$ sudo apt-get update  
$ sudo apt-get upgrade
```

## 安装搜狗输入法：

```
$ sudo add-apt-repository ppa:fcitx-team/nightly	#添加fcitx源
$ sudo apt-get install fcitx  
#下载搜狗输入法双击安装
```
<!--more-->

## 安装JDK：  

```
#下载最近的jdk包  
$ sudo mv jdk.tar.gz /usr/lib/jvm/  
$ cd /usr/lib/jvm/  
$ sudo tar -zxvf jdk.tar.gz  
$ sudo rm jdk.tar.gz  

$ sudo update-alternatives --install /usr/bin/java java /usr/lib/jvm/jdk/bin/java 300  
$ sudo update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/jdk/bin/javac 300

#选择默认java库
$ sudo update-alternatives --config java  
$ java -version	#测试  
```
## 安装Eclispe  

```sh
#下载最新的elipse包  

$ sudo mv eclipse.tar.gz /opt/
$ cd /opt/
$ sudo tar -zxvf eclipse.tar.gz
$ sudo rm eclipse.tar.gz

#若想从命令行启动  
$ sudo ln -sf /opt/eclipse/eclipse /usr/bin

#创建桌面图标  
$ sudo vi /usr/share/applications/eclipse.desktop
#添加如下内容  
[Desktop Entry]
Type=Application
Name=Eclipse
Comment=Eclipse Integrated Development Environment
Icon=/opt/eclipse/icon.xpm
Exec=eclipse
Terminal=false
Categories=Development;IDE;Java;  
```

## 配置Android SDK:  

```  
#从官网下载最新的SDK

#将文件解压到目标文件夹/opt/

$ cd /opt/android-sdk/tools

$ sudo  ./android #运行Android SDK Manager,下载platfor-tools和bulid-tools

$ sudo chmod 777 android-sdk	#修改所有文件的权限,使之可以运行

#64位的ubuntu 13.10及以上版本需要安装32位相关的库
$ sudo dpkg --add-architecture i386
$ sudo apt-get update
$ sudo apt-get install libncurses5:i386 libstdc++6:i386 zlib1g:i386

$ sudo vi /ect/profile  
#添加如下内容
export PATH=/opt/android-sdk-linux/platform-tools:$PATH  
export PATH=/opt/android-sdk-linux/tools:$PATH
$ source /ect/profile

```

## Firefox Flash插件

```
#下载最新的Flash Player

$ tar -zxvf flash.tar.gz
#获取flashplayer.so

$ sudo cp libflashplayer.so /usr/lib/mozilla/plugins
```

## 常用命令  
1. 取消某个软件(tomcat)开机自启动。

```
sudo update-rc.d tomcat disable
```