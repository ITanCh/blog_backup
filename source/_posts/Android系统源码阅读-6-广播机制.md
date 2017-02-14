title: Android系统源码阅读(6)广播机制
date: 2017-02-14 13:57:56
tags: [Android,技术]
---
# Android系统源码阅读（6）：广播机制

>该系列只记录阅读代码时遇到的问题和心得体会，具体代码讲解可以参考老罗的《Android系统源代码情景分析》，我就不班门弄斧了。我编译的AOSP版本：6.0.1_r50。    

## 注册广播接收器

### Step1. Activity开始注册
![这里写图片描述](http://img.blog.csdn.net/20160821154639757)

<!--more-->

### Step2. ActivityManagerService处理注册
![这里写图片描述](http://img.blog.csdn.net/20160821154752054)

## 发送广播  

### Step1. Activity发送广播
![这里写图片描述](http://img.blog.csdn.net/20160821154940083)

### Step2. ActivityManagerService处理广播消息
![这里写图片描述](http://img.blog.csdn.net/20160821155023461)

### Step3. Activity接收广播
![这里写图片描述](http://img.blog.csdn.net/20160821155114881)

