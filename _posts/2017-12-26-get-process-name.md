---
layout: post
title: "获取进程名的几种方式"
description: ""
header_image: /assets/img/2017-12-26-02.png
keywords: "进程名"
tags: [Android]
---
{% include JB/setup %}
![img](/assets/img/2017-12-26-02.png)

## 背景

获取Android指定pid的进程名该怎么操作？如果你曾经遇到过类似需求一定知道有多种办法，那么这些获取进程名的方法速度上有什么差异么？用那种方式最合适呢？

本文存粹为了对比集中获取进程的方式，做一个简单分析。

## 0x1 ActivityManager

第一种通过系统的`ActivityManager`来获取当前运行的所有进程，通过遍历比较pid来找出目标进程名。

```java
private String processNameFromAMS(int pid) {
    String processName = "";
    ActivityManager manager = (ActivityManager) this.getSystemService(Context.ACTIVITY_SERVICE);
    for (ActivityManager.RunningAppProcessInfo processInfo : manager.getRunningAppProcesses()) {
        if (processInfo.pid == pid) {
            processName = processInfo.processName;
            break;
        }
    }
    return processName;
}
```

不过系统自5.x之后开始限制`getRunningAppProcesses`，只能返回当前app的相关进程。
[ActivityManager.html#getRunningAppProcesses()](https://developer.android.com/reference/android/app/ActivityManager.html#getRunningAppProcesses())

除了`ActivityManager`之外，类似的也可以去找其他的Manager

## 0x2 /proc/pid/cmdline

另外一种比较通用的方式是读取Linux的系统文件，因为Android本身是基于Linux的，而在Linux中一切都是文件，所以通过进程相关的文件就可以读取到指定进程名。

```java
private String processNameFromLinuxFile(int pid) {
    String processName = "";
    BufferedReader cmdlineReader = null;
    try {
        cmdlineReader = new BufferedReader(new InputStreamReader(
                new FileInputStream("/proc/" + pid + "/cmdline"),
                "iso-8859-1"));
        int c;
        StringBuilder builder = new StringBuilder();
        while ((c = cmdlineReader.read()) > 0) {
            builder.append((char) c);
        }
        builder.trimToSize();
        processName = builder.toString();
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        if (cmdlineReader != null) {
            try {
                cmdlineReader.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    return processName;
}
```

## 0x3 ActivityThread

个人认为通过Linux文件读取是最好的办法，稳定且不受版本限制，但是读取一个文件还是有开销的，即使进行线程名缓存复用，还是免不了首次开销。笔者在mi 4上测试了一下大概2毫秒左右。

其实除了通过Linux文件还可以通过`ActivityThread`的成员变量来获取，不过需要借助反射技术.

`Application`/`mLoadedApk`/`mActivityThread`#`getProcessName`

```java
private String processNameFromReflect(Application app) {
    String processName = null;
    try {
        Field loadedApkField = app.getClass().getField("mLoadedApk");
        loadedApkField.setAccessible(true);
        Object loadedApk = loadedApkField.get(app);

        Field activityThreadField = loadedApk.getClass().getDeclaredField("mActivityThread");
        activityThreadField.setAccessible(true);
        Object activityThread = activityThreadField.get(loadedApk);

        Method getProcessName = activityThread.getClass().getDeclaredMethod("getProcessName", null);
        processName = (String) getProcessName.invoke(activityThread, null);
    } catch (Exception e) {
        e.printStackTrace();
    }
    return processName;
}
```

好奇的你一定想知道为什么`ActivityThread`可以变相获取到进程名？

## 0x4 ActivityThread 跟踪

这个实际上有点意思，app启动后开辟了一个进程，在经过一系列的套路之后，会把当前的进程名字回调给Java层保存。

android/app/Application.java

```java
public class Application extends ContextWrapper implements ComponentCallbacks2 {
    private ArrayList<ComponentCallbacks> mComponentCallbacks =
            new ArrayList<ComponentCallbacks>();
    private ArrayList<ActivityLifecycleCallbacks> mActivityLifecycleCallbacks =
            new ArrayList<ActivityLifecycleCallbacks>();
    private ArrayList<OnProvideAssistDataListener> mAssistCallbacks = null;

    /** @hide */
    public LoadedApk mLoadedApk;
    // ...
```

android/app/LoadedApk.java

```java
public final class LoadedApk {

    private static final String TAG = "LoadedApk";

    private final ActivityThread mActivityThread;
    final String mPackageName;
    private ApplicationInfo mApplicationInfo;
    // ...
}
```

android/app/ActivityThread.java

```java
AppBindData mBoundApplication;

public String getProcessName() {
    return mBoundApplication.processName;
}

```

所以我们反射取得就是这个`mBoundApplication.processName`的值，接下可以再翻一下源代码，看看这个赋值是在`handleBindApplication`中：

```java
private void handleBindApplication(AppBindData data) {
    // Register the UI Thread as a sensitive thread to the runtime.
    VMRuntime.registerSensitiveThread();
    if (data.trackAllocation) {
        DdmVmInternal.enableRecentAllocations(true);
    }

    // Note when this process has started.
    Process.setStartTimes(SystemClock.elapsedRealtime(), SystemClock.uptimeMillis());

    mBoundApplication = data;
    mConfiguration = new Configuration(data.config);
    mCompatConfiguration = new Configuration(data.config);

    mProfiler = new Profiler();
    if (data.initProfilerInfo != null) {
        mProfiler.profileFile = data.initProfilerInfo.profileFile;
        mProfiler.profileFd = data.initProfilerInfo.profileFd;
        mProfiler.samplingInterval = data.initProfilerInfo.samplingInterval;
        mProfiler.autoStopProfiler = data.initProfilerInfo.autoStopProfiler;
    }

    // send up app name; do this *before* waiting for debugger
    Process.setArgV0(data.processName);
    android.ddm.DdmHandleAppName.setAppName(data.processName,
                                            UserHandle.myUserId());
    // ...
}
```

调用是在我们的Handler中, 同时我们也可以知道，Android内的组件基本都是通过这个`H`内部类来实现消息分发，比如Activity的创建，销毁，Window的展示，Application的绑定:

android/app/ActivityThread.java

```java
final ApplicationThread mAppThread = new ApplicationThread();
final Looper mLooper = Looper.myLooper();
final H mH = new H();
final ArrayMap<IBinder, ActivityClientRecord> mActivities = new ArrayMap<>();
// List of new activities (via ActivityRecord.nextIdle) that should
// be reported when next we idle.
ActivityClientRecord mNewActivities = null;
```

android/app/ActivityThread.java

```java
private class H extends Handler {
    public static final int LAUNCH_ACTIVITY         = 100;
    // ...
    public static final int DESTROY_ACTIVITY        = 109;
    public static final int BIND_APPLICATION        = 110;
    public static final int EXIT_APPLICATION        = 111;
    // ...
    public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
            // ...
            case DESTROY_ACTIVITY:
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityDestroy");
                handleDestroyActivity((IBinder)msg.obj, msg.arg1 != 0,
                        msg.arg2, false);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                break;
            case BIND_APPLICATION:
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                AppBindData data = (AppBindData)msg.obj;
                handleBindApplication(data);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                break;
            case EXIT_APPLICATION:
                if (mInitialApplication != null) {
                    mInitialApplication.onTerminate();
                }
                Looper.myLooper().quit();
                break;
            // ...
        }
    }
}
```

接下来就是看看什么地方发送了Handler的`BIND_APPLICATION`消，相关代码仍然在`android/app/ActivityThread.java`中。
有一个内部类叫做`ApplicationThread`:

android/app/ActivityThread.java

```java
private class ApplicationThread extends ApplicationThreadNative {

    public final void bindApplication(String processName, ApplicationInfo appInfo,
            List<ProviderInfo> providers, ComponentName instrumentationName,
            ProfilerInfo profilerInfo, Bundle instrumentationArgs,
            IInstrumentationWatcher instrumentationWatcher,
            IUiAutomationConnection instrumentationUiConnection, int debugMode,
            boolean enableBinderTracking, boolean trackAllocation,
            boolean isRestrictedBackupMode, boolean persistent, Configuration config,
            CompatibilityInfo compatInfo, Map<String, IBinder> services, Bundle coreSettings) {

        if (services != null) {
            // Setup the service cache in the ServiceManager
            ServiceManager.initServiceCache(services);
        }

        setCoreSettings(coreSettings);

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
        sendMessage(H.BIND_APPLICATION, data);
    }

    public final void scheduleExit() {
        sendMessage(H.EXIT_APPLICATION, null);
    }
}
```

`bindApplication`是定义在`IApplicationThread`接口中一个方法，由`ApplicationThread`实现，他继承自`ApplicationThreadNative`,`ApplicationThreadNative`本身是一个`Binder`, 会在`onTransact`中根据code的取值来判断是否是`BIND_APPLICATION_TRANSACTION`。

```java
@Override
public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException {
    switch (code) {
    // ...
    case BIND_APPLICATION_TRANSACTION:
    {
        data.enforceInterface(IApplicationThread.descriptor);
        String packageName = data.readString();
        ApplicationInfo info =
            ApplicationInfo.CREATOR.createFromParcel(data);
        List<ProviderInfo> providers =
            data.createTypedArrayList(ProviderInfo.CREATOR);
        ComponentName testName = (data.readInt() != 0)
            ? new ComponentName(data) : null;
        ProfilerInfo profilerInfo = data.readInt() != 0
                ? ProfilerInfo.CREATOR.createFromParcel(data) : null;
        Bundle testArgs = data.readBundle();
        IBinder binder = data.readStrongBinder();
        IInstrumentationWatcher testWatcher = IInstrumentationWatcher.Stub.asInterface(binder);
        binder = data.readStrongBinder();
        IUiAutomationConnection uiAutomationConnection =
                IUiAutomationConnection.Stub.asInterface(binder);
        int testMode = data.readInt();
        boolean enableBinderTracking = data.readInt() != 0;
        boolean trackAllocation = data.readInt() != 0;
        boolean restrictedBackupMode = (data.readInt() != 0);
        boolean persistent = (data.readInt() != 0);
        Configuration config = Configuration.CREATOR.createFromParcel(data);
        CompatibilityInfo compatInfo = CompatibilityInfo.CREATOR.createFromParcel(data);
        HashMap<String, IBinder> services = data.readHashMap(null);
        Bundle coreSettings = data.readBundle();
        bindApplication(packageName, info, providers, testName, profilerInfo, testArgs,
                testWatcher, uiAutomationConnection, testMode, enableBinderTracking,
                trackAllocation, restrictedBackupMode, persistent, config, compatInfo, services,
                coreSettings);
        return true;
    }
    // ...
    }
}
```

可以看到一个进程名的传递最后涉及到了Binder，最终回调到Java层的`ActivityThread`中。
