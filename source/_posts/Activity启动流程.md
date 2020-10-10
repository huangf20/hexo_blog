---
title: Activity启动流程
date: 2020-10-09 15:55:50
summary: 日常开发中似乎只要知道Activity的生命周期就够用了，但是深入了解的话还是要从源码中探索
categories: Android
tags:
  - Android
---
{% asset_img 1.jpeg Activity启动流程图 %}
本篇主要讲的是从一个App启动，到Activity执行onCreate()的流程，
## 一切从main()方法开始  
Android中，一个应用程序的开始可以说是从ActivityThread.java中的main()方法中开始的，也就是java程序方法的入口：
{% codeblock lang:java %}
public static void main(String[] args) {
        ...

        Looper.prepareMainLooper();
        //初始化Looper
        ...
        ActivityThread thread = new ActivityThread();
        //实例化一个ActivityThread
        ...
        thread.attach(false, startSeq);
        //这个方法最后就是为了发送出创建Application的消息
        ...
        Looper.loop();
        //主线程进入无限循环状态，等待接收消息
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
{% endcodeblock %}
main()方法中主要做的事情有：  
1.初始化主线程的 **Looper** 、主 **Handler** 。并使主线程进入等待接收Message消息的无限循环状态。  
2.调用 **attach()** 方法，主要就是为了发送出初始化Application的消息。  

## 创建Application的消息如何发送？
ActivityThread的 **attach()** 方法最终的目的是发送出一条创建Application的消息—— **H.BIND_APPLICATION** ，到主线程的主 **Handler** 中。  
attach()方法中关键代码：  
{% codeblock lang:java %}
private void attach(boolean system, long startSeq) {
            ...
            final IActivityManager mgr = ActivityManager.getService();
            //获得IActivityManager实例
            try {
                mgr.attachApplication(mAppThread, startSeq);
                //又是一个方法
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
            ...
    }
{% endcodeblock %}

查看**ActivityManager**中的 **getService()** 方法：
{% codeblock lang:java %}
public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    //获得IBinder实例
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
{% endcodeblock %}  
可以看到, **getService()** 返回的是一个静态常量，它是一个单例。在其中终于获得了前面一直在用的IBinder实例。
获取 **IBinder** 的目的就是为了通过这个 **IBinder** 和 **ActivityManager** 进行通讯，进而ActivityManager会调度发送 **H.BIND_APPLICATION** 即初始化Application的Message消息。  

再来看看attachApplication(mAppThread)方法
{% codeblock lang:java %}
@Override
    public final void attachApplication(IApplicationThread thread, long startSeq) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid, callingUid, startSeq);
            Binder.restoreCallingIdentity(origId);
        }
    }
{% endcodeblock %}  
attachApplicationLocked（）方法是内部的一个方法，将thread传回**ActivityManager**。  

### ApplicationThread mAppThread
在ActivityThread.java 中可以看到这个变量：
{% codeblock lang:java %}
final ApplicationThread mAppThread = new ApplicationThread();
{% endcodeblock %}  
ApplicationThread.java:
{% codeblock lang:java %}
private class ApplicationThread extends IApplicationThread.Stub {
    ...
    }
{% endcodeblock %} 
attach()方法中出现的两个对象。ApplicationThread作为IApplicationThread的一个子类，承担了最后发送Activity生命周期、及其它一些消息的任务。

### ActivityManagerService调度发送初始化消息
ActivityManagerService中有一这样的方法：
{% codeblock lang:java %}
private final boolean attachApplicationLocked(IApplicationThread thread,int pid, int callingUid, long startSeq) {
     
    if (app.instr != null) {
                thread.bindApplication(processName, appInfo, providers,
                        app.instr.mClass,
                        profilerInfo, app.instr.mArguments,
                        app.instr.mWatcher,
                        app.instr.mUiAutomationConnection, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, isAutofillCompatEnabled);
            } else {
                thread.bindApplication(processName, appInfo, providers, null, profilerInfo,
                        null, null, null, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, isAutofillCompatEnabled);
            }
 }
{% endcodeblock %} 
ApplicationThread以IApplicationThread的身份到了ActivityManagerService中，经过一系列的操作，最终被调用了自己的bindApplication()方法，发出初始化Applicationd的消息。  
{% codeblock lang:java %}
public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableBinderTracking, boolean trackAllocation,
                boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
                String buildSerial, boolean autofillCompatibilityEnabled) {

            AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.instrumentationUiAutomationConnection = instrumentationUiConnection;
            data.debugMode = debugMode;
            data.enableBinderTracking = enableBinderTracking;
            data.trackAllocation = trackAllocation;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfilerInfo = profilerInfo;
            data.buildSerial = buildSerial;
            data.autofillCompatibilityEnabled = autofillCompatibilityEnabled;
    
                
            sendMessage(H.BIND_APPLICATION, data);
        }
{% endcodeblock %} 
最后发了一条**H.BIND_APPLICATION**消息，接着程序开始了。  

## 收到初始化消息之后的世界
收到消息后都发生了些什么。上面我们已经找到初始化Applicaitond的消息是在哪发送的了。现在，需要看一看收到消息后都发生了些什么。现在找到第一个消息：**H.BIND_APPLICATION**。一旦接收到这个消息就开始创建Application了。这个过程是在handleBindApplication()中完成的。
{% codeblock lang:java %}
private void handleBindApplication(AppBindData data) {
    ...
    try {
                final ClassLoader cl = instrContext.getClassLoader();
                mInstrumentation = (Instrumentation)
                    cl.loadClass(data.instrumentationName.getClassName()).newInstance();
        
                //通过反射创建Instrumentation实例
            } catch (Exception e) {
                throw new RuntimeException(
                    "Unable to instantiate instrumentation "
                    + data.instrumentationName + ": " + e.toString(), e);
    ...       
    Application app = data.info.makeApplication(data.restrictedBackupMode, null); 
    //通过LoadedApp命令创建Application实例
    ...
    mInstrumentation.callApplicationOnCreate(app);
    //让仪器调用Application的onCreate()方法
    ...
	}
}
{% endcodeblock %}

### Instrumentation类
看看文档是怎么说的：  
用于实现应用程序检测代码的基类。运行时打开检测后，将为您实例化该类在任何应用程序代码之前，允许您监视所有系统与应用程序的交互。仪器通过一个AndroidManifest.xml的&lt；instrumentation&gt；标记。  
{% asset_img 2.png Instrumentation类 %}  
打开这个类你可以发现，最终Apllication的创建，Activity的创建，以及生命周期都会经过这个对象去执行。简单点说，就是把这些操作包装了一层。通过操作Instrumentation进而实现上述的功能。  
同时我们可看到，Instrumentation是通过反射来来实现的，而反射的ClassName就是通过Binder传递过来的。  

再来看一下**callApplicationOnCreate（）**方法  
{% codeblock lang:java %}
public void callApplicationOnCreate(Application app) {
        app.onCreate();
    }
{% endcodeblock %}
调用了一下Application的onCreate()方法,这就是为什么它能够起到监控的作用。  

再来看一下**makeApplication（）**方法
{% codeblock lang:java %}
public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
    ...
    String appClass = mApplicationInfo.className;
    //得到className以便反射
    //
    ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
    //通过Instrumentation创建Application
    ...
           
}
{% endcodeblock %}
在取得Application的实际类名之后，最终的创建工作还是交由Instrumentation去完成。  

### 目光移回Instrumentation中的newAppliction()
{% codeblock lang:java %}
public Application newApplication(ClassLoader cl, String className, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        Application app = getFactory(context.getPackageName())
                .instantiateApplication(cl, className);
        //反射创建
        app.attach(context);
        //Application被创建后第一个调用的方法,绑定Context。
        return app;
    }
{% endcodeblock %}



## LaunchActivity
当Application初始化完成后，系统会更具Manifests中的配置的启动Activity发送一个Intent去启动相应的Activity，处理是在ActivityThread中的handleLaunchActivity()中进行的。
{% codeblock lang:java %}
public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {

    final Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            reportSizeConfigurations(r);
            if (!r.activity.mFinished && pendingActions != null) {
                pendingActions.setOldState(r.state);
                pendingActions.setRestoreInstanceState(true);
                pendingActions.setCallOnPostCreate(true);
            }
        } else {
            // If there was an error, for any reason, tell the activity manager to stop us.
            try {
                ActivityManager.getService()
                        .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                                Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }

        return a;
}
{% endcodeblock %}

查看一下 **performLaunchActivity( )** 方法
{% codeblock lang:java %}
 private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
     ...
     java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
     ////通过仪表来创建Activity
     ...
     Application app = r.packageInfo.makeApplication(false, mInstrumentation);
     //获取Application
     ...
     activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);
     //绑定Context
     ...
     if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
	//是否可持久化,通过Instrumentation来创建Activity
 }
 {% endcodeblock %}
 
 到这里，从Application创建开始，到第一个Activity onCreate()结束的整个流程就结束了。