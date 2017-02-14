title: Android系统源码阅读(7):Content Provider的启动
date: 2017-02-14 14:01:17
tags: [Android,技术]
---

>该系列只记录阅读代码时遇到的问题和心得体会，具体代码讲解可以参考老罗的《Android系统源代码情景分析》，我就不班门弄斧了。我编译的AOSP版本：6.0.1_r50。   

## 基本知识

 - Content Provider运行在一个独立的应用程序进程中，它本事就是一个Android应用程序。
 - Binder进程间通信只适合传递体积较小的数据结构，不适合大量数据的传输。所以需要采用匿名共享内存来传输体积较大的数据。
 - Cursor对象通过匿名共享内存来传输数据，而Bundle是通过Binder传递数据。
 
<!--more-->

## *1*. 用户开始调用Provider

首先，Activity集成的ContextWrapper中有成员函数`getContentResolver`，用该函数可以获得一个ContentResolver对象。该ContentResolver对象是在ContextImpl对象建立时创建的一个ApplicationContentResolver对象。

下面从ContentResolver.acquireProvider开始。
![这里写图片描述](http://img.blog.csdn.net/20160902220256938)
 1. 步骤1的主要目的是验证URI是否正确，然后就会将acquireProvider的任务交给子类来实现，这里它将任务传递给ApplicationContentResolver。
 2. 步骤2直接将任务给主线程处理。
 3. 步骤3将会尝试从已有的Provider中获取目标Provider，ActivityThread将已经访问过的Provider放入mProviderMap中，如果其中含有需要获取的Provider，只需要增加对其的引用数目即可。如果没有，则需要请求ActivityManager来获取Provider（见1.4），ActivityManager最后会返回一个ContentProviderHolder（见2.2，见6.1）。最后，需要将获得的ContentProviderHolder进行install(见1.5)。
 4. 这里假设没有现成的Provider，则需要交给ActivityManager进行处理。
 5. 这一步和5.7是一样的函数，区别在于这里的holder不为null，所以不需要自己通过类加载创建Provider，只需要将这个Provider放入mProviderRefCountMap中，对其引用加1。

## *2*. ActivityManager处理请求
![这里写图片描述](http://img.blog.csdn.net/20160902220401299)
 1. 这一步检验调用者caller是否为空，如果为空，则说明没有应用在请求Provider，则无需继续进行。
 2. 这一步所做的处理比较多。首先，Manager首先根据name在mProviderMap中查找是否已经启动。如果没有，则根据Class来查找是否已经启动。如果仍然没有，则构建`ContentProviderRecord`，里面包含了`ProviderInfo`,`ApplicationInfo`等重要信息。然后，如果是*multiprocess*属性为`true`，则将该`ContentProviderRecord`直接返回给调用者，让其实例化。在我们的例子中，我们假设该属性为false，需要创建新的进程来启动该Provider。下面就要开始启动新的进程，会在2.3中详细讲述，然后将Record放入mProvider。最后，ActivityManager进入循环等待provider启动，在发现provider启动后（见6.1），则退出循环，将ContentProviderHolder返回给用户（见1.3）。
 3. 这一步就是进入启动新进程的准备工作。

## *3*. 新进程的启动
![这里写图片描述](http://img.blog.csdn.net/20160902220422830)
这里的启动过程和原来一样。

## *4*. ActivityManager处理新的进程
![这里写图片描述](http://img.blog.csdn.net/20160902220440500)
 1. 获取新进程pid。
 2. 这一步先根据pid获得对应的ProcessRecord，然后填充ProcessRecord的信息。然后，获取需要在该进程中启动的Provider的列表，将在4.3中详解。接下来依次启动该进程中的Content Provider（在4.4中详解）、Activity、Service和Broadcast Receiver。
 3. 这一步通过PackageManager获得所有的该应用进程下需要启动的Provider，然后为其创建ContentProviderRecord，放入mProviderMap。
 4. 准备向新进程发送消息。

## *5*. Content Provider在新进程中启动
![这里写图片描述](http://img.blog.csdn.net/20160902220538532)
 1. 将收到的数据封装成一个AppBindData，准备发送到主线程中。
 2. 发送消息。
 3. 发送消息。
 4. 处理异步消息。
 5. 准备启动所有的Provider。
 6. 这一步将传递过来的每一个`ProviderInfo`通过`installProvider`函数（见5.7）封装成了`ContentProviderHolder`。然后将所有生成的`ContentProviderHolder`告知给ActivityManager（见5.11）。
 7. 这一步根据Provider的类名，使用ClassLoader实例化一个ContentProvider对象，并对其进行一定的初始化（见5.8、5.9）工作。然后，创建对应的ProviderClientRecord，将其放入mLocalProviders和mLocalProvidersByName。
 8. 获取该Provider的类型为Transport的Binder本地对象。这个Binder将会告知给ActivityManagerService，然后有它再告知给需要使用该Provider的其它组件。然后其它进程组件就可以使用这个Binder来使用这个provider了。
 9. 这里会设置该provider的context，同时设置读写权限。最后会调用重写的onCreate函数。
 10. 调用自己实现的onCreate函数。
 11. 将自己的Provider告知给ActivityManager。

## *6*. ActivityManager发布Provider
![这里写图片描述](http://img.blog.csdn.net/20160902220606286)
 1. 这一步会将ActivityManager利用获得的ContentProviderHolder对记录的ContentProviderRecord的provider进行填充，同时将该ContentProviderRecord放入mProviderMap。最后将该Record移除mLaunchingProviders。在2.2中等待的ActivityManagerService线程循环等待发现provider不为null，退出循环，返回ContentProviderHolder给用户（见1.3）。


