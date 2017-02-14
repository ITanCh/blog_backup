title: Nexus7搞机教程
date: 2017-02-14 13:28:37
tags: [技术,Android]
---

## 系统准备
这里，我准备了Nexus 7（wifi第二版）作为测试机型，Android版本4.4.2。由于该机器是一个新机器，还没激活，需要在WLAN界面联网激活，否则无法进入系统。我已经尝试过使用代理IP，或者使用翻墙的其它手机作为热点，都以失败告终。所以，在这里我要介绍一种绕过Nexus 7激活的方法。

## 刷入recovery
该过程用于Nexus 7绕过激活和设备root，有个好使的recovery也很必要。

首先保证电脑已经安装[ADB](https://developer.android.com/studio/command-line/adb.html)环境，这里不再赘述安装过程。确保`fastboot`命令可以在终端执行即可。

我在这里推荐TWRP recovery，用起来很顺手。在 [TeamWin - TWRP](https://twrp.me/Devices/) 找到自己需要的recovery，我在这里选择[Asus Nexus 7 2013 Wi-Fi (flo)](https://twrp.me/devices/asusnexus72013wifi.html)，这个一定要和自己移动设备匹配。

下载好`twrp-xxxx.img`文件，关机后，按Power+Volumn down来重启手机，进入fastboot模式。这时，保证设备和电脑通过USB连接，同时使用命令，可以在终端显示设备名称：
```sh
$ adb devices
```
如果手机初始状态为加锁状态，所以首先要解锁：
```
$ fastboot oem unlock 
```
解锁成功后，然后刷入recovery：
```sh
$ fastboot flash recovery twrp.img
```
刷入recovery，成功会显示一些Okey字样。成功刷入以后，关闭设备，使用Power+Volumn up进入recovery模式，等待一小会，如果长时间停留在twrp初始界面，可以点击音量上或下键来观察是否已经进入。如果recovery无法进入，有可能是刷入的recovery和设备版本不一致。

Recovery不仅仅用于Nexus 7绕过激活，一会root手机时，也需要用到。

## 激活Nexus 7
Twrp recovery提供了很人性化的界面，可以使用点击操作。你需要首先利用recovery的挂在功能(mount)，将/system分区挂在上，然后在计算机终端进入设备系统文件：   
```shell
$ adb shell
$ cd /system
```
这时可以看到文件`build.prop`，可以种`cat`或者`vi`来看里面内容，其中重要语句`ro.setupwizard.network_required=true`，这个就是要求网络验证的配置。这里，无论你用何种方法，把`true`改为`false`，保存即可。

重启手机，如果卡在等待界面，则按音量键，看时候可以进入（其实我也不知其中原委，只是这样试了一下，尽量耐心等待一段时间）。进入后会发现进入了正常的初始化步骤，到达WLAN连接激活时，发现下面神奇的多了一个按钮“跳过”！过程中可能会出现一些网络错误什么的，不必关心，只要一步步初始化，就可以最终进入系统。

## Root
手机已经root请跳过此步骤。

可以使用国内的root大师、精灵之流的root软件进行root，但是并不推荐，因为它们会附加安装一些应用。这里我推荐使用[SuperSU](http://forum.xda-developers.com/showthread.php?t=1538053)结合recovery进行root。

首先下载SuperSU [zip](http://download.chainfire.eu/supersu-stable)文件，下载完毕后，将其传入手机某个位置，然后重启进入recovery模式。

在recovery选择install 这个zip文件即可。重启后可以发现应用SuperSU，可以利用它管理手机权限。


## 刷入Android其它版本
Android官方提供了很方便的image资源，其中 [Nexus factory image](https://developers.google.com/android/nexus/images#nakasi)列出了可用的所有Nexus镜像，寻找到适合的镜像进行下载，建议验证一下MD5校验码。下载完毕后，可以按照官网上的教程刷入系统，过程比较简单。

我在这里刷入了Android 4.3 (JSS15Q)版本，降低了Nexus系统的版本。刷机完毕后，可以重复上面步骤，刷入需要的的工具。需要注意的是在绕过激活时，4.3版本和4.4版本略有不同。在4.3版本中，前面步骤相同，最后修改`build.prop`时，只要加入` ro.setupwizard.mode=DISABLED`该配置即可绕过激活。