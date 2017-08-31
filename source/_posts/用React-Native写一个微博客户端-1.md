title: 用React-Native写一个微博客户端(1)
date: 2017-04-06 20:01:14
tags: [Android,技术,React Native]
---

> 用React-Native技术实现一个简单的微博客户端。项目开源地址：[github](https://github.com/ITanCh/Welone)。项目还在持续更新中😊。   

开发环境：macOS

<!--more-->

部分效果：  

![](http://7xky03.com1.z0.glb.clouddn.com/welone_gif_2.gif?imageView2/0/h/300)

![](http://7xky03.com1.z0.glb.clouddn.com/welone_gif_1.gif?imageView2/0/h/300) 

## 安装

如何安装[官网](https://facebook.github.io/react-native/docs/getting-started.html)很清楚，如果网页打不开可以参考[国内文档](http://reactnative.cn/docs/0.40/getting-started.html#content)。

首先安装node和watchman。

{% codeblock lang:shell %}
brew install node 
brew install watchman
{% endcodeblock %}

然后安装react native cli。

{% codeblock lang:shell %}
npm install -g react-native-cli
{% endcodeblock %}

安装Android开发环境就不再赘述。 

创建项目、运行项目。

{% codeblock lang:shell %}
react-native init AwesomeProject 
cd AwesomeProject 
react-native run-android
{% endcodeblock %}

这样可以直接将应用在手机上运行起来，并且可以通过晃动手机（😓）来唤出命令界面，进行动态的更新js实现的功能。但是这样的运行的应用性能并不高，仅仅适用于调试，所以还是需要打包进行发布。

## 打包发布

生成签名密钥。

{% codeblock lang:shell %}
keytool -genkey -v -keystore my-release-key.keystore -alias tianchi -keyalg RSA -keysize 2048 -validity 10000
{% endcodeblock %}  
`-alias`后的参数你可以起个好听的名字，比如“tianchi”。这里设置了10000天的有效期。   

设置gradle变量。多谢这些成熟的构建工具和框架，react-native在创建android项目时已经帮你生成了一个很完善的gradle配置文件，下面需要对其进行修改来满足自己的定制需求。

把生成的`my-release-key.keystore`文件放到`android/app`下面，然后在用户Home下创建并且编辑`~/.gradle/gradle.properties`文件：

{% codeblock lang:shell %}
MYAPP_RELEASE_STORE_FILE=my-release-key.keystore
MYAPP_RELEASE_KEY_ALIAS=tianchi
MYAPP_RELEASE_STORE_PASSWORD=******
MYAPP_RELEASE_KEY_PASSWORD=******
{% endcodeblock %}

添加签名文件到`android/app/build.gradle`：

{% codeblock lang:groovy %}
    //Attention!!! You shuld use your own release key!!!
    signingConfigs {
        release {
            storeFile file(MYAPP_RELEASE_STORE_FILE)
            storePassword MYAPP_RELEASE_STORE_PASSWORD
            keyAlias MYAPP_RELEASE_KEY_ALIAS
            keyPassword MYAPP_RELEASE_KEY_PASSWORD
        }
    }

    buildTypes {
        release {
            ...
            signingConfig signingConfigs.release
        }
    }
{% endcodeblock %}

混淆压缩apk。在`android/app/build.gradle`文件中，修改：

{% codeblock lang:groovy %}
//Run Proguard to shrink the Java bytecode in release builds.
def enableProguardInReleaseBuilds = true
{% endcodeblock %}


打包和安装。  
{% codeblock lang:shell %}
./gradlew assembleRelease
./gradlew installRelease
{% endcodeblock %}


## 引入微博api

进入微[博开发平台](http://open.weibo.com/apps)，申请一个开发者账号，创建一个移动应用，关键是得到一个app key。然后研究weibo提供的android sdk[文档](https://github.com/sinaweibosdk/weibo_android_sdk)。

这里需要特别注意的是，要想成功使用微博的api，你首先要在微博开发平台>应用信息>测试信息中填写你的测试账号，否则无法登陆。同时，将你的开发的app安装到手机上，用微博提供的`app_signatures.apk`获得该应用的Android签名，然后填写到微博开发平台>应用信息>基本信息中的`Android包名：com.welone`和`Android签名:xxxx你获得的签名xxxxxxx`。完成以上两个步骤才能正常使用微博的api，呵呵。  

下面需要通过gradle引入微博的api，官网最近版本已经提供了gradle的引入方式，但是移除了open api的功能。因为我是用的是旧的版本，并且使用了open api，所以引入了一个其他开发者提供的微博api仓库。在`android/app/build.gradle`中修改如下：

{% codeblock lang:groovy %}

repositories {
    jcenter()
    maven { url "https://jitpack.io" }
}

//...

dependencies {
    //...
    compile 'com.github.8tory:weibo-android-sdk:-SNAPSHOT'
}

{% endcodeblock %}

在构建的时候会自动的下载该库进行编译。


## 让你的UI更好看

使用react-native提供的component比较朴素，当然自己可以通过调制创建出漂亮的组建。这里，有[NativeBase](http://nativebase.io/docs/v0.5.13/getting-started#)已经帮我们创建好了，官网提供了很详细的安装教程，这里先安装好，我们将来要使用。其中`react-native-vector-icons`组件需要特殊的配置一下。


> 这里主要介绍了开始编写一个react-native的微博客户端的准备工作，下面一章将会详细介绍。