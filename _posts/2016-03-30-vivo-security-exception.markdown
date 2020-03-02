---
layout: post
title:  "Vivo X5SL上的SecurityException: xxx cannot kill pkg ..."
date: "2016-03-30"
header-img: "img/post-bg-04.jpg"
author: oopsoome
categories: "Android"
---
> 真是被Android国产机型的各种问题ROM操得不要不要的，你敢相信么，正常地使用ContentProvider也会造成Crash，下面就是完整的分析

{:.no_toc}
* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

### 问题的由来

在工作中，发现我司开发的App出现了不少如下类似的Crash

    java.lang.SecurityException: 10087 cannot kill pkg: xxx.xxx.xx.xx
    	at android.os.Parcel.readException(Parcel.java:1472)
    	at android.os.Parcel.readException(Parcel.java:1426)
    	at android.app.ActivityManagerProxy.removeContentProvider(ActivityManagerNative.java:3091)
    	at android.app.ActivityThread.completeRemoveProvider(ActivityThread.java:4930)
    	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1497)
    	at android.os.Handler.dispatchMessage(Handler.java:110)
    	at android.os.Looper.loop(Looper.java:193)
    	at android.app.ActivityThread.main(ActivityThread.java:5351)
    	at java.lang.reflect.Method.invokeNative(Native Method)
    	at java.lang.reflect.Method.invoke(Method.java:515)
    	at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:829)
    	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:645)
    	at dalvik.system.NativeStart.main(Native Method)

我发誓！我们的App绝对没有尝试kill掉任何竞对的App，而且，上面的stacktrace也没有看到任何包含我们app代码的stack，就是说，Crash是在系统的Framework层上挂掉的，我怀疑问题出现在rom本身。通过归类整理，发现：<font color="red">所有这样类似的Crash，都是在vivo机型上出现的!!!</font>  

### Crash分析

好了，到这里的时候，可以甩锅了。反正不是我们的问题。然而，锅甩出去了，问题并没有解决。
先去搜索了一下aosp代码，搜索到`cannot kill pkg:`这几个关键字只在killApplicationWithAppId这个函数 中出现 

```java
public void killApplicationWithAppId(String pkg, int appid, String reason) {
    if (pkg != null) {
        if (appid < 0) {
            Slog.w(TAG, "Invalid appid specified for pkg : " + pkg);
            return;
        }
        int callerUid = Binder.getCallingUid();
        if (callerUid == 1000) {
            Message msg = this.mHandler.obtainMessage(KILL_APPLICATION_MSG);
            msg.arg1 = appid;
            msg.arg2 = POWER_CHECK_DELAY;
            Bundle bundle = new Bundle();
            bundle.putString("pkg", pkg);
            bundle.putString(AutobrightInfo.KEY_REASON, reason);
            msg.obj = bundle;
            this.mHandler.sendMessage(msg);
            return;
        }
        throw new SecurityException(callerUid + " cannot kill pkg: " + pkg);
    }
}
```
从上面的代码我们知道，killApplicationWithAppId这个函数会检查Binder的callerUid，如果调用者的身份不是1000（1000是system的uid），则抛出一个异常。
下载了出问题机型的rom，解压，从system.img中提取出framework.jar和service.jar，反编译，搜索了一下对killApplicationWithAppId这个函数的调用，发现相比aosp，vivo rom的代码在`ActivityManagerService::updateOomAdjLocked(...)`中，也会调用到了killApplicationWithAppId这个函数  

看到这里，有豁然开朗的感觉，肯定是应用层的一些代码，也会调用到updateOomAdjLocked这个函数，从而最终可能调用到killApplicationWithAppId，但是应用不一定是系统应用，从而导致了异常的发生。  

Vivo Rom中updateOomAdjLocked这个函数，被改得特别地长（相对于AOSP来说），但是逻辑不难理解，所做的工作其实就是确定当前后台运行的进程的数量，如果数量大于限制个数，就根据LRU，把不在`白名单`中最久未使用的进程，通过killApplicationWithAppId函数kill掉。  

从上面的stacktrace看，触发killApplicationWithAppId的逻辑，应该和ContentProvider有关。经过一翻grep + cscope f...，下面是我发现的最后会调用到updateOomAdjLocked的路径。  

![vivoroute]({{ site.url }}/assets/vivo_updateoomadjlock_route.png)

下面是上述调用路径中，一些关键性的代码

#### ContentResolver.java

```java
public final Cursor query(final Uri uri, String[] projection,
        String selection, String[] selectionArgs, String sortOrder,
        CancellationSignal cancellationSignal) {
    IContentProvider unstableProvider = acquireUnstableProvider(uri);
    IContentProvider stableProvider = null;
    Cursor qCursor = null;
    try {
    	 // ...    
    } catch (RemoteException e) {
	    // ...
    } finally {
        // ...
        if (unstableProvider != null) {
            releaseUnstableProvider(unstableProvider);
        }
        if (stableProvider != null) {
            releaseProvider(stableProvider);
        }
    }
}
```

#### ContextImpl.java 

```java
private static final class ApplicationContentResovler extends ContentResolver {
    private final ActivityThread mMainThread;
    // ...
    @Override
    public boolean releaseProvider(IContentProvider provider) {
        // 下层入口
        return mMainThread.releaseProvider(provider, true);
    }
}
```

#### ActivityThread.java

```java
public final boolean releaseProvider(IContentProvider provider, boolean stable) {
    // ...
    synchronized (mProviderMap) {
        // ...
        if (lastRef) {
            if (!prc.removePending) {
                // 下层入口
                prc.removePending = true;
                Message msg = mH.obtainMessage(H.REMOVE_PROVIDER, prc);
                mH.sendMessage(msg);
            } else {
                Slog.w(TAG, "Duplicate remove pending of provider " + prc.holder.info.name);
            }
        }
        return true;
    }
}

private class H extands Handler {
    // ...
    private static final int REMOVE_PROVIDER = 131;
    // ...
    public void handleMessage(Message msg) {
        switch (msg.what) {
            // ...
            case REMOVE_PROVIDER:
                // 下层入口
                completeRemoveProvider((ProviderRefCount)msg.obj);
                break;
            // ...
        }
    }
}

final void completeRemoveProvider(ProviderRefCount prc) {
    // ...
    try {
        if (DEBUG_PROVIDER) {
            Slog.v(TAG, "removeProvider: Invoking ActivityManagerNative."
                    + "removeContentProvider(" + prc.holder.info.name + ")");
        }
        // 下层入口
        ActivityManagerNative.getDefault().removeContentProvider(
                prc.holder.connection, false);
    } catch (RemoteException e) {
        //do nothing content provider object is dead any way
    }
}
```

#### ActivityManagerService.java

```java
public void removeContentProvider(IBinder connection, boolean stable) {
    // ...
    try {
        synchronized (this) {
            // ...
            if (decProviderCountLocked(conn, null, null, stable)) {
                // 下层入口
                updateOomAdjLocked();
            }
        }
    } finally {
        // ...
    }
}
```

### 感想
你敢相信么，一个移动操作系统，被改到四大核心组建都不可用的状态？Vivo得有多坑爹才能做到这种程度。并且，更蛋疼的是这个异常不能被catch，异常发生在ActivityThread的一个异步线程中。要么vivo自己修复这个问题（我觉得vivo不会这么做的，国内厂商，能靠谱地给旧手机升级系统的有几家？），要么，放弃ContentProvider，再要么，放弃Vivo 🙈

下面就是vivo rom中的updateOomAdjLocked函数，一个神奇的函数  
代码来自反编译，仅供研究和学习之用，侵删。

{::options parse_block_html="true" /}

<details><summary markdown="span">updateOomAdjLocked</summary>
``` java
final void updateOomAdjLocked() {
    int cachedProcessLimit;
    int emptyProcessLimit;
    int memFactor;
    ActivityRecord TOP_ACT = resumedAppLocked();
    ProcessRecord TOP_APP = TOP_ACT != null ? TOP_ACT.app : null;
    long now = SystemClock.uptimeMillis();
    long oldTime = now - BATTERY_STATS_TIME;
    int N = this.mLruProcesses.size();
    this.mAdjSeq += SHOW_ERROR_MSG;
    this.mNewNumServiceProcs = POWER_CHECK_DELAY;
    this.mNewNumAServiceProcs = POWER_CHECK_DELAY;
    if (this.mProcessLimit <= 0) {
        cachedProcessLimit = POWER_CHECK_DELAY;
        emptyProcessLimit = POWER_CHECK_DELAY;
    } else if (this.mProcessLimit == SHOW_ERROR_MSG) {
        emptyProcessLimit = SHOW_ERROR_MSG;
        cachedProcessLimit = POWER_CHECK_DELAY;
    } else {
        emptyProcessLimit = ProcessList.computeEmptyProcessLimit(this.mProcessLimit);
        cachedProcessLimit = this.mProcessLimit - emptyProcessLimit;
    }
    int numEmptyProcs = (N - this.mNumNonCachedProcs) - this.mNumCachedHiddenProcs;
    if (numEmptyProcs > cachedProcessLimit) {
        numEmptyProcs = cachedProcessLimit;
    }
    int emptyFactor = numEmptyProcs / SHOW_FACTORY_ERROR_MSG;
    if (emptyFactor < SHOW_ERROR_MSG) {
        emptyFactor = SHOW_ERROR_MSG;
    }
    int cachedFactor = (this.mNumCachedHiddenProcs > 0 ? this.mNumCachedHiddenProcs : SHOW_ERROR_MSG) / SHOW_FACTORY_ERROR_MSG;
    if (cachedFactor < SHOW_ERROR_MSG) {
        cachedFactor = SHOW_ERROR_MSG;
    }
    int stepCached = POWER_CHECK_DELAY;
    int stepEmpty = POWER_CHECK_DELAY;
    int numCached = POWER_CHECK_DELAY;
    int numEmpty = POWER_CHECK_DELAY;
    int numTrimming = POWER_CHECK_DELAY;
    this.mNumNonCachedProcs = POWER_CHECK_DELAY;
    this.mNumCachedHiddenProcs = POWER_CHECK_DELAY;
    int curCachedAdj = 9;
    int nextCachedAdj = 9 + SHOW_ERROR_MSG;
    int curEmptyAdj = 9;
    int nextEmptyAdj = 9 + SHOW_NOT_RESPONDING_MSG;
    ArrayList<String> protectAppList = new ArrayList();
    ArrayList<String> keepList = new ArrayList();
    boolean needkill = VALIDATE_TOKENS;
    int i = N - 1;
    while (i >= 0) {
        ProcessRecord app = (ProcessRecord) this.mLruProcesses.get(i);
        if (!(app.killedByAm || app.thread == null)) {
            app.procStateChanged = VALIDATE_TOKENS;
            boolean wasKeeping = app.keeping;
            computeOomAdjLocked(app, ENTER_VISIT_MODE, TOP_APP, SHOW_ACTIVITY_START_TIME, now);
            if (app.curAdj >= ENTER_VISIT_MODE) {
                switch (app.curProcState) {
                    case H.WINDOW_FREEZE_TIMEOUT /*11*/:
                    case SERVICE_TIMEOUT_MSG /*12*/:
                        app.curRawAdj = curCachedAdj;
                        app.curAdj = app.modifyRawOomAdj(curCachedAdj);
                        if (DEBUG_LRU) {
                        }
                        if (curCachedAdj != nextCachedAdj) {
                            stepCached += SHOW_ERROR_MSG;
                            if (stepCached >= cachedFactor) {
                                stepCached = POWER_CHECK_DELAY;
                                curCachedAdj = nextCachedAdj;
                                nextCachedAdj += SHOW_NOT_RESPONDING_MSG;
                                if (nextCachedAdj > IM_FEELING_LUCKY_MSG) {
                                    nextCachedAdj = IM_FEELING_LUCKY_MSG;
                                    break;
                                }
                            }
                        }
                        break;
                    default:
                        app.curRawAdj = curEmptyAdj;
                        app.curAdj = app.modifyRawOomAdj(curEmptyAdj);
                        if (DEBUG_LRU) {
                        }
                        if (curEmptyAdj != nextEmptyAdj) {
                            stepEmpty += SHOW_ERROR_MSG;
                            if (stepEmpty >= emptyFactor) {
                                stepEmpty = POWER_CHECK_DELAY;
                                curEmptyAdj = nextEmptyAdj;
                                nextEmptyAdj += SHOW_NOT_RESPONDING_MSG;
                                if (nextEmptyAdj > IM_FEELING_LUCKY_MSG) {
                                    nextEmptyAdj = IM_FEELING_LUCKY_MSG;
                                    break;
                                }
                            }
                        }
                        break;
                }
            }
            applyOomAdjLocked(app, wasKeeping, TOP_APP, SHOW_ACTIVITY_START_TIME, VALIDATE_TOKENS, now);
            switch (app.curProcState) {
                case SHOW_NOT_RESPONDING_MSG /*2*/:
                case SHOW_FACTORY_ERROR_MSG /*3*/:
                case UPDATE_CONFIGURATION_MSG /*4*/:
                case H.REPORT_APPLICATION_TOKEN_DRAWN /*9*/:
                    if ((app.info.flags & SHOW_ERROR_MSG) == 0) {
                        if (!protectAppList.contains(app.info.packageName)) {
                            protectAppList.add(app.info.packageName);
                        }
                    }
                    this.mNumNonCachedProcs += SHOW_ERROR_MSG;
                    break;
                case H.FINISHED_STARTING /*7*/:
                    if ((app.info.flags & SHOW_ERROR_MSG) == 0) {
                        if (!(keepList.contains(app.info.packageName) || this.mAllowServiceAppList.contains(app.info.packageName))) {
                            keepList.add(app.info.packageName);
                            if (keepList.size() > ProcessList.MAX_SERVICE_APPS) {
                                needkill = SHOW_ACTIVITY_START_TIME;
                                if (DEBUG_PROCESSES) {
                                    Slog.d(TAG, "CONTROL_PROCESS_RESTART needkill set true ");
                                }
                            }
                            if (DEBUG_PROCESSES && i == 0) {
                                Slog.d(TAG, "CONTROL_PROCESS_RESTART size: " + keepList.size());
                                break;
                            }
                        }
                    }
                    break;
                case H.WINDOW_FREEZE_TIMEOUT /*11*/:
                case SERVICE_TIMEOUT_MSG /*12*/:
                    this.mNumCachedHiddenProcs += SHOW_ERROR_MSG;
                    numCached += SHOW_ERROR_MSG;
                    if (numCached > cachedProcessLimit) {
                        killUnneededProcessLocked(app, "cached #" + numCached);
                        break;
                    }
                    break;
                case UPDATE_TIME_ZONE /*13*/:
                    if (numEmpty > ProcessList.TRIM_EMPTY_APPS && app.lastActivityTime < oldTime) {
                        killUnneededProcessLocked(app, "empty for " + (((BATTERY_STATS_TIME + oldTime) - app.lastActivityTime) / 1000) + "s");
                        break;
                    }
                    numEmpty += SHOW_ERROR_MSG;
                    if (numEmpty > emptyProcessLimit) {
                        killUnneededProcessLocked(app, "empty #" + numEmpty);
                        break;
                    }
                    break;
                default:
                    this.mNumNonCachedProcs += SHOW_ERROR_MSG;
                    break;
            }
            if (app.isolated && app.services.size() <= 0) {
                killUnneededProcessLocked(app, "isolated not needed");
            }
            if (app.curProcState >= 9 && !app.killedByAm) {
                numTrimming += SHOW_ERROR_MSG;
            }
        }
        i--;
    }
    if (needkill) {
        ProcessRecord selectApp = null;
        long selectTime = now;
        for (i = this.mLruProcesses.size() - 1; i >= 0; i--) {
            app = (ProcessRecord) this.mLruProcesses.get(i);
            if (!app.killedByAm && app.thread != null && app.curProcState == 7 && (app.info.flags & SHOW_ERROR_MSG) == 0 && app.curAdj >= GC_BACKGROUND_PROCESSES_MSG) {
                if (!(protectAppList.contains(app.info.packageName) || this.mAllowServiceAppList.contains(app.info.packageName) || selectTime - app.lastActivityTime <= 0)) {
                    selectTime = app.lastActivityTime;
                    selectApp = app;
                }
            }
        }
        if (selectApp != null) {
            if (DEBUG_PROCESSES) {
                Slog.d(TAG, "CONTROL_PROCESS_RESTART kill selectApp " + selectApp);
            }
            killApplicationWithAppId(selectApp.info.packageName, selectApp.pid, "service apps too much");
        }
    }
    this.mNumServiceProcs = this.mNewNumServiceProcs;
    int numCachedAndEmpty = numCached + numEmpty;
    if (numCached > ProcessList.TRIM_CACHED_APPS || numEmpty > ProcessList.TRIM_EMPTY_APPS) {
        memFactor = POWER_CHECK_DELAY;
    } else if (numCachedAndEmpty <= SHOW_FACTORY_ERROR_MSG) {
        memFactor = SHOW_FACTORY_ERROR_MSG;
    } else if (numCachedAndEmpty <= GC_BACKGROUND_PROCESSES_MSG) {
        memFactor = SHOW_NOT_RESPONDING_MSG;
    } else {
        memFactor = SHOW_ERROR_MSG;
    }
    if (DEBUG_OOM_ADJ) {
        Slog.d(TAG, "oom: memFactor=" + memFactor + " last=" + this.mLastMemoryLevel + " allowLow=" + this.mAllowLowerMemLevel + " numProcs=" + this.mLruProcesses.size() + " last=" + this.mLastNumProcesses);
    }
    if (memFactor > this.mLastMemoryLevel && (!this.mAllowLowerMemLevel || this.mLruProcesses.size() >= this.mLastNumProcesses)) {
        memFactor = this.mLastMemoryLevel;
        if (DEBUG_OOM_ADJ) {
            Slog.d(TAG, "Keeping last mem factor!");
        }
    }
    this.mLastMemoryLevel = memFactor;
    this.mLastNumProcesses = this.mLruProcesses.size();
    boolean allChanged = this.mProcessStats.setMemFactorLocked(memFactor, !this.mSleeping ? SHOW_ACTIVITY_START_TIME : VALIDATE_TOKENS, now);
    int trackerMemFactor = this.mProcessStats.getMemFactorLocked();
    if (memFactor != 0) {
        int fgTrimLevel;
        if (this.mLowRamStartTime == 0) {
            this.mLowRamStartTime = now;
        }
        int step = POWER_CHECK_DELAY;
        switch (memFactor) {
            case SHOW_NOT_RESPONDING_MSG /*2*/:
                fgTrimLevel = 10;
                break;
            case SHOW_FACTORY_ERROR_MSG /*3*/:
                fgTrimLevel = IM_FEELING_LUCKY_MSG;
                break;
            default:
                fgTrimLevel = GC_BACKGROUND_PROCESSES_MSG;
                break;
        }
        int factor = numTrimming / SHOW_FACTORY_ERROR_MSG;
        int minFactor = SHOW_NOT_RESPONDING_MSG;
        if (this.mHomeProcess != null) {
            minFactor = SHOW_NOT_RESPONDING_MSG + SHOW_ERROR_MSG;
        }
        if (this.mPreviousProcess != null) {
            minFactor += SHOW_ERROR_MSG;
        }
        if (factor < minFactor) {
            factor = minFactor;
        }
        int curLevel = 80;
        for (i = POWER_CHECK_DELAY; i < N; i += SHOW_ERROR_MSG) {
            app = (ProcessRecord) this.mLruProcesses.get(i);
            if (allChanged || app.procStateChanged) {
                setProcessTrackerState(app, trackerMemFactor, now);
                app.procStateChanged = VALIDATE_TOKENS;
            }
            if (app.curProcState >= 9 && !app.killedByAm) {
                if (app.trimMemoryLevel < curLevel && app.thread != null) {
                    try {
                        if (DEBUG_SWITCH || DEBUG_OOM_ADJ) {
                            Slog.v(TAG, "Trimming memory of " + app.processName + " to " + curLevel);
                        }
                        app.thread.scheduleTrimMemory(curLevel);
                    } catch (RemoteException e) {
                    }
                }
                app.trimMemoryLevel = curLevel;
                step += SHOW_ERROR_MSG;
                if (step >= factor) {
                    step = POWER_CHECK_DELAY;
                    switch (curLevel) {
                        case 60:
                            curLevel = FREE_MEM_PROFILE_MSG;
                            break;
                        case 80:
                            curLevel = 60;
                            break;
                        default:
                            break;
                    }
                }
            } else if (app.curProcState == WAIT_FOR_DEBUGGER_MSG) {
                if (app.trimMemoryLevel < FREE_MEM_PROFILE_MSG && app.thread != null) {
                    try {
                        if (DEBUG_SWITCH || DEBUG_OOM_ADJ) {
                            Slog.v(TAG, "Trimming memory of heavy-weight " + app.processName + " to " + FREE_MEM_PROFILE_MSG);
                        }
                        app.thread.scheduleTrimMemory(FREE_MEM_PROFILE_MSG);
                    } catch (RemoteException e2) {
                    }
                }
                app.trimMemoryLevel = FREE_MEM_PROFILE_MSG;
            } else {
                if ((app.curProcState >= UPDATE_CONFIGURATION_MSG || app.systemNoUi) && app.pendingUiClean) {
                    if (app.trimMemoryLevel < PROC_START_TIMEOUT_MSG && app.thread != null) {
                        try {
                            if (DEBUG_SWITCH || DEBUG_OOM_ADJ) {
                                Slog.v(TAG, "Trimming memory of bg-ui " + app.processName + " to " + PROC_START_TIMEOUT_MSG);
                            }
                            app.thread.scheduleTrimMemory(PROC_START_TIMEOUT_MSG);
                        } catch (RemoteException e3) {
                        }
                    }
                    app.pendingUiClean = VALIDATE_TOKENS;
                }
                if (app.trimMemoryLevel < fgTrimLevel && app.thread != null) {
                    try {
                        if (DEBUG_SWITCH || DEBUG_OOM_ADJ) {
                            Slog.v(TAG, "Trimming memory of fg " + app.processName + " to " + fgTrimLevel);
                        }
                        app.thread.scheduleTrimMemory(fgTrimLevel);
                    } catch (RemoteException e4) {
                    }
                }
                app.trimMemoryLevel = fgTrimLevel;
            }
        }
    } else {
        if (this.mLowRamStartTime != 0) {
            this.mLowRamTimeSinceLastIdle += now - this.mLowRamStartTime;
            this.mLowRamStartTime = 0;
        }
        for (i = N - 1; i >= 0; i--) {
            app = (ProcessRecord) this.mLruProcesses.get(i);
            if (allChanged || app.procStateChanged) {
                setProcessTrackerState(app, trackerMemFactor, now);
                app.procStateChanged = VALIDATE_TOKENS;
            }
            if ((app.curProcState >= UPDATE_CONFIGURATION_MSG || app.systemNoUi) && app.pendingUiClean) {
                if (app.trimMemoryLevel < PROC_START_TIMEOUT_MSG && app.thread != null) {
                    try {
                        if (DEBUG_SWITCH || DEBUG_OOM_ADJ) {
                            Slog.v(TAG, "Trimming memory of ui hidden " + app.processName + " to " + PROC_START_TIMEOUT_MSG);
                        }
                        app.thread.scheduleTrimMemory(PROC_START_TIMEOUT_MSG);
                    } catch (RemoteException e5) {
                    }
                }
                app.pendingUiClean = VALIDATE_TOKENS;
            }
            app.trimMemoryLevel = POWER_CHECK_DELAY;
        }
    }
    if (this.mAlwaysFinishActivities) {
        this.mStackSupervisor.scheduleDestroyAllActivities(null, "always-finish");
    }
    if (allChanged) {
        requestPssAllProcsLocked(now, VALIDATE_TOKENS, this.mProcessStats.isMemFactorLowered());
    }
    if (this.mProcessStats.shouldWriteNowLocked(now)) {
        this.mHandler.post(new Runnable() {
            public void run() {
                synchronized (ActivityManagerService.this) {
                    ActivityManagerService.this.mProcessStats.writeStateAsyncLocked();
                }
            }
        });
    }
    if (DEBUG_OOM_ADJ) {
        Slog.d(TAG, "Did OOM ADJ in " + (SystemClock.uptimeMillis() - now) + "ms");
    }
}
```
</details>
{::options parse_block_html="false" /}
