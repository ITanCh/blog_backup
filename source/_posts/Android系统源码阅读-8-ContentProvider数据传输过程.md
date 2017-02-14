title: Android系统源码阅读(8):ContentProvider数据传输过程
date: 2017-02-14 14:03:37
tags: [Android,技术]
---

> 该系列只记录阅读代码时遇到的问题和心得体会，具体代码讲解可以参考老罗的《Android系统源代码情景分析》，我就不班门弄斧了。我编译的AOSP版本：6.0.1_r50。 

## *1*. 用户开始查询
![这里写图片描述](http://img.blog.csdn.net/20160905214338676)
 1. 用户调用`query`函数进行查询。首先尝试获得provider，这里分为两种，先是unstable，后是stable，暂时没有分清这两者之间的区别。在尝试获取Provider的过程中，将会类似于第7篇文章中的过程。这里假设该用户已经有该provider的记录，获取过程见1.2。在取得provider后，见1.5。
 2. 继续获取Provider。
 3. 继续获取Provider。
 4. ActivityThread将所有已经获得的Provider放在mProviderMap中。这里假设需要获取的Provider存在，返回一个`IContentProvider`类型的provider，这个接口指向了Content Provider的一个Transport代理对象，即ContentProviderProxy对象。
 5. 使用provider的代理进行查询。这里会创建一个`BulkCursorToCursorAdaptor`对象。该版本没有创建CursorWindow对象，即没有创建匿名内存，和旧版本有些区别。那么，新版本中在哪里创建共享的内存区呢？最后，准备通过进程间通信发送消息，等待返回的BulkCursorDescriptor，然后可以获得对应的BulkCursorToCursorAdaptor，即一个AbstractWindowedCursor。

<!--more-->

## *2*. Content Provider处理事务
![这里写图片描述](http://img.blog.csdn.net/20160905214406127)
 1. 该函数负责处理`QUERY_TRANSACTION`事务。首先将传入的数据解析出来，然后利用其子类Transport进行查询，从2.2~2.6，获得一个SQLiteCursor对象。然后用SQLiteCursor封装成CursorToBulkCursorAdapter对象。然后创建BulkCursorDescriptor，从2.7~2.11，目的就是创建window，将数据放入window。然后将BulkCursorDescriptor存入reply，里面包含了SQLiteCursor的binder本地对象，因为SQLiteCursor里又含有window，最后返回给用户后，用户可以获得window里存储的内容。
 2. 这里会检查用户是否有权限访问该数据，如果有权限，则调用自己实现的ContentProvider子类的query函数进行查询。
 3. 该函数需要在子类中重载，实现自己的查询过程。这里，我们假设使用了数据库查询。
 4. 这里会首先根据传入的参数创建一条SQL查询语句，然后在数据库中进行查询。
 5. 这一步会创建一个数据查询driver，用来执行查询。
 6. 创建一个SQLiteCursor对象，这个对象包含了database、driver、query。下面就是将SQLiteCursor返回，一直返回到2.1中。
 7. 该步骤会对创建的BulkCursorDescriptor进行一定的初始化。
 8. 获得用户需要的数据条数。开始时为-1，说明尚未执行查询过程，所以下面要执行查询数据，同时将数据放入内存共享区window。
 9. 因为window还没创建，所以第一件事是先创建共享内存区window，见2.10。然后填充window，见2.11。
 10. 这一步会创建新的window或者清空旧的window以重复利用。这里的window以数据库的路径为名称。创建CursorWindow的过程会调用native函数，见3.1~3.3，所以，新的Android版本放弃了原来在用户中创建window的模式，而是在Content Provider中统一创建、管理window。
 11. 填充window，这里同样会使用sql的一些方法，最后落脚于调用native函数，见4.1~4.3。执行到这里，需要查询的数据已经被存储于window中。


## *3*. Provider创建window的native过程
![这里写图片描述](http://img.blog.csdn.net/20160906191744953)
 1. 在2.10中，会创建CursorWindow对象，所以，SQLiteCursor调用了CursorWindow的构造函数。在该步骤中，会设置window的大小，该大小会在资源文件xml中设置。然后创建window的过程就交给c++代码完成。
 2. `android_database_CursorWindow.cpp`位于*frameworks/base/core/jni/*目录下。这里会创建一个CursorWindow对象window，见3.3。然后用`reinterpret_cast`函数将window指针类型变为jlong返回。
 3. 这里就会创建匿名内存块。这里使用了匿名内存分配的机制，暂不深究。

## *4*. Provider填充window的native过程	
![这里写图片描述](http://img.blog.csdn.net/20160906191803016)
 1. 在2.11中，会对步骤3中创建的window进行填充。这一步会获得一个SQLiteConnection对象，然后通过这个connection进行数据库操作。
 2. 这里开始调用native code执行数据查询，同时填充window。
 3. 这里会通过sqlite3一行一行的将数据copy入window，这里暂不深究sqlite3如何在c++中的使用过程。

	


