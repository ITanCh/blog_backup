title: 用React-Native写一个微博客户端(2)
date: 2017-04-08 16:47:49
tags: [Android,技术,React Native]
---

> 用React-Native技术实现一个简单的微博客户端。项目开源地址：[github](https://github.com/ITanCh/Welone)。项目还在持续更新中😊。 

## 主界面设计

进入应用，首先看到的就是主界面，这里采用了经典的底部3个tab的主界面设计风格，如下所示。

<!--more-->

![3tab](http://7xky03.com1.z0.glb.clouddn.com/welone_3tab.gif?imageView2/0/h/300)

react-native 框架已经为我们生成了一个简单的主界面于`index.android.js`，代码如下：

```js
export default class welone extends Component {
  render() {
    return (
      <View style={styles.container}>
        <Text style={styles.welcome}>
          Welcome to React Native!
        </Text>
        <Text style={styles.instructions}>
          To get started, edit index.ios.js
        </Text>
        <Text style={styles.instructions}>
          Press Cmd+R to reload,{'\n'}
          Cmd+D or shake for dev menu
        </Text>
      </View>
    );
  }
}
//...

AppRegistry.registerComponent('welone', () => welone);

```


这里我们所要做的就是将render()函数中的内容替换成我们的componet。这里我新建了一个MainTab组建，并且将它放置于`./app/comp/MainTab.android.js`中，android后缀表明编译android应用时选择该js文件作为编译对象，这样你也可以实现MainTab.ios.js，从而可以针对不同的系统采用不同的设计内容。这里我新建了一个app目录用来存放组件相关的代码和资源，我的文件组织目录如下图所示。

![文件结构](http://7xky03.com1.z0.glb.clouddn.com/welone_file.png?imageView2/0/h/200)

修改后的`index.android.js`如下：

```js
//...

import MainTab from './app/comp/MainTab';

export default class welone extends Component {
  render() {
    //主界面的MainTab组件
    let mainComponent = MainTab;
    return (
      //这里是用了Navigator组件，目的是为了以后的导航栏作用
      <Navigator
        initialRoute={{ component: mainComponent }}
        configureScene={(route) => {
          return Navigator.SceneConfigs.VerticalDownSwipeJump;
        }}
        renderScene={(route, navigator) => {
          //这里的组建就是MainTab
          let Comp = route.component;
          //MainTab
          return <Comp {...route.params} navigator={navigator} />
        }}
      />
    );
  }
}

//...
```

先忽略Navigator组件在这里的作用，这里可以先理解为以MainTab作为当前的视图的根view：

```js
render() {
    return <MainTab />
}
```

在`app/comp/MainTab.android.js`里，我采用了同样经典的三段式配合3个tab来使用。

```js
//...
export default class MainTab extends Component {

    //...
    render() {
        return (
            <Container theme={TianTheme}>
                //头部标题会跟着tab的切换变化
                <Header>
                 //标题内容
                </Header>

                //主体，显示内容
                <Content style={{ backgroundColor: '#f8f9f9' }}>
                   //...
                </Content>
                
                //底部，显示3个tab
                <Footer style={{ borderTopWidth: 1, borderTopColor: '#e5e8e8' }}>
                    <FooterTab>
                        <Button onPress={() => this.onPressTab(TAB_LOVE)}>
                            <Icon name='ios-heart' />
                        </Button>
                        <Button onPress={() => this.onPressTab(TAB_ME)}>
                            <Icon name='ios-chatbubbles' />
                        </Button>
                        <Button onPress={() => this.onPressTab(TAB_SET)}>
                            <Icon name='ios-settings' />
                        </Button>
                    </FooterTab>
                </Footer>
            </Container>
        );
    }
}
```

其中Container、Header、Content、Footer等组件皆来自于Native-Base。我的建议是能用原始的react-native的组建就用原始的，Native-Base提供的组件虽然美观，但是缺少了定制的灵活性。

当点击tab时，调用回调函数`onPressTab()`触发view的变化。根据传入的参数确定用户点击的哪一个tab，然后修改当前的状态。注意，这里涉及修改当前组件状态的操作`this.setState({ activeTab: tab })`，所以回调函数采用`<Button onPress={() => this.onPressTab(TAB_SET)}>`，这里是用箭头函数指明了该函数运行的上下文位于该组件中。在不涉及修改state时并无大碍，如果涉及修改state，一定要采用异步操作。记住千万不要在`render`函数中直接修改state。

```js
    onPressTab(tab) {
        //...
        this.setState({ activeTab: tab })
    }
```

## 内容显示

在MainTab组件中，显示内容的部分需要根据用户的选择依次显示“关注”、“消息”和“设置”三个板块，为此我设计了一个`WeiboList`组件来显示具体内容，并且作为Content的子组件镶嵌进MainTab。

```js
                <Content style={{ backgroundColor: '#f8f9f9' }}>
                    <View style={{ marginHorizontal: 7 }}>
                        <WeiboList
                            navigator={this.props.navigator}
                            //保留weibolist组件的引用
                            ref={(weiboList) => { this.weiboList = weiboList }}
                            //指明当前用户选择的哪一个tab
                            tab={this.state.activeTab}
                        />
                    </View>
                </Content>
```


这里为WeiboList组件添加了属性`tab`，描述当前用户选择的tab类型。同时，为MainTab保留了一个weibolist的应用，为的是从MainTab组件中调用WeiboList的函数，当然这种设计方式不算优美，将来我会用其它方式进行改进。

在WeiboList组件中，实现render()函数如下：

```js
 render() {
        if (this.props.tab === TAB_LOVE) {
            //关注人的微博列表
            return (
                <ListView
                    //设置数据源
                    dataSource={this.state.loveSource}
                    //列表中的子组件
                    renderRow={(rowData) => <WeiboCard weiData={rowData} navigator={this.props.navigator} />}
                    //列表底部组件
                    renderFooter={() => this.getEnd(GET_MORE_WEIBO)}
                />
            );
        } else if (this.props.tab === TAB_ME) {
            //用户消息
            //...
        } else {
            //用户设置
            //...
        }
    }
```

根据用户选择的板块，返回不同的组件内容。这里主要介绍一下微博内容列表的实现，关于微博登录认证的模块，可以参考源代码。

为了显示列表结构的微博内容，使用了react-native自带的`ListView`组件，ListView可以滚动的现实View子组件列表。它需要设置两个重要属性，`dataSource`表示数据数组，`renderRow`则会将数组中的每一项取出来，渲染一个子组件。这里我设计了`WeiboCard`来显示微博的具体内容。

dataSourece的数据源来自`this.state.loveSource`，该状态在WeiboList中进行初始化。

```js
    constructor(props) {
        super(props);

        //创建一个DataSource，并且设置判断当前视图是否发生滚动的判断函数，以此来减少绘制列表的消耗。
        let ds1 = new ListView.DataSource({ rowHasChanged: (r1, r2) => r1 !== r2 });

        this.state = {
            loveSource: ds1,
        };
        //用来存储当前获取的微博内容数组
        this.loveData = [];
    }
```

到此为止，视图已经设计完毕，但是还没有数据。接下来需要完成获取数据的部分。


## 数据获取

这里采用的微博api的Android版本，是一个java库，幸运的是react-native提供了接口让我们开发原生模块在js中调用。当然，也可以研究一下是否可以直接调用微博的js库。

先看一下微博api中提供了什么样的接口供开发者调用。在`com/sina/weibo/sdk/openapi/StatusesAPI.java`中，提供了一些与获取微博内容有关的函数，并且提供了详细的文档说明，这里需要使用的就是如下接口。

```java
    /**
     * 获取当前登录用户及其所关注用户的最新微博。
     * 
     * @param since_id    若指定此参数，则返回ID比since_id大的微博（即比since_id时间晚的微博），默认为0
     * @param max_id      若指定此参数，则返回ID小于或等于max_id的微博，默认为0。
     * @param count       单页返回的记录条数，默认为50。
     * @param page        返回结果的页码，默认为1。
     * @param base_app    是否只获取当前应用的数据。false为否（所有数据），true为是（仅当前应用），默认为false。
     * @param featureType 过滤类型ID，0：全部、1：原创、2：图片、3：视频、4：音乐，默认为0。
     *                    <li>{@link #FEATURE_ALL}
     *                    <li>{@link #FEATURE_ORIGINAL}
     *                    <li>{@link #FEATURE_PICTURE}
     *                    <li>{@link #FEATURE_VIDEO}
     *                    <li>{@link #FEATURE_MUSICE}
     * @param trim_user   返回值中user字段开关，false：返回完整user字段、true：user字段仅返回user_id，默认为false。
     * @param listener    异步请求回调接口
     */
    public void friendsTimeline(long since_id, long max_id, int count, int page, boolean base_app,
            int featureType, boolean trim_user, RequestListener listener) {
        WeiboParameters params = 
                buildTimeLineParamsBase(since_id, max_id, count, page, base_app, trim_user, featureType);
        requestAsync(sAPIList.get(READ_API_FRIENDS_TIMELINE), params, HTTPMETHOD_GET, listener);
    }  
```

在Android项目文件夹下添加了微博模块：`android/app/src/main/java/com/welone/weibo/WeiboModule.java`。实现了如下函数：

```java
   /**
     * get the status of your friends
     *
     * @param sinceId
     * @param maxId
     * @param successCallback
     * @param errorCallback
     */
    @ReactMethod
    public void getTimeline(String sinceId, String maxId, Callback successCallback, Callback errorCallback) {
        //获取认证token
        Oauth2AccessToken token = AccessTokenKeeper.getAccessToken();
        if (token != null && token.isSessionValid()) {
            if (mStatusesAPI == null) {
                mStatusesAPI = new StatusesAPI(getCurrentActivity(), Constants.APP_KEY, token);
            }
            try {
                long since = Long.parseLong(sinceId);
                long max = Long.parseLong(maxId);
                //调用了friendsTimeline的同步版本，因为该函数已经是异步调用,
                //没必要再嵌套一次异步调用
                String info = mStatusesAPI.friendsTimelineSync(since, max, 20, 1, false, 0, false);
                if (info != null && info.length() > 0) {
                    //获取内容成功
                    successCallback.invoke(info);
                } else {
                    //获取内容失败
                    errorCallback.invoke("Get weibo error..");
                }
            } catch (com.sina.weibo.sdk.exception.WeiboException e) {
                errorCallback.invoke("Please open the Internet :)");
            }
        } else {
            errorCallback.invoke("Get user information: token error..");
        }
    }
```

其中`@ReactMethod`标签指明该函数为一个react-native函数，将会暴露在js部分使用。设计好函数接口，需要讲模块添加到应用包里面。

在android/app/src/main/java/com/welone/weibo/WeiboPackage.java中，
创建一个ReactPackage：

```java
public class WeiboPackage implements ReactPackage {
    //添加自己定制的原生功能模块
    @Override
    public List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
        List<NativeModule> modules = new ArrayList<>();
        //这里目前就一个weibo模块
        modules.add(new WeiboModule(reactContext));
        return modules;
    }

    //可以添加js模块在原生代码中使用
    @Override
    public List<Class<? extends JavaScriptModule>> createJSModules() {
        return Collections.emptyList();
    }

    //定制原生view提供给js部分使用
    @Override
    public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
        return Collections.emptyList();
    }
}

```

从上面代码可以看出，react-native不仅可以定制功能模块，还可以定制原生view给js使用。

在android/app/src/main/java/com/welone/MainApplication.java下添加ReactPackage：

```java
    private final ReactNativeHost mReactNativeHost = new ReactNativeHost(this) {
        @Override
        protected boolean getUseDeveloperSupport() {
            return BuildConfig.DEBUG;
        }

        @Override
        protected List<ReactPackage> getPackages() {
            return Arrays.<ReactPackage>asList(
                    //react-native自己的包
                    new MainReactPackage(),
                    //因为使用native-base引入的包
                    new VectorIconsPackage(),
                    //自己定义的微博包
                    new WeiboPackage()
            );
        }
    };
```

写好原生部分的模块，接下来考虑如何在js部分进行使用。首先创建文件app/comp/module/WeiboModule.android.js，将定义的模块引入并且暴露。

```js
'use strict';

import { NativeModules } from 'react-native';

export default NativeModules.WeiboModule;
```

在WeiboList模块调用getTimeline函数：

```js
    //Get the weibo of the user's friends
    getTimeLine(type) {

        let since = 0;
        let max = 0;
        //调用原生模块实现的函数，参数要一一对应
        WeiboModule.getTimeline(
            since.toString(),
            max.toString(),
            //成功的回调函数
            (success) => {
                if (success.length > 0) {
                    let tl = JSON.parse(success);
                    let statuses = tl.statuses;
                    if (statuses.length > 0) {
                       
                        //获取内容数组
                        this.loveData = statuses;
                        ToastAndroid.showWithGravity(`有${statuses.length}条新微博 :)`, ToastAndroid.SHORT, ToastAndroid.CENTER);
 
                        //更新列表
                        this.setState({
                            loveSource: this.state.loveSource.cloneWithRows(this.loveData)
                        });
                    } 
                }
            },
            //失败的回调函数
            (err) => {
                ToastAndroid.showWithGravity(err, ToastAndroid.SHORT, ToastAndroid.CENTER);
                this.gettingTimeLine = false;
            }
        );
    }
```

实现getTimeLine函数后可以在适当的地方调用，获取微博内容。

到此为止，实现了应用获取微博内容，并且进行显示的功能。其它功能，比如获取评论等，实现过程类似。