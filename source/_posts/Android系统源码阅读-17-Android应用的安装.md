title: Android系统源码阅读(17):Android应用的安装
date: 2017-02-14 15:00:58
tags: [Android,技术]
---

> 学到的才是自己的，干活都是扯淡


## 1. 应用的安装

PackageManagerService负责管理应用的安装。在第14章中讲到，SystemService会启动PackageManagerService，那么我们就从SystemService启动PackageManagerService开始分析。

<!--more-->

### 1.1 PackageManagerService.main

创建PackageManagerService对象。

*frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java*  :
{% codeblock lang:java %}
public static PackageManagerService main(Context context, Installer installer, boolean factoryTest, boolean onlyCore) {
     //先创建一个PackageManagerService对象
     PackageManagerService m = new PackageManagerService(context, installer, factoryTest, onlyCore);
     //将他注册到ServiceManager里面
     ServiceManager.addService("package", m);
     return m;
 }
{% endcodeblock %}   

### 1.2 PackageManagerService.PackageManagerService

PackageManagerService的构造函数。

*frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java* :
{% codeblock lang:java %}
  public PackageManagerService(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        //...

        mSettings = new Settings(mPackages);

	//...
        synchronized (mInstallLock) {
        // writer
        synchronized (mPackages) {
            //一个PackageManagerService自己的Looper
            mHandlerThread = new ServiceThread(TAG,
                    Process.THREAD_PRIORITY_BACKGROUND, true /*allowIo*/);
　　　　　　　//线程启动，开启自己的消息循环队列
            mHandlerThread.start();
            //并且创建了一个Handler该线程
            mHandler = new PackageHandler(mHandlerThread.getLooper());

　　　　　　　//获取文件路径
            File dataDir = Environment.getDataDirectory();
            mAppDataDir = new File(dataDir, "data");
            mAppInstallDir = new File(dataDir, "app");
            mAppLib32InstallDir = new File(dataDir, "app-lib");
            mAsecInternalPath = new File(dataDir, "app-asec").getPath();
            mUserAppDataDir = new File(dataDir, "user");
            mDrmAppPrivateInstallDir = new File(dataDir, "app-private");

            sUserManager = new UserManagerService(context, this,
                    mInstallLock, mPackages);

	　　 //　获取权限配置文件的中的权限
            // Propagate permission configuration in to package manager.
            ArrayMap<String, SystemConfig.PermissionEntry> permConfig
                    = systemConfig.getPermissions();
            for (int i=0; i<permConfig.size(); i++) {
                SystemConfig.PermissionEntry perm = permConfig.valueAt(i);
                BasePermission bp = mSettings.mPermissions.get(perm.name);
                if (bp == null) {
                    bp = new BasePermission(perm.name, "android", BasePermission.TYPE_BUILTIN);
                    mSettings.mPermissions.put(perm.name, bp);
                }
                if (perm.gids != null) {
                    bp.setGids(perm.gids, perm.perUser);
                }
            }

	　　 //获取共享库
            ArrayMap<String, String> libConfig = systemConfig.getSharedLibraries();
            for (int i=0; i<libConfig.size(); i++) {
                mSharedLibraries.put(libConfig.keyAt(i),
                        new SharedLibraryEntry(libConfig.valueAt(i), null));
            }

           　//恢复上一次应用安装信息，见1.3
            mRestoredSettings = mSettings.readLPw(this, sUserManager.getUsers(false),
                    mSdkVersion, mOnlyCore);

            /**
             * Add everything in the in the boot class path to the
             * list of process files because dexopt will have been run
             * if necessary during zygote startup.
             */
            final String bootClassPath = System.getenv("BOOTCLASSPATH");
            final String systemServerClassPath = System.getenv("SYSTEMSERVERCLASSPATH");



            File frameworkDir = new File(Environment.getRootDirectory(), "framework");

            // Gross hack for now: we know this file doesn't contain any
            // code, so don't dexopt it to avoid the resulting log spew.
            alreadyDexOpted.add(frameworkDir.getPath() + "/framework-res.apk");

            // Gross hack for now: we know this file is only part of
            // the boot class path for art, so don't dexopt it to
            // avoid the resulting log spew.
            alreadyDexOpted.add(frameworkDir.getPath() + "/core-libart.jar");

            /**
             * 将一些应用dexopt转化
             * There are a number of commands implemented in Java, which
             * we currently need to do the dexopt on so that they can be
             * run from a non-root shell.
             */

           　//...

            //设备厂商提供的应用
            // Collect vendor overlay packages.
            // (Do this before scanning any apps.)
            // For security and version matching reason, only consider
            // overlay packages if they reside in VENDOR_OVERLAY_DIR.
            File vendorOverlayDir = new File(VENDOR_OVERLAY_DIR);
            scanDirLI(vendorOverlayDir, PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags | SCAN_TRUSTED_OVERLAY, 0);
　　　　　　　// 资源型的程序
            // Find base frameworks (resource packages without code).
            scanDirLI(frameworkDir, PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR
                    | PackageParser.PARSE_IS_PRIVILEGED,
                    scanFlags | SCAN_NO_DEX, 0);
　　　　　　  //特权应用
            // Collected privileged system packages.
            final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
            scanDirLI(privilegedAppDir, PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR
                    | PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);

            // Collect ordinary system packages.
            final File systemAppDir = new File(Environment.getRootDirectory(), "app");
            scanDirLI(systemAppDir, PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

            // Collect all vendor packages.
            File vendorAppDir = new File("/vendor/app");
            try {
                vendorAppDir = vendorAppDir.getCanonicalFile();
            } catch (IOException e) {
                // failed to look up canonical path, continue with original one
            }
            scanDirLI(vendorAppDir, PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

            // Collect all OEM packages.
            final File oemAppDir = new File(Environment.getOemDirectory(), "app");
            scanDirLI(oemAppDir, PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

            if (DEBUG_UPGRADE) Log.v(TAG, "Running installd update commands");
            mInstaller.moveFiles();

            // Prune any system packages that no longer exist.
            final List<String> possiblyDeletedUpdatedSystemApps = new ArrayList<String>();
           　//...


            //look for any incomplete package installations
            ArrayList<PackageSetting> deletePkgsList = mSettings.getListOfIncompleteInstallPackagesLPr();
            //clean up list
            for(int i = 0; i < deletePkgsList.size(); i++) {
                //clean up here
                cleanupInstallFailedPackage(deletePkgsList.get(i));
            }
            //delete tmp files
            deleteTempPackageFiles();

            // Remove any shared userIDs that have no associated packages
            mSettings.pruneSharedUsersLPw();
            //...

            // Now that we know all the packages we are keeping,
            // read and update their last usage times.
            mPackageUsage.readLP();

            // If the platform SDK has changed since the last time we booted,
            // we need to re-grant app permission to catch any new ones that
            // appear.  This is really a hack, and means that apps can in some
            // cases get permissions that the user didn't initially explicitly
            // allow...  it would be nice to have some better way to handle
            // this situation.
            int updateFlags = UPDATE_PERMISSIONS_ALL;
            if (ver.sdkVersion != mSdkVersion) {
                Slog.i(TAG, "Platform changed from " + ver.sdkVersion + " to "
                        + mSdkVersion + "; regranting permissions for internal storage");
                updateFlags |= UPDATE_PERMISSIONS_REPLACE_PKG | UPDATE_PERMISSIONS_REPLACE_ALL;
            }
　　　　　　　　　　　　//为申请了特定资源的访问权限的应用分配用户组id
            updatePermissionsLPw(null, null, StorageManager.UUID_PRIVATE_INTERNAL, updateFlags);
            ver.sdkVersion = mSdkVersion;

            //将应用安装信息保存到本地的配置文件
            // can downgrade to reader
            mSettings.writeLPr();


        } // synchronized (mPackages)
        } // synchronized (mInstallLock)

        // Now after opening every single application zip, make sure they
        // are all flushed.  Not really needed, but keeps things nice and
        // tidy.
        Runtime.getRuntime().gc();

        // Expose private service for system components to use.
        LocalServices.addService(PackageManagerInternal.class, new PackageManagerInternalImpl());
    }
{% endcodeblock %}   

### 1.3 Settings.readLPw

读取保存的应用信息。
*frameworks/base/services/core/java/com/android/server/pm/Settings.java* :
{% codeblock lang:java %}
boolean readLPw(PackageManagerService service, List<UserInfo> users, int sdkVersion,
            boolean onlyCore) {
        FileInputStream str = null;
        if (mBackupSettingsFilename.exists()) {
            try {
                //获取/data/system/packages-backup.xml文件
                str = new FileInputStream(mBackupSettingsFilename);
                //如果backup不存在，则获取/data/system/packages.xml文件
                //...
            } catch (java.io.IOException e) {
                // We'll try for the normal settings file.
            }
        }
	//...
        try {
            if (str == null) {
                //...
                str = new FileInputStream(mSettingsFilename);
            }
            //用parser解析xml文件
            XmlPullParser parser = Xml.newPullParser();
            parser.setInput(str, StandardCharsets.UTF_8.name());
            //...
            int outerDepth = parser.getDepth();
            while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                    && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
                if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                    continue;
                }
                 
                //获取应用的各项参数
                String tagName = parser.getName();
                if (tagName.equals("package")) {
                    //获取上次分配给他的应用用户ID，见1.4
                    readPackageLPw(parser);
                } else if (tagName.equals("permissions")) {
                    readPermissionsLPw(mPermissions, parser);
                } else if (tagName.equals("permission-trees")) {
                    readPermissionsLPw(mPermissionTrees, parser);
                } else if (tagName.equals("shared-user")) {
                    //获取上次分配的共享用户ID，见1.6
                    readSharedUserLPw(parser);
                } else if {
                  //...
                } else {
                    Slog.w(PackageManagerService.TAG, "Unknown element under <packages>: "
                            + parser.getName());
                    XmlUtils.skipCurrentTag(parser);
                }
            }

            str.close();

        } catch (XmlPullParserException e) {
           //...
        } catch (java.io.IOException e) {
           //....
        }
        //...
        return true;
    }
{% endcodeblock %}   

### 1.4 Settings.readPackageLPw

读取package信息。

*frameworks/base/services/core/java/com/android/server/pm/Settings.java* :
{% codeblock lang:java %}
 private void readPackageLPw(XmlPullParser parser) throws XmlPullParserException, IOException {
        String name = null;
        //...
        String idStr = null;
        String sharedIdStr = null;
	//...
     
        try {
            name = parser.getAttributeValue(null, ATTR_NAME);
            realName = parser.getAttributeValue(null, "realName");
            idStr = parser.getAttributeValue(null, "userId");
            //...
            //名字不能为null
            if (name == null) {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Error in package manager settings: <package> has no name at "
                                + parser.getPositionDescription());
            } else if (codePathStr == null) {
                PackageManagerService.reportSettingsProblem(Log.WARN,
                        "Error in package manager settings: <package> has no codePath at "
                                + parser.getPositionDescription());
            } else if (userId > 0) {
                //该应用上次已经分配过ID，所以这次还使用该id,见1.5
                packageSetting = addPackageLPw(name.intern(), realName, new File(codePathStr),
                        new File(resourcePathStr), legacyNativeLibraryPathStr, primaryCpuAbiString,
                        secondaryCpuAbiString, cpuAbiOverrideString, userId, versionCode, pkgFlags,
                        pkgPrivateFlags);
                //...
            } else if (sharedIdStr != null) {
                userId = sharedIdStr != null ? Integer.parseInt(sharedIdStr) : 0;
                if (userId > 0) {
                    //如果上次使用的是共享id，则说明上次该app没有独立的id
                    //所以这次不能直接将该id分配给他，要先将其保存起来以后处理
                    packageSetting = new PendingPackage(name.intern(), realName, new File(
                            codePathStr), new File(resourcePathStr), legacyNativeLibraryPathStr,
                            primaryCpuAbiString, secondaryCpuAbiString, cpuAbiOverrideString,
                            userId, versionCode, pkgFlags, pkgPrivateFlags);
                    //...
                    mPendingPackages.add((PendingPackage) packageSetting);
                   //...
                } else {
                   //...
                }
            } else {
                //...
            }
        } catch (NumberFormatException e) {
            //...
        }
      //...
    }
{% endcodeblock %}   

### 1.5 Setttings.addPackageLPw

为该package分配UID，创建保存package信息的PackageSetting对象。

*frameworks/base/services/core/java/com/android/server/pm/Settings.java* :
{% codeblock lang:java %}
    PackageSetting addPackageLPw(String name, String realName, File codePath, File resourcePath,
            String legacyNativeLibraryPathString, String primaryCpuAbiString, String secondaryCpuAbiString,
            String cpuAbiOverrideString, int uid, int vc, int pkgFlags, int pkgPrivateFlags) {
        //每一个app的信息都保存于一个PackageSetting对象中
        PackageSetting p = mPackages.get(name);
        if (p != null) {
            //该应用已经有对应的PackageSetting对象了，无需再次添加
            if (p.appId == uid) {
                return p;
            }
            PackageManagerService.reportSettingsProblem(Log.ERROR,
                    "Adding duplicate package, keeping first: " + name);
            return null;
        }
        //为该app创建一个PackageSetting
        p = new PackageSetting(name, realName, codePath, resourcePath,
                legacyNativeLibraryPathString, primaryCpuAbiString, secondaryCpuAbiString,
                cpuAbiOverrideString, vc, pkgFlags, pkgPrivateFlags);
        p.appId = uid;
        //在系统中保留值为uid的Linux用户ID，会在下面详解
        if (addUserIdLPw(uid, p, name)) {
            //mPackages保存创建的app PackageSetting对象
            mPackages.put(name, p);
            return p;
        }
        return null;
    }
{% endcodeblock %}   

在系统中保留指定的UID。这里可以看出，一共可以分配10000个uid给应用程序，小于10000的uid是分配给特权用户的。这些特权用户的uid可以通过共享的形式给其它应用使用。例如，一个想修改系统时间的的应用可以共享"android.uid.system"的特权用户的uid，即在配置文件中将它的android:sharedUserId的属性设置为“android.uid.system”。uid不能重复，但是shareduid是可以共享的。

*frameworks/base/services/core/java/com/android/server/pm/Settings.java* :
{% codeblock lang:java %}
    private boolean addUserIdLPw(int uid, Object obj, Object name) {
        //分配的uid不能超过LAST_APPLICATION_UID，19999
        if (uid > Process.LAST_APPLICATION_UID) {
            return false;
        }
        
        //分配的uid应该大于等于FIRST_APPLICATION_UID，10000
        if (uid >= Process.FIRST_APPLICATION_UID) {
            int N = mUserIds.size();
            final int index = uid - Process.FIRST_APPLICATION_UID;
            while (index >= N) {
                //将中间没分配的uid先置为空
                mUserIds.add(null);
                N++;
            }
            if (mUserIds.get(index) != null) {
                //该uid重复了，分配失败
                return false;
            }
            //正式分配uid，所有的uid被mUserIds管理
            mUserIds.set(index, obj);
        } else {
            //这里是特权用户的uid
            if (mOtherUserIds.get(uid) != null) {
                //uid已经分配过，返回失败
                return false;
            }
            mOtherUserIds.put(uid, obj);
        }
        return true;
    }
{% endcodeblock %}   
到这里，应用的uid已经分配完成。下面回到1.3，看如何读取shareduid。

### 1.6 Setttings.readSharedUserLPw

读取共享uid。

*frameworks/base/services/core/java/com/android/server/pm/Settings.java* :
{% codeblock lang:java %}
    private void readSharedUserLPw(XmlPullParser parser) throws XmlPullParserException,IOException {
        String name = null;
        String idStr = null;
        int pkgFlags = 0;
        int pkgPrivateFlags = 0;
        SharedUserSetting su = null;
        try {
            name = parser.getAttributeValue(null, ATTR_NAME);
            idStr = parser.getAttributeValue(null, "userId");
            int userId = idStr != null ? Integer.parseInt(idStr) : 0;
            //该共享id是系统应用还是用户类型的应用
            if ("true".equals(parser.getAttributeValue(null, "system"))) {
                pkgFlags |= ApplicationInfo.FLAG_SYSTEM;
            }
            if (name == null) {
                //...
            } else if (userId == 0) {
               //...
            } else {
                //在系统中为该应用申请该userId，见1.7
                if ((su = addSharedUserLPw(name.intern(), userId, pkgFlags, pkgPrivateFlags))
                        == null) {
                   //...
                }
            }
        } catch (NumberFormatException e) {
	    //...
        }
	   //....
        } else {
            XmlUtils.skipCurrentTag(parser);
        }
    }
{% endcodeblock %}   

### 1.7 Settings.addSharedUserLPw

添加共享用户。

*frameworks/base/services/core/java/com/android/server/pm/Settings.java* :
{% codeblock lang:java %}
SharedUserSetting addSharedUserLPw(String name, int uid, int pkgFlags, int pkgPrivateFlags) {
        SharedUserSetting s = mSharedUsers.get(name);
        if (s != null) {
            if (s.userId == uid) {
                return s;
            }
            //添加uid和已经添加的不同，则失败返回
            return null;
        }
        //mSharedUsers中没有该共享用户id，则创建一个SharedUserSetting
        s = new SharedUserSetting(name, pkgFlags, pkgPrivateFlags);
        s.userId = uid;
        //在系统中为它分配该uid，重复1.6
        if (addUserIdLPw(uid, s, name)) {
	    //分配成功，记录下来
            mSharedUsers.put(name, s);
            return s;
        }
        return null;
    }
{% endcodeblock %}   

在1.4中，mPendingPackages保存了一些使用共享用户id的package。现在已经解析完毕共享用户的信息，在1.3 readLPw函数中将会处理这些package了。

*frameworks/base/services/core/java/com/android/server/pm/Settings.java* :
{% codeblock lang:java %}
for (int i = 0; i < N; i++) {
            final PendingPackage pp = mPendingPackages.get(i);
            Object idObj = getUserIdLPr(pp.sharedId);
            if (idObj != null && idObj instanceof SharedUserSetting) {
                //获得共享用户信息，说明该package使用的共享用户id是有效的，将在下面小节分析
                PackageSetting p = getPackageLPw(pp.name, null, pp.realName,
                        (SharedUserSetting) idObj, pp.codePath, pp.resourcePath,
                        pp.legacyNativeLibraryPathString, pp.primaryCpuAbiString,
                        pp.secondaryCpuAbiString, pp.versionCode, pp.pkgFlags, pp.pkgPrivateFlags,
                        null, true /* add */, false /* allowInstall */);
                if (p == null) {
                    //...
                    continue;
                }
                p.copyFrom(pp);
            } else if (idObj != null) {
               //该package使用了一个非共享的id
            } else {
               //使用的共享id不存在
            }
        }
        mPendingPackages.clear();
{% endcodeblock %}   

### 1.8 PackageManagerService.scanDirLI

到此，PackageManagerService已经将上一次保存的应用信息恢复完毕，在1.2中接下来需要进一步安装各个目录下的应用。

*frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java* :
{% codeblock lang:java %}
    private void scanDirLI(File dir, int parseFlags, int scanFlags, long currentTime) {
        final File[] files = dir.listFiles();
        //依次访问文件夹中的每个apk文件
        for (File file : files) {
            final boolean isPackage = (isApkFile(file) || file.isDirectory())
                    && !PackageInstallerService.isStageName(file.getName());
            if (!isPackage) {
                // Ignore entries which are not packages
                continue;
            }
            try {
                scanPackageLI(file, parseFlags | PackageParser.PARSE_MUST_BE_APK,
                        scanFlags, currentTime, null);
            } catch (PackageManagerException e) {
                //删除无效的文件
                // Delete invalid userdata apps
                if ((parseFlags & PackageParser.PARSE_IS_SYSTEM) == 0 &&
                        e.error == PackageManager.INSTALL_FAILED_INVALID_APK) {
                    if (file.isDirectory()) {
                        mInstaller.rmPackageDir(file.getAbsolutePath());
                    } else {
                        file.delete();
                    }
                }
            }
        }
    }
{% endcodeblock %}   


### 1.9 PackageManagerService.scanPackageLI

*frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java*  :
{% codeblock lang:java %}
    private PackageParser.Package scanPackageLI(File scanFile, int parseFlags, int scanFlags,
            long currentTime, UserHandle user) throws PackageManagerException {
        //...
        PackageParser pp = new PackageParser();
        //...
        final PackageParser.Package pkg;
        try {
            //解析目标文件
            pkg = pp.parsePackage(scanFile, parseFlags);
        } catch (PackageParserException e) {
            //...
        }
        //...
 }
{% endcodeblock %}   

### 1.10 PackageParser.parsePackage

*frameworks/base/core/java/android/content/pm/PackageParser.java*  :
{% codeblock lang:java %}
  public Package parsePackage(File packageFile, int flags) throws PackageParserException {
        if (packageFile.isDirectory()) {
            //一个目录下是一个应用
            return parseClusterPackage(packageFile, flags);
        } else {
            //一个整体的应用
            return parseMonolithicPackage(packageFile, flags);
        }
    }
{% endcodeblock %}   

### 1.11 PackageParser.parseMonolithicPackage

*frameworks/base/core/java/android/content/pm/PackageParser.java*  :
{% codeblock lang:java %}
   public Package parseMonolithicPackage(File apkFile, int flags) throws PackageParserException {
        //...
        final AssetManager assets = new AssetManager();
        try {
            final Package pkg = parseBaseApk(apkFile, assets, flags);
            pkg.codePath = apkFile.getAbsolutePath();
            return pkg;
        } finally {
            IoUtils.closeQuietly(assets);
        }
  }
{% endcodeblock %}   

### 1.12 PackageParser.parseBaseApk

*frameworks/base/core/java/android/content/pm/PackageParser.java* :
{% codeblock lang:java %}
    private Package parseBaseApk(File apkFile, AssetManager assets, int flags)
            throws PackageParserException {
        //...
        final String apkPath = apkFile.getAbsolutePath();
        mArchiveSourcePath = apkFile.getAbsolutePath();
        final int cookie = loadApkIntoAssetManager(assets, apkPath, flags);

        Resources res = null;
        XmlResourceParser parser = null;
        try {
            res = new Resources(assets, mMetrics, null);
            assets.setConfiguration(0, 0, null, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
                    Build.VERSION.RESOURCES_SDK_INT);
            //获取AndroidManifest.xml文件
            parser = assets.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);

            //解析AndroidManifest.xml文件
            final Package pkg = parseBaseApk(res, parser, flags, outError);
           //...
            return pkg;
        } catch (PackageParserException e) {
           //...
        } finally {
            IoUtils.closeQuietly(parser);
        }
    }
{% endcodeblock %}   
*frameworks/base/core/java/android/content/pm/PackageParser.java* :
{% codeblock lang:java %}
    private Package parseBaseApk(Resources res, XmlResourceParser parser, int flags,
            String[] outError) throws XmlPullParserException, IOException {
        final boolean trustedOverlay = (flags & PARSE_TRUSTED_OVERLAY) != 0;

        AttributeSet attrs = parser;

        final String pkgName;
        final String splitName;
        try {
            Pair<String, String> packageSplit = parsePackageSplitNames(parser, attrs, flags);
　　　　　　　　　　　 //获取package名称
            pkgName = packageSplit.first;
            splitName = packageSplit.second;
        } catch (PackageParserException e) {
            //...
        }

        //创建Package对象
        final Package pkg = new Package(pkgName);
        boolean foundApp = false;

        TypedArray sa = res.obtainAttributes(attrs,
                com.android.internal.R.styleable.AndroidManifest);
　　　　　　　　//应用版本信息
        pkg.mVersionCode = pkg.applicationInfo.versionCode = sa.getInteger(
                com.android.internal.R.styleable.AndroidManifest_versionCode, 0);
        //...
　　　　　　　　//用户共享id
        String str = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifest_sharedUserId, 0);
        if (str != null && str.length() > 0) {
            String nameError = validateName(str, true, false);
            if (nameError != null && !"android".equals(pkgName)) {
                outError[0] = "<manifest> specifies bad sharedUserId name \""
                    + str + "\": " + nameError;
                mParseError = PackageManager.INSTALL_PARSE_FAILED_BAD_SHARED_USER_ID;
                return null;
            }
            //设置共享id后表示要和其它应用共享一个id
            pkg.mSharedUserId = str.intern();
            pkg.mSharedUserLabel = sa.getResourceId(
                    com.android.internal.R.styleable.AndroidManifest_sharedUserLabel, 0);
        }

        //...

        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > outerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            if (tagName.equals("application")) {
                //...
                //进一步解析application项的内容
                if (!parseBaseApplication(pkg, res, parser, attrs, flags, outError)) {
                    return null;
                }
		//...
            } else if (tagName.equals("uses-permission")) {
                //获取申请的权限
                if (!parseUsesPermission(pkg, res, parser, attrs)) {
                    return null;
                }

            }
	   //...
	}
        return pkg;
    }
{% endcodeblock %}   
一个应用可以申请多个权限，应用权限是和用户组ID对应的。应用申请权限就是获得该用户组id的过程。

### 1.13 

*framework/base/core/java/android/content/pm/PackageParser.java*
{% codeblock lang:java %}
private boolean parseBaseApplication(Package owner, Resources res,
            XmlPullParser parser, AttributeSet attrs, int flags, String[] outError)
        throws XmlPullParserException, IOException {
      
        final int innerDepth = parser.getDepth();
        int type;
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > innerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }
         
            String tagName = parser.getName();
            //解析出activity
            if (tagName.equals("activity")) {
                Activity a = parseActivity(owner, res, parser, attrs, flags, outError, false,
                        owner.baseHardwareAccelerated);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.activities.add(a);

            } else if (tagName.equals("receiver")) {
                //解析出receiver
                Activity a = parseActivity(owner, res, parser, attrs, flags, outError, true, false);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.receivers.add(a);

            } else if (tagName.equals("service")) {
                //解析service
                Service s = parseService(owner, res, parser, attrs, flags, outError);
                if (s == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.services.add(s);

            } else if (tagName.equals("provider")) {
                //解析provider
                Provider p = parseProvider(owner, res, parser, attrs, flags, outError);
                if (p == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.providers.add(p);

            } 
        }
            //...
        return true;
    }
{% endcodeblock %}   

### 1.14 下面步骤笼统讲一下

到这里，PackageManagerService已经解析完成一个apk文件。下面回到1.9 scanPackageLI中，继续下面的工作。1.9步中会调用重载函数scanPackageLI继续解析获得的package。重载函数会调用scanPackageDirtyLI来分析package。

这一步会为package分配id，当然该package可能使用的是一个共享id。首先，每个package信息会被保存在一个PackageSetting对象中，然后将其放到mPackages这个HashMap中。

因为在前面步骤中已经从上一次安装的应用信息中读取了一些package信息，所以需要先在HashMap中确定一下这个新解析的应用是否是已经在mPackages中。如果已经存在，说明这是一个老应用，就直接将PackageSetting 返回。如果不在，则新建一个PackageSetting，下面开始为它分配uid：   	
	1. 使用共享id。将pkg的uid设置为想要共享的uid。
	2. 使用原来的uid。禁用的系统程序使用它原来的uid。
	3. 使用新的uid。创建一个新的uid给pkg。
	4. 使用first application uid。所有应用使用同一个uid。

下面来看一下如何为新的应用分配新uid。mUserIds管理了所有的分配给用户的uid，这里会在规定的范围内找到一个uid分配。 

到此为止，所有的安装了应用都存在PackageManagerService的mPackages中，下面需要依次为这些应用分配权限。在设备`/system/etc/permissions/platform.xml`文件中，表述了该设备的资源访问权限列表。如：

{% codeblock lang:xml %}
    <permission name="android.permission.BLUETOOTH" >
        <group gid="net_bt" />
    </permission
{% endcodeblock %}   

gid指明了该资源权限所在的用户组，当然一个权限可以拥有多个用户组。应用获取权限就是加入相应的用户组。Package的申请的权限已经保存到它对应的PackageSetting对象中，如果当前package使用的是共享uid，则它获得权限与它共享的linux用户的资源权限相同。对于有自己独立uid的应用，它会首先获得一组默认的用户权限，这是所有应用都具有的基本权限。然后会一一验证应用申请的权限是否合法，如果合法，则会为其增加该项权限。

现在，应用已经被安装、分配uid和分配权限，接下来需要将这些应用的信息保存下来以备下次使用。保存的位置就是`/data/system/packages.xml`。注意，在保存uid时，userId和sharedUserId只能有一个。

Android系统就是通过用户id和用户组id来限制应用的资源访问权限，防止破坏其它应用的数据。分配的uid和gid会在创建应用进程时使用，是该进程在特定的用户组下运行。







