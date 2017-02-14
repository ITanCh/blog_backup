title: Android系统源码阅读(18)Android应用的显示
date: 2017-02-14 15:04:48
tags: [Android,技术]
---

## 1. 启动ActivityManagerService

在前面第14章讲到，在System进程启动时，会启动系统的一些基本服务。启动就有ActivityManagerService和PackageManagerService。在SystemServer中如下启动ActivityManagerService。

*frameworks/base/services/java/com/android/server/SystemServer.java* :
{% codeblock lang:java %}
 // Activity manager runs the show.
 mActivityManagerService = mSystemServiceManager.startService(ActivityManagerService.Lifecycle.class).getService();
{% endcodeblock %}   

startService就是创建了ActivityManagerService的实例对象mActivityManagerService，然后启动了ActivityManagerService的looper线程。在SystemServer启动了基本服务后，就会调用mActivityManagerService的systemReady函数来启动HomeActivity。
<!--more-->

## 2. 启动HomeActivity

目前仍在SystemServer主线程中调用的systemReady函数。该函数又会接着调用startHomeActivityLocked函数来启动HomeActivity，具体如下：

*frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java*  :
{% codeblock lang:java %}
  boolean startHomeActivityLocked(int userId, String reason) {
        //...
        //先创建一个启动Home的Intent，Intent的category为android.intent.category.home
        Intent intent = getHomeIntent();
        ActivityInfo aInfo =
            resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
        if (aInfo != null) {
            intent.setComponent(new ComponentName(
                    aInfo.applicationInfo.packageName, aInfo.name));
            // Don't do this if the home app is currently being
            // instrumented.
            aInfo = new ActivityInfo(aInfo);
            aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);
            //因为是第一次启动home，所以还没有相应的进程，所以app为null
            ProcessRecord app = getProcessRecordLocked(aInfo.processName,
                    aInfo.applicationInfo.uid, true);
            if (app == null || app.instrumentationClass == null) {
                intent.setFlags(intent.getFlags() | Intent.FLAG_ACTIVITY_NEW_TASK);
                //这里开始启动HomeActivity
                mStackSupervisor.startHomeActivity(intent, aInfo, reason);
            }
        }
        return true;
    }
{% endcodeblock %}   

这里交给了mStackSupervisor.startHomeActivity函数来启动HomeActivity，然后就会调用startActivityLocked函数。到这里，就开始重复第2章中，启动普通应用的activity的过程了，不再赘述。其中，在Launcher的Manifest中指明了它的类型为android.intent.category.home，所以启动的就是Laucher应用。

## 3. Launcher展示应用

下面从Launcher的onCreate函数开始讲起，部分代码如下：

*packages/apps/Launcher3/src/com/android/launcher3/Launcher.java* :
{% codeblock lang:java %}
LauncherAppState app = LauncherAppState.getInstance();
//将该launcher设置为mModel的callback函数
mModel = app.setLauncher(this);

if (!mRestoring) {
    if (DISABLE_SYNCHRONOUS_BINDING_CURRENT_PAGE) {
        // If the user leaves launcher, then we should just load items asynchronously when
        // they return.
        mModel.startLoader(PagedView.INVALID_RESTORE_PAGE);
     } else {
        // We only load the page synchronously if the user rotates (or triggers a
        // configuration change) while launcher is in the foreground
        //开始加载应用信息
        mModel.startLoader(mWorkspace.getRestorePage());
      }
}
{% endcodeblock %}   
其中mModel是一个LauncherModel对象，它负责加载应用信息。在重载的startLoader函数中：

{% codeblock lang:java %}
 public void startLoader(int synchronousBindPage, int loadFlags) {
        //...
        synchronized (mLock) {
            //...
            // Don't bother to start the thread if we know it's not going to do anything
            if (mCallbacks != null && mCallbacks.get() != null) {
                //需要重新加载了，所以先把原来的加载任务停了
                // If there is already one running, tell it to stop.
                stopLoaderLocked();
                //创建了一个加载任务
                mLoaderTask = new LoaderTask(mApp.getContext(), loadFlags);
                if (synchronousBindPage != PagedView.INVALID_RESTORE_PAGE
                    //...
                } else {
                    //交给workThread来处理这个任务
                    sWorkerThread.setPriority(Thread.NORM_PRIORITY);
                    //sWorker是sWorkerThread的Handler
                    sWorker.post(mLoaderTask);
                }
            }
        }
    }
{% endcodeblock %}   

因为加载所有安装的应用信息是一个比较耗时的过程，所以应该交给一个异步线程来处理。这里先创建了一个加载任务mLoaderTask，然后将它交给sWorkerThread来处理。sWorkerThread是一个HandlerThread，我们知道HandlerThread是一个独立的线程，并且有着自己的looper。mLoaderTask加载任务执行一次就可以结束，但是为什么需要用一个HandlerThread在这里不断等待着其它任务呢？因为桌面上应用可以动态的添加和删除，所以应用的加载可能是频繁的。这里使用HandlerThread可以避免重复的创建加载线程。

下面我们就看LoaderTask.run函数中具体做了哪些任务。

*packages/apps/Launcher3/src/com/android/launcher3/LauncherModel.java*  :
{% codeblock lang:java %}
      public void run() {
            synchronized (mLock) {
                if (mStopped) {
                    return;
                }
                mIsLoaderTaskRunning = true;
            }
            // Optimize for end-user experience: if the Launcher is up and // running with the
            // All Apps interface in the foreground, load All Apps first. Otherwise, load the
            // workspace first (default).
            keep_running: {
                if (DEBUG_LOADERS) Log.d(TAG, "step 1: loading workspace");
                //启动workspace
                loadAndBindWorkspace();
                if (mStopped) {
                    break keep_running;
                }
                waitForIdle();
                // second step
                if (DEBUG_LOADERS) Log.d(TAG, "step 2: loading all apps");
                //加载应用
                loadAndBindAllApps();
            }
            //...
        }
{% endcodeblock %}   

WorkSpace就是Android桌面中的分屏，每个分屏有若干应用。先加载工作区，再加载应用更符合用户的心理预期？函数loadAndBindAllApps会判断是否已经加载过应用，如果已经加载过信息，则直接将app显示在桌面上即可；如果没有，则需要先调用函数loadAllApps，具体内容如下：

*packages/apps/Launcher3/src/com/android/luancher3/LauncherModel.java*  :
{% codeblock lang:java %}
 private void loadAllApps() {
            final Callbacks oldCallbacks = mCallbacks.get();

            // Clear the list of apps
            mBgAllAppsList.clear();
            for (UserHandleCompat user : profiles) {
                // Query for the set of apps
                //获取所有的app信息，这里最终还是从PackageManagerService获取的应用信息
                final List<LauncherActivityInfoCompat> apps = mLauncherApps.getActivityList(null, user);
         
                // Create the ApplicationInfos
                for (int i = 0; i < apps.size(); i++) {
                    LauncherActivityInfoCompat app = apps.get(i);
                    //把每个应用信息封装成AppInfo
                    // This builds the icon bitmaps.
                    mBgAllAppsList.add(new AppInfo(mContext, app, user, mIconCache));
                }
            }
            //...
            // Huh? Shouldn't this be inside the Runnable below?
            final ArrayList<AppInfo> added = mBgAllAppsList.added;
            mBgAllAppsList.added = new ArrayList<AppInfo>();
             
            //这里一直运行在Launcher中创建的一个HandlerThread子线程中
            //获取了应用信息后，需要通知主线程进行ui更新，mHandler是主线程的Handler
            // Post callback on main thread
            mHandler.post(new Runnable() {
                public void run() {
                    //这个callback就是launcher
                    final Callbacks callbacks = tryGetCallbacks(oldCallbacks);
                    if (callbacks != null) {
                       //显示所有应用
                        callbacks.bindAllApplications(added);
                       //...
                    } else {
                        //...
                    }
                }
            });
            //...
        }
{% endcodeblock %}   

mLauncherApps.getActivityList获取app信息的方法是通过向PackageManager发送请求的方式实现的。下面回到Launcher主线程，开始显示应用图标。首先看一眼这个回调函数：

*packages/apps/Launcher3/src/com/android/launcher3/Launcher.java* :
{% codeblock lang:java %}
   /**
     * Add the icons for all apps.
     *
     * Implementation of the method from LauncherModel.Callbacks.
     */
    public void bindAllApplications(final ArrayList<AppInfo> apps) {
        //...
        if (mAppsView != null) {
            //mAppsView管理着所有应用的view
            mAppsView.setApps(apps);
        }
        //...
    }
{% endcodeblock %}   

mAppsView是一个view的容器，维持了一个AlphabeticalAppsList列表，保存应用的信息。当向其中添加新的应用信息后，会对home上的应用进行更新。当home上的图标被点击后，会触发启动该应用。