title: Android系统源码阅读(3):子Activity在进程内的启动过程
date: 2017-02-14 13:51:41
tags: [Android,技术]
---

>该系列只记录阅读代码时遇到的问题和心得体会，具体代码讲解可以参考老罗的《Android系统源代码情景分析》，我就不班门弄斧了。我编译的AOSP版本：6.0.1_r50。

## 子Activity在进程内的启动过程  

### Step1. Activity开始启动另一个Activity    
![这里写图片描述](http://img.blog.csdn.net/20160817155832276)   
这里和从Launcher启动过程几乎没分别，但是启动有些参数的设置还是有区别的。

<!--more-->

### Step2. ActivityManagerService准备  
![这里写图片描述](http://img.blog.csdn.net/20160817160041764)
同样这些会先停止正在最前端的Activity。

### Step3. 旧Activity停止    
![这里写图片描述](http://img.blog.csdn.net/20160817160321578)
同样这里需要在完成停止过程后告知ActivityManagerService。

### Step4.  ActivityManagerService继续准备 
![这里写图片描述](http://img.blog.csdn.net/20160817160534704) 
万事俱备，只待真的启动Activity。

### Step5. 新Activity创建
![这里写图片描述](http://img.blog.csdn.net/20160817160656204)     
到这里启动完毕。

## Task, Process and Activity
这几个概念真的很让人混淆，最基本的一句话是它们几个是完全不同的抽象概念。

先说Process和Activity。一个Application可以多个Activity和多个Process。两个Activity可以运行在相同的Process里，也可以在不同的Process里，这里可以在AndroidManifest.xml中设置`android:process`，默认情况下是运行在同一个Process里的。并且该进程中的Activity、Service都是运行在主线程中，所以Service也不是一个独立的线程概念，不可以直接在Service中运行耗时的任务。

Task同样和Process没有一一对应的关系。Task是描述Activity跳转状态的栈，利用后退操作是在同一个栈中进行Activity的回退；同样，某一个Activity启动了其它Application的Activity（比如你的app调用了系统自带的相册app），这两个Activity同样会放置于同一个栈中，虽然这两个acitivity明显不在同一个Application中，也有可能不在同一个Process中。Task在用户通过任务管理器来切换时，则会发生交换。将前台的Task放到后面，你选择的Task则会置于最前端。

