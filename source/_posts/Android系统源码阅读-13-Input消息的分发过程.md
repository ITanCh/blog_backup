title: Android系统源码阅读(13):Input消息的分发过程
date: 2017-02-14 14:29:02
tags: [Android,技术]
---
# Android系统源码阅读（13）：Input消息的分发过程

> 请对照AOSP版本：6.0.1_r50。学校电脑好渣，看源码时卡半天

---------
先回顾一下前两篇文章。在设备没有事件输入的时候，InputReader和InputDispatcher都处于睡眠状态。当输入事件发生，InputReader首先被激活，然后发送读取消息，激活Dispatcher。Dispatcher被激活以后，将消息发送给当前激活窗口的主线程，然后睡眠等待主线程处理完这个事件。主线程被激活后，会处理相应的消息，处理完毕后反馈给Dispatcher，从而Dispatcher可以继续发送消息。

## 1. InputReader获取事件
回顾一下第11章4.2中，InputReader线程在获取事件以后，会调用`processEventsLocked(mEventBuffer, count);`处理事件。

![这里写图片描述](http://img.blog.csdn.net/20160927222615741)

<!--more-->

### 1.1
这里先根据event的种类进行分门别类的处理。

*frameworks/native/services/inputflinger/InputReader.cpp :*
{% codeblock lang:cpp %}
void InputReader::processEventsLocked(const RawEvent* rawEvents, size_t count) {
    for (const RawEvent* rawEvent = rawEvents; count;) {
        int32_t type = rawEvent->type;
        size_t batchSize = 1;
      
        if (type < EventHubInterface::FIRST_SYNTHETIC_EVENT) {
            //如果这里获得是合成事件
            //这里一次要将该输入设备中的一组事件都获取出来
            int32_t deviceId = rawEvent->deviceId;
            while (batchSize < count) {
                if (rawEvent[batchSize].type >= EventHubInterface::FIRST_SYNTHETIC_EVENT
                        || rawEvent[batchSize].deviceId != deviceId) {
                    break;
                }
                batchSize += 1;
            }
            //处理这些事件
            processEventsForDeviceLocked(deviceId, rawEvent, batchSize);
        } else {
            //这些是一些设备状况的事件，没必要将这些消息发送出去，留给自己处理就可以了
            switch (rawEvent->type) {
            case EventHubInterface::DEVICE_ADDED:
                addDeviceLocked(rawEvent->when, rawEvent->deviceId);
                break;
            case EventHubInterface::DEVICE_REMOVED:
                removeDeviceLocked(rawEvent->when, rawEvent->deviceId);
                break;
            case EventHubInterface::FINISHED_DEVICE_SCAN:
                handleConfigurationChangedLocked(rawEvent->when);
                break;
            default:
                ALOG_ASSERT(false); // can't happen
                break;
            }
        }
        count -= batchSize;
        rawEvent += batchSize;
    }
}
{% endcodeblock %}

### 1.2
准备将事件交给设备进行处理。

*frameworks/native/services/inputflinger/InputReader.cpp :*
{% codeblock lang:cpp %}
void InputReader::processEventsForDeviceLocked(int32_t deviceId,
        const RawEvent* rawEvents, size_t count) {  
    //判断设备是否是已知的
    ssize_t deviceIndex = mDevices.indexOfKey(deviceId);
    if (deviceIndex < 0) {
        ALOGW("Discarding event for unknown deviceId %d.", deviceId);
        return;
    }
    //获得设备
    InputDevice* device = mDevices.valueAt(deviceIndex);
    if (device->isIgnored()) {
        //ALOGD("Discarding event for ignored deviceId %d.", deviceId);
        return;
    }
    //交给设备进行处理
    device->process(rawEvents, count);
}

{% endcodeblock %}

### 1.3

用device中的mapper去映射传入的事件，然后再处理。

*frameworks/native/services/inputflinger/InputReader.cpp :*
{% codeblock lang:cpp %}
void InputDevice::process(const RawEvent* rawEvents, size_t count) {
    // Process all of the events in order for each mapper.
    // We cannot simply ask each mapper to process them in bulk because mappers may
    // have side-effects that must be interleaved.  For example, joystick movement events and
    // gamepad button presses are handled by different mappers but they should be dispatched
    // in the order received.
    //一个设备可能有多种类型的event，所有有多个mapper，但是需要保持event的顺序性
    //所以这里采用先循环event，再循环mapper的方式
    size_t numMappers = mMappers.size();
    for (const RawEvent* rawEvent = rawEvents; count--; rawEvent++) {

        if (mDropUntilNextSync) {
            if (rawEvent->type == EV_SYN && rawEvent->code == SYN_REPORT) {
                mDropUntilNextSync = false;
                //..
            } else {
                //..
            }
        } else if (rawEvent->type == EV_SYN && rawEvent->code == SYN_DROPPED) {
            //..
            mDropUntilNextSync = true;
            reset(rawEvent->when);
        } else {
            for (size_t i = 0; i < numMappers; i++) {
                InputMapper* mapper = mMappers[i];
                //用每一种mapper尝试处理event
                mapper->process(rawEvent);
            }
        }
    }
}
{% endcodeblock %}

### 1.4

这里InputMapper种类很多，每个事件处理方法各不相同，所以在这里不再详述。其中有如下的InputMapper：
> SwitchInputMapper, VibratorInputMapper, KeyboardInputMapper,  CursorInputMapper, TouchInputMapper, SingleTouchInputMapper, MultiTouchInputMapper, JoystickInputMapper

这些方法处理到最后，会调用`getListener()->notifyXXX(&args)`，让Dispatcher进行分发，XXX根据不同的Mapper有相应的名字。在创建InputReader时，将InputDispatcher作为参数传入，同时建立了QueuedInputListener来存放这个InputDispatcher，估计是准备将来处理多个InputDispatcher。所以这里getListener获取的就是当初建立的InputDispatcher对象。

### 1.5
这里虽然已经开始调用InputDispatcher的函数，但是还是在InputReader线程中。这里开始向InputDispatcher的队列中插入事件，并且把InputDispatcher唤醒了。因为notifyXXX函数同样是针对不同的输入有着不同的处理，所以不再详述，截取一段MotionEvent的代码片段。

*frameworks/native/services/inputflinger/InputDispatcher.cpp :*
{% codeblock lang:cpp %}
   //针对每种event，都会想将其封装成一个EventEntry
   // Just enqueue a new motion event.
   MotionEntry* newEntry = new MotionEntry(args->eventTime,
                args->deviceId, args->source, policyFlags,
                args->action, args->actionButton, args->flags,
                args->metaState, args->buttonState,
                args->edgeFlags, args->xPrecision, args->yPrecision, args->downTime,
                args->displayId,
                args->pointerCount, args->pointerProperties, args->pointerCoords, 0, 0);
  //然后加入队列
  needWake = enqueueInboundEventLocked(newEntry);
  
  //...
  //唤醒Looper线程
  if (needWake) {
        mLooper->wake();
    }
{% endcodeblock %}

将一个事件放入队列之后，会根据needWake参数决定是否要唤醒线程。如果要唤醒，则调用Looper的wake函数就可以了，和原来道理一样。在有些时候，有事件添加进去，不一定要唤醒线程，比如线程正在等待应用反馈事件处理完毕的消息。

### 1.6
实实在在的将这个EventEntry放入队列mInboundQueue中了。

*frameworks/native/services/inputflinger/InputDispatcher.cpp :*
{% codeblock lang:cpp %}
bool InputDispatcher::enqueueInboundEventLocked(EventEntry* entry) {
    //如果队列空了，需要唤醒
    bool needWake = mInboundQueue.isEmpty();
    //将事件加入队列
    mInboundQueue.enqueueAtTail(entry);
    traceInboundQueueLengthLocked();

    switch (entry->type) {
    //这里会优化App切换的事件，如果上一个App还有事件没处理完，也没反馈事件处理完毕消息
    //则清空之前的事件，切换下一个应用
    case EventEntry::TYPE_KEY: {
        // Optimize app switch latency.
        // If the application takes too long to catch up then we drop all events preceding
        // the app switch key.
        KeyEntry* keyEntry = static_cast<KeyEntry*>(entry);
        if (isAppSwitchKeyEventLocked(keyEntry)) {
            if (keyEntry->action == AKEY_EVENT_ACTION_DOWN) {
                mAppSwitchSawKeyDown = true;
            } else if (keyEntry->action == AKEY_EVENT_ACTION_UP) {
                if (mAppSwitchSawKeyDown) {
#if DEBUG_APP_SWITCH
                    ALOGD("App switch is pending!");
#endif
                    mAppSwitchDueTime = keyEntry->eventTime + APP_SWITCH_TIMEOUT;
                    mAppSwitchSawKeyDown = false;
                    needWake = true;
                }
            }
        }
        break;
    }

    //当一个非当前激活app的点击事件发生，会清空之前的事件
    //从这个新的点击事件开始
    case EventEntry::TYPE_MOTION: {
        // Optimize case where the current application is unresponsive and the user
        // decides to touch a window in a different application.
        // If the application takes too long to catch up then we drop all events preceding
        // the touch into the other window.
        MotionEntry* motionEntry = static_cast<MotionEntry*>(entry);
        if (motionEntry->action == AMOTION_EVENT_ACTION_DOWN
                && (motionEntry->source & AINPUT_SOURCE_CLASS_POINTER)
                && mInputTargetWaitCause == INPUT_TARGET_WAIT_CAUSE_APPLICATION_NOT_READY
                && mInputTargetWaitApplicationHandle != NULL) {
            int32_t displayId = motionEntry->displayId;
            int32_t x = int32_t(motionEntry->pointerCoords[0].
                    getAxisValue(AMOTION_EVENT_AXIS_X));
            int32_t y = int32_t(motionEntry->pointerCoords[0].
                    getAxisValue(AMOTION_EVENT_AXIS_Y));
            sp<InputWindowHandle> touchedWindowHandle = findTouchedWindowAtLocked(displayId, x, y);
            if (touchedWindowHandle != NULL
                    && touchedWindowHandle->inputApplicationHandle
                            != mInputTargetWaitApplicationHandle) {
                // User touched a different application than the one we are waiting on.
                // Flag the event, and start pruning the input queue.
                mNextUnblockedEvent = motionEntry;
                needWake = true;
            }
        }
        break;
    }
    }
    return needWake;
}
{% endcodeblock %}
这里做了两种优化，主要是在当前App窗口处理事件过慢，同时你又触发其他App的事件时，Dispatcher就会丢弃先前的事件，从这个开始唤醒Dispatcher。这样做很合情合理，用户在使用时，会遇到App由于开发者水平有限导致处理事件过慢情况，这时用户等的不耐烦，则应该让用户轻松的切换到其它App，而不是阻塞在那。所以，事件无法响应只会发生在App内部，而不会影响应用的切换，从而提升用户体验。App的质量问题不会影响系统的运转，Android在这点上做的很人性。

## 2. InputDispatcher分发事件

在第11章中3.2中，Dispatcher调用函数`dispatchOnceInnerLocked(&nextWakeupTime);`来分配队列中的事件。

![这里写图片描述](http://img.blog.csdn.net/20160927223056508)

### 2.1 
从队列中获取event，然后准备处理。

*frameworks/native/services/inputflinger/InputDispatcher.cpp :*
{% codeblock lang:cpp %}
void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
    nsecs_t currentTime = now();

    // Reset the key repeat timer whenever normal dispatch is suspended while the
    // device is in a non-interactive state.  This is to ensure that we abort a key
    // repeat if the device is just coming out of sleep.
    if (!mDispatchEnabled) {
        resetKeyRepeatLocked();
    }

    // If dispatching is frozen, do not process timeouts or try to deliver any new events.
    if (mDispatchFrozen) {
        return;
    }

    //对App切换的情况的优化
    // Optimize latency of app switches.
    // Essentially we start a short timeout when an app switch key (HOME / ENDCALL) has
    // been pressed.  When it expires, we preempt dispatch and drop all other pending events.
    bool isAppSwitchDue = mAppSwitchDueTime <= currentTime;
    if (mAppSwitchDueTime < *nextWakeupTime) {
        //如果有切换App的event，且时间小于设定的时间
        //则将等待事件设为小者
        *nextWakeupTime = mAppSwitchDueTime;
    }

    // Ready to start a new event.
    // If we don't already have a pending event, go grab one.
    if (! mPendingEvent) {
        if (mInboundQueue.isEmpty()) {
            //队列是空的情况
            if (isAppSwitchDue) {
                // The inbound queue is empty so the app switch key we were waiting
                // for will never arrive.  Stop waiting for it.
                resetPendingAppSwitchLocked(false);
                isAppSwitchDue = false;
            }

            // Synthesize a key repeat if appropriate.
            //如果有连续重复事件发生，则制造重复事件
            if (mKeyRepeatState.lastKeyEntry) {
                if (currentTime >= mKeyRepeatState.nextRepeatTime) {
                    mPendingEvent = synthesizeKeyRepeatLocked(currentTime);
                } else {
                    if (mKeyRepeatState.nextRepeatTime < *nextWakeupTime) {
                        *nextWakeupTime = mKeyRepeatState.nextRepeatTime;
                    }
                }
            }

            // Nothing to do if there is no pending event.
            //如果真无事可做，下次睡眠可能比较久
            if (!mPendingEvent) {
                return;
            }
        } else {
            // Inbound queue has at least one entry.
            //从队列中获取一个Event
            mPendingEvent = mInboundQueue.dequeueAtHead();
            traceInboundQueueLengthLocked();
        }

        // Poke user activity for this event.
        if (mPendingEvent->policyFlags & POLICY_FLAG_PASS_TO_USER) {
            pokeUserActivityLocked(mPendingEvent);
        }
        //重置ANRT
        // Get ready to dispatch the event.
        resetANRTimeoutsLocked();
    }

    // Now we have an event to dispatch.
    // All events are eventually dequeued and processed this way, even if we intend to drop them.
    bool done = false;
    DropReason dropReason = DROP_REASON_NOT_DROPPED;
    if (!(mPendingEvent->policyFlags & POLICY_FLAG_PASS_TO_USER)) {
        dropReason = DROP_REASON_POLICY;
    } else if (!mDispatchEnabled) {
        dropReason = DROP_REASON_DISABLED;
    }

    if (mNextUnblockedEvent == mPendingEvent) {
        mNextUnblockedEvent = NULL;
    }

    //分类处理Event
    switch (mPendingEvent->type) {
    case EventEntry::TYPE_CONFIGURATION_CHANGED: {
        ConfigurationChangedEntry* typedEntry =
                static_cast<ConfigurationChangedEntry*>(mPendingEvent);
        done = dispatchConfigurationChangedLocked(currentTime, typedEntry);
        dropReason = DROP_REASON_NOT_DROPPED; // configuration changes are never dropped
        break;
    }

    case EventEntry::TYPE_DEVICE_RESET: {
        DeviceResetEntry* typedEntry =
                static_cast<DeviceResetEntry*>(mPendingEvent);
        done = dispatchDeviceResetLocked(currentTime, typedEntry);
        dropReason = DROP_REASON_NOT_DROPPED; // device resets are never dropped
        break;
    }

    case EventEntry::TYPE_KEY: {
        KeyEntry* typedEntry = static_cast<KeyEntry*>(mPendingEvent);
        if (isAppSwitchDue) {
            if (isAppSwitchKeyEventLocked(typedEntry)) {
                //这个Event就是SwitchEvent，我已经处理
                //重置App切换的状态为false
                resetPendingAppSwitchLocked(true);
                isAppSwitchDue = false;
            } else if (dropReason == DROP_REASON_NOT_DROPPED) {
                //如果是其他事件，则丢弃，因为正在switch
                dropReason = DROP_REASON_APP_SWITCH;
            }
        }
        if (dropReason == DROP_REASON_NOT_DROPPED
                && isStaleEventLocked(currentTime, typedEntry)) {
            dropReason = DROP_REASON_STALE;
        }
        if (dropReason == DROP_REASON_NOT_DROPPED && mNextUnblockedEvent) {
            dropReason = DROP_REASON_BLOCKED;
        }
        //分发KeyEvent
        done = dispatchKeyLocked(currentTime, typedEntry, &dropReason, nextWakeupTime);
        break;
    }

    case EventEntry::TYPE_MOTION: {
        MotionEntry* typedEntry = static_cast<MotionEntry*>(mPendingEvent);
        if (dropReason == DROP_REASON_NOT_DROPPED && isAppSwitchDue) {
            dropReason = DROP_REASON_APP_SWITCH;
        }
        if (dropReason == DROP_REASON_NOT_DROPPED
                && isStaleEventLocked(currentTime, typedEntry)) {
            dropReason = DROP_REASON_STALE;
        }
        if (dropReason == DROP_REASON_NOT_DROPPED && mNextUnblockedEvent) {
            dropReason = DROP_REASON_BLOCKED;
        }
        //分发Motion Event
        done = dispatchMotionLocked(currentTime, typedEntry,
                &dropReason, nextWakeupTime);
        break;
    }

    default:
        ALOG_ASSERT(false);
        break;
    }

    if (done) {
        if (dropReason != DROP_REASON_NOT_DROPPED) {
            dropInboundEventLocked(mPendingEvent, dropReason);
        }
        mLastDropReason = dropReason;
        //将mPendingEvent置为null
        releasePendingEventLocked();
        //我已经处理的event，所以需要将睡眠设置时间小一点
        *nextWakeupTime = LONG_LONG_MIN;  // force next poll to wake up immediately
    }
}
{% endcodeblock %}

### 2.2
这一步根据Event的种类，略有不同。有`dispatchMotionLocked`和`dispatchKeyLocked`等。主要过程类似，首先判断是否需要丢弃该event，然后获得目标Window，再向目标window发送event。代码片如下：

*frameworks/native/services/inputflinger/InputDispatcher.cpp :*
{% codeblock lang:cpp %}
    //..
    // Identify targets.
    Vector<InputTarget> inputTargets;
    //..
    injectionResult = findTouchedWindowTargetsLocked(currentTime, entry, inputTargets, nextWakeupTime, &conflictingPointerActions);
    //..
    dispatchEventLocked(currentTime, entry, inputTargets);
{% endcodeblock %}

这里调用`findTouchedWindowTargetsLocked()`来获取目标Window。在前面12章3.6中，将mFocusedWindowHandle参数设置为了当前激活的Window，所以目前返回的inputTargets就是将mFocusedWindowHandle封装后的结果。

### 2.3
向每一个目标发送event。

*frameworks/native/services/inputflinger/InputDispatcher.cpp :*
{% codeblock lang:cpp %}
void InputDispatcher::dispatchEventLocked(nsecs_t currentTime,
        EventEntry* eventEntry, const Vector<InputTarget>& inputTargets) {

    pokeUserActivityLocked(eventEntry);
    //向每一个目标发送event
    for (size_t i = 0; i < inputTargets.size(); i++) {
        const InputTarget& inputTarget = inputTargets.itemAt(i);
        //获取目标的connection
        ssize_t connectionIndex = getConnectionIndexLocked(inputTarget.inputChannel);
        if (connectionIndex >= 0) {
            sp<Connection> connection = mConnectionsByFd.valueAt(connectionIndex);
            prepareDispatchCycleLocked(currentTime, connection, eventEntry, &inputTarget);
        } else {
          //..
        }
    }
}
{% endcodeblock %}
 
### 2.4

*frameworks/native/services/inputflinger/InputDispatcher.cpp :*
{% codeblock lang:cpp %}
void InputDispatcher::prepareDispatchCycleLocked(nsecs_t currentTime,
        const sp<Connection>& connection, EventEntry* eventEntry, const InputTarget* inputTarget) {

    // Skip this event if the connection status is not normal.
    // We don't want to enqueue additional outbound events if the connection is broken.
    if (connection->status != Connection::STATUS_NORMAL) {
        return;
    }
   
    // Split a motion event if needed.
    if (inputTarget->flags & InputTarget::FLAG_SPLIT) {
        //Motion event一般为连续的基本event组合而成
        //所以可以分割
        MotionEntry* originalMotionEntry = static_cast<MotionEntry*>(eventEntry);
        if (inputTarget->pointerIds.count() != originalMotionEntry->pointerCount) {
            MotionEntry* splitMotionEntry = splitMotionEvent(
                    originalMotionEntry, inputTarget->pointerIds);
            if (!splitMotionEntry) {
                return; // split event was dropped
            }
            enqueueDispatchEntriesLocked(currentTime, connection,
                    splitMotionEntry, inputTarget);
            splitMotionEntry->release();
            return;
        }
    }

    // Not splitting.  Enqueue dispatch entries for the event as is.
    enqueueDispatchEntriesLocked(currentTime, connection, eventEntry, inputTarget);
}
{% endcodeblock %}

### 2.5

这一步先判断目标Connection是否空，然后向其队列加入event。如果原来队列为空，说明可以进一步分发event；如果不为空，说明旧event还没有处理完毕，则不进一步分发。

*frameworks/native/services/inputflinger/InputDispatcher.cpp :*
{% codeblock lang:cpp %}
    bool wasEmpty = connection->outboundQueue.isEmpty();

    // Enqueue dispatch entries for the requested modes.
    //将event放入outboundQueue中，省略..

    // If the outbound queue was previously empty, start the dispatch cycle going.
    if (wasEmpty && !connection->outboundQueue.isEmpty()) {
        startDispatchCycleLocked(currentTime, connection);
    }
{% endcodeblock %}

### 2.6
循环的取出队列中的event，然后交给connection发送。

*frameworks/native/services/inputflinger/InputDispatcher.cpp :*
{% codeblock lang:cpp %}
void InputDispatcher::startDispatchCycleLocked(nsecs_t currentTime,
        const sp<Connection>& connection) {

    //将outboundQueue中的entry一一进行处理
    while (connection->status == Connection::STATUS_NORMAL
            && !connection->outboundQueue.isEmpty()) {
        //获取首部的entry
        DispatchEntry* dispatchEntry = connection->outboundQueue.head;
        dispatchEntry->deliveryTime = currentTime;

        // Publish the event.
        status_t status;
        EventEntry* eventEntry = dispatchEntry->eventEntry;
        switch (eventEntry->type) {
        case EventEntry::TYPE_KEY: {
            KeyEntry* keyEntry = static_cast<KeyEntry*>(eventEntry);
            //..
            
            // Publish the key event.
            status = connection->inputPublisher.publishKeyEvent(dispatchEntry->seq,
                    keyEntry->deviceId, keyEntry->source,
                    dispatchEntry->resolvedAction, dispatchEntry->resolvedFlags,
                    keyEntry->keyCode, keyEntry->scanCode,
                    keyEntry->metaState, keyEntry->repeatCount, keyEntry->downTime,
                    keyEntry->eventTime);
            break;
        }

        case EventEntry::TYPE_MOTION: {
            MotionEntry* motionEntry = static_cast<MotionEntry*>(eventEntry);

            // Set the X and Y offset depending on the input source.
            //..
            
            // Publish the motion event.
            status = connection->inputPublisher.publishMotionEvent(dispatchEntry->seq,
                    motionEntry->deviceId, motionEntry->source,
                    dispatchEntry->resolvedAction, motionEntry->actionButton,
                    dispatchEntry->resolvedFlags, motionEntry->edgeFlags,
                    motionEntry->metaState, motionEntry->buttonState,
                    xOffset, yOffset, motionEntry->xPrecision, motionEntry->yPrecision,
                    motionEntry->downTime, motionEntry->eventTime,
                    motionEntry->pointerCount, motionEntry->pointerProperties,
                    usingCoords);
            break;
        }

        default:
            return;
        }

        // Check the result.
    
        // Re-enqueue the event on the wait queue.
        //从outboundQueue中移除
        connection->outboundQueue.dequeue(dispatchEntry);
        traceOutboundQueueLengthLocked(connection);
        //加入waitQueue
        connection->waitQueue.enqueueAtTail(dispatchEntry);
        traceWaitQueueLengthLocked(connection);
    }
}
{% endcodeblock %}

### 2.7

这一步同样有多种情况，有publishMotionEvent和publishKeyEvent。基本思路相近。先封装成message，最后都调用了`mChannel->sendMessage(&msg)`。
*frameworks/native/libs/input/InputTransport.cpp :*
{% codeblock lang:cpp %}
    InputMessage msg;
    msg.header.type = InputMessage::TYPE_MOTION;
    msg.body.motion.seq = seq;
    msg.body.motion.deviceId = deviceId;
    msg.body.motion.source = source;
    msg.body.motion.action = action;
    msg.body.motion.actionButton = actionButton;
    msg.body.motion.flags = flags;
    msg.body.motion.edgeFlags = edgeFlags;
    msg.body.motion.metaState = metaState;
    msg.body.motion.buttonState = buttonState;
    msg.body.motion.xOffset = xOffset;
    msg.body.motion.yOffset = yOffset;
    msg.body.motion.xPrecision = xPrecision;
    msg.body.motion.yPrecision = yPrecision;
    msg.body.motion.downTime = downTime;
    msg.body.motion.eventTime = eventTime;
    msg.body.motion.pointerCount = pointerCount;
    for (uint32_t i = 0; i < pointerCount; i++) {
        msg.body.motion.pointers[i].properties.copyFrom(pointerProperties[i]);
        msg.body.motion.pointers[i].coords.copyFrom(pointerCoords[i]);
    }
    return mChannel->sendMessage(&msg);
{% endcodeblock %}

### 2.8
这一步就要像保存的文件描述符`mFd`中写入数据了。

*frameworks/native/libs/input/InputTransport.cpp :*
{% codeblock lang:cpp %}
    size_t msgLength = msg->size();
    ssize_t nWrite;
    do {
        nWrite = ::send(mFd, msg, msgLength, MSG_DONTWAIT | MSG_NOSIGNAL);
    } while (nWrite == -1 && errno == EINTR);
{% endcodeblock %}
到这里，Dispatcher终于把event通过当初建立的Channel pair发送给了应用window。这里和旧版本很不同，当初需要先将event放入共享内存，然后发送一个信号进行通知。

## 3. 当前激活Window获得消息
这一节比较复杂，需要回忆大量的前面几章的细节和一定的逻辑推理（连蒙带猜）能力。

先来回忆一下第12章4.5节，在InputChannel注册到Client时，最后一步做了什么。

*frameworks/base/core/jni/android_view_InputEventReceiver.cpp ：*
{% codeblock lang:cpp %}
 mMessageQueue->getLooper()->addFd(fd, 0, events, this, NULL);
{% endcodeblock %}
这里给主线程的Looper添加的一个需要监听fd，这个fd是Client端的InputChannel的文件描述符。`addFd`函数的第4个参数是一个回调函数，这里this指NativeInputEventReceiver对象，它是LooperCallback的子类。`addFd`函数新建了一个Request对象，如下：

*system/core/libutils/Looper.cpp :*
{% codeblock lang:cpp %}
Request request;
request.fd = fd;
request.ident = ident;
request.events = events;
request.seq = mNextRequestSeq++;
//回调函数指明是上一步传入的NativeInputEventReceiver对象
request.callback = callback;
request.data = data;
//..
//将request以fd为关键字加入mRequests
mRequests.add(fd, request);
{% endcodeblock %}

再次回忆第10章1.6，也就是主线程被阻塞的地方。代码如下：
*system/core/libutils/Looper.cpp :*
{% codeblock lang:cpp %}
int Looper::pollInner(int timeoutMillis) {

    // Adjust the timeout based on when the next message is due.
    //..
    //清空mResponses数组
    // Poll.
    int result = POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;

    // We are about to idle.
    mPolling = true;
    //等待epoll监测到IO事件
    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    //..

    // Handle all events.
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeEventFd) {
            if (epollEvents & EPOLLIN) {
                //这里处理的唤醒事件
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
            }
        } else {
	    //这里处理的是input事件
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                //将event加入mResponses
                pushResponse(events, mRequests.valueAt(requestIndex));
            } else {
              //..
            }
        }
    }
Done: ;

    //遍历每一个存在mResponses的event
    // Invoke all response callbacks.
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
            // Invoke the callback.  Note that the file descriptor may be closed by
            // the callback (and potentially even reused) before the function returns so
            // we need to be a little careful when removing the file descriptor afterwards.
            //这一步开始调用回调函数，说明该文件描述符下有人输入的event
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                removeFd(fd, response.request.seq);
            }

            // Clear the callback reference in the response structure promptly because we
            // will not clear the response vector itself until the next poll.
            response.request.callback.clear();
            result = POLL_CALLBACK;
        }
    }
    return result;
}
{% endcodeblock %}
这一步处理wake事件已经在第10章第1节讲述过了，这里需要处理的另一种事件input event。对input event，先将其放入mResponses数组，然后依次调用他们的回调函数。这里就是NativeInputEventReceiver的handleEvent函数了。

![这里写图片描述](http://img.blog.csdn.net/20160930132531316)

### 3.1 
Event分为Input和Output，这里是事件输入，所以先看Input分支。

*frameworks/base/core/jni/android_view_InputEventReceiver.cpp ：*
{% codeblock lang:cpp %}
int NativeInputEventReceiver::handleEvent(int receiveFd, int events, void* data) {

    if (events & ALOOPER_EVENT_INPUT) {
        JNIEnv* env = AndroidRuntime::getJNIEnv();
        status_t status = consumeEvents(env, false /*consumeBatches*/, -1, NULL);
        mMessageQueue->raiseAndClearException(env, "handleReceiveCallback");
        return status == OK || status == NO_MEMORY ? 1 : 0;
    }

    if (events & ALOOPER_EVENT_OUTPUT) {
      //..
    }
    return 1;
}
{% endcodeblock %}

### 3.2
开始从目标Channel中读出event，然后包装成java层的对象，开始调用java函数进行处理。

*frameworks/base/core/jni/android_view_InputEventReceiver.cpp ：*
{% codeblock lang:cpp %}
status_t NativeInputEventReceiver::consumeEvents(JNIEnv* env,
        bool consumeBatches, nsecs_t frameTime, bool* outConsumedBatch) {

    ScopedLocalRef<jobject> receiverObj(env, NULL);
    bool skipCallbacks = false;
    //循环从Channel中读出event
    for (;;) {
        uint32_t seq;
        InputEvent* inputEvent;
        //这一步开始从Channel中读取event，存入inputEvent中
        status_t status = mInputConsumer.consume(&mInputEventFactory,
                consumeBatches, frameTime, &seq, &inputEvent);
        if (status) {
            if (status == WOULD_BLOCK) {
                if (!skipCallbacks && !mBatchedInputEventPending
                        && mInputConsumer.hasPendingBatch()) {
                    // There is a pending batch.  Come back later.
                    //..
                return OK;
                }
            }
            return status;
        }

        if (!skipCallbacks) {
            if (!receiverObj.get()) {
                //这里的mReceiverWeakGlobal是java层的InputEventReceiver
                receiverObj.reset(jniGetReferent(env, mReceiverWeakGlobal));
                //..
            }

            //根据不同的InputEvent种类，将其转化为java层的inputEventObj
            jobject inputEventObj;
            switch (inputEvent->getType()) {
            case AINPUT_EVENT_TYPE_KEY:
                inputEventObj = android_view_KeyEvent_fromNative(env,
                        static_cast<KeyEvent*>(inputEvent));
                break;
            case AINPUT_EVENT_TYPE_MOTION: {
                MotionEvent* motionEvent = static_cast<MotionEvent*>(inputEvent);
                if ((motionEvent->getAction() & AMOTION_EVENT_ACTION_MOVE) && outConsumedBatch) {
                    *outConsumedBatch = true;
                }
                inputEventObj = android_view_MotionEvent_obtainAsCopy(env, motionEvent);
                break;
            }
            default:
                assert(false); // InputConsumer should prevent this from ever happening
                inputEventObj = NULL;
            }

            if (inputEventObj) {
                //开始调用java层的dispatchInputEvent函数
                env->CallVoidMethod(receiverObj.get(),
                        gInputEventReceiverClassInfo.dispatchInputEvent, seq, inputEventObj);
                env->DeleteLocalRef(inputEventObj);
            } else {
              //..
            }
        }

        if (skipCallbacks) {
            //不需要调用回调函数，则可以直接反馈完成信号
            mInputConsumer.sendFinishedSignal(seq, false);
        }
    }
}
{% endcodeblock %}
我们先看如何从Channel中获取event的，3.3。然后讲解java层的分发过程，3.4。

### 3.3
从Chuannel中读取一个（一组）event。

*frameworks/native/libs/input/InputTransport.cpp :*
{% codeblock lang:cpp %}
status_t InputConsumer::consume(InputEventFactoryInterface* factory,
        bool consumeBatches, nsecs_t frameTime, uint32_t* outSeq, InputEvent** outEvent) {

    *outSeq = 0;
    *outEvent = NULL;

    // Fetch the next input message.
    // Loop until an event can be returned or no additional events are received.
    while (!*outEvent) {
        if (mMsgDeferred) {
            // mMsg contains a valid input message from the previous call to consume
            // that has not yet been processed.
            mMsgDeferred = false;
        } else {
            // Receive a fresh message.
            //从Channel中读取一个Message
            status_t result = mChannel->receiveMessage(&mMsg);
            if (result) {
                // Consume the next batched event unless batches are being held for later.
                //..
            }
        }
        //根据message种类构造event
        switch (mMsg.header.type) {
        case InputMessage::TYPE_KEY: {
            KeyEvent* keyEvent = factory->createKeyEvent();
            if (!keyEvent) return NO_MEMORY;
            
            initializeKeyEvent(keyEvent, &mMsg);
            *outSeq = mMsg.body.key.seq;
            *outEvent = keyEvent;
            break;
        }
        case AINPUT_EVENT_TYPE_MOTION: {
            //对Motion event，需要处理成批的event事件
            //..
            MotionEvent* motionEvent = factory->createMotionEvent();
            if (! motionEvent) return NO_MEMORY;

            updateTouchState(&mMsg);
            initializeMotionEvent(motionEvent, &mMsg);
            *outSeq = mMsg.body.motion.seq;
            *outEvent = motionEvent;
            break;
        }
        default:
            return UNKNOWN_ERROR;
        }
    }
    return OK;
}
{% endcodeblock %}

`mChannel->receiveMessage(&mMsg)`函数在InputChannel中主要如下实现，调用socket函数recv从mFd中读出数据。

*frameworks/native/libs/input/InputTransport.cpp :*
{% codeblock lang:cpp %}
nRead = ::recv(mFd, msg, sizeof(InputMessage), MSG_DONTWAIT);
{% endcodeblock %}

### 3.4
回到java层，在获得event后，就需要进行分发了。receiverObj指向的是一个InputEventReceiver对象，这里其实是它的子类WindowInputEventReceiver对象，见第12章第4节。`dispatchInputEvent`函数还是继承的父类的，没有重写。

### 3.5
这一步直接调用了下一步。

### 3.6
将event插入等待事件队列的尾部，然后开始调度这些消息。

*frameworks/base/core/java/android/view/ViewRootImpl.java :*
{% codeblock lang:java %}
  void enqueueInputEvent(InputEvent event,
            InputEventReceiver receiver, int flags, boolean processImmediately) {
        adjustInputEventForCompatibility(event);
        //将event和receiver封装在一起
        QueuedInputEvent q = obtainQueuedInputEvent(event, receiver, flags);

        // Always enqueue the input event in order, regardless of its time stamp.
        // We do this because the application or the IME may inject key events
        // in response to touch events and we want to ensure that the injected keys
        // are processed in the order they were received and we cannot trust that
        // the time stamp of injected events are monotonic.
        //找到尾部插入
        QueuedInputEvent last = mPendingInputEventTail;
        if (last == null) {
            mPendingInputEventHead = q;
            mPendingInputEventTail = q;
        } else {
            last.mNext = q;
            mPendingInputEventTail = q;
        }
        mPendingInputEventCount += 1;

        if (processImmediately) {
            //立即处理所有的InputEvent
            doProcessInputEvents();
        } else {
            //发送一个提醒消息
            scheduleProcessInputEvents();
        }
    }
{% endcodeblock %}
这里会有两种不同的处理event的方式，一个是立即处理；另一种是向主线程Looper发送`MSG_PROCESS_INPUT_EVENTS`消息。这里选择立即处理。

### 3.7
这里会把等待在队列中的event，一口气全处理了。这一步会循环拿出队列中的每一个event，然后调用下一步进行处理。

### 3.8
准备交给stage处理。

*frameworks/base/core/java/android/view/ViewRootImpl.java :*
{% codeblock lang:java %}
    private void deliverInputEvent(QueuedInputEvent q) {
	//...
	//下面开始从某个stage开始
	//寻找适合的stage进行处理
        InputStage stage;
        if (q.shouldSendToSynthesizer()) {
            stage = mSyntheticInputStage;
        } else {
            stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
        }
        if (stage != null) {
            stage.deliver(q);
        } else {
            finishInputEvent(q);
        }
    }
{% endcodeblock %}

### 3.9

下面就讲解一下stage是什么东西。stage可以说是处理event的不同阶段，如果上一个stage处理不了，就交给下一个stage处理，总有一个stage可以将event处理掉。这里stage的处理顺序图如下所示：

![这里写图片描述](http://img.blog.csdn.net/20160927223222813)

 1. NativePrelmeInputStage: Delivers pre-ime input events to a native activity. Does not support pointer events. 其实我也不清楚到底是干什么的，没有想到具体的应用场景。
 2. ViewPreImeInputStage: Delivers pre-ime input events to the view hierarchy. Does not support pointer events. 这一步只处理KeyEvent，在InputMethod处理这个KeyEvent之前，可以截获这个event。典型的例子是处理BACK key。
 3. ImeInputStage: Delivers input events to the ime. Does not support pointer events. 将event交给IputMethodManager处理。
 4. EarlyPostImeInputStage: Performs early processing of post-ime input events. 在交给下一阶段之前，先处理筛选一些event。
 5. NativePostImeInputStage: Delivers post-ime input events to a native activity. 尝试让InputQueue发送event给native activity。
 6. ViewPostImeInputStage: Delivers post-ime input events to the view hierarchy. 将event发送给view的层次结构中。
 7. SyntheticInputStage: Performs synthesis of new input events from unhandled input events. 最后一个阶段，处理综合事件，比如trackball, joystick等。

这里我们暂且研究第6种stage，这个stage和View有直接关系。

### 3.10
这里根据event的类型分别进行处理。我们先关注一下PointerEvent如何处理的。

### 3.11
将event交给mView来处理，这里mView就是一个DecorView对象。

### 3.12
这一步是DecorView继承自View的方法，将event分为TouchEvent和GenericMotionEvent来处理。先看TouchEvent如何处理。

### 3.13
DecorView重写了该函数。这里调用`getCallback`函数来获取一个回调对象。该函数是PhoneWindow继承自Window类的方法，获得是mCallback。那么这个mCallback到底是谁呢？

回顾一下Activity的创建过程，在Activity创建以后会调用attach函数对Activity进行一定的初始化，其中就创建了PhoneWindow，同时设置了callback为Activity它自己。所以这一步获得是一个Activity对象，然后调用它的函数继续分发event。

### 3.14

Event交到Activity手中进行处理。为什么会先交给Activity处理？目的是让开发者可以重写这个函数，从而可以在分发这个事件之前进行截获。

{% codeblock lang:java %}
    /**
     * Called to process touch screen events.  You can override this to
     * intercept all touch screen events before they are dispatched to the
     * window.  Be sure to call this implementation for touch screen events
     * that should be handled normally.
     *
     * @param ev The touch screen event.
     *
     * @return boolean Return true if this event was consumed.
     */
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        //绕一圈有交给PhoneWindow处理
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        //没有view可以处理，那么activity自己处理
        //默认就是放弃，也可以重写实现特定功能
        return onTouchEvent(ev);
    }
{% endcodeblock %}

### 3.15
什么也没做，将event传回DecorView，让它处理。

###3.16
这里DecorView又交给dispatchTouchEvent处理。这里的dispatchTouchEvent是源自父类ViewGroup的函数，而不是自己重写的函数。

###3.17
ViewGroup是View的子类。它管理了一组View在mChildren数组中，按照设计模式的说法叫Composite模式。

![这里写图片描述](http://img.blog.csdn.net/20160927225350157)

这一步会依次访问每个子view，判断他们是否可以处理该event，如果能就交给它处理；没人能处理就自己处理。无论哪种方式，都会调用下一步`dispatchTransformedTouchEvent`函数。

*frameworks/base/core/java/android/view/ViewGroup.java :*
{% codeblock lang:java %}
    public boolean dispatchTouchEvent(MotionEvent ev) {
        //..
        // If the event targets the accessibility focused view and this is it, start
        // normal event dispatch. Maybe a descendant is what will handle the click.
        if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
            ev.setTargetAccessibilityFocus(false);
        }

        boolean handled = false;
        //如果Window没有被遮住，才进行以下过程
        if (onFilterTouchEventForSecurity(ev)) {
            //..
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) {
                    //..
                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        //根据z坐标进行排序，从小到大的顺序
                        final ArrayList<View> preorderedList = buildOrderedChildList();
                        //也可以自己定制顺序
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        //所有子View存放在mChildren中
                        final View[] children = mChildren;
                        //按z的值，从大到小开始遍历
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = customOrder
                                    ? getChildDrawingOrder(childrenCount, i) : i;
                            final View child = (preorderedList == null)
                                    ? children[childIndex] : preorderedList.get(childIndex);

                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            //如果指定了一个view去获得这个event，一直循环到那个view为止
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }

                            //判断该view是否可以接受这个event，是否在这个view范围内
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            //如果该child正在处理上一个event
                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }
                            //尝试去向该child发送event
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                //..
                                //将child加入mFirstTouchTarget为首的队列头部
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }
                            //..
                        }
                    }
                }
            }

            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
	        //没有child可以处理，那么就自己处理
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        //..
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        //..
                    }
                    predecessor = target;
                    target = next;
                }
            }
            //..
        }
        return handled;
    }
{% endcodeblock %}

### 3.18

ViewGroup会决定自己处理还是交给child处理。在ViewGroup自己处理，或者child为不再是一个ViewGroup时，则开始调用View的dispatchTouchEvent函数。

*frameworks/base/core/java/android/view/ViewGroup.java :*
{% codeblock lang:java %}
//省略了计算坐标便宜的过程
if (child == null) {
	//ViewGroup决定自己处理
	handled = super.dispatchTouchEvent(event);
} else {
	//交给child处理，child可能是个view
	//也可能还是个ViewGroup，这就重复3.17步骤
        handled = child.dispatchTouchEvent(event);
}
{% endcodeblock %}

### 3.19
这一步就会调用用户自己实现的Listener；如果没有Listener，则会调用view默认的处理函数。

*frameworks/base/core/java/android/view/View.java :*
{% codeblock lang:java %}
  public boolean dispatchTouchEvent(MotionEvent event) {
        // If the event should be handled by accessibility focus first.
        if (event.isTargetAccessibilityFocus()) {
            // We don't have focus or no virtual descendant has it, do not handle the event.
            if (!isAccessibilityFocusedViewOrHost()) {
                //该View不可访问的状态，返回
                return false;
            }
        }

        boolean result = false;
        //..
        //先判断该View是否被遮挡，没被遮挡才进行处理
        if (onFilterTouchEventForSecurity(event)) {
            //noinspection SimplifiableIfStatement
            //获得注册的Listener，看是否有注册OnTouchListener
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                      //调用Listener的回调函数，处理event
                result = true;
            }
            //View也可以采用默认的处理onTouchEvent
            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }
        //..
        return result;
    }
{% endcodeblock %}
默认处理函数onTouchEvent会处理一些基本的操作，比如button的按下松开的效果，滚动容器产生滚动的效果等。

### 3.20
在InputStage完成处理event的任务后，会开始返回完成的消息。

### 3.21
开始调用c++函数。

### 3.22
将指针转化为NativeInputEventReceiver指针，交给它来处理。

*frameworks/base/core/jni/android_view_InputEventReceiver.cpp ：*
{% codeblock lang:cpp %}
sp<NativeInputEventReceiver> receiver = reinterpret_cast<NativeInputEventReceiver*>(receiverPtr);

status_t status = receiver->finishInputEvent(seq, handled);
{% endcodeblock %}
### 3.23
InputConsumer依次处理每个sequence，发送完成信号。

### 3.24
向channel发送完成消息。

{% codeblock lang:cpp %}
status_t InputConsumer::sendUnchainedFinishedSignal(uint32_t seq, bool handled) {
    InputMessage msg;
    msg.header.type = InputMessage::TYPE_FINISHED;
    msg.body.finished.seq = seq;
    msg.body.finished.handled = handled;
    return mChannel->sendMessage(&msg);
}
{% endcodeblock %}
向Channel写入事件后，处于睡眠的InputDispatcher被唤醒，开始分发下一个event。


## 最后说两句

到这里，Input event的处理流程已经分析完了，心好累。
















