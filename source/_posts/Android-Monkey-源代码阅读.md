title: Android Monkey 源代码阅读
date: 2017-02-14 14:09:32
tags: [Android,技术]
---

## 0. Monkey基本信息
目前该工具位于源代码的位置：`development/cmds/monkey`。

生成的jar包位于：`out/traget/product/generic/system/framework/monkey.jar`。

在设备中，启动monkey的脚本位于`/system/bin/monkey`:
```
# Script to start "monkey" on the device, which has a very rudimentary
# shell.
#
base=/system
export CLASSPATH=$base/framework/monkey.jar
trap "" HUP
exec app_process $base/bin com.android.commands.monkey.Monkey $*
```

<!--more-->

## 1. Monkey开始启动


![这里写图片描述](http://img.blog.csdn.net/20160916145356103)
 1. main函数只是设置了进程的名称，主要过程在run函数中执行。
 2. 获取参数，初始化参数和随机数。然后它会获取系统的一些服务，见1.3。获取需要启动的main activity，见1.4。创建一个MonkeySourceRoandom对象mEventSource，由他管理随机事件的生成，这里首先让它生成了一个启动main activity的事件放入了队列。monkey还支持通过脚本、网络获取事件，这里只说默认的随机情况。下面，它启动Network Monitor。接着，启动Monkey的循环过程，见1.5。等待测试结束后，则输出测试报告。
 3. 获取一些系统服务，有ActivityManager、WindowManager、PackageManager。给ActivityManager设置了一个ActivityController对象，该对象提供了Activity启动、crash获取、应用无响应等功能。最后，还注册了一个网络状态的监听器。
 4. 从PackageManager中获取符合条件的main activity，然后添加入`mMainApps`中。
 5. 这一步就会循环的生成测试事件见1.6、触发事件1.10。
 6. 从队列中获取第一个事件返回。如果队列为空，则生成一个放入队列，见1.7。
 7. 这里先生成一个随机数，根据随机数范围，决定具体生成哪一个事件。这里有点击、拖动、缩放（前三个，见1.8）、轨迹球（已经不常用了，可以忽略）、旋转、权限（见1.9）和键盘事件。
 8. 这里会首先利用DisplayManager获得屏幕的实际大小，然后随机的生成生成点坐标。Touch只需要一点就够了，而Drag则需要生成一系列的点，这里利用了randomWalk函数来随机的移动来生成下一个点坐标，然后将这一系列点坐标封装成事件放入队列。两指缩放同样使用了和Drag类似的方法，只不过是将一个点变为两个点。
 9. 从目标package中的权限列表中获取一个，然后生成一个MonkeyPersissionEvent。
 10. MonkeyEvent是一个抽象类，具体的实现在其子类中。MotionEvent通过InputManager注入事件。旋转MonkeyRotationEvent通过WindowManager进行旋转。MonkeyPermissionEvent利用PackageManager授予、撤销应用的一些权限。MonkeyKeyEvent同样是使用InputManager进行注入。
 11. 这里还有一些其它事件。MonkeyActivityEvent的事件是由ActivityManager启动的。MonkeyCommandEvent则启动一个进程，执行命令即可。MonkeyFlipEvent，唤起键盘的操作，这里是向`/dev/input/event0`写入了一组字节。MonkeyGetAppFrameRateEvent和MonkeyGetFrameRateEvent，获取应用帧频，通过命令行执行来实现。MonkeyInstrumentationEvent, 利用ActivityManager启动Instrumentation组件。MonkeyNoopEvent，什么也不做，呵呵。MonkeyPowerEvent，从log信息中获取电量信息。MonkeyThrottleEvent和MonkeyWaitEvent，Thread.sleep休眠一段时间。
