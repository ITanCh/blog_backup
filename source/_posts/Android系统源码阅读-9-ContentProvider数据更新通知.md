title: Android系统源码阅读(9):ContentProvider数据更新通知
date: 2017-02-14 14:05:43
tags: [Android,技术]
---

## *1.* 用户注册内容观察者
![这里写图片描述](http://img.blog.csdn.net/20160908190740714)
 1. 用户（比如一个Activity）想要实时获得某项内容的变化，需要注册相应的观察者。这个观察者可以自定，但是需要继承`ContentObserver`类，这个类的构造函数需要一个Handle参数，这个Handle就是用来向用户发送数据变化消息的。然后将observer作为参数，利用ContentResolver的这个函数进行注册。
 2. 在1.1中想要注册observer，首先要利用该步骤获取`ContentService`。ContentService是一个系统服务，由ServiceManager管理。这一步获取的实际上市ContentService的Binder代理对象。
 3. 在1.1中注册的observer实际传递的是observer的Transport类型的Binder代理对象，所以这一步要从observer中获得这个transport。
 4. 这一步为进程间函数调用。这一步会将observer加入到mRootNode中，因为ContentService用一棵树来维护所有注册到这里的观测者。用树结构是因为URI天然的树状结构。
 5. 这个函数是一个在树上寻找和添加节点的递归函数。在按照URI的层次结构寻找目标节点时，可能会创造新的节点。直到匹配了最后的节点，将observer封装在ObserverEntry中，添加进该节点。

<!--more-->

## *2*. Content Provider发送更新消息
![这里写图片描述](http://img.blog.csdn.net/20160908190715901)
1. Content Provider如果想要发送更新通知，需要调用ontifyChange函数。这一步以新的URI、和Observer为参数，这里将Observer设置为null。这一步会做一些检测URI、observer是否为空的工作。最后，仍然是获取一个ContentService的进程间通讯代理，然后调用ContentService的notifyChange函数。
2. 在这一步中，会从根节点mRootNode中收集所有的Observer，见2.3。然后依次调用这些Observer的onChange函数，见2.4。
3.  这一步会根据URI指示的路径，依次遍历树上的节点。对于经过的每一个节点，需要将该节点上所有对这个消息感兴趣的observer都封装成ObserverCall添加进入。
4. 这一步也是进程间的函数调用。这里Transport又将消息抛给ContentObserver处理。
5. ContentObserver将消息封装进一个NotificationRunnable函数，然后利用mHandler 将其post给应用程序主线程处理。
6. 这个函数是在主线程中运行。这将要调用它父类的onChange函数。
7. 运行自己重写的ContentObserver的onChange函数，消息发送完毕。