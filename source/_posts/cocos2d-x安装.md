title: Cocos2d-x 安装
date: 2017-02-14 13:02:41
tags: [技术,Cocos2d-x]
---

>hello cocos2d 

--------

## 环境
安装环境：Mac OS X
运行目标：Android
Cocos2d版本：cocos2d-x-3.13.1，Cocos Console 2.1

<!--more-->

## Cocos2d-x 安装

### 1. 基本工具
安装Cocos2d-x，最基础的是安装命令行工具。在安装Cocos之前，需要一些其它基本工具：

- JAVA SDK
- Android SDK
- Android NDK
- Apache Ant
- Python 2.7.5

这些基本工具最好已经配置在环境变量中。

### 2. Cocos2d-x 

下载最新的[Cocos2d-x](http://www.cocos2d-x.org/download)，然后解压。找到`setup.py`，然后运行

```sh
python setup.py
```

然后它会提醒你补足一些环境变量的绝对地址，最后在你原有的Home目录下的`.bash_profile`基础上增加一些环境变量，很贴心。

在我的mac上，补足的环境变量如下：
```sh
# Add environment variable COCOS_CONSOLE_ROOT for cocos2d-x
export COCOS_CONSOLE_ROOT=/Users/Tianchi/Tool/cocos2d-x-3.13.1/tools/cocos2d-console/bin
export PATH=$COCOS_CONSOLE_ROOT:$PATH

# Add environment variable COCOS_X_ROOT for cocos2d-x
export COCOS_X_ROOT=/Users/Tianchi/Tool
export PATH=$COCOS_X_ROOT:$PATH

# Add environment variable COCOS_TEMPLATES_ROOT for cocos2d-x
export COCOS_TEMPLATES_ROOT=/Users/Tianchi/Tool/cocos2d-x-3.13.1/templates
export PATH=$COCOS_TEMPLATES_ROOT:$PATH

# Add environment variable NDK_ROOT for cocos2d-x
export NDK_ROOT=/Users/Tianchi/Tool/ndk
export PATH=$NDK_ROOT:$PATH

# Add environment variable ANDROID_SDK_ROOT for cocos2d-x
export ANDROID_SDK_ROOT=/Users/Tianchi/Tool/sdk
export PATH=$ANDROID_SDK_ROOT:$PATH
export PATH=$ANDROID_SDK_ROOT/tools:$ANDROID_SDK_ROOT/platform-tools:$PATH

# Add environment variable ANT_ROOT for cocos2d-x
export ANT_ROOT=/usr/local/apache-ant-1.9.4/bin
export PATH=$ANT_ROOT:$PATH
```

是否安装成功：
```
$ cocos -v
cocos2d-x-3.13.1
Cocos Console 2.1
```

### 3. New Project

创建一个项目：

```sh
cocos new <game name> -p <package identifier> -l <language> -d <location>
```

样例：
```sh
cocos new MyGame -p com.MyCompany.MyGame -l cpp -d ~/MyCompany

cocos new MyGame -p com.MyCompany.MyGame -l lua -d ~/MyCompany

cocos new MyGame -p com.MyCompany.MyGame -l js -d ~/MyCompany
```

编译项目：

```sh
cocos compile -s <path to your project> -p <platform> -m <mode> -o <output directory>
```

样例：

```sh
cocos compile -s ~/MyCompany/MyGame -p ios -m release -o ~/MyCompany/MyGame/bin

cocos compile -s ~/MyCompany/MyGame -p android -m release -o ~/MyCompany/MyGame/bin

cocos compile -s c:\MyCompany\MyGame -p win32 -m release -o c:\MyCompany\MyGame\bin
```

编译Android应用，需要指定目标版本
```sh
cocos compile -p android --ap android-22
```

编译成功后，就会在项目目录下的`proj.android/bin`下发现apk文件，可以进行安装。
