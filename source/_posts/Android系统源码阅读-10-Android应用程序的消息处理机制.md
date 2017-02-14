title: Android系统源码阅读(10):Android应用程序的消息处理机制
date: 2017-02-14 14:07:55
tags: [Android,技术]
---

## 基础知识

>原来写好的博客被CSDN给坑了，法克，只能靠回忆重写。    

1. Android应用程序的四种组件皆运行于ActivityThread之中。ActivityThread包含有程序入口main，同时它会启动一个循环，这个循环会轮询消息队列，来处理发送给它的消息。而四种组件则被这个线程统一管理。所以，ActivityThread这个线程是一个动态的过程，像一个无休止的转动的法条，而四个组件则像是被驱动四个齿轮，需要它们转动时才会在法条的带动下进行转动。
2. 主线程中有一个Looper，Looper是一个循环，其中又有一个MessageQueue来管理消息队列。MessageQueue又有一个mPtr变量，记录了c++层对应的NativeMessageQueue的位置。NativeMessageQueue也有一个c++层的Looper，它负责管理pip，而pip则是进程间通信的通道。 

<!--more-->

## 1. 线程消息循环
![这里写图片描述](http://img.blog.csdn.net/20160916145806480)

 1. ActivityThread会由此进入循环。通过一个无限循环，从MessageQueue里获取Message，这里可能会被阻塞，见1.2。获取消息后，分发消息到响应的的组件。
 2. 调用nativePollOnce函数来判断是否有新的消息，见1.3。如果有新消息，则判断时机是否已到，时机到了则将Message返回。如果时机未到，或者还没有消息，则先执行一些`IdleHandler`。
 3. 这一步将ptr翻译成一个NativeMessageQueue的对象指针，然后将任务交给它来处理。目前该c++文件位于`frameworks/base/core/jni`目录下。
 4. NativeMessage又将任务交给c++层的Looper来处理。
 5. Looper.cpp文件位于`system/core/libutils`。这里同样一个循环，不断调用pollInner来判断时候有新消息。
 6. Looper中有一个epoll实例，epoll是用来监听IO读写的一种机制，这里来监听pip是否有读写事件。这里使用epoll_wait来等待epoll中的读写事件，如果没有则会根据设定睡眠一段时间。如果有事件，则对比该事件描述符是否为mWakeEventFd文件描述符，如果是，则调用awoken。
 7. 尝试从mWakeEventFd文件描述符中读出数据，里面的数据并没有什么实际意义。

## 2. 线程消息发送过程

Handler用来向一个消息队列发送消息。它内部有mLooper和mQueue，在构造Handler时，会将它所在线程中的Looper获取，放入mLooper。
![这里写图片描述](http://img.blog.csdn.net/20160918112806764)
 1. 发送一个消息。
 2. 加入延迟的发送消息。
 3. 在某个时间点发送消息，这里会将这个消息和发送时刻放入队列。
 4. 将message的target设为该Handler，然后将message放入mQueue。
 5. 消息队列是根据时间进行排序，所以这一步会根据该消息的时间，将消息插入到适合的位置。如果队列头发生变化，就要进入native世界，唤醒等待的人。
 6. 这一函数对应c++中的`android_os_MessageQueue_nativeWake`。这里会将传入的ptr转化为一个NativeMessageQueue的指针。
 7. 调用mLooper的wake函数。
 8. 这一步会向mWakeEventFd描述写入一个数据"1"，并没有什么实际意义，就是想触发一个IO事件，然后让等待这个IO事件的epoll_wait触发。


## 3. 线程消息处理
![这里写图片描述](http://img.blog.csdn.net/20160918165035114)
1. 再次回到looper函数，在第1节中，线程会阻塞在该函数中，直到获取了一个消息。如果获得的消息为null，则退出循环。否则，获取消息的target，从2.4可知target为一个Handler，下面看这个Handler如何派发消息。
2. 这里要处理三种情况：第一种，message提供了自己的回调函数，见3.3；第二种，使用Handler提供回调函数mCallback，见3.4；第三种，由Handler继续处理见3.5。
3. 这里message的callback参数指向的是一个Runnable的对象，所以这一步直接启动这个对象的run函数即可。
4. CallBack接口中只有handleMessage函数，这里需要在定义Handler时，设定一个实现CallBack接口的对象。
5. 同样，这是一个需要子类来具体实现的函数。一般我们在定义一个Handler时，定义的是一个Handler子类，在handleMessage函数中实现自己的功能。

## 4. 接着说两句

在1.2中涉及到的IdleHandler，在没有消息需要处理时，会调返回这个空闲Handler。那么，我们来看一下这个Handler如何工作的。

在MessageQueue中，有mIdleHandlers来存放IdleHandler。在next函数中，如果没有获得一个消息，则会开始处理idler。这里会判断是否有idler在mIdleHandlers中，没有或者原来已经发送过一个了，则无需发送，所以一个线程的一次next调用，最多只会发送一次idler。

如果有idler，且为第一次发送，则开始处理这些idler。这里会调用IdleHandler的queueIdler函数。同样，这里需要自己实现IdlerHandler接口，来处理一些不紧急的事情。