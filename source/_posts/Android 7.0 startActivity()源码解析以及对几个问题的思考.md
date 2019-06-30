---
title: Android 7.0 startActivity()源码解析以及对几个问题的思考
tags: Android源码解析
categories: Android
abbrlink: b3e682b8
date: 2017-12-05 23:00:23
---

#### 一、本文需要解决的问题
本文并不是非常详细地解释startActivity()源码每行代码的具体作用（实际上也根本做不到），所以我省略了很多代码，只保留了最核心的代码。我研究这段源码的目的是为了解决以下几个我在开发应用的过程中所思考的问题：
1. 是通过何种方式生成一个新的Activity类的，是通过java反射生成的吗？
2. Activity的生命周期回调方法是通过哪个类调用的，在什么时候调用的？
3. 界面的绘制是在执行Activity#onResume()之后还是之前？
4. 在之前的学习中，我了解到应用程序的真正入口是ActivityThread类，那么ActivityThread#main()方法是在哪里调用的？

<!--more-->

#### 二、相关解析（基于Android 7.1源码）
```java
// 这里解析的是在已有进程中启动一个新Activity的情况
Intent intent = new Intent(this, SubActivity.class);
intent.startActivity();
```
（1）Activity本地调用：
Activity#startActivity() 
&nbsp; --> Activity#startActivityForResult() 
```java
@Override
public void startActivityForResult(String who, Intent intent, int requestCode, @Nullable Bundle options) {
    // 省略代码
    Instrumentation.ActivityResult ar = 
                mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, who,
                intent, requestCode, options);
    // 省略代码
}
```

（2）Instrumentation#execStartActivity
Instrumentation类相当于一个管家，它的职责是管理各个应用程序和系统的交互，Instrumentation将在任何应用程序运行前初始化，每个进程只会存在一个Instrumentation对象，且每个Activity都有此对象的实际引用，可以通过它监测系统与应用程序之间的所有交互。
```java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
    // 省略代码
    int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
    checkStartActivityResult(result, intent);
    // 省略代码
}
```

（3）ActivityManagerNative，ActivityManagerService，ActivityManagerProxy类之间的关系
ActivityManagerNative#getDefault() 
&nbsp; --> ActivityManagerProxy#startActivity()
```java
public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IActivityManager.descriptor);
    data.writeStrongBinder(caller != null ? caller.asBinder() : null);
    data.writeString(callingPackage);
    intent.writeToParcel(data, 0);
    data.writeString(resolvedType);
    data.writeStrongBinder(resultTo);
    data.writeString(resultWho);
    data.writeInt(requestCode);
    data.writeInt(startFlags);
    if (profilerInfo != null) {
        data.writeInt(1);
        profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
    } else {
        data.writeInt(0);
    }
    if (options != null) {
        data.writeInt(1);
        options.writeToParcel(data, 0);
    } else {
        data.writeInt(0);
    }
    mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
    reply.readException();
    int result = reply.readInt();
    reply.recycle();
    data.recycle();
    return result;
}
```
如果学过AIDL的话，对上面这段代码会比较熟悉，其实上面是发起了一次跨进程调用。这里涉及到一种设计模式叫作代理模式，这里不详细介绍这种设计模式，简单总结一下：
- ActivityManagerProxy相当于Proxy
- ActivityManagerNative就相当于Stub
- ActivityManagerService是ActivityManagerNative的具体实现，换句话说，就是AMS才是服务端的具体实现！

（4）ActivityMangerService
ActivityMangerService#startActivity()
&nbsp; --> ActivityMangerService#startActivityAsUser()
```java
@Override
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
    enforceNotIsolatedCaller("startActivity");
    userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
                userId, false, ALLOW_FULL_ONLY, "startActivity", null);
    // TODO: Switch to user app stacks here.
    return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, bOptions, false, userId, null, null);
}
```

（5）ActivityStarter
ActivityStarter类的注释：
```java
/**
 * Controller for interpreting how and then launching activities.
 *
 * This class collects all the logic for determining how an intent and flags should be turned into
 * an activity and associated task and stack.
 */
```
简单来说，ActivityStarter类主要负责处理Activity的Intent和Flags, 还有关联相关的Stack和TaskRecord
ActivityStarter#startActivityMayWait()
&nbsp; --> ActivityStarter#startActivityLocked()
&nbsp; &nbsp; --> ActivityStarter#startActivityUnchecked()

其中：
**startActivityMayWait()：**获取Activity的启动信息，包括ResolveInfo和ActivityInfo，以及获取CallingPid和CallingUid；
**startActivityLocked()：**创建一个ActivityRecord；
**startActivityUnchecked()：**设置TaskRecord, 完成后执行ActivityStackSupervisor类的resumeFocusedStackTopActivityLocked方法

（6）ActivityStackSupervisor和ActivityStack
ActivityStackSupervisor#resumeFocusedStackTopActivityLocked()
&nbsp; --> ActivityStackSupervisor#resumeTopActivityUncheckedLocked()
&nbsp; &nbsp; --> ActivityStack#resumeTopActivityInnerLocked()
&nbsp; &nbsp; &nbsp; --> ActivityStackSupervisor#startSpecificActivityLocked()
```java
void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
    // Is this activity's application already running?
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

    r.task.stack.setLaunchTime(r);

    if (app != null && app.thread != null) {
         try {
             if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
                 // Don't add this if it is a platform component that is marked
                 // to run in multiple processes, because this is actually
                 // part of the framework so doesn't make sense to track as a
                 // separate apk in the process.
                 app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                            mService.mProcessStats);
             }
             // ！！！
             realStartActivityLocked(r, app, andResume, checkConfig);
             return;
        } catch (RemoteException e) {
             Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
        }

        // If a dead object exception was thrown -- fall through to
        // restart the application.
    }

    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
}
```
这里会判断进程是否存在，由于我们是在原有进程中启动一个新的activity，所以会调用 realStartActivityLocked()方法。
ActivityStackSupervisor#realStartActivityLocked()
```java
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException  {
    // 省略代码
    app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(task.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);
}  
```

（7）关于IApplicationThread，ApplicationThreadProxy，ApplicationThreadNative，ApplicationThread
上述代码中，app.thread为IApplicationThread类型，继承了IInterface，我们查看IApplicationThread的直接实现ApplicationThreadNative
ApplicationThreadNative#scheduleLaunchActivity()
```java
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
            ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
            CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
            int procState, Bundle state, PersistableBundle persistentState,
            List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
            boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) throws RemoteException {
    Parcel data = Parcel.obtain();
    data.writeInterfaceToken(IApplicationThread.descriptor);
    intent.writeToParcel(data, 0);
    data.writeStrongBinder(token);
    data.writeInt(ident);
    info.writeToParcel(data, 0);
    curConfig.writeToParcel(data, 0);
    if (overrideConfig != null) {
        data.writeInt(1);
        overrideConfig.writeToParcel(data, 0);
    } else {
        data.writeInt(0);
    }
    compatInfo.writeToParcel(data, 0);
    data.writeString(referrer);
    data.writeStrongBinder(voiceInteractor != null ? voiceInteractor.asBinder() : null);
    data.writeInt(procState);
    data.writeBundle(state);
    data.writePersistableBundle(persistentState);
    data.writeTypedList(pendingResults);
    data.writeTypedList(pendingNewIntents);
    data.writeInt(notResumed ? 1 : 0);
    data.writeInt(isForward ? 1 : 0);
    if (profilerInfo != null) {
        data.writeInt(1);
        profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
    } else {
        data.writeInt(0);
    }
    mRemote.transact(SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
    data.recycle();
}
```
同样的，这里也发起了一次跨进程调用。
- ApplicationThreadProxy相当于Proxy
- ApplicationThreadNative相当于Stub
- ApplicationThread相当于服务器端，代码真正的实现者！

（8）ActivityThread.ApplicationThread类
ActivityThread.ApplicationThread#scheduleLaunchActivity
```java
@Override
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

    updateProcessState(procState, false);

    ActivityClientRecord r = new ActivityClientRecord();

    r.token = token;
    r.ident = ident;
    r.intent = intent;
    r.referrer = referrer;
    r.voiceInteractor = voiceInteractor;
    r.activityInfo = info;
    r.compatInfo = compatInfo;
    r.state = state;
    r.persistentState = persistentState;

    r.pendingResults = pendingResults;
    r.pendingIntents = pendingNewIntents;

    r.startsNotResumed = notResumed;
    r.isForward = isForward;

    r.profilerInfo = profilerInfo;

    r.overrideConfig = overrideConfig;
    updatePendingConfiguration(curConfig);

    sendMessage(H.LAUNCH_ACTIVITY, r);
}
```
这里会调用sendMessage，最后调用到mH.sendMessage(msg);
mH为H类的一个实例，H就是Handler的一个子类，发送消息之后，我们来查看H类的handleMessage()方法：
源码里面就是根据msg.what来执行对应的操作：
```java
case LAUNCH_ACTIVITY: {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
    r.packageInfo = getPackageInfoNoCheck(
    r.activityInfo.applicationInfo, r.compatInfo);
    handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
} 
```

（9） ActivityThread#handleLaunchActivity()
```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;

    if (r.profilerInfo != null) {
        mProfiler.setProfiler(r.profilerInfo);
        mProfiler.startProfiling();
    }

    // Make sure we are running with the most recent config.
    handleConfigurationChanged(null, null);

    if (localLOGV) Slog.v(TAG, "Handling launch of " + r);

    // Initialize before creating the activity
    WindowManagerGlobal.initialize();
        
    // ！！！
    Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        reportSizeConfigurations(r);
        Bundle oldState = r.state;
        handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);

        if (!r.activity.mFinished && r.startsNotResumed) {
            // The activity manager actually wants this one to start out paused, because it
            // needs to be visible but isn't in the foreground. We accomplish this by going
            // through the normal startup (because activities expect to go through onResume()
            // the first time they run, before their window is displayed), and then pausing it.
            // However, in this case we do -not- need to do the full pause cycle (of freezing
            // and such) because the activity manager assumes it can just retain the current
            // state it has.
            performPauseActivityIfNeeded(r, reason);

            // We need to keep around the original state, in case we need to be created again.
            // But we only do this for pre-Honeycomb apps, which always save their state when
            // pausing, so we can not have them save their state when restarting from a paused
            // state. For HC and later, we want to (and can) let the state be saved as the
            // normal part of stopping the activity.
            if (r.isPreHoneycomb()) {
                r.state = oldState;
            }
        }
     } else {
         // If there was an error, for any reason, tell the activity manager to stop us.
         try {
             ActivityManagerNative.getDefault()
                    .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                            Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
         } catch (RemoteException ex) {
             throw ex.rethrowFromSystemServer();
         }
     }
}
```
这里会调用performLaunchActivity()方法。

（10）ActivityThread#performLaunchActivity()
```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // System.out.println("##### [" + System.currentTimeMillis() + "] ActivityThread.performLaunchActivity(" + r + ")");
    ActivityInfo aInfo = r.activityInfo;
    if (r.packageInfo == null) {
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
    }

    ComponentName component = r.intent.getComponent();
    if (component == null) {
        component = r.intent.resolveActivity(mInitialApplication.getPackageManager());
        r.intent.setComponent(component);
    }

    if (r.activityInfo.targetActivity != null) {
        component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
    }

    Activity activity = null;
    try {
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
        }
    }

    try {
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);

        if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
        if (localLOGV) Slog.v(
                    TAG, r + ": app=" + app
                    + ", appName=" + app.getPackageName()
                    + ", pkg=" + r.packageInfo.getPackageName()
                    + ", comp=" + r.intent.getComponent().toShortString()
                    + ", dir=" + r.packageInfo.getAppDir());

        if (activity != null) {
            Context appContext = createBaseContextForActivity(r, activity);
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
            Configuration config = new Configuration(mCompatConfiguration);
            if (r.overrideConfig != null) {
                config.updateFrom(r.overrideConfig);
            }
            if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
            Window window = null;
            if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                window = r.mPendingRemoveWindow;
                r.mPendingRemoveWindow = null;
                r.mPendingRemoveWindowManager = null;
            }
            activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window);

            if (customIntent != null) {
                activity.mIntent = customIntent;
            }
            r.lastNonConfigurationInstances = null;
            activity.mStartedActivity = false;
            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                activity.setTheme(theme);
            }

            activity.mCalled = false;
            if (r.isPersistable()) {
                // 11.1
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                // 11.1
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            if (!activity.mCalled) {
                throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
            }
            r.activity = activity;
            r.stopped = true;
            if (!r.activity.mFinished) {
                // 11.2
                activity.performStart();
                r.stopped = false;
            }
            if (!r.activity.mFinished) {
                if (r.isPersistable()) {
                    if (r.state != null || r.persistentState != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
                    }
                } else if (r.state != null) {
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                }
            }
            if (!r.activity.mFinished) {
                activity.mCalled = false;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnPostCreate(activity, r.state,
                                r.persistentState);
                } else {
                    mInstrumentation.callActivityOnPostCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPostCreate()");
                }
            }
        }
        r.paused = true;

        mActivities.put(r.token, r);

    } catch (SuperNotCalledException e) {
        throw e;
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
           throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
       }
    }
    return activity;
}
```
这里会收集要启动的Activity的相关信息，主要是package和component信息，然后通过ClassLoader将要启动的Activity类加载出来。

这里就解决了我的第一个问题：**新的Activity类是通过类加载器方式即通过反射的方式生成的**，我们可以看一下mInstrumentation.newActivity()方法：
```java
public Activity newActivity(ClassLoader cl, String className, Intent intent)
            throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    return (Activity)cl.loadClass(className).newInstance();
}
```
最后调用mInstrumentation.callActivityOnCreate()

（11）
11.1 Instrumentation#callActivityOnCreate()
```java
public void callActivityOnCreate(Activity activity, Bundle icicle,
            PersistableBundle persistentState) {
    prePerformCreate(activity);
    activity.performCreate(icicle, persistentState);
    postPerformCreate(activity);
}
```
Activity#performCreate()
```java
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
    restoreHasCurrentPermissionRequest(icicle);
    onCreate(icicle, persistentState);
    mActivityTransitionState.readState(icicle);
    performCreateCommon();
}
```
这里就会显式调用Activity的生命周期方法onCreate()！

11.2 Activity#performStart()
```java
final void performStart() {
    // 省略代码
    // ！！！
    mInstrumentation.callActivityOnStart(this);
    // 省略代码
}
```
在mInstrumentation.callActivityOnStart(this)方法里面就会显式调用Activtiy的onStart()方法！

到这里我们也可以基本解决第二个问题：**Activity的生命周期方法是通过Instrumentation类调用callActivityOnXXX方法最终调用Activity的onCreate等方法，调用时机为ActivityThread#performLaunchActivitiy()方法中。**

那么还有一个问题，我们知道启动一个Activity，所经历的生命周期为onCreate() --> onStart() --> onResume()
那么onResume()方法在哪里调用的呢？
我们回到前面的ActivityThread#handleLaunchActivity()：
```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    // 省略代码
    Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        reportSizeConfigurations(r);
        Bundle oldState = r.state;
        // ！！！
        handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
    // 省略代码
}
```
在我们调用performLaunchActivity之后返回新生成的Activity实例之后，接下来就会调用handleResumeActivity()方法
ActivityThread#handleResumeActivity()
```java
final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
    ActivityClientRecord r = mActivities.get(token);
    if (!checkAndUpdateLifecycleSeq(seq, r, "resumeActivity")) {
        return;
    }
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;
    // TODO Push resumeArgs into the activity for consideration
    // ！！！
    r = performResumeActivity(token, clearHide, reason);
    if (r != null) {
        final Activity a = r.activity;
        if (localLOGV) Slog.v(TAG, "Resume " + r + " started activity: " + 
                a.mStartedActivity + ", hideForNow: " + r.hideForNow + ", finished: " + a.mFinished);
        final int forwardBit = isForward ? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;
        // If the window hasn't yet been added to the window manager,
        // and this guy didn't finish itself or start another activity,
        // then go ahead and add the window.
        boolean willBeVisible = !a.mStartedActivity;
        if (!willBeVisible) {
            try {
                willBeVisible = ActivityManagerNative.getDefault().willActivityBeVisible(a.getActivityToken());
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (r.mPreserveWindow) {
                a.mWindowAdded = true;
                r.mPreserveWindow = false;
                // Normally the ViewRoot sets up callbacks with the Activity
                // in addView->ViewRootImpl#setView. If we are instead reusing
                // the decor view we have to notify the view root that the
                // callbacks may have changed.
                ViewRootImpl impl = decor.getViewRootImpl();
                if (impl != null) {
                    impl.notifyChildRebuilt();
                }
            }
            if (a.mVisibleFromClient && !a.mWindowAdded) {
                a.mWindowAdded = true;
                wm.addView(decor, l);
            }
            // If the window has already been added, but during resume
            // we started another activity, then don't yet make the
            // window visible.
        } else if (!willBeVisible) {
            if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");
            r.hideForNow = true;
        }
        // Get rid of anything left hanging around.
        cleanUpPendingRemoveWindows(r, false /* force */ );
        // The window is now visible if it has been added, we are not
        // simply finishing, and we are not starting another activity.
        if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
            if (r.newConfig != null) {
                performConfigurationChangedForActivity(r, r.newConfig, REPORT_TO_ACTIVITY);
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Resuming activity " + r.activityInfo.name + 
                                " with newConfig " + r.activity.mCurrentConfig);
                r.newConfig = null;
            }
            if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward=" + isForward);
            WindowManager.LayoutParams l = r.window.getAttributes();
            if ((l.softInputMode & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION) != forwardBit) {
                l.softInputMode = (l.softInputMode & 
                              (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)) 
                              | forwardBit;
                if (r.activity.mVisibleFromClient) {
                    ViewManager wm = a.getWindowManager();
                    View decor = r.window.getDecorView();
                    wm.updateViewLayout(decor, l);
                }
            }
            r.activity.mVisibleFromServer = true;
            mNumVisibleActivities++;
            if (r.activity.mVisibleFromClient) {
                r.activity.makeVisible();
            }
        }
        if (!r.onlyLocalRequest) {
            r.nextIdle = mNewActivities;
            mNewActivities = r;
            if (localLOGV) Slog.v(TAG, "Scheduling idle handler for " + r);
            Looper.myQueue().addIdleHandler(new Idler());
        }
        r.onlyLocalRequest = false;
        // Tell the activity manager we have resumed.
        if (reallyResume) {
            try {
                ActivityManagerNative.getDefault().activityResumed(token);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }
    } else {
        // If an exception was thrown when trying to resume, then
        // just end this activity.
        try {
            ActivityManagerNative.getDefault().
                      finishActivity(token, Activity.RESULT_CANCELED, null, Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
        } catch (RemoteException ex)
        throw ex.rethrowFromSystemServer();
    }
}
```
这里会调用：
ActivityThread#performResumeActivity()
&nbsp; --> Activity#performResume()
&nbsp; &nbsp; --> Instrumentation#callActivityOnResume()
&nbsp; &nbsp; &nbsp; --> Activity#onResume()
另外，观察执行handleResumeActivity()之后的代码，会发现程序会开始获取DecorView，执行addView()方法，里面最终会调用到ViewRootImpl#performTraversals()，即开始绘制view界面！
这里我们就解决了第三个问题：**界面的绘制是在执行Activity#onResume()之后！**

#### 三、关于第四个问题
那么我们还差第四个问题没有解决！为什么我们这个流程都没有涉及到调用main方法呢，是因为在一开始，我们分析的情况是在已有的App进程中启动一个新的Activity，而通过我们上文的分析，我们知道ActivityThread类才是应用的主线程类，一个app应用进程有且只有一个ActivityThread类，也就是我们一直说的应用程序主线程（UI线程）。所以，在分析源码时，我们已经假设ActivityThread类已经存在实例，所以不会再调用main方法。

那么什么情况下会调用到main方法呢，其实我们每次在手机的桌面点击一个应用图标打开应用时，其实是通由Launcher启动起来的。

（1）关于Launcher
Launcher本身也是一个应用程序，其它的应用程序安装后，就会Launcher的界面上出现一个相应的图标，点击这个图标时，Launcher就会对应的应用程序启动起来。Launcher其实也是Activity的一个子类。
```java
public final class Launcher extends Activity  
        implements View.OnClickListener, OnLongClickListener, LauncherModel.Callbacks, AllAppsView.Watcher { 
    ...
}
```
所以本质上也是调用了startActivity()方法启动一个新的Activity！
根据上述的流程，一直到ActivityStackSupervisor#startSpecificActivityLocked()这里，代码的调用流程就会开始发生变化！
ActivityStackSupervisor#startSpecificActivityLocked()
```java
void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig) {
    // Is this activity's application already running?
    ProcessRecord app = mService.getProcessRecordLocked(r.processName, r.info.applicationInfo.uid, true);
    r.task.stack.setLaunchTime(r);
    if (app != null && app.thread != null) {
        try {
            if ((r.info.flags & ActivityInfo.FLAG_MULTIPROCESS) == 0 || !"android".equals(r.info.packageName)) {
                // Don't add this if it is a platform component that is marked
                // to run in multiple processes, because this is actually
                // part of the framework so doesn't make sense to track as a
                // separate apk in the process.
                app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode, mService.mProcessStats);
            }
            // ！！！
            realStartActivityLocked(r, app, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity " + r.intent.getComponent().flattenToShortString(), e);
        }
        // If a dead object exception was thrown -- fall through to
        // restart the application.
    }
    mService.startProcessLocked(r.processName, r.info.applicationInfo, 
            true, 0, "activity", r.intent.getComponent(), false, false, true);
}
```
上文说过。这里会判断进程是否存在，而这次，app为空，所以会跳出if判断，直接到下面的mService.startProcessLocked()方法，这里的mService为ActivityManagerService类的一个实例！
startProcessLocked()通过几次重载函数的调用，最终调用到这里：
```java
private final void startProcessLocked(ProcessRecord app, String hostingType, String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
    // 省略代码
    // Start the process.  It will either succeed and return a result containing
    // the PID of the new process, or else throw a RuntimeException.
    boolean isActivityProcess = (entryPoint == null);
    if (entryPoint == null) entryPoint = "android.app.ActivityThread";
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " + app.processName);
    checkTime(startTime, "startProcess: asking zygote to start proc");
    // ！！！
    Process.ProcessStartResult startResult = Process.start(entryPoint, app.processName, uid, 
                uid, gids, debugFlags, mountExternal, 
                app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet, 
                app.info.dataDir, entryPointArgs);
    // 省略代码
}
```

（2）
Process#start()
&nbsp; --> Process#startViaZygote()
```java
private static ProcessStartResult startViaZygote(final String processClass,
                                  final String niceName,
                                  final int uid, final int gid,
                                  final int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] extraArgs)
                                  throws ZygoteStartFailedEx {
        ArrayList < String > argsForZygote = new ArrayList < String > ();
    // --runtime-args, --setuid=, --setgid=,
    // and --setgroups= must go first
    argsForZygote.add("--runtime-args");
    argsForZygote.add("--setuid=" + uid);
    argsForZygote.add("--setgid=" + gid);
    if ((debugFlags & Zygote.DEBUG_ENABLE_JNI_LOGGING) != 0) {
        argsForZygote.add("--enable-jni-logging");
    }
    if ((debugFlags & Zygote.DEBUG_ENABLE_SAFEMODE) != 0) {
        argsForZygote.add("--enable-safemode");
    }
    if ((debugFlags & Zygote.DEBUG_ENABLE_DEBUGGER) != 0) {
        argsForZygote.add("--enable-debugger");
    }
    if ((debugFlags & Zygote.DEBUG_ENABLE_CHECKJNI) != 0) {
        argsForZygote.add("--enable-checkjni");
    }
    if ((debugFlags & Zygote.DEBUG_GENERATE_DEBUG_INFO) != 0) {
        argsForZygote.add("--generate-debug-info");
    }
    if ((debugFlags & Zygote.DEBUG_ALWAYS_JIT) != 0) {
        argsForZygote.add("--always-jit");
    }
    if ((debugFlags & Zygote.DEBUG_NATIVE_DEBUGGABLE) != 0) {
        argsForZygote.add("--native-debuggable");
    }
    if ((debugFlags & Zygote.DEBUG_ENABLE_ASSERT) != 0) {
        argsForZygote.add("--enable-assert");
    }
    if (mountExternal == Zygote.MOUNT_EXTERNAL_DEFAULT) {
        argsForZygote.add("--mount-external-default");
    } else if (mountExternal == Zygote.MOUNT_EXTERNAL_READ) {
        argsForZygote.add("--mount-external-read");
    } else if (mountExternal == Zygote.MOUNT_EXTERNAL_WRITE) {
        argsForZygote.add("--mount-external-write");
    }
    argsForZygote.add("--target-sdk-version=" + targetSdkVersion);
    //TODO optionally enable debuger
    //argsForZygote.add("--enable-debugger");
    // --setgroups is a comma-separated list
    if (gids != null && gids.length > 0) {
        StringBuilder sb = new StringBuilder();
        sb.append("--setgroups=");
        int sz = gids.length;
        for (int i = 0; i < sz; i++) {
            if (i != 0) {
                sb.append(',');
            }
            sb.append(gids[i]);
        }
        argsForZygote.add(sb.toString());
    }
    if (niceName != null) {
        argsForZygote.add("--nice-name=" + niceName);
    }
    if (seInfo != null) {
        argsForZygote.add("--seinfo=" + seInfo);
    }
    if (instructionSet != null) {
        argsForZygote.add("--instruction-set=" + instructionSet);
    }
    if (appDataDir != null) {
        argsForZygote.add("--app-data-dir=" + appDataDir);
    }
    argsForZygote.add(processClass);
    if (extraArgs != null) {
        for (String arg: extraArgs) {
            argsForZygote.add(arg);
        }
    }
    return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
}
```
上面方法中设置一些参数之后，调用最后的zygoteSendArgsAndGetResult()，方法的作用是向Zygote发送创建进程请求，内部与Zygote进行Socket通信。具体代码逻辑在ZygoteInit#runSelectLoop()：

（3）ZygoteInit#runSelectLoop()：
```java
private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
    ArrayList < FileDescriptor > fds = new ArrayList < FileDescriptor > ();
    ArrayList < ZygoteConnection > peers = new ArrayList < ZygoteConnection > ();
    fds.add(sServerSocket.getFileDescriptor());
    peers.add(null);
    while (true) {
        StructPollfd[] pollFds = new StructPollfd[fds.size()];
        for (int i = 0; i < pollFds.length; ++i) {
            pollFds[i] = new StructPollfd();
            pollFds[i].fd = fds.get(i);
            pollFds[i].events = (short) POLLIN;
        }
        try {
            Os.poll(pollFds, -1);
        } catch (ErrnoException ex) {
            throw new RuntimeException("poll failed", ex);
        }
        for (int i = pollFds.length - 1; i >= 0; --i) {
            if ((pollFds[i].revents & POLLIN) == 0) {
                continue;
            }
            if (i == 0) {
                ZygoteConnection newPeer = acceptCommandPeer(abiList);
                peers.add(newPeer);
                fds.add(newPeer.getFileDesciptor());
            } else {
                boolean done = peers.get(i).runOnce();
                if (done) {
                    peers.remove(i);
                    fds.remove(i);
                }
            }
        }
    }
}
```
上面方法当中，通过acceptCommandPeer()方法创建一个新的ZygoteConnection，调用runOnce()方法处理请求。
ZygoteConnection#runOnce()：
```java
boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {
    String args[];
    Arguments parsedArgs = null;
    FileDescriptor[] descriptors;
    try {
        // 读取参数
        args = readArgumentList();
        descriptors = mSocket.getAncillaryFileDescriptors();
    } catch (IOException ex) {
        Log.w(TAG, "IOException on command socket " + ex.getMessage());
        closeSocket();
        return true;
    }
    if (args == null) {
        // EOF reached.
        closeSocket();
        return true;
    }
    /** the stderr of the most recent request, if avail */
    PrintStream newStderr = null;
    if (descriptors != null && descriptors.length >= 3) {
        newStderr = new PrintStream(new FileOutputStream(descriptors[2]));
    }
    int pid = -1;
    FileDescriptor childPipeFd = null;
    FileDescriptor serverPipeFd = null;
    try {
        // 省略代码
        // ！！！
        pid = Zygote.forkAndSpecialize(parsedArgs.uid,
                      parsedArgs.gid,parsedArgs.gids,parsedArgs.debugFlags, 
                      rlimits, parsedArgs.mountExternal, parsedArgs.seInfo, parsedArgs.niceName,
                      fdsToClose, parsedArgs.instructionSet, parsedArgs.appDataDir);
    } catch (ErrnoException ex) {
        logAndPrintError(newStderr, "Exception creating pipe", ex);
    } catch (IllegalArgumentException ex) {
        logAndPrintError(newStderr, "Invalid zygote arguments", ex);
    } catch (ZygoteSecurityException ex) {
        logAndPrintError(newStderr, "Zygote security policy prevents request: ", ex);
    }
    try {
        if (pid == 0) {
            // in child
            IoUtils.closeQuietly(serverPipeFd);
            serverPipeFd = null;
            // ！！！
            handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);
            // should never get here, the child is expected to either
            // throw ZygoteInit.MethodAndArgsCaller or exec().
            return true;
        } else {
            // in parent...pid of < 0 means failure
            IoUtils.closeQuietly(childPipeFd);
            childPipeFd = null;
            return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
        }
    } finally {
        IoUtils.closeQuietly(childPipeFd);
        IoUtils.closeQuietly(serverPipeFd);
    }
}
```
这里会调用Zygote.forkAndSpecialize()方法生成一个新的进程，如果生成成功，则pid的值为0，然后调用handleChildProc()方法：
``` java
private void handleChildProc(Arguments parsedArgs,
            FileDescriptor[] descriptors, FileDescriptor pipeFd, PrintStream newStderr)
            throws ZygoteInit.MethodAndArgsCaller {
    /**
     * By the time we get here, the native code has closed the two actual Zygote
     * socket connections, and substituted /dev/null in their place.  The LocalSocket
     * objects still need to be closed properly.
     */

    closeSocket();
    ZygoteInit.closeServerSocket();

    if (descriptors != null) {
        try {
            Os.dup2(descriptors[0], STDIN_FILENO);
            Os.dup2(descriptors[1], STDOUT_FILENO);
            Os.dup2(descriptors[2], STDERR_FILENO);

            for (FileDescriptor fd: descriptors) {
                IoUtils.closeQuietly(fd);
            }
            newStderr = System.err;
        } catch (ErrnoException ex) {
            Log.e(TAG, "Error reopening stdio", ex);
        }
    }

    if (parsedArgs.niceName != null) {
        Process.setArgV0(parsedArgs.niceName);
    }

    // End of the postFork event.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    if (parsedArgs.invokeWith != null) {
        WrapperInit.execApplication(parsedArgs.invokeWith,
                    parsedArgs.niceName, parsedArgs.targetSdkVersion,
                    VMRuntime.getCurrentInstructionSet(),
                    pipeFd, parsedArgs.remainingArgs);
    } else {
        RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion,
                    parsedArgs.remainingArgs, null /* classLoader */);
    }
}
```
这里会根据invokeWith参数决定使用哪种执行方式，我们只要知道SystemServer和apk都是通过RuntimeInit类生成的即可。

（4）RuntimeInit#zygoteInit()
```java
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
    if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
    redirectLogStreams();

    commonInit();
    nativeZygoteInit();
    applicationInit(targetSdkVersion, argv, classLoader);
}
```
最后会调用applicationInit()方法：
```java
private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
    // 省略代码

    // Remaining arguments are passed to the start class's static main
    invokeStaticMain(args.startClass, args.startArgs, classLoader);
}
```
最关键的就是invokeStaticMain()方法：
```java
private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
    Class<?> cl;

    try {
        cl = Class.forName(className, true, classLoader);
    } catch (ClassNotFoundException ex) {
        throw new RuntimeException(
                    "Missing class when invoking static main " + className,
                    ex);
    }

    Method m;
    try {
        m = cl.getMethod("main", new Class[] { String[].class });
    } catch (NoSuchMethodException ex) {
        throw new RuntimeException(
                    "Missing static main on " + className, ex);
    } catch (SecurityException ex) {
        throw new RuntimeException(
                    "Problem getting static main on " + className, ex);
    }

    int modifiers = m.getModifiers();
    if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
        throw new RuntimeException(
                    "Main method is not public and static on " + className);
    }

    /*
     * This throw gets caught in ZygoteInit.main(), which responds
     * by invoking the exception's run() method. This arrangement
     * clears up all the stack frames that were required in setting
     * up the process.
     */
    throw new ZygoteInit.MethodAndArgsCaller(m, argv);
}
```
上述方法中，通过反射的方式获取main方法，最后抛出一个MethodAndArgsCaller，它继承于Exception，同时他也是实现了Runnable接口。看 throw new ZygoteInit.MethodAndArgsCaller()的代码注释我们可以知道，它会在ZygoteInit.main()方法中被捕获，执行run方法。
```java
public static class MethodAndArgsCaller extends Exception
            implements Runnable {
    /** method to call */
    private final Method mMethod;

    /** argument array */
    private final String[] mArgs;

    public MethodAndArgsCaller(Method method, String[] args) {
        mMethod = method;
        mArgs = args;
    }

    public void run() {
        try {
            mMethod.invoke(null, new Object[] { mArgs });
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(ex);
        } catch (InvocationTargetException ex) {
            Throwable cause = ex.getCause();
            if (cause instanceof RuntimeException) {
                throw (RuntimeException) cause;
            } else if (cause instanceof Error) {
                throw (Error) cause;
            }
            throw new RuntimeException(ex);
        }
    }
}
```
我们可以知道，最终它将会调用ActivityThread类的main方法！
所以，我们解决了第四个问题，**ActivityThread的main方法是在生成一个新的app进程过程中调用的，具体是通过与Zygote通信，之后通过RuntimeInit类采用反射的方式调用ActivityThread#main()方法，即生成app中的主线程（UI线程）！**
