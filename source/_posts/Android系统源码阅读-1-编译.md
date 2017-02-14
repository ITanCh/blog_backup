title: Android系统源码阅读(1):编译
date: 2017-02-14 13:31:31
tags: [Android,技术]
---
## 1. 编译过程
参考[官方](https://source.android.com/source/building.html)，注意细节。 

编译版本`6.0.1_r50`。

### 1.1 下载第三方二进制文件
如果想要将编译好的img刷入物理设备，而不是虚拟机，一定要先下载好这些二进制文件，放在AOSP的根目录下。

[二进制文件下载](https://developers.google.com/android/nexus/drivers)。下载完毕后并解压后，会有sh文件生成；运行sh文件，会在AOSP目录下生成`vendor/`文件夹，必要的文件会放在里面。然后再执行编译。

<!--more-->

### 1.2 编译
先清理一下旧的生成文件，个人认为很有必要。
```sh
make clobber
``` 

然后设置环境变量:  
```sh
$ source build/envsetup.sh
# 或者
$ . build/envsetup.sh
```

设置java 环境变量，这里根据你的AOSP选择java版本，设置适当的java路径:
```
$ export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64
$ export PATH=$JAVA_HOME/bin:$PATH
```
设置缓存区域，根据自己情况设置：
```
export USE_CCACHE=1
export CCACHE_DIR=/home/tianchi/Project/.ccache 
prebuilts/misc/linux-x86/ccache/ccache -M 50G
```
选择需要编译的目标：
```
$ launch
#选择你想要编译的版本
```

开始编译，设置编译时用到的内核数目，这里写４：
```
$ make -j4
```
稍等几个小时，呵呵，奇迹就会发生。生成的结果位于`out/target/product/flo/`，`flo`是我Nexus 7的型号，和launch时选择的版本相同。

## 2. 刷入真机
将编译好的img文件刷入设备比较容易，关键点在于前面步骤：下载了第三方包，选择了正确的编译版本。

重启设备进入fastboot模式，使用命令，这里刷入的Nexus7：
```
$ fastboot -p flo flashall 
```

## 3. 启动emulator问题   

想要在虚拟机里跑起来，需要在`launch`步骤中选择正确的虚拟机版本，默认第一个。

相关工具emulator和kernel-qemu默认放置目录已经不在`out/host/linux-xxx/bin`，已经迁移到`prebuilts/android-emulator/linux-x86_64/emulator`和`/prebuilts/qemu-kernel/x86_64/kernel-qemu`等相对应的位置。  

常用的启动命令：   
```sh
prebuilts/android-emulator/linux-x86_64/emulator -kernel ./prebuilts/qemu-k ernel/x86_64/kernel-qemu -sysdir out/target/product/generic/ -system system.img -data userdata.img -ramdisk ramdisk.img
```  
该命令启动会出现各种各样的问题，例如：   
```
qemu: could not load initrd 'ramdisk.img' 
```  
虽然有人已经给出了相关解决方案，比如去掉`-ramdisk ramdisk.img`，或者修改`chmod -R 777 out/target/product/generic/`来增加权限，或者修改`/prebuilts/qemu-k ernel/x86_64/kernel-qemu`为合适的系统版本。这些解决方法有的不太有效，总之不是很优美，其实官方已经提供了一个启动emulator的省心方法：   
```sh
# 首先加入基本的环境变量
$ source build/envsetup.sh

# 选择需要启动的版本，这里因为我编译的1，所以一定要选择1.
$ lunch

You're building on Linux

Lunch menu... pick a combo:
     1. aosp_arm-eng
     2. aosp_arm64-eng
     3. aosp_mips-eng
     4. aosp_mips64-eng
     5. aosp_x86-eng
     6. aosp_x86_64-eng
     7. aosp_deb-userdebug
     8. aosp_flo-userdebug
     9. full_fugu-userdebug
     10. aosp_fugu-userdebug
     11. mini_emulator_arm64-userdebug
     12. m_e_arm-userdebug
     13. mini_emulator_mips-userdebug
     14. mini_emulator_x86_64-userdebug
     15. mini_emulator_x86-userdebug
     16. aosp_flounder-userdebug
     17. aosp_angler-userdebug
     18. aosp_bullhead-userdebug
     19. aosp_hammerhead-userdebug
     20. aosp_hammerhead_fp-userdebug
     21. aosp_shamu-userdebug
 
# 启动emulator
$ emulator
```

## 4. 在Android Studio中阅读源码
Android工程师很地道，考虑到了如何方便的将项目导入AndroidStudio。在编译完成，环境变量配置好的前提下，进行下面步骤。

### 4.1 生成idegen

编译idegen:    
```
$ mmm development/tools/idegen
```

运行idegen:   
```
$ development/tools/idegen/idegen.sh
```
然后，你就回在AOSP目录下发现｀android.ipr｀等文件。

### 4.2 导入
打开AndroidStudio，选择导入已有的项目，选择android.ipr。这里需要导入一段时间。

导入后会发现错误百出。首先打开Project Structure，选择｀SDKs｀添加一个SDK，我这里需要1.7：
![这里写图片描述](http://img.blog.csdn.net/20160930163851944)
清理ClassPath下的所有jar包，然后保存。

选择`Project`，选择Project SDK为新建的SDK，Project language level也设置为对应的等级。
![这里写图片描述](http://img.blog.csdn.net/20160930164516796)

选择`Modules`，在`Dependencies`标签下，删除jar 依赖，最后如下：
![这里写图片描述](http://img.blog.csdn.net/20160930164739735)

然后在`Modules`下，`Sources`标签下，选择`out/target/common/R`文件，选择右键Source。
![这里写图片描述](http://img.blog.csdn.net/20160930165042758)
这里一般情况下，会因为R文件过大，导致依然报错。这时需要修改AndroidStudio应用目录下`Android_Studio/bin/idea.properties`文件，找到filesize，将参数修改大一些。
```
idea.max.intellisense.filesize=5000
```

## Android stack   
这个图应该时时回顾一下。
![这里写图片描述](http://img.blog.csdn.net/20160805212247341)