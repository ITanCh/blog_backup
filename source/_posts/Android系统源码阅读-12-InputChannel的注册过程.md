title: Android系统源码阅读(12):InputChannel的注册过程
date: 2017-02-14 14:24:58
tags: [Android,技术]
---

> 请对照AOSP版本：6.0.1_r50。 

InputManager可以获得输入事件并分发，Activity需要处理这些输入事件。那么，这两者之间如何建立的连接呢？这就需要InputChannel作为桥梁建立两者之间的通道。
 
## 1. ViewRootImpl创建InputChannel
这里ViewRoot类已经消失了，由ViewRootImpl替代。Activity在创建时会将自己的DecorView设置给对应的ViewRootImpl。

![这里写图片描述](http://img.blog.csdn.net/20160924164736978)

<!--more-->

### 1.1 
这一步会创建client的InputChannel，并且将当前启动的Activity的窗口传递给WindowManagerService。

*frameworks/base/core/java/android/view/ViewRootImpl.java*
{% codeblock lang:java %}
    /**
     * We have one child
     */
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                //将view设置为传入的DecorView
                mView = view;
                //..

                mAdded = true;
                int res; /* = WindowManagerImpl.ADD_OKAY; */

                //新建InputChannel
                if ((mWindowAttributes.inputFeatures
                        & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                    mInputChannel = new InputChannel();
                }
                try {
                    mOrigWindowType = mWindowAttributes.type;
                    mAttachInfo.mRecomputeGlobalAttributes = true;
                    collectViewAttributes();
                    //mWindowSession是一个Binder代理对象
                    //它引用了运行在WindowManagerService中的一个类型为Session的Binder本地对象
                    //向WindowManagerService添加正在启动的Activity的窗口
                    //这里还会将InputChannel传递过去
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
                } catch (RemoteException e) {
                    //..
                } finally {
                    //..
                }
                //..

                if (view instanceof RootViewSurfaceTaker) {
                    mInputQueueCallback =
                        ((RootViewSurfaceTaker)view).willYouTakeTheInputQueue();
                }
                if (mInputChannel != null) {
                    if (mInputQueueCallback != null) {
                        mInputQueue = new InputQueue();
                        mInputQueueCallback.onInputQueueCreated(mInputQueue);
                    }
                    //将InputChannel和主线程关联起来，在下面会详细讲解
                    mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                            Looper.myLooper());
                }
                view.assignParent(this);
            }
        }
    }
{% endcodeblock %}

### 1.2 

这一步是进程间的请求，从应用进程转到WindowManagerService进程，对应于1.1中的函数addToDisplay。这里会补足一些参数，开始调用下一步函数。

### 1.3

这一步也是调整一些参数，然后交给WindowManagerService来处理。

### 1.4
这里会将传入的Window存入Map来进行统一管理，同时创建了一对server/client端的InputChannel。

*frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java :*
{% codeblock lang:java %}
   public int addWindow(Session session, IWindow client, int seq,
            WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
            Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
            InputChannel outInputChannel) {

            //先对添加的window做一些检查，省略..

            //创建了一个WindowSatete对象
            WindowState win = new WindowState(this, session, client, token,
                    attachedWindow, appOp[0], seq, attrs, viewVisibility, displayContent);

            //..
            
            if (outInputChannel != null && (attrs.inputFeatures
                    & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                String name = win.makeInputChannelName();
                //创建了一个InputChannel对，在1.5中详细讲解
                InputChannel[] inputChannels = InputChannel.openInputChannelPair(name);
                //一个放在WindowState里作为server端的InputChannel
                win.setInputChannel(inputChannels[0]);
                //一个转化为client传递过来的outInputChannel
                inputChannels[1].transferTo(outInputChannel);
                //从上一篇文章中的1.1可知，InputManager作为参数传入
                //WindowManagerService的构造函数，并且存放在mInputManager中
                //下面章节会详细讲述如何注册server端的InputChannel
                mInputManager.registerInputChannel(win.mInputChannel, win.mInputWindowHandle);
            }

            // From now on, no exceptions or errors allowed!
            //将win放入mWindowMap，以client的binder为关键字
            mWindowMap.put(client.asBinder(), win);
          
            //将win加入相应的list，省略..

            mInputMonitor.setUpdateInputWindowsNeededLw();

            boolean focusChanged = false;
            if (win.canReceiveKeys()) {
                focusChanged = updateFocusedWindowLocked(UPDATE_FOCUS_WILL_ASSIGN_LAYERS,
                        false /*updateInputWindows*/);
                if (focusChanged) {
                    imMayMove = false;
                }
            }

            if (imMayMove) {
                moveInputMethodWindowsIfNeededLocked(false);
            }

            assignLayersLocked(displayContent.getWindowList());
            // Don't do layout here, the window must call
            // relayout to be displayed, so we'll do it there.

            if (focusChanged) {
                mInputMonitor.setInputFocusLw(mCurrentFocus, false /*updateInputWindows*/);
            }
            mInputMonitor.updateInputWindowsLw(false /*force*/);

        }

        return res;
    }
{% endcodeblock %}

### 1.5
这一步会将任务交给c++层来处理。

### 1.6

这一步对应的c++的函数为`android_view_InputChannel_nativeOpenInputChannelPair`，它会创建两个InputChannel，并返回。

*frameworks/base/core/jni/android_view_InputChannel.cpp :*
{% codeblock lang:cpp %}
static jobjectArray android_view_InputChannel_nativeOpenInputChannelPair(JNIEnv* env,
        jclass clazz, jstring nameObj) {
    //将java String变为char*
    const char* nameChars = env->GetStringUTFChars(nameObj, NULL);
    String8 name(nameChars);
    env->ReleaseStringUTFChars(nameObj, nameChars);

    sp<InputChannel> serverChannel;
    sp<InputChannel> clientChannel;
    //创建c++层的两个Channel，将在下一步详细讲解
    status_t result = InputChannel::openInputChannelPair(name, serverChannel, clientChannel);
    
    jobjectArray channelPair = env->NewObjectArray(2, gInputChannelClassInfo.clazz, NULL);

    //创建java的server channel
    jobject serverChannelObj = android_view_InputChannel_createInputChannel(env,
            new NativeInputChannel(serverChannel));
    //创建java的client channel
    jobject clientChannelObj = android_view_InputChannel_createInputChannel(env,
            new NativeInputChannel(clientChannel));
    //存入java数组
    env->SetObjectArrayElement(channelPair, 0, serverChannelObj);
    env->SetObjectArrayElement(channelPair, 1, clientChannelObj);
    return channelPair;
}
{% endcodeblock %}

### 1.7
注意，这一步真的要创建ChannelPair了。

*frameworks/native/libs/input/InputTransport.cpp :*
{% codeblock lang:cpp %}
status_t InputChannel::openInputChannelPair(const String8& name,
        sp<InputChannel>& outServerChannel, sp<InputChannel>& outClientChannel) {
    int sockets[2];
    //这个socketpair建立的是一个可以双向通信的管道，创建一对套接字描述符
    if (socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets)) {
        status_t result = -errno;
        ALOGE("channel '%s' ~ Could not create socket pair.  errno=%d",
                name.string(), errno);
        outServerChannel.clear();
        outClientChannel.clear();
        return result;
    }

    //设置管道缓存的大小
    int bufferSize = SOCKET_BUFFER_SIZE;
    setsockopt(sockets[0], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[0], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[1], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[1], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));

    //开始new 两个InputChannel
    String8 serverChannelName = name;
    serverChannelName.append(" (server)");
    outServerChannel = new InputChannel(serverChannelName, sockets[0]);

    String8 clientChannelName = name;
    clientChannelName.append(" (client)");
    outClientChannel = new InputChannel(clientChannelName, sockets[1]);
    return OK;
}
{% endcodeblock %}
socketpair创建了一对无名的套接字描述符（只能在AF_UNIX域中使用），描述符存储于一个二元数组s[2] .这对套接字可以进行双工通信，每一个描述符既可以读也可以写。这个在同一个进程中也可以进行通信，向s[0]中写入，就可以从s[1]中读取（只能从s[1]中读取），也可以在s[1]中写入，然后从s[0]中读取；但是，若没有在0端写入，而从1端读取，则1端的读取操作会阻塞，即使在1端写入，也不能从1读取，仍然阻塞；反之亦然。[该段解释来自](http://liulixiaoyao.blog.51cto.com/1361095/533469/)。

这里new了两个InputChannel，该类的构造函数如下：

*frameworks/native/libs/input/InputTransport.cpp :*
{% codeblock lang:cpp %}
InputChannel::InputChannel(const String8& name, int fd) :
        mName(name), mFd(fd) {
   //..
   int result = fcntl(mFd, F_SETFL, O_NONBLOCK);
   //..
}
{% endcodeblock %}
这里将描述符mFd设置为nonblock。

显然，这里和Android 2.3版本有着很大区别。在旧版本中使用的是匿名共享内存和两个pipe来实现双向的通信。显然，新版本利用的linux系统的新机制，更为简洁高效。

以上步骤实在新Activity建立，窗口开始创建时执行的。这里主要就让WindowManagerService建立一对InputChannel，将Activity的Window和InputDispatcher建立起连接，从而传输输入事件。

## 2. Server端注册InputChannel

在1.4中，创建了一对InputChannel，其中Server端的InputChannel会注册进InputManagerService。

{% codeblock lang:cpp %}
mInputManager.registerInputChannel(win.mInputChannel, win.mInputWindowHandle);
{% endcodeblock %}
![这里写图片描述](http://img.blog.csdn.net/20160924164829942)

### 2.1
这一步就检验了一下传入的InputChannel是否为空。下面一言不合就开始调用native函数。

### 2.2

*frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp :*
{% codeblock lang:cpp %}
    //将指针转化为NativeInputManager
    NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);
    //将java的InputChannel对象转化为一个c++层的InputChannel对象
    sp<InputChannel> inputChannel = android_view_InputChannel_getInputChannel(env,
            inputChannelObj);
    //将InputWindowHandle转化为c++层的InputWindowHandler
    sp<InputWindowHandle> inputWindowHandle =
            android_server_InputWindowHandle_getHandle(env, inputWindowHandleObj);

    //将注册任务交给NativeInputManager
    status_t status = im->registerInputChannel(env, inputChannel, inputWindowHandle, monitor);
    //..
{% endcodeblock %}

### 2.3
这一步通过NativeInputManager中的InputManager，InputManager通过InputDispatcher来注册InputChannel。
*frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp :*
{% codeblock lang:cpp %}
mInputManager->getDispatcher()->registerInputChannel(
            inputChannel, inputWindowHandle, monitor);
{% endcodeblock %}

### 2.4

终于，将Server端的InputChannel交给了InputDispatcher。

*frameworks/native/services/inputflinger/InputDispatcher.cpp :*
{% codeblock lang:cpp %}
        AutoMutex _l(mLock);
        //判断是否该InputChannel已经添加过
        if (getConnectionIndexLocked(inputChannel) >= 0) {
            ALOGW("Attempted to register already registered input channel '%s'",
                    inputChannel->getName().string());
            return BAD_VALUE;
        }
        //创建一个Connection
        sp<Connection> connection = new Connection(inputChannel, inputWindowHandle, monitor);

        //获得InputChannel的文件描述符
        int fd = inputChannel->getFd();
        //将Connection 以文件描述符为关键字，添加入m
        mConnectionsByFd.add(fd, connection);

        //..
        //将文件描述符添加Looper，让Looper监控InputChannel的IO事件
        //当IO事件发生时，就会调用回调函数handleReceiveCallback
        mLooper->addFd(fd, 0, ALOOPER_EVENT_INPUT, handleReceiveCallback, this);
{% endcodeblock %}

到这一步，InputDispatcher已经将InputChannel的Server端管理起来了。当Dispatcher将一个事件通过Channel发送给应用程序窗口以后，会进入休眠状态，直到应用窗口再次通过这个Channel返回一个消息将其激活，Dispatcher才准备发送下一个消息。

这里创建的Connection，Connection中又创建了InputPublisher。InputPublisher可以直接将消息通过这个InputChannel发送出去。

## 3. 向InputManagerService注册当前激活的应用程序窗口

再次回顾一下1.4的，在1.4中WindowManagerService在焦点发生改变时，需要改变Focused Window。这里会在InputMonitor中注册当前激活的窗口。

![这里写图片描述](http://img.blog.csdn.net/20160924164924027)

### 3.1

在InputMonitor中，有mInputFocus保存着当前激活的窗口，这里会将mInputFocus设置为传入的newWindow。然后调用updateInputWindowLW继续更新激活的窗口。

### 3.2

在1.4中，每个window会被加入一个Window List。这里会遍历这些List中的Windows，然后将这些windows进一步交给InputManagerService处理。

*rameworks/base/services/core/java/com/android/server/wm/InputMonitor.java :*
{% codeblock lang:java %}
        // Populate the input window list with information about all of the windows that
        // could potentially receive input.
        // As an optimization, we could try to prune the list of windows but this turns
        // out to be difficult because only the native code knows for sure which window
        // currently has touch focus.
        //..

        // Add all windows on the default display.
        //mService指WindowManagerService
        final int numDisplays = mService.mDisplayContents.size();
        for (int displayNdx = 0; displayNdx < numDisplays; ++displayNdx) {
            WindowList windows = mService.mDisplayContents.valueAt(displayNdx).getWindowList();
            for (int winNdx = windows.size() - 1; winNdx >= 0; --winNdx) {
                final WindowState child = windows.get(winNdx);
                final InputChannel inputChannel = child.mInputChannel;
                final InputWindowHandle inputWindowHandle = child.mInputWindowHandle;
                if (inputChannel == null || inputWindowHandle == null || child.mRemoved) {
                    // Skip this window because it cannot possibly receive input.
                    continue;
                }

                //..
                
                final int flags = child.mAttrs.flags;
                final int privateFlags = child.mAttrs.privateFlags;
                final int type = child.mAttrs.type;
                //只有一个window可以获得Focus
                final boolean hasFocus = (child == mInputFocus);
                final boolean isVisible = child.isVisibleLw();

                //..
                addInputWindowHandleLw(inputWindowHandle, child, flags, type, isVisible, hasFocus,
                        hasWallpaper);
            }
        }

        // Send windows to native code.
        //将这些windows交给InputManager继续进行注册
        mService.mInputManager.setInputWindows(mInputWindowHandles);

        // Clear the list in preparation for the next round.
        clearInputWindowHandlesLw();
{% endcodeblock %}

### 3.3
这一步InputManagerService将任务交给了c++层。

### 3.4
进入c++层，将这些windowHandles交给NativeInputManager。

*frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp :*
{% codeblock lang:cpp %}
static void nativeSetInputWindows(JNIEnv* env, jclass /* clazz */,
        jlong ptr, jobjectArray windowHandleObjArray) {
    //将ptr转化为指针
    NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);
    //windowHandleObjArray是传入的windowHandles
    im->setInputWindows(env, windowHandleObjArray);
}
{% endcodeblock %}

### 3.5
这一步先将java的windowHandle变为c++对象，然后这些windowHandle被交给InputDispatcher处理。

*frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp :*
{% codeblock lang:cpp %}
  if (windowHandleObjArray) {
        jsize length = env->GetArrayLength(windowHandleObjArray);
        //将java对象转化为c++对象
        for (jsize i = 0; i < length; i++) {
            jobject windowHandleObj = env->GetObjectArrayElement(windowHandleObjArray, i);
            if (! windowHandleObj) {
                break; // found null element indicating end of used portion of the array
            }

            sp<InputWindowHandle> windowHandle =
                    android_server_InputWindowHandle_getHandle(env, windowHandleObj);
            if (windowHandle != NULL) {
                windowHandles.push(windowHandle);
            }
            env->DeleteLocalRef(windowHandleObj);
        }
    }

    //将这些windowHandles交给Dispatcher处理
    mInputManager->getDispatcher()->setInputWindows(windowHandles);
{% endcodeblock %}

### 3.6
InputDispatcher跟新WindowHandle，并且更新Focused window。

*frameworks/native/services/inputflinger/InputDispatcher.cpp ：*
{% codeblock lang:cpp %}
    { // acquire lock
        AutoMutex _l(mLock);

        //清理旧的WindowHandle
        Vector<sp<InputWindowHandle> > oldWindowHandles = mWindowHandles;
        mWindowHandles = inputWindowHandles;

        sp<InputWindowHandle> newFocusedWindowHandle;
        bool foundHoveredWindow = false;
        //找到新的Focused WindowHandle
        for (size_t i = 0; i < mWindowHandles.size(); i++) {
            const sp<InputWindowHandle>& windowHandle = mWindowHandles.itemAt(i);
            if (!windowHandle->updateInfo() || windowHandle->getInputChannel() == NULL) {
                mWindowHandles.removeAt(i--);
                continue;
            }
            if (windowHandle->getInfo()->hasFocus) {
                newFocusedWindowHandle = windowHandle;
            }
            if (windowHandle == mLastHoverWindowHandle) {
                foundHoveredWindow = true;
            }
        }

        if (!foundHoveredWindow) {
            mLastHoverWindowHandle = NULL;
        }

        if (mFocusedWindowHandle != newFocusedWindowHandle) {
            //旧的FocusedWindow和新的不同，则需要停止向旧的Channel中发送消息
            if (mFocusedWindowHandle != NULL) {
                sp<InputChannel> focusedInputChannel = mFocusedWindowHandle->getInputChannel();
                if (focusedInputChannel != NULL) {
                    synthesizeCancelationEventsForInputChannelLocked(
                            focusedInputChannel, options);
                }
            }
            //设置新的Focused Window
            mFocusedWindowHandle = newFocusedWindowHandle;
        }

        // Release information for windows that are no longer present.
        // This ensures that unused input channels are released promptly.
        // Otherwise, they might stick around until the window handle is destroyed
        // which might not happen until the next GC.
        for (size_t i = 0; i < oldWindowHandles.size(); i++) {
            const sp<InputWindowHandle>& oldWindowHandle = oldWindowHandles.itemAt(i);
            if (!hasWindowHandleLocked(oldWindowHandle)) {
                //释放已经不存在的旧Window
                oldWindowHandle->releaseInfo();
            }
        }
    } // release lock

    // Wake up poll loop since it may need to make new input dispatching choices.
    mLooper->wake();
{% endcodeblock %}

到这里，InputDispatcher获取了和激活的Window的双向通信通道，同时Dispatcher也知道了是哪个Window处于Focused状态。只要Client端将通信通道建立完毕，则Dispatcher可以向Activity的Window发送消息了。

## 4. Client端注册InputChannel

{% codeblock lang:java %}
 mInputEventReceiver = new WindowInputEventReceiver(mInputChannel, Looper.myLooper());
{% endcodeblock %}
在1.1中，通过WindowSession添加Window和InputChannel后，mInputChannel对象已经转化为双向通道中的client端通道了，从1.4可以知道。然后，该步骤又创建了一个WindowInputEventReceiver类的对象mInputEventReceiver，它将mInputChannel和主线程绑定在一起。

下面就看一下WindowInputEventReceiver的构造过程。

![这里写图片描述](http://img.blog.csdn.net/20160924165000028)

### 4.1 
WindowInputEventReceiver是InputEventReceiver的子类，具体构造过程在InputEventReceiver中。

### 4.2
这一步又开始调用native函数。

*frameworks/base/core/java/android/view/InputEventReceiver.java:*
{% codeblock lang:java %}
  mInputChannel = inputChannel;
  mMessageQueue = looper.getQueue();
  mReceiverPtr = nativeInit(new WeakReference<InputEventReceiver>(this), inputChannel, mMessageQueue);
{% endcodeblock %}

### 4.3
创建c++层的NativeInputEventReceiver。

*frameworks/base/core/jni/android_view_InputEventReceiver.cpp :*
{% codeblock lang:cpp %}
static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak,
        jobject inputChannelObj, jobject messageQueueObj) {
    //将java对象转化为c++对象
    sp<InputChannel> inputChannel = android_view_InputChannel_getInputChannel(env,
            inputChannelObj);
    //传入的java MessageQueue转化为c++的MessageQueue
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);

    //..

    //创建了一个NativeInputEventReceiver，构造函数在下面
    sp<NativeInputEventReceiver> receiver = new NativeInputEventReceiver(env,
            receiverWeak, inputChannel, messageQueue);
    //对其进行初始化
    status_t status = receiver->initialize();

    //返回一个NativeInputEventReceiver的指针
    receiver->incStrong(gInputEventReceiverClassInfo.clazz); // retain a reference for the object
    return reinterpret_cast<jlong>(receiver.get());
}
{% endcodeblock %}
NativeInputEventReceiver的构造函数。

*frameworks/base/core/jni/android_view_InputEventReceiver.cpp :*
{% codeblock lang:cpp %}
NativeInputEventReceiver::NativeInputEventReceiver(JNIEnv* env,
        jobject receiverWeak, const sp<InputChannel>& inputChannel,
        const sp<MessageQueue>& messageQueue) :
        mReceiverWeakGlobal(env->NewGlobalRef(receiverWeak)),
        mInputConsumer(inputChannel), mMessageQueue(messageQueue),
        mBatchedInputEventPending(false), mFdEvents(0) {
    if (kDebugDispatchCycle) {
        ALOGD("channel '%s' ~ Initializing input event receiver.", getInputChannelName());
    }
}
{% endcodeblock %}

### 4.4
这一步调用函数setFdEvents，参数为`ALOOPER_EVENT_INPUT`。

### 4.5
将传入的Channel让主线程监听起来，以便处理传入的消息。

*frameworks/base/core/jni/android_view_InputEventReceiver.cpp*
{% codeblock lang:cpp %}
void NativeInputEventReceiver::setFdEvents(int events) {'
    if (mFdEvents != events) {
        mFdEvents = events;
        //获取Channel的文件描述符
        int fd = mInputConsumer.getChannel()->getFd();
        if (events) {
	    //获取Looper，给其添加监听fd的IO事件的请求
	    //MessageQueue的Looper为主线程Looper，如果Looper进入睡眠
	    //则会被Channel上写入的事件唤醒，从而可以处理新来的消息
            mMessageQueue->getLooper()->addFd(fd, 0, events, this, NULL);
        } else {
            mMessageQueue->getLooper()->removeFd(fd);
        }
    }
}
{% endcodeblock %}
这里监听的文件描述符发生IO事件时，调用的回调函数就是NativeInputEventReceiver自己，因为它为LooperCallback子类。

同时注意这里的Looper就是应用主线程的Looper，这里向其epoll多添加了一个文件描述符进行监听，因为epoll可以同时监听多个epoll的IO事件。同时设定了监听的类型为ALOOPER_EVENT_INPUT。

回忆下第10章的1.6步骤中epoll_wait进入的睡眠后，会监听mWakeEventFd文件描述符的事件。不仅如此，这一步还会监听更多的文件描述符的事件，这里就包括建立的InputChannel的文件描述符。所以在主线程通过调用pollInner进入睡眠以后，被唤醒后会判断发生事件的文件描述符是哪一个。

*system/core/libutils/Looper.cpp :*
{% codeblock lang:cpp %}
  if (fd == mWakeEventFd) {
            if (epollEvents & EPOLLIN) {
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
            }
        } else {
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                pushResponse(events, mRequests.valueAt(requestIndex));
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on fd %d that is "
                        "no longer registered.", epollEvents, fd);
            }
        }
{% endcodeblock %}

到这里，InputChannel在Client端也进行了监听，整个完整的window所在的应用主线程和InputDispatcher线程之间的双向Channel已经建立完毕，同时InputDispatcher知道哪一个window处于激活状态，因为它知道向哪一个window发送消息。
