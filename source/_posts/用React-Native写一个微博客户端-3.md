title: 用React-Native写一个微博客户端(3)
date: 2017-04-10 21:21:46
tags: [Android,技术,React Native]
---

> 用React-Native技术实现一个简单的微博客户端。项目开源地址：[github](https://github.com/ITanCh/Welone)。项目还在持续更新中😊。 

这一章节主要讲一下在设计该app时遇到的一些问题。

## 导航

Android应用在设计的时候，Activity之间的跳转形成了一个task，该task通过一个栈结构来体现。当用户点击back键，当前的activity会自然的销毁，同时位于栈顶的activity会占据屏幕。这种设计方式很符合用户的操作习惯，所以在通过react-native设计app时，也需要有组件来实现这种功能，所以这里就使用了`Navigator`组件。

<!--more-->

在app中，当用户点击一个微博时，界面会跳进所点击微博的内容，显示微博内容详情，并且可以查看微博评论。当用户点击叉号，会退到浏览微博的页面。

![导航](http://7xky03.com1.z0.glb.clouddn.com/welone_nav.gif?imageView2/0/h/300)

再看一下写在`android/welone/index.android.js`中的导航组件。  
```js
export default class welone extends Component {
  render() {

    let mainComponent = MainTab;
    return (
      <Navigator
        //初始化route的属性
        initialRoute={{ component: mainComponent }}
        //设置界面切换的动画效果
        configureScene={(route) => {
          return Navigator.SceneConfigs.VerticalDownSwipeJump;
        }}
        //根据route属性，渲染当前界面组件
        renderScene={(route, navigator) => {
          let Comp = route.component;
          return <Comp {...route.params} navigator={navigator} />
        }}
      />
    );
  }
}
```   

这里将MainTab设置为当前初始的主界面，并且将`navigator`作为属性传递给MainTab，同时还传递了一个`route.params`属性，这里该属性为空，所以暂时没有什么作用，以后会用到。

通过props，我将navigator对象依次传递到了`WeiboCard`，也就是显示每一个微博内容的卡片。当点击该卡片时，就会触发界面跳转的功能。

```js
    //start the weibo content
    startWeiboContent(data) {
        const { navigator } = this.props;
        if (navigator) {
            //向导航栈中放入需要跳转的组件
            navigator.push({
                component: WeiboContent,
                //将微博内容作为参数传递给了下一个界面，可以参考上一段代码
                params: {
                    data: data
                }
            })
        }
    }
```

这里调用了navigator的push函数，放入了需要占据屏幕的界面`WeiboContent`。这时，MainTab就会被覆盖，显示WeiboContent界面。

在WeiboContent界面，如果需要返回上一个界面MainTab，点击叉号，则会调用如下函数。

```js
    //navigate to the parent
    pressBack() {
        const { navigator } = this.props;
        if (navigator) {
            navigator.pop();
        }
    }
```

navigator只需要pop一下就可以了。

## 数据传递

在React中，父组件向子组件传递数据，主要的方式就是通过设置子组件的props，组件内部的状态通过states来保存。props是外部给组件的属性，states时组件自己的属性。数据流从父节点流向子节点。这种貌似完美的设计方式，在有些情况下会不方便。

比如，子节点需要更新父节点的状态、或者调用父亲节点的功能。则需要将函数通过props将函数传递到子节点。如下代码所示：  

在父节点WeiboContent中，将函数getComments作为属性传递给了子组件WeiboCard。   
```js
 <WeiboCard weiData={this.props.data} getComments={this.getComments.bind(this)} />
```

在WeiboCard中，调用传入的getComments来更新评论内容.上段代码中的bind指明了该函数在当前组件的上下文中执行。

```js
    getComments() {
        if (this.props.getComments) {
            this.props.getComments(NEW_COMMENT);
        }
    }
```

如果父组件想要调用子组件的函数呢？这就用到`ref`属性。

在父节点MainTab中，保留了子组件的引用，如下所示。  
```js
  <WeiboList
  navigator={this.props.navigator}
  ref={(weiboList) => { this.weiboList = weiboList }}
  tab={this.state.activeTab}
  />
```  
将WeiboList组件保存在weibolist变量中，这样父组件就可以调用子节点的函数了。如下：  
```js
    onPressTab(tab) {
      if (this.weiboList != null) {
        this.weiboList.getTimeLine(NEW_WEIBO);
      }
    }
```

显然这种设计方式是不优美的，破坏了组件之间的独立性。以后的章节中，我将会寻找更好的保存组件状态的方法。
