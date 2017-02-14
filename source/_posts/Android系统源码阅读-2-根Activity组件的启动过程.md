title: Android系统源码阅读(2):根Activity组件的启动过程
date: 2017-02-14 13:34:58
tags: [Android,技术]
---

> 该系列只记录阅读代码时遇到的问题和心得体会，具体代码讲解可以参考老罗的《Android系统源代码情景分析》，我就不班门弄斧了。我编译的AOSP版本：6.0.1_r50。
     
## 代码摘抄

###  两个版本的Launcher
目前版本已经从Launcher2（packages/apps/Launcher2/src/com/android/launcher2）进化为Launcher3（packages/apps/Launcher3/src/com/android/launcher3）。两个版本的Launcher有着一定的差异，老罗书中以Launcher2为出发点。

<!--more-->

**packages/apps/Launcher2/src/com/android/launcher2/Launcher.java**:
{% codeblock lang:java %}
    boolean startActivity(View v, Intent intent, Object tag) {
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);

        try {
            // Only launch using the new animation if the shortcut has not opted out (this is a
            // private contract between launcher and may be ignored in the future).
            boolean useLaunchAnimation = (v != null) &&
                    !intent.hasExtra(INTENT_EXTRA_IGNORE_LAUNCH_ANIMATION);
            UserHandle user = (UserHandle) intent.getParcelableExtra(ApplicationInfo.EXTRA_PROFILE);
            LauncherApps launcherApps = (LauncherApps)
                    this.getSystemService(Context.LAUNCHER_APPS_SERVICE);
            if (useLaunchAnimation) {
                ActivityOptions opts = ActivityOptions.makeScaleUpAnimation(v, 0, 0,
                        v.getMeasuredWidth(), v.getMeasuredHeight());
                if (user == null || user.equals(android.os.Process.myUserHandle())) {
                    // Could be launching some bookkeeping activity
                    startActivity(intent, opts.toBundle());
                } else {
                    launcherApps.startMainActivity(intent.getComponent(), user,
                            intent.getSourceBounds(),
                            opts.toBundle());
                }
            } else {
                if (user == null || user.equals(android.os.Process.myUserHandle())) {
                    startActivity(intent);
                } else {
                    launcherApps.startMainActivity(intent.getComponent(), user,
                            intent.getSourceBounds(), null);
                }
            }
            return true;
        } catch (SecurityException e) {
            Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
            Log.e(TAG, "Launcher does not have the permission to launch " + intent +
                    ". Make sure to create a MAIN intent-filter for the corresponding activity " +
                    "or use the exported attribute for this activity. "
                    + "tag="+ tag + " intent=" + intent, e);
        }
        return false;
    }
    
    boolean startActivitySafely(View v, Intent intent, Object tag) {
        boolean success = false;
        try {
            success = startActivity(v, intent, tag);
        } catch (ActivityNotFoundException e) {
            Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
            Log.e(TAG, "Unable to launch. tag=" + tag + " intent=" + intent, e);
        }
        return success;
    }
{% endcodeblock %}

**packages/apps/Launcher3/src/com/android/launcher3/Launcher.java**:
{% codeblock lang:java %}
    private boolean startActivity(View v, Intent intent, Object tag) {
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        try {
            // Only launch using the new animation if the shortcut has not opted out (this is a
            // private contract between launcher and may be ignored in the future).
            boolean useLaunchAnimation = (v != null) &&
                    !intent.hasExtra(INTENT_EXTRA_IGNORE_LAUNCH_ANIMATION);
            LauncherAppsCompat launcherApps = LauncherAppsCompat.getInstance(this);
            UserManagerCompat userManager = UserManagerCompat.getInstance(this);

            UserHandleCompat user = null;
            if (intent.hasExtra(AppInfo.EXTRA_PROFILE)) {
                long serialNumber = intent.getLongExtra(AppInfo.EXTRA_PROFILE, -1);
                user = userManager.getUserForSerialNumber(serialNumber);
            }

            Bundle optsBundle = null;
            if (useLaunchAnimation) {
                ActivityOptions opts = null;
                if (Utilities.ATLEAST_MARSHMALLOW) {
                    int left = 0, top = 0;
                    int width = v.getMeasuredWidth(), height = v.getMeasuredHeight();
                    if (v instanceof TextView) {
                        // Launch from center of icon, not entire view
                        Drawable icon = Workspace.getTextViewIcon((TextView) v);
                        if (icon != null) {
                            Rect bounds = icon.getBounds();
                            left = (width - bounds.width()) / 2;
                            top = v.getPaddingTop();
                            width = bounds.width();
                            height = bounds.height();
                        }
                    }
                    opts = ActivityOptions.makeClipRevealAnimation(v, left, top, width, height);
                } else if (!Utilities.ATLEAST_LOLLIPOP) {
                    // Below L, we use a scale up animation
                    opts = ActivityOptions.makeScaleUpAnimation(v, 0, 0,
                                    v.getMeasuredWidth(), v.getMeasuredHeight());
                } else if (Utilities.ATLEAST_LOLLIPOP_MR1) {
                    // On L devices, we use the device default slide-up transition.
                    // On L MR1 devices, we a custom version of the slide-up transition which
                    // doesn't have the delay present in the device default.
                    opts = ActivityOptions.makeCustomAnimation(this,
                            R.anim.task_open_enter, R.anim.no_anim);
                }
                optsBundle = opts != null ? opts.toBundle() : null;
            }

            if (user == null || user.equals(UserHandleCompat.myUserHandle())) {
                // Could be launching some bookkeeping activity
                startActivity(intent, optsBundle);
            } else {
                // TODO Component can be null when shortcuts are supported for secondary user
                launcherApps.startActivityForProfile(intent.getComponent(), user,
                        intent.getSourceBounds(), optsBundle);
            }
            return true;
        } catch (SecurityException e) {
            Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
            Log.e(TAG, "Launcher does not have the permission to launch " + intent +
                    ". Make sure to create a MAIN intent-filter for the corresponding activity " +
                    "or use the exported attribute for this activity. "
                    + "tag="+ tag + " intent=" + intent, e);
        }
        return false;
    }

    public boolean startActivitySafely(View v, Intent intent, Object tag) {
        boolean success = false;
        if (mIsSafeModeEnabled && !Utilities.isSystemApp(this, intent)) {
            Toast.makeText(this, R.string.safemode_shortcut_error, Toast.LENGTH_SHORT).show();
            return false;
        }
        try {
            success = startActivity(v, intent, tag);
        } catch (ActivityNotFoundException e) {
            Toast.makeText(this, R.string.activity_not_found, Toast.LENGTH_SHORT).show();
            Log.e(TAG, "Unable to launch. tag=" + tag + " intent=" + intent, e);
        }
        return success;
    }
{% endcodeblock %}

### Android的Singleton模式的实现
在**frameworks/base/core/java/android/app/ActivityManagerNative.java**中`IActivityManager`是一个单利模式，所以Android实现了一个很标准的Singleton。在老罗版本里，没有使用这个类。

**framework/base/core/java/android/util/Singleton.java**:
{% codeblock lang:java %}
/**
 * Singleton helper class for lazily initialization.
 *
 * Modeled after frameworks/base/include/utils/Singleton.h
 *
 * @hide
 */
public abstract class Singleton<T> {
    private T mInstance;

    protected abstract T create();

    public final T get() {
        synchronized (this) {
            if (mInstance == null) {
                mInstance = create();
            }
            return mInstance;
        }
    }
}
{% endcodeblock %}

### ActivityManagerService

显然，AcitivityManagerService的位置由`frameworks/base/services/java/com/android/service/am/ActivityManagerService.java`转移至`frameworks/base/services/core/java/com/android/service/am/ActivityManagerService.java`。同时`startActivity`函数也有所改变：

**frameworks/base/services/core/java/com/android/service/am/ActivityManagerService.java**:
{% codeblock lang:java %}
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, options,
            UserHandle.getCallingUserId());
    }

    @Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {
        enforceNotIsolatedCaller("startActivity");
        userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
                false, ALLOW_FULL_ONLY, "startActivity", null);
        // TODO: Switch to user app stacks here.
        return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, options, false, userId, null, null);
    }
{% endcodeblock %}

主要的变化是添加了用户的概念，同时加入了mStackSupervisor（ActivityStackSuperVisor）来管理ActivityStack。

## 根Activity启动过程   
下面将用顺序图来展示根Activity是如何从桌面启动的。

### Step1. Launcher启动一个APP
![这里写图片描述](http://img.blog.csdn.net/20160816201736649)
当用户点击某一个应用图标，意味着需要启动一个应用的根Activity。Launcher也是一个应用，和普通应用启动Activity过程类似。这里展示了在Launcher中的进行的步骤，最后通过Binder与ActivityMangerService进行进程间通信，告知它来启动Activity。

### Step2. ActivityMangerService进行准备工作   
![这里写图片描述](http://img.blog.csdn.net/20160817161257468)
这里主要工作就行将需要启动的Activity放置于栈顶，等待启动。在启动新Activity前，需要将旧的Activity先停止下来。

### Step3. 旧Activity停止
![这里写图片描述](http://img.blog.csdn.net/20160816203017385)
在这里，需要停止的Activity就是Launcher的Activity。在完成停止的过程后，需要告知ActivityMangerService。

### Step4. ActivityMangerService继续准备
![这里写图片描述](http://img.blog.csdn.net/20160816203325622)
这里会发现该Activity还没有进程可以运行，所以需要先启动一个新的ActvityThread进程。

### Step5. ActivityThread进程启动
![这里写图片描述](http://img.blog.csdn.net/20160816203708629)
等进程启动完毕后，同样需要ActivityMangerService，以便它将等待该进程的Activity启动起来。

### Step6. ActivityMangerService继续启动
![这里写图片描述](http://img.blog.csdn.net/20160816205231942)
这里会将等待该进程的Activity和Service启动起来。

### Step7. 新Activity启动
![这里写图片描述](http://img.blog.csdn.net/20160816205319833)
到这里，启动过程结束