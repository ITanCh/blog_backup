title: Eclipse架构
date: 2017-04-11 17:24:23
tags: [eclipse,架构,记录]
---


> 矫情的话在前一篇文章已经说了，迁移完这篇文章就把github上的这个仓库删了。

> an integrated development environment(IDE) for anything and nothing in particular.  
> eclipse是一个集成开发环境，它既是所有事物的开发环境，又不为某一特定事物的开发环境。     
  
### 最初的愿景  
eclispe 不仅仅局限于一个工具集合，而是一个框架。作为一个框架，它应该具有模块化和可扩展的特点。eclipse 是想作为一个基于组件的平台，一个可以为其他开发者建立自己的工具而服务的平台。    

<!--more-->

---

## Early eclipse  
早期的eclispe是为了将开发者从繁琐的开发过程中解放出来，从而开发者可以更加专注于开发新的工具。   
早期主要有三大元素：`Platform` `JDT(Java Development Tools)` `PDE(Plug-in Development Environment)`  

### 1.Platform   
Platform 是由`plugin`组成的。plugin是Eclipse组件模型的基础，它是由一个JAR文件和manifest描述文件组成。manifest存在`plug-in.xml`文件里。  

#### requires  
plugin通过requires来说明对其他plugin的依赖。  

#### extension vs extension point  
开发者可以通过extension和extension point 机制来对Ecipse进行功能上的扩展。它们之间的关系可以用插座和插头来比喻。详细参考[1](http://wiki.eclipse.org/FAQ_What_are_extensions_and_extension_points%3F) [2](http://www.vogella.com/tutorials/EclipseExtensionPoint/article.html#extensionpoints)。    

![extension](http://7xky03.com1.z0.glb.clouddn.com/extensions.png)  
**extension point**  
插座：如果一个plugin希望其它plugin能够对其进行扩展，则需要定义extension point。extension point说明了对其扩展的extension需要遵守的规约。只有特定的插头才能符合条件。  

种类：说明性的，没有具体代码；提供可被重写的接口；将关联的元素组织在一起，例如`org.eclipse.ui.views`extension point将view组织在一起管理。  
**extension**    
插头：为某个plugin特定的extension point扩展功能。例如`org.eclipse.ui.views`extension point下，定义的一些view。

#### plugin的激活过程
当Eclipse启动的时候，runtime platform会扫描你安装的所有plugin的manifest，在内存里建立plugin注册表，并且该信息会存在硬盘中，下次启动可以直接加载。plugin通过注册被发现，但是直到它们被使用的时候才真的被激活加载，这种方式叫做`lazy activation`。  

#### 早期Eclipse架构  
>everything is a plugin    

![early eclipse architecture](http://7xky03.com1.z0.glb.clouddn.com/platform.png)   

**workbench**  
workbench是组织Eclipse怎么样在桌面上进行显示的UI元素。它由`perspective` `views` `editors`组成。  
Eclipse workbench是建立在`Standard Widget Toolkit(SWT)`和`JFace`之上。  

**SWT**  
Widget toolkits被分为两类：  
1.`native`:使用系统调用来实现UI组件。“像素优先”，和桌面上得其他应用看上去很和谐。但是随着系统的不断变化，widget更新起来很困难，并且造成不一致性，导致程序可移植性也很差。  
2.`emulated`:独立于操作系统。试图模仿操作系统的风格，很大的灵活性。但是运行很慢，和系统风格不协调。  

`Abstract Window Toolkit(AWT)`是Java提供的一个native的WT库,但是错误百出，饱受诟病。然后又出现了`Swing`，一个emulated的WT，同样也是有很多错误，耗费时间和内存。所以这些都不是理想的选择。  

最后选择了SWT，SWT在当时比较成功。SWT是native的，它画质精良，让人不禁惊叹：“*I can't believe it's a Java program.*”。   

**JFace**  
JFace 是建立在SWT上的UI组件，但是它是纯JAVA代码，没有使用系统的代码。  

**Help**  
可以扩展`org.eclipse.help.toc`来写自己的help。  

**Team**  
代码库支持等。   


Eclipse 的一个目的就是鼓励开发人员来扩展eclipse 的功能，实现自己的工具，所以一个稳定的API是十分必要的，所以有句话说的好：“*API is forever*”。  

### 2.Java Development Tools(JDT)  
JDT 提供了Java编辑器，导航，重构，debugger，编译和增量编译等功能。  

其中为了摆脱命令行编译的繁琐，所以eclipse团队写了一个独立的编译器，并且该编译器可以进行增量编译。通过一个队列，将所有有变化的类进行重新编译，然后因为该变化而牵连的类也要放入队列等待重新编译，最后直到队列为空为止。  

### 3.Plug-in Development Environment(PDE)  
PDE提供了开发、建立、配置和测试plugin的工具。eclipse团队自己写了`PDE Build`来创建plugin。   

-----

## Eclipse 3.0:Runtime,RCP and Robots   

### 1.Runtime  
该版本对原来的requires,extensions和extension points组件模型进行了改进，决定用一个现有的方案来替代旧的方法，最后选择了OSGi。  

#### OSGi(Open Service Gateway Initiative)  
为什么选择OSGi?  
1.OSGi拥有语义上的版本控制策略来处理依赖问题。对外的接口也是明确指明的。  
2.拥有自己的类加载器。  
3.可以被广泛的接受。  
4.OSGi社区也是充满活力的。  
5.这个框架实现了一个优雅、完整和动态的组件模型。应用程序（称为bundle）无需重新引导可以被远程安装、启动、升级和卸载（其中Java包／类的管理被详细定义）。  

选择OSGi后Eclipse `plugin`被称为`bundle`。`plugin.xml`仍然保留extensions和extension points这些信息，其他的信息被转移到`META-INF/MANIFEST.MF`。

**依赖问题**  
OSGI框架会通过检测bundle的manifest里的依赖信息来生成该bundle的classpath。同样，manifest中会指明哪些package是对外开放的。  

**生命周期**  
OSGi是一个动态框架，依旧保持原来Eclipse中的`lazy activation`，需要的时候类才会被加载。
![bundle lifecycle](http://7xky03.com1.z0.glb.clouddn.com/bundlelifecycle.png)       


**版本命名**  
一套规范的四部分命名规则。  

**Service**  
提供了进一步的bundle之间的解耦。extension是被启动时就注册，而service是动态注册。使用service的bundle需要import该package，并且由框架决定该service的启用。  

**workbench**  
Eclipse 启动自己的extensin是`org.eclipse.ui.ide.workbench`，是对extension point`org.eclipse.core.runtime.application`的扩展。  

### 2.Rich Client Platform(RCP)  

因为RCP不需要所有的IDE功能，所以对Eclipse进行了重构，将一些bundle分割，方便开发RCP应用。  
![rcp](http://7xky03.com1.z0.glb.clouddn.com/rcp.png)      

------
## Eclipse 3.4  

### Feature  
feature是一组被打包的bundle,它们能够被统一的建立和安装。`update manager`对一个bundle的升级需要升级整个feature。  
`feature.xml`中说明了包含的bundle和一些性质。  

### p2 Concepts 
p2是关于`installation units(IU)`的概念。IU说明了安装组件的名字、id、组件的功能和它的依赖。它可以对安装的组件进行版本控制，并且方便的让组件从一个版本更新到另一个版本。将冲突在安装时发现，而不是在运行时发现。  
![p2](http://7xky03.com1.z0.glb.clouddn.com/p2.png)         

**profile**  
你现有的安装组件的清单。说明了一些运行环境，安装位置等。  
**director**  
执行版本控制。`planner`检测现有的profile，决定更新一个安装组件的必须要执行的操作。`engine`执行把新的组件安装到硬盘上。  
**touchpoint**  
处理运行环境。  

----  
## Eclipse 4.0  
新版本的目的是简化Eclipse模型，吸引新的社区加入，让平台能够利用新的网络技术。  

### Model Workbench  
model的变化会立即在workbench中展现出来。  

### Cascading Style Sheet Sytling  
可以使用`CSS stylesheets`来展现界面。  

### Dependency Injection  
服务编程模型包含：producer，consumer，broker。中间人是负责管理生产者和消费者。  
![context](http://7xky03.com1.z0.glb.clouddn.com/context.png)      
生产者向context中添加service和对象。service向消费者注入context。消费者声明自己的需求。

### Application Services  
为用户提供更加简单独立的API，让服务也可以用于其他语言。  

----  
## Conclusion  
基于组件架构的Eclipse不断吸收新的技术来发展，并且兼顾旧的版本。所以Eclipse社区不断壮大，用户也不断增加。    
