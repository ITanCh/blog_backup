title: Android系统源码阅读(5):Service在进程内绑定
date: 2017-02-14 13:56:11
tags: [Android,技术]
---

# Android系统源码阅读（5）：Service在进程内绑定   

>该系列只记录阅读代码时遇到的问题和心得体会，具体代码讲解可以参考老罗的《Android系统源代码情景分析》，我就不班门弄斧了。我编译的AOSP版本：6.0.1_r50。 

## Step1. Activity开始启动Service   
![这里写图片描述](http://img.blog.csdn.net/20160818225346176)

<!--more-->
## Step2. ActivityManagerService中准备
![这里写图片描述](http://img.blog.csdn.net/20160819144636114)

这里需要注意有两次进程间通信，先讲第7步 (Step3)，再讲第9步(Step4)。

## Step3. Service创建   
![这里写图片描述](http://img.blog.csdn.net/20160818225823568)

## Step4. Service公布IBinder   
![这里写图片描述](http://img.blog.csdn.net/20160818225912992)

## Step5. ActivityManagerService公布Service   
![这里写图片描述](http://img.blog.csdn.net/20160819144711212)

## Step6. Activity建立Connection   
![这里写图片描述](http://img.blog.csdn.net/20160818230007631)
