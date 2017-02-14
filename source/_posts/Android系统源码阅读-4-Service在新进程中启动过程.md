title: Android系统源码阅读(4):Service在新进程中启动过程
date: 2017-02-14 13:54:43
tags: [Android,技术]
---
# Android系统源码阅读（4）：Service在新进程中启动过程   

>该系列只记录阅读代码时遇到的问题和心得体会，具体代码讲解可以参考老罗的《Android系统源代码情景分析》，我就不班门弄斧了。我编译的AOSP版本：6.0.1_r50。   

## Step1. Activity开始启动Service  
![这里写图片描述](http://img.blog.csdn.net/20160818135053514)  

<!--more-->
## Step2. ActivityManagerService准备   
![这里写图片描述](http://img.blog.csdn.net/20160818135157155)    

## Step3. ActivityThread新进程启动    
![这里写图片描述](http://img.blog.csdn.net/20160818135237858)   

## Step4. ActivityManagerService启动等待的Service   
![这里写图片描述](http://img.blog.csdn.net/20160818135341778)

## Step5. Server创建   
![这里写图片描述](http://img.blog.csdn.net/20160818135416357)
