title: AngularJS--SB Admin
date: 2016-01-15 10:56:03
tags: [AngularJS,SB Admin,web,Yeoman,JavaScript]
---
从SB Admin开始学习AngularJS!

[SB Admin v2.0 rewritten in AngularJS](https://github.com/start-angular/sb-admin-angular)是一个SB Admin的AngularJS的实现，源代码、安装和运行可以参考以上链接。

## 一些工具  

[Nodejs](https://nodejs.org)升级：
{% codeblock lang:sh %}
$ sudo npm cache clean -f
$ sudo npm install -g n
$ sudo n stable
$ sudo ln -sf /usr/local/n/versions/node/<VERSION>/bin/node /usr/bin/node
{% endcodeblock %}

npm升级：
{% codeblock lang:sh %}
$ sudo npm install --global npm@latest
{% endcodeblock %}  

[Bower](http://bower.io/)是一个对web开发进行包管理的工具。  

[GRUNT](http://www.gruntjs.net/)的目的就是自动化。对于需要反复重复的任务，例如压缩（minification）、编译、单元测试、linting等，自动化工具可以减轻你的劳动，简化你的工作。当你在 Gruntfile 文件正确配置好了任务，任务运行器就会自动帮你或你的小组完成大部分无聊的工作。  

[Yeoman](http://yeoman.io/)生成web应用的基本架构。
<!--more-->

## Yeoman入门
![yeoman](http://7xky03.com1.z0.glb.clouddn.com/yeoman.png)  
### 1. 安装Bower、GRUNT和Yeoman：  
{% codeblock lang:sh %}
$ sudo npm install --global yo bower grunt-cli
{% endcodeblock %}

### 2. 安装generator
{% codeblock lang:sh %}
$ sudo npm install --global generator-karma generator-angular
{% endcodeblock %}  

### 3. 运行generator
{% codeblock lang:sh %}
$ yo angular
{% endcodeblock %}

### 4. Yeoman总结
Yeoman确实是一个很强大的web应用架构工具，他可以帮你完成大部分繁琐的工作，从而你可以更专心的实现你的功能。

## 源代码分析
### 1. index.html
位于app/目录下的index.html是整个网页的主要入口。  
{% codeblock lang:html %}
    <div ng-app="sbAdminApp">
        <div ui-view></div>
    </div>
{% endcodeblock %}

### 2. app.js
位于app/scripts/目录下，应用的主要module。  

**module**  
首先，我们可以看到sbAdminApp使用了如下module：  
- **oc.lazyLoad**：自动的加载一些module。
- **ui.router**：AngularUI Router是一个路由框架，可以利用它将应用接口组织成为一个有穷状态机，不同于Angular自带的`$route`（根据URL进行路由），UI-Router则是根据状态进行路由。  
- **ui.bootstrap**：用AngularJS开发的Bootstrap组件。  

**config**
**stateProvider**：由*ui.router*提供，功能类似于Angular v1的router，但是它只专注于state。state就是应用中的某一个状态，到达某一个状态时，会将template插入到相应位置。该部分的主要工作是将state、html和controller对应起来。

`resolve`可以提供给controller一些定制的内容，采用`key:value`的格式。在这里，使用ocLazyLoad加载了一些模块。

首先加载的是`sbAdminApp`模块中的header部分，然后依次为header-notification、sidebar和sidebar-search。这些部分在加载的过程中，采用了`module.directive`函数，这个函数可以对指定的元素附加一些行为，如：
{% codeblock lang:js %}
angular.module('sbAdminApp')
.directive('header',function(){
	return {
			templateUrl:'scripts/directives/header/header.html',
			restrict: 'E',
			replace: true,
		}
});
{% endcodeblock %}  
这里就是在`main.html`的`<header></header>`元素替换为`header.html`的内容。  

从源代码中不难发现，header、header-notification、sidebar和sidebar-search文件的加载是按照一定的顺序的，因为前后顺序在View中表现为嵌套的父子关系。

然后，加载了`toggle-switch`，一个开关模块；`ngAnimate`，动画效果；`ngCookies`，读写浏览器cookies；`ngResource`，加载资源对象；`ngSanitize`，可以对HTML进行清洁；`ngTouch`，对可触摸设备提供服务。

其它state类似。

**ocLazyLoadProvider**：来自于*oc.lazyLoad*，`events:true`可以使得加载module时可以发送广播。  

 **urlRouterProvider**：由*ui.router*提供，在这里只是用它来处理其它状态下的情况，统一重定向至`/dashboard/home`。
