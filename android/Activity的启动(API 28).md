# Activity 的启动（API 28）

## 启动一个Activity

启动一个Activity可以通过下面的几种方法来启动：
-  `Activity.startActivity()` 
-  `Activity.startActivityForResult()` 
-  `ContextImpl.startActivity()`


Activity中前两种方法：
```java
    @Override
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }

    @Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }

    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
            // TODO Consider clearing/flushing other event sources and events for child windows.
        } else {
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                // Note we want to go through this method for compatibility with
                // existing applications that may have overridden it.
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
```

ContextImpl中的启动方法：
```java
    @Override
    public void startActivity(Intent intent) {
        warnIfCallingFromSystemProcess();
        startActivity(intent, null);
    }
    
    @Override
    public void startActivity(Intent intent, Bundle options) {
        warnIfCallingFromSystemProcess();

        // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
        // generally not allowed, except if the caller specifies the task id the activity should
        // be launched in. A bug was existed between N and O-MR1 which allowed this to work. We
        // maintain this for backwards compatibility.
        final int targetSdkVersion = getApplicationInfo().targetSdkVersion;

        if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
                && (targetSdkVersion < Build.VERSION_CODES.N
                        || targetSdkVersion >= Build.VERSION_CODES.P)
                && (options == null
                        || ActivityOptions.fromBundle(options).getLaunchTaskId() == -1)) {
            throw new AndroidRuntimeException(
                    "Calling startActivity() from outside of an Activity "
                            + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                            + " Is this really what you want?");
        }
        mMainThread.getInstrumentation().execStartActivity(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intent, -1, options);
    }
```

这几种方法最后都会都会调用到主线程初始化时与AMS进行通信时，记录在ActivityThread中的Instrumentation对象来启动Activity。

```java
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    ActivityResult result = null;
                    if (am.ignoreMatchingSpecificIntents()) {
                        result = am.onStartActivity(intent);
                    }
                    if (result != null) {
                        am.mHits++;
                        return result;
                    } else if (am.match(who, null, intent)) {
                        am.mHits++;
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            int result = ActivityManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

在Instrumentation.execStartActivity()方法中，首先会拿到当前主线程在对应的ActivityThread对象的远程代理对象ApplicationThread，这个远程代理对象之后会通过Binder远程通信，传递到Framework层的AMS中去。而主线程中的Instrumentation对象是在AMS初始化对应的主线程时进行将主线程与AMS提前建立远程通信时初始化的。

之后会对对应Instrumentation设置的ActivityMonitors进行一个判断，看是否存在已经设置了ActivityMonitor对象。一般需要使用反射获取ActivityThread中的Instrumentation对象，然后调用`Instrumentation.addMonitor()`方法来添加。同时，ActivityMonitor对象是用来监控Activity的启动，一般是用在单元测试中。

之后就会远程调用ActivityManagerService.startActivity()方法，来进行远程调用，将Activity的启动交与ActivityManagerService控制。在AMS进行Activity的启动时，会去执行ActivityRecord的创建（根据Intent去通过PackageManagerService进行查找对于的要启动的Activity）、Activity对应的IToken的创建（关联Activity Window，之后会与WindowManagerService相关联）、根据传入的token、target以及Intent Flag来判断由谁来接收新创建的Activity所返回的Result、根据Intent的启动stack相关的Flag来确定Activity的堆栈、最后通过ActivityThread的远程代理对象ApplicationThread远程回调来创建一个Activty实例。

### Instrumentation中获取对应Intent的resolvedType

上面说的startActivity最终会调用到Instrumentation.execStartActivity()来进行远程BInder调用，将启动Activity的工作交付给AMS。其中，Instrumentation会根据当前传入的Intent去获取传给AMS的参数resolvedType。这里是调用方法`Intent.resolveTypeIfNeed()`来完成的：

```java
            int result = ActivityManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
```
下面看到`Intent.resolveTypeIfNeed()`方法
```java
    public @Nullable String resolveTypeIfNeeded(@NonNull ContentResolver resolver) {
        if (mComponent != null) {
            return mType;
        }
        return resolveType(resolver);
    }

    public @Nullable String resolveType(@NonNull ContentResolver resolver) {
        if (mType != null) {
            return mType;
        }
        if (mData != null) {
            if ("content".equals(mData.getScheme())) {
                return resolver.getType(mData);
            }
        }
        return null;
    }
```

这里会首先根据当前Intent是否设置了`mComponent`字段，如果设置了，则会直接返回当前Intent所设置的`mType`字段。如果没有设置`mComponent`字段，则会调用`Intent.resolveType()`方法来继续处理。而在`Intent.resolveType()`方法中，也会依次去从Intent中的`mType`字段和`mData`字段的Type部分中去取得相应的resolveType的值。如果都没有设置，则会返回为null。

这里的`mComponent`字段其实就是当前Intent设置的显示要启动的组件信息，包含包名和完整类名。而`mType`字段则是通过Intent.setType()和Intent.setDataAndType()等方法去设置的，可以认为是查找一个Intent对应组件的一个隐性条件。也就是说通过`Intent.resolveTypeIfNeed()`方法去获取的resolveType的值，都取决于当前Intent中设置的`mType`字段，这个字段在Android manifest文件中的对应值为:
```xml
            <intent-filter>
                <data android:mimeType="111" />
            </intent-filter>
```

## ActivityManagerService.startActivity()

```java
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }

    @Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
                true /*validateIncomingUser*/);
    }

    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
            boolean validateIncomingUser) {
        enforceNotIsolatedCaller("startActivity");

        userId = mActivityStartController.checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        // TODO: Switch to user app stacks here.
        return mActivityStartController.obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setMayWait(userId)
                .execute();
    }
```
AMS.startActivity()最终会调用到一个ActivityStartController对象来继续处理Activity创建的流程。
首先，这里会去获取远程调用AMS中当前方法的线程的UID，然后通过ActivityStartController.obtainStarted()方法来创建一个ActivityStarter对象，来处理此次的Activity的启动。这里会通过工厂模式封装的ActivityStarter去设置各种需要用到启动Activity的参数，然后调用`ActivityStarted.execute()`方法来启动Activity。

```java
    ActivityStarter obtainStarter(Intent intent, String reason) {
        return mFactory.obtain().setIntent(intent).setReason(reason);
    }
```

注：在API 28时，已将ActivityStarter这个类封装成了一个工厂模式，通过调用`ActivityStarted.execute()`方法来启动Activity，而不是之前的那种直接调用`ActivityStarter.startActivityMayWait()`方法来启动。


## ActivityStarter.execute()启动Activity

之前看到在ActivityManagerService.startActivity()方法中，最后调用了ActivityStarter.execute()方法来启动Activity。这里会将一次Activity的启动描述为一个Request对象，然后根据Request对象的mayWait字段来判断要执行哪一种启动：

```java
    int execute() {
        try {
            // TODO(b/64750076): Look into passing request directly to these methods to allow
            // for transactional diffs and preprocessing.
            if (mRequest.mayWait) {
                return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                        mRequest.callingPackage, mRequest.intent, mRequest.resolvedType,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                        mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                        mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                        mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup);
            } else {
                return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                        mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                        mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                        mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                        mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                        mRequest.outActivity, mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup);
            }
        } finally {
            onExecutionComplete();
        }
    }
```

这里两种方法，最后都会调用同一个ActivityStatrer.startActivity()方法。通常情况下，会走到ActivityStarter.startActivityMayWait()方法。

### ActivityStatrer.startActivityMayWait()
之前说的，在远程调用AMS.startActivity()中会通过工厂模式根据传入的参数建立一个ActivityStarter对象，然后调用到刚刚创建的对象的ActivityStarter.startActivity()方法中，最后调用到ActivityStarter.startActivityMayWait()方法。接下来看到`ActivityStarter.startActivityMayWait()方法`：

```java
    private int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult,
            Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
            int userId, TaskRecord inTask, String reason,
            boolean allowPendingRemoteAnimationRegistryLookup) {
        // Refuse possible leaked file descriptors
        if (intent != null && intent.hasFileDescriptors()) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }
        mSupervisor.getActivityMetricsLogger().notifyActivityLaunching();
        boolean componentSpecified = intent.getComponent() != null;

        final int realCallingPid = Binder.getCallingPid();
        final int realCallingUid = Binder.getCallingUid();

        int callingPid;
        if (callingUid >= 0) {
            callingPid = -1;
        } else if (caller == null) {
            callingPid = realCallingPid;
            callingUid = realCallingUid;
        } else {
            callingPid = callingUid = -1;
        }

        // Save a copy in case ephemeral needs it
        final Intent ephemeralIntent = new Intent(intent);
        // Don't modify the client's object!
        intent = new Intent(intent);
        if (componentSpecified
                && !(Intent.ACTION_VIEW.equals(intent.getAction()) && intent.getData() == null)
                && !Intent.ACTION_INSTALL_INSTANT_APP_PACKAGE.equals(intent.getAction())
                && !Intent.ACTION_RESOLVE_INSTANT_APP_PACKAGE.equals(intent.getAction())
                && mService.getPackageManagerInternalLocked()
                        .isInstantAppInstallerComponent(intent.getComponent())) {
            // intercept intents targeted directly to the ephemeral installer the
            // ephemeral installer should never be started with a raw Intent; instead
            // adjust the intent so it looks like a "normal" instant app launch
            intent.setComponent(null /*component*/);
            componentSpecified = false;
        }

        ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId,
                0 /* matchFlags */,
                        computeResolveFilterUid(
                                callingUid, realCallingUid, mRequest.filterCallingUid));
        
        ...
        
        // Collect information about the target of the Intent.
        ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);

        synchronized (mService) {
            final ActivityStack stack = mSupervisor.mFocusedStack;
            stack.mConfigWillChange = globalConfig != null
                    && mService.getGlobalConfiguration().diff(globalConfig) != 0;
            if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION,
                    "Starting activity when config will change = " + stack.mConfigWillChange);

            final long origId = Binder.clearCallingIdentity();

            ...

            final ActivityRecord[] outRecord = new ActivityRecord[1];
            int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
                    callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
                    ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
                    allowPendingRemoteAnimationRegistryLookup);

            Binder.restoreCallingIdentity(origId);

            if (stack.mConfigWillChange) {
                // If the caller also wants to switch to a new configuration,
                // do so now.  This allows a clean switch, as we are waiting
                // for the current activity to pause (so we will not destroy
                // it), and have not yet started the next activity.
                mService.enforceCallingPermission(android.Manifest.permission.CHANGE_CONFIGURATION,
                        "updateConfiguration()");
                stack.mConfigWillChange = false;
                if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION,
                        "Updating to new configuration after starting activity.");
                mService.updateConfigurationLocked(globalConfig, null, false);
            }

            if (outResult != null) {
                outResult.result = res;

                final ActivityRecord r = outRecord[0];

                switch(res) {
                    case START_SUCCESS: {
                        mSupervisor.mWaitingActivityLaunched.add(outResult);
                        do {
                            try {
                                mService.wait();
                            } catch (InterruptedException e) {
                            }
                        } while (outResult.result != START_TASK_TO_FRONT
                                && !outResult.timeout && outResult.who == null);
                        if (outResult.result == START_TASK_TO_FRONT) {
                            res = START_TASK_TO_FRONT;
                        }
                        break;
                    }
                    case START_DELIVERED_TO_TOP: {
                        outResult.timeout = false;
                        outResult.who = r.realActivity;
                        outResult.totalTime = 0;
                        outResult.thisTime = 0;
                        break;
                    }
                    case START_TASK_TO_FRONT: {
                        // ActivityRecord may represent a different activity, but it should not be
                        // in the resumed state.
                        if (r.nowVisible && r.isState(RESUMED)) {
                            outResult.timeout = false;
                            outResult.who = r.realActivity;
                            outResult.totalTime = 0;
                            outResult.thisTime = 0;
                        } else {
                            outResult.thisTime = SystemClock.uptimeMillis();
                            mSupervisor.waitActivityVisible(r.realActivity, outResult);
                            // Note: the timeout variable is not currently not ever set.
                            do {
                                try {
                                    mService.wait();
                                } catch (InterruptedException e) {
                                }
                            } while (!outResult.timeout && outResult.who == null);
                        }
                        break;
                    }
                }
            }

            mSupervisor.getActivityMetricsLogger().notifyActivityLaunched(res, outRecord[0]);
            return res;
        }
    }
```

在此方法中，首先会通过Binder来获取本次远程调用的进程的Pid（进程id）和Uid（每个App的唯一标识）。然后借助PackageManagerService（在App安装后，PackageManagerService会取收集每一个安装的App中的manifest文件中的所有组件信息）和Intent携带的信息和对应Intent的resolveType（上文讲过，取决于当前Intent的`mType`字段，对应于manifest文件中的mineType标签）来去查找相应的组件信息，然后生成对应的ResolveInfo对象。这其中会涉及到PKMS中对Intent中携带的显性条件和隐性条件的判断与处理。

ResolveInfo中会包含对应要启动的组件信息对象，这里面可能会包含又ActivityInfo、ServiceInfo、ProviderInfo等几种类型的组件信息，其中至少有一个不为Null。这里对应不为空的为ActivityInfo。

所以在借助PKMS完成对应Intent所要启动的组件的查找和对应ResolveInfo的创建后，当前情况下会将ResolveInfo中的ActivityInfo取出来。然后就会调用到之前在ActivityStarter.exectu()方法中调用到的另外一种情况下的`ActivityStarter.startActivity()`方法。在此之前，还会预先创建一个长度为1的ActivityRecord数组，作为调用该方法的参数，用来保存之后将于启动的Activity组件的对应ActivityRecord对象。


## ActivityStarter.startActivity()

之前说的，在远程调用`AMS.startActivity()`中会通过工厂模式根据传入的参数建立一个ActivityStarter对象，然后调用到刚刚创建的对象的`ActivityStarter.startActivity()`方法中，再调用到`ActivityStarter.startActivityMayWait()`方法。而在`ActivityStarter.startActivityMayWait()`中，会借助PKMS来获取到要启动的组件的ResolveInfo和ActivityInfo和创建一个长度为1用来保存ActivityRecord对象的数组后，会继续调用`ActivityStarter.startActivity()`方法来完成Activity的启动。

```java
    private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            SafeActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
            ActivityRecord[] outActivity, TaskRecord inTask, String reason,
            boolean allowPendingRemoteAnimationRegistryLookup) {

        if (TextUtils.isEmpty(reason)) {
            throw new IllegalArgumentException("Need to specify a reason.");
        }
        mLastStartReason = reason;
        mLastStartActivityTimeMs = System.currentTimeMillis();
        mLastStartActivityRecord[0] = null;

        mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
                aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
                callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
                inTask, allowPendingRemoteAnimationRegistryLookup);

        if (outActivity != null) {
            // mLastStartActivityRecord[0] is set in the call to startActivity above.
            outActivity[0] = mLastStartActivityRecord[0];
        }

        return getExternalResult(mLastStartActivityResult);
    }
```

首先在`ActivityStarter.startActivity()`方法中会调用到另外一个很长的重写`ActivityStarter.startActivity()`方法，在此方法中完成启动前一系列操作，下面将分段介绍此方法。

### `ActivityStarter.startActivity()`-part1

```java
        ActivityRecord sourceRecord = null;
        ActivityRecord resultRecord = null;
        if (resultTo != null) {
            sourceRecord = mSupervisor.isInAnyStackLocked(resultTo);
            if (DEBUG_RESULTS) Slog.v(TAG_RESULTS,
                    "Will send result to " + resultTo + " " + sourceRecord);
            if (sourceRecord != null) {
                if (requestCode >= 0 && !sourceRecord.finishing) {
                    resultRecord = sourceRecord;
                }
            }
        }

        final int launchFlags = intent.getFlags();

        if ((launchFlags & Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0 && sourceRecord != null) {
            // Transfer the result target from the source activity to the new
            // one being started, including any failures.
            if (requestCode >= 0) {
                SafeActivityOptions.abort(options);
                return ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT;
            }
            resultRecord = sourceRecord.resultTo;
            if (resultRecord != null && !resultRecord.isInStackLocked()) {
                resultRecord = null;
            }
            resultWho = sourceRecord.resultWho;
            requestCode = sourceRecord.requestCode;
            sourceRecord.resultTo = null;
            if (resultRecord != null) {
                resultRecord.removeResultsLocked(sourceRecord, resultWho, requestCode);
            }
            if (sourceRecord.launchedFromUid == callingUid) {
                // The new activity is being launched from the same uid as the previous
                // activity in the flow, and asking to forward its result back to the
                // previous.  In this case the activity is serving as a trampoline between
                // the two, so we also want to update its launchedFromPackage to be the
                // same as the previous activity.  Note that this is safe, since we know
                // these two packages come from the same uid; the caller could just as
                // well have supplied that same package name itself.  This specifially
                // deals with the case of an intent picker/chooser being launched in the app
                // flow to redirect to an activity picked by the user, where we want the final
                // activity to consider it to have been launched by the previous app activity.
                callingPackage = sourceRecord.launchedFromPackage;
            }
        }
```

这里首先会通过传递过来的参数`resultTo`和`requestCode`的值来判断此次Activity的启动是否需要将Result返回到某个Activity。这两个参数的意义如下：
- `resultTo`：类型为IBinder，其来自于执行远程调用的Activity的Window Token，也就是对应ActivityRecord中的`IApplicationToken`，表示是由哪个Activity来发起此次Activity的启动。
- `requestCode`：类型为int类型，如果值>=0，则说明此要启动的Activity是有result要进行返回，此值作为接受result的Activity中对此result的唯一标识。

当`resultTo`不为Null以及`requestCode >= 0`时，说明当前要启动的Activity是可以返回result的。这里会初始化两个ActivityRecord对象：`sourceRecord`和`resultRecord`，分别表示`resultTo`所代表的Activity所在AMS端对应的ActivityRecord和之后将要接收此次启动Activity的Result的Activity。通常情况下`sourceRecord`和`resultRecord`是一个ActivityRecord。

但如果此次Intent设置了`Intent.FLAG_ACTIVITY_FORWARD_RESULT`，那么就会进行对`resultRecord`的修改：

无论 `requestCode`的值是任何值，`sourceRecord`都是指向`resultTo`所代表的Activity所在AMS端对应的ActivityRecord。而只有当 `requestCode`不<0时，Intent才允许设置`Intent.FLAG_ACTIVITY_FORWARD_RESULT`，否则此次启动就会报错。当经过判断`requestCode`的值小于0，那么`resultRecord`的值就会是Null。

此时如果设置了`Intent.FLAG_ACTIVITY_FORWARD_RESULT`，那么就会将`resultRecord`的值赋值为`sourceRecord.resultTo`，同时将当前的`requestCode`和`resultWho`赋值为`sourceRecord`对应的字段，同时将`sourceRecord.resultTo`置为Null。这样做的意义就是将进行本次Activity启动的Activity自身所要进行结果进行接收的Activity传递给了将要启动的Activity。

这样做的效果其实和设置`requestCode`大于等于0时是一致的，但`requestCode`大于等于0时不能提前finish()关掉`resultTo`所指向的Activity，而设置了`Intent.FLAG_ACTIVITY_FORWARD_RESULT`是可以的，这样在多个Activity之间传递Result时，是可以提前关闭中间可以关闭Activity，而不需要将result一级级的往外传递。


### `ActivityStarter.startActivity()`-part2

```java
        boolean abort = !mSupervisor.checkStartAnyActivityPermission(intent, aInfo, resultWho,
                requestCode, callingPid, callingUid, callingPackage, ignoreTargetSecurity,
                inTask != null, callerApp, resultRecord, resultStack);
        abort |= !mService.mIntentFirewall.checkStartActivity(intent, callingUid,
                callingPid, resolvedType, aInfo.applicationInfo);
```

在完成了对当前要启动的Activity是否要返回Result和返回Result由谁来接受之后，回通过调用`checkStartAnyActivityPermission`()方法来完成对启动当前Activity的权限进行检测。

```java
    boolean checkStartAnyActivityPermission(Intent intent, ActivityInfo aInfo,
            String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, boolean ignoreTargetSecurity, boolean launchingInTask,
            ProcessRecord callerApp, ActivityRecord resultRecord, ActivityStack resultStack) {
        final boolean isCallerRecents = mService.getRecentTasks() != null &&
                mService.getRecentTasks().isCallerRecents(callingUid);
        final int startAnyPerm = mService.checkPermission(START_ANY_ACTIVITY, callingPid,
                callingUid);
        if (startAnyPerm == PERMISSION_GRANTED || (isCallerRecents && launchingInTask)) {
            // If the caller has START_ANY_ACTIVITY, ignore all checks below. In addition, if the
            // caller is the recents component and we are specifically starting an activity in an
            // existing task, then also allow the activity to be fully relaunched.
            return true;
        }
        final int componentRestriction = getComponentRestrictionForCallingPackage(
                aInfo, callingPackage, callingPid, callingUid, ignoreTargetSecurity);
        final int actionRestriction = getActionRestrictionForCallingPackage(
                intent.getAction(), callingPackage, callingPid, callingUid);
        if (componentRestriction == ACTIVITY_RESTRICTION_PERMISSION
                || actionRestriction == ACTIVITY_RESTRICTION_PERMISSION) {
            if (resultRecord != null) {
                resultStack.sendActivityResultLocked(-1,
                        resultRecord, resultWho, requestCode,
                        Activity.RESULT_CANCELED, null);
            }
            final String msg;
            if (actionRestriction == ACTIVITY_RESTRICTION_PERMISSION) {
                msg = "Permission Denial: starting " + intent.toString()
                        + " from " + callerApp + " (pid=" + callingPid
                        + ", uid=" + callingUid + ")" + " with revoked permission "
                        + ACTION_TO_RUNTIME_PERMISSION.get(intent.getAction());
            } else if (!aInfo.exported) {
                msg = "Permission Denial: starting " + intent.toString()
                        + " from " + callerApp + " (pid=" + callingPid
                        + ", uid=" + callingUid + ")"
                        + " not exported from uid " + aInfo.applicationInfo.uid;
            } else {
                msg = "Permission Denial: starting " + intent.toString()
                        + " from " + callerApp + " (pid=" + callingPid
                        + ", uid=" + callingUid + ")"
                        + " requires " + aInfo.permission;
            }
            Slog.w(TAG, msg);
            throw new SecurityException(msg);
        }

        if (actionRestriction == ACTIVITY_RESTRICTION_APPOP) {
            final String message = "Appop Denial: starting " + intent.toString()
                    + " from " + callerApp + " (pid=" + callingPid
                    + ", uid=" + callingUid + ")"
                    + " requires " + AppOpsManager.permissionToOp(
                            ACTION_TO_RUNTIME_PERMISSION.get(intent.getAction()));
            Slog.w(TAG, message);
            return false;
        } else if (componentRestriction == ACTIVITY_RESTRICTION_APPOP) {
            final String message = "Appop Denial: starting " + intent.toString()
                    + " from " + callerApp + " (pid=" + callingPid
                    + ", uid=" + callingUid + ")"
                    + " requires appop " + AppOpsManager.permissionToOp(aInfo.permission);
            Slog.w(TAG, message);
            return false;
        }

        return true;
    }
```

在此方法中，有三处对启动Activity的权限检测：
- 首先判断当前远程调用的线程是否声明了`START_ANY_ACTIVITY`权限。如果声明了，则相当于任何Activity都能启动，也就不会存在权限问题，这里就会直接返回true，表明没有权限问题。
- 在当前远程调用的线程没有声明`START_ANY_ACTIVITY`，则会调用`getComponentRestrictionForCallingPackage()`方法，来判断当前线程是否声明了**当前要启动的Activity在manifest文件中对应Activity标签声明的权限**。当没有在Manifest文件中对应Activity标签中声明权限，这里也可以认为不存在权限问题。
- 在检测完对应Activity标签中声明的权限后，还会去通过Intent的Action来检测是否有对应的权限。

在上述情况都检测完成，并不存在有权限问题时，还会在通过IntentFirewall来判断是否设置了拦截Intent的rule，看是否需要中断此次的Activity的启动。


### `ActivityStarter.startActivity()`-part3

在完成对Activity的result相关处理和对创建Activity的权限进行判断后，接下来即将进入Activity启动的过程，所以这里会对将要启动的Activity创建一个对应的ActivityRecord对象，并把这个对象赋值到之前传入的一个ActivityRecord数组中去。

```java
        ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
                callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
                resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
                mSupervisor, checkedOptions, sourceRecord);
        if (outActivity != null) {
            outActivity[0] = r;
        }
```

在创建ActivityRecord对象时，会创建一个唯一的与之对应的`Token`类型的对象。这个**Token对象**将会回传到App层去，是此Activity对应的Window在WindowManagerService处的唯一标识。这个**Token对**象会在稍后从AMS传递到WMS中，生成一个对应的AppWindowToken，同时还会再ViewRootImpl.setView()方法中远程调用WMS.addWindow()方法时，通过参数WindowManager.LayoutParams()来传递到WMS中去，查找之前创建AppWindowToken对象。

### `ActivityStarter.startActivity()`-part4

在完成了ActivityRecord对象的创建和与之对应的唯一Token对象的创建后，最后会调用另外一个ActivityStarter.startActivity()方法来启动Activity，此方法中回带上之前创建的ActivityRecord对象。

```java
        return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
                true /* doResume */, checkedOptions, inTask, outActivity);
```

```java
    private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                ActivityRecord[] outActivity) {
        int result = START_CANCELED;
        try {
            mService.mWindowManager.deferSurfaceLayout();
            result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, outActivity);
        } finally {
            // If we are not able to proceed, disassociate the activity from the task. Leaving an
            // activity in an incomplete state can lead to issues, such as performing operations
            // without a window container.
            final ActivityStack stack = mStartActivity.getStack();
            if (!ActivityManager.isStartResultSuccessful(result) && stack != null) {
                stack.finishActivityLocked(mStartActivity, RESULT_CANCELED,
                        null /* intentResultData */, "startActivity", true /* oomAdj */);
            }
            mService.mWindowManager.continueSurfaceLayout();
        }

        postStartActivityProcessing(r, result, mTargetStack);

        return result;
    }
```


在此方法中会调用`ActivityStarter.startActivityUnchecked()`来完成接下来的Activity的启动工作。


## ActivityStarter.startActivityUnchecked()

目前Activity的启动到此已经经历了如下几个步骤：
- 在App远程调用AMS来启动Activity时，会经历获取远程调用线程UID和PID；
- 然后借助PKMS来获取对应的ResolveInfo和ActivityInfo；
- 然后还会确定当前要启动的Activity是否能够返回Result，如果能，还会去判断由哪一个ActivityRecord来作为接受Result；
- 接着检测启动当前Activity的权限是否都通过了；
- 创建一个新的与当前要启动的Activity对应的ActivityRecord，其中会初始化一个Token对象，作为Activity在WMS端的标识；

接下来来看到`ActivityStarter.startActivityUnchecked()`的处理，这个函数也非常的长，这里也将分段解析。

### ActivityStarter.startActivityUnchecked()-part1

```java
        computeLaunchingTaskFlags();
```

在ActivityStarter.startActivityUnchecked()方法中首先会调用`ActivityStarter.computeLaunchingTaskFlags()`方法，主要是根据现有的SourceRecord和自身的LaunchMode来判断修改自身mLaunchFlags。

```java
    private void computeLaunchingTaskFlags() {
        // If the caller is not coming from another activity, but has given us an explicit task into
        // which they would like us to launch the new activity, then let's see about doing that.
        if (mSourceRecord == null && mInTask != null && mInTask.getStack() != null) {
            final Intent baseIntent = mInTask.getBaseIntent();
            final ActivityRecord root = mInTask.getRootActivity();
            if (baseIntent == null) {
                ActivityOptions.abort(mOptions);
                throw new IllegalArgumentException("Launching into task without base intent: "
                        + mInTask);
            }

            // If this task is empty, then we are adding the first activity -- it
            // determines the root, and must be launching as a NEW_TASK.
            if (isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK)) {
                if (!baseIntent.getComponent().equals(mStartActivity.intent.getComponent())) {
                    ActivityOptions.abort(mOptions);
                    throw new IllegalArgumentException("Trying to launch singleInstance/Task "
                            + mStartActivity + " into different task " + mInTask);
                }
                if (root != null) {
                    ActivityOptions.abort(mOptions);
                    throw new IllegalArgumentException("Caller with mInTask " + mInTask
                            + " has root " + root + " but target is singleInstance/Task");
                }
            }

            // If task is empty, then adopt the interesting intent launch flags in to the
            // activity being started.
            if (root == null) {
                final int flagsOfInterest = FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_MULTIPLE_TASK
                        | FLAG_ACTIVITY_NEW_DOCUMENT | FLAG_ACTIVITY_RETAIN_IN_RECENTS;
                mLaunchFlags = (mLaunchFlags & ~flagsOfInterest)
                        | (baseIntent.getFlags() & flagsOfInterest);
                mIntent.setFlags(mLaunchFlags);
                mInTask.setIntent(mStartActivity);
                mAddingToTask = true;

                // If the task is not empty and the caller is asking to start it as the root of
                // a new task, then we don't actually want to start this on the task. We will
                // bring the task to the front, and possibly give it a new intent.
            } else if ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
                mAddingToTask = false;

            } else {
                mAddingToTask = true;
            }

            mReuseTask = mInTask;
        } else {
            mInTask = null;
            // Launch ResolverActivity in the source task, so that it stays in the task bounds
            // when in freeform workspace.
            // Also put noDisplay activities in the source task. These by itself can be placed
            // in any task/stack, however it could launch other activities like ResolverActivity,
            // and we want those to stay in the original task.
            if ((mStartActivity.isResolverActivity() || mStartActivity.noDisplay) && mSourceRecord != null
                    && mSourceRecord.inFreeformWindowingMode())  {
                mAddingToTask = true;
            }
        }

        if (mInTask == null) {
            if (mSourceRecord == null) {
                // This activity is not being started from another...  in this
                // case we -always- start a new task.
                if ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) == 0 && mInTask == null) {
                    Slog.w(TAG, "startActivity called from non-Activity context; forcing " +
                            "Intent.FLAG_ACTIVITY_NEW_TASK for: " + mIntent);
                    mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
                }
            } else if (mSourceRecord.launchMode == LAUNCH_SINGLE_INSTANCE) {
                // The original activity who is starting us is running as a single
                // instance...  this new activity it is starting must go on its
                // own task.
                mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
            } else if (isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK)) {
                // The activity being started is a single instance...  it always
                // gets launched into its own task.
                mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
            }
        }
    }
```

这个方法中会对此次要启动的Activity是否来自于另外一个Activity的启动，还是非Activity的启动，如Service。这里是主要是通过SourceRecord是否为空来判断是否由另外一个Activity启动。当一个Activity是由Service启动时，其远程调用的参数`Token resultTo`为Null，所以其SourceRecord为Null，且其Intent一定事设置了`FLAG_ACTIVITY_NEW_TASK`标志位。

正常情况下，由其他Activity远程调用的走到这里的Activity的，其SourceRecord不会为Null，所以这里会走到**else分支**的情形。这里会将对应的`ActivityStarter.mInTask`置为Null，然后继续向下判断：
- 当SourceRecord为NULL，则说明启动此Activity的组件不是一个Activity，这是需要强制为`mLaunchFlags`添加`FLAG_ACTIVITY_NEW_TASK`标志位，但一般这种情况下已经添加过了；
- 当SourceRecord的`mLaunchFlags`中含有`LAUNCH_SINGLE_INSTANCE`标志位，则说明SourceRecord是一个特殊的Activity，其ActivityStack中有且只有他自身一个Activity，由其启动的其他Activity只能在其他已有或新建的Task中，所以这里也会为`mLaunchFlags`添加`FLAG_ACTIVITY_NEW_TASK`标志位；
- 当启动当前Activity的`mLaunchFlags`中含有`LAUNCH_SINGLE_INSTANCE`或`LAUNCH_SINGLE_TASK`标志位，那么也会强制加上`FLAG_ACTIVITY_NEW_TASK`标志位。


### ActivityStarter.startActivityUnchecked()-part2

```java
    computeSourceStack();
```

在此部分依旧是之调用了一个`ActivityStarter.computeSourceStack()`方法，主要功能是计算是否能够复用的SourceRecord所在的ActivityStack。

```java
    private void computeSourceStack() {
        if (mSourceRecord == null) {
            mSourceStack = null;
            return;
        }
        if (!mSourceRecord.finishing) {
            mSourceStack = mSourceRecord.getStack();
            return;
        }

        // If the source is finishing, we can't further count it as our source. This is because the
        // task it is associated with may now be empty and on its way out, so we don't want to
        // blindly throw it in to that task.  Instead we will take the NEW_TASK flow and try to find
        // a task for it. But save the task information so it can be used when creating the new task.
        if ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) == 0) {
            Slog.w(TAG, "startActivity called from finishing " + mSourceRecord
                    + "; forcing " + "Intent.FLAG_ACTIVITY_NEW_TASK for: " + mIntent);
            mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
            mNewTaskInfo = mSourceRecord.info;

            // It is not guaranteed that the source record will have a task associated with it. For,
            // example, if this method is being called for processing a pending activity launch, it
            // is possible that the activity has been removed from the task after the launch was
            // enqueued.
            final TaskRecord sourceTask = mSourceRecord.getTask();
            mNewTaskIntent = sourceTask != null ? sourceTask.intent : null;
        }
        mSourceRecord = null;
        mSourceStack = null;
    }
```

具体的判断如下：
- 当SourceRecord为Null，则说明不是由Activity来发起的远程调用，此时mSourceStack为Null；
- 当SourceRecord不是处于即将finish的状态，则优先复用SourceRecord所在的TaskRecord所在的ActivityStack；
- 当SourceRecord处于即将finish的状态，且当前启动Activity的`mLaunchFlags`**不带有**`FLAG_ACTIVITY_NEW_TASK`标志位，此时由于SourceRecord所在的TaskRecord为空，所以不会直接复用此SourceRecord所在的TaskRecord，而是为其`mLaunchFlags`加上`FLAG_ACTIVITY_NEW_TASK`标志位，表示要尝试寻找一个TaskRecord。然后将`mSourceStack`和`mSourceRecord`置为Null。

这里就完成了对

### ActivityStarter.startActivityUnchecked()-part3

```java
        mIntent.setFlags(mLaunchFlags);

        ActivityRecord reusedActivity = getReusableIntentActivity();
```

这一部分首先给当前用来启动Activity的Intent设置了之前第一部分处理过的`mLaunchFlags`，然后调用了`ActivityStarter.getReusableIntentActivity()`方法。此方法的具体作用是根据现有的`mLaunchFlags`等条件，来判断当前要启动的Activity是在现有的ActivityStack中寻找一个已经存在的ActivityRecord进行复用，还是将之前新创建的ActivityRecord插入到某个TaskRecord中去。当此方法返回Null时，则说明没有可以复用的ActivityRecord。

```java
    /**
     * Decide whether the new activity should be inserted into an existing task. Returns null
     * if not or an ActivityRecord with the task into which the new activity should be added.
     */
    private ActivityRecord getReusableIntentActivity() {
        // We may want to try to place the new activity in to an existing task.  We always
        // do this if the target activity is singleTask or singleInstance; we will also do
        // this if NEW_TASK has been requested, and there is not an additional qualifier telling
        // us to still place it in a new task: multi task, always doc mode, or being asked to
        // launch this as a new task behind the current one.
        boolean putIntoExistingTask = ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0 &&
                (mLaunchFlags & FLAG_ACTIVITY_MULTIPLE_TASK) == 0)
                || isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK);
        // If bring to front is requested, and no result is requested and we have not been given
        // an explicit task to launch in to, and we can find a task that was started with this
        // same component, then instead of launching bring that one to the front.
        putIntoExistingTask &= mInTask == null && mStartActivity.resultTo == null;
        ActivityRecord intentActivity = null;
        if (mOptions != null && mOptions.getLaunchTaskId() != -1) {
            final TaskRecord task = mSupervisor.anyTaskForIdLocked(mOptions.getLaunchTaskId());
            intentActivity = task != null ? task.getTopActivity() : null;
        } else if (putIntoExistingTask) {
            if (LAUNCH_SINGLE_INSTANCE == mLaunchMode) {
                // There can be one and only one instance of single instance activity in the
                // history, and it is always in its own unique task, so we do a special search.
               intentActivity = mSupervisor.findActivityLocked(mIntent, mStartActivity.info,
                       mStartActivity.isActivityTypeHome());
            } else if ((mLaunchFlags & FLAG_ACTIVITY_LAUNCH_ADJACENT) != 0) {
                // For the launch adjacent case we only want to put the activity in an existing
                // task if the activity already exists in the history.
                intentActivity = mSupervisor.findActivityLocked(mIntent, mStartActivity.info,
                        !(LAUNCH_SINGLE_TASK == mLaunchMode));
            } else {
                // Otherwise find the best task to put the activity in.
                intentActivity = mSupervisor.findTaskLocked(mStartActivity, mPreferredDisplayId);
            }
        }
        return intentActivity;
    }
```
首先，如果LaunchMode设置了`single_task`或这`single_instance`模式或者mLaunchFlag中含有`FLAG_ACTIVITY_NEW_TASK`但不含有`FLAG_ACTIVITY_MULTIPLE_TASK`多任务栈标志位时，这里都会在现有的ActivityStack中去尝试寻找一个可以复用的ActivityRecord。同时，如果此次启动Activity的ActivityOptions中有指明Activity要运行的TaskRecord的Id，则会尝试去找到对应的TaskRecord，然后拿到其TaskRecord的顶部ActivityRecord作为复用。

此方法最后一般都会返回某个TaskRecord的顶部ActivityRecord来作为尝试复用的ActivityRecord，其实除了想要尝试复用ActivityRecord，也是想要复用TaskRecord。


### ActivityStarter.startActivityUnchecked()-part4

```java
        if (reusedActivity != null) {
            // When the flags NEW_TASK and CLEAR_TASK are set, then the task gets reused but
            // still needs to be a lock task mode violation since the task gets cleared out and
            // the device would otherwise leave the locked task.
            if (mService.getLockTaskController().isLockTaskModeViolation(reusedActivity.getTask(),
                    (mLaunchFlags & (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))
                            == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))) {
                Slog.e(TAG, "startActivityUnchecked: Attempt to violate Lock Task Mode");
                return START_RETURN_LOCK_TASK_MODE_VIOLATION;
            }

            // True if we are clearing top and resetting of a standard (default) launch mode
            // ({@code LAUNCH_MULTIPLE}) activity. The existing activity will be finished.
            final boolean clearTopAndResetStandardLaunchMode =
                    (mLaunchFlags & (FLAG_ACTIVITY_CLEAR_TOP | FLAG_ACTIVITY_RESET_TASK_IF_NEEDED))
                            == (FLAG_ACTIVITY_CLEAR_TOP | FLAG_ACTIVITY_RESET_TASK_IF_NEEDED)
                    && mLaunchMode == LAUNCH_MULTIPLE;

            // If mStartActivity does not have a task associated with it, associate it with the
            // reused activity's task. Do not do so if we're clearing top and resetting for a
            // standard launchMode activity.
            if (mStartActivity.getTask() == null && !clearTopAndResetStandardLaunchMode) {
                mStartActivity.setTask(reusedActivity.getTask());
            }

            if (reusedActivity.getTask().intent == null) {
                // This task was started because of movement of the activity based on affinity...
                // Now that we are actually launching it, we can assign the base intent.
                reusedActivity.getTask().setIntent(mStartActivity);
            }

            // This code path leads to delivering a new intent, we want to make sure we schedule it
            // as the first operation, in case the activity will be resumed as a result of later
            // operations.
            if ((mLaunchFlags & FLAG_ACTIVITY_CLEAR_TOP) != 0
                    || isDocumentLaunchesIntoExisting(mLaunchFlags)
                    || isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK)) {
                final TaskRecord task = reusedActivity.getTask();

                // In this situation we want to remove all activities from the task up to the one
                // being started. In most cases this means we are resetting the task to its initial
                // state.
                final ActivityRecord top = task.performClearTaskForReuseLocked(mStartActivity,
                        mLaunchFlags);

                // The above code can remove {@code reusedActivity} from the task, leading to the
                // the {@code ActivityRecord} removing its reference to the {@code TaskRecord}. The
                // task reference is needed in the call below to
                // {@link setTargetStackAndMoveToFrontIfNeeded}.
                if (reusedActivity.getTask() == null) {
                    reusedActivity.setTask(task);
                }

                if (top != null) {
                    if (top.frontOfTask) {
                        // Activity aliases may mean we use different intents for the top activity,
                        // so make sure the task now has the identity of the new intent.
                        top.getTask().setIntent(mStartActivity);
                    }
                    deliverNewIntent(top);
                }
            }

	        .....
        }
```

在第四部分，会对之前几部分处理完得到的`mLaunchFlags`、`mLaunchMode`、`reusedActivity`等字段，在进行一些启动Activity前的处理，如对`FLAG_ACTIVITY_CLEAR_TOP`标志位的特殊处理和对已存在TaskRecord中的同一个Activity的ActivityRecord的onNewIntent用等。

 首先此处会判断要启动的Activity的launchMode是否为默认`LAUNCH_MULTIPLE`而且同时设置了`FLAG_ACTIVITY_CLEAR_TOP`标志位，用本地变量`clearTopAndResetStandardLaunchMode`标识。当`clearTopAndResetStandardLaunchMode`为true时，如果在之前找到的`reusedActivity`所对应的TaskRecord中含有同一个类名的ActivityRecord，则会首先移除此已存在的ActivityRecord之上的所有其他ActivityRecord，同时finish掉此已存在的ActivityRecord，之后再重新创建新的实例。

当设置了`FLAG_ACTIVITY_CLEAR_TOP`标志位或`mLaunchMode`为`LAUNCH_SINGLE_INSTANCE`和`LAUNCH_SINGLE_TASK`两者其一，这种情况下会优先去再reusedActivity所在的TaskRecord中是否存在同一个Activity类的实例，这里是借助`TaskRecord.performClearTaskForReuseLocked()`方法来查找的。如果存在这样的实列，则会清空此ActivityRecord实例之上的其他ActivityRecord，然后将此次启动的Activity的Intent通过远程回调`Activity.onNewIntent()`方法来传递到已经存在的实例。此外，这里要排除上面说的launchMode为默认`LAUNCH_MULTIPLE`而且同时设置了`FLAG_ACTIVITY_CLEAR_TOP`标志位的情况，在`TaskRecord.performClearTaskForReuseLocked()`方法中就会对这种情况进行一个特殊处理。

```java
    final ActivityRecord performClearTaskLocked(ActivityRecord newR, int launchFlags) {
        int numActivities = mActivities.size();
        for (int activityNdx = numActivities - 1; activityNdx >= 0; --activityNdx) {
            ActivityRecord r = mActivities.get(activityNdx);
            if (r.finishing) {
                continue;
            }
            if (r.realActivity.equals(newR.realActivity)) {
                // Here it is!  Now finish everything in front...
                final ActivityRecord ret = r;

                for (++activityNdx; activityNdx < numActivities; ++activityNdx) {
                    r = mActivities.get(activityNdx);
                    if (r.finishing) {
                        continue;
                    }
                    ActivityOptions opts = r.takeOptionsLocked();
                    if (opts != null) {
                        ret.updateOptionsLocked(opts);
                    }
                    if (mStack != null && mStack.finishActivityLocked(
                            r, Activity.RESULT_CANCELED, null, "clear-task-stack", false)) {
                        --activityNdx;
                        --numActivities;
                    }
                }

                // Finally, if this is a normal launch mode (that is, not
                // expecting onNewIntent()), then we will finish the current
                // instance of the activity so a new fresh one can be started.
                if (ret.launchMode == ActivityInfo.LAUNCH_MULTIPLE
                        && (launchFlags & Intent.FLAG_ACTIVITY_SINGLE_TOP) == 0
                        && !ActivityStarter.isDocumentLaunchesIntoExisting(launchFlags)) {
                    if (!ret.finishing) {
                        if (mStack != null) {
                            mStack.finishActivityLocked(
                                    ret, Activity.RESULT_CANCELED, null, "clear-task-top", false);
                        }
                        return null;
                    }
                }

                return ret;
            }
        }

        return null;
    }
```

### ActivityStarter.startActivityUnchecked()-part5

```java
        if (reusedActivity != null) {
			
			......
			
            mSupervisor.sendPowerHintForLaunchStartIfNeeded(false /* forceSend */, reusedActivity);

            reusedActivity = setTargetStackAndMoveToFrontIfNeeded(reusedActivity);

	        ......
        }
```

此部分首先调用了`ActivityStarter.setTargetStackAndMoveToFrontIfNeeded()`来设置当前要启动的Activity的对应目标ActivityStack的值，这里会将目标ActivityStack设置为reusedActivity所在的TaskRecord所对应的ActivityStack。

同时，当目前的前台ActivityStack的值和当前目标ActivityStack的值不一致，则需要将将目标ActivityStack设置为前台ActivityStack,，同时这里也会将目标ActivityStack中的目标TaskRecord设置为前台TaskRecord。这里会调用`ActivityStack.moveTaskToFrontLocked()`方法来完成这个操作。

```java
    final void moveTaskToFrontLocked(TaskRecord tr, boolean noAnimation, ActivityOptions options,
            AppTimeTracker timeTracker, String reason) {
        if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "moveTaskToFront: " + tr);

        final ActivityStack topStack = getDisplay().getTopStack();
        final ActivityRecord topActivity = topStack != null ? topStack.getTopActivity() : null;
        final int numTasks = mTaskHistory.size();
        final int index = mTaskHistory.indexOf(tr);
        if (numTasks == 0 || index < 0)  {
            // nothing to do!
            if (noAnimation) {
                ActivityOptions.abort(options);
            } else {
                updateTransitLocked(TRANSIT_TASK_TO_FRONT, options);
            }
            return;
        }

        if (timeTracker != null) {
            // The caller wants a time tracker associated with this task.
            for (int i = tr.mActivities.size() - 1; i >= 0; i--) {
                tr.mActivities.get(i).appTimeTracker = timeTracker;
            }
        }

        try {
            // Defer updating the IME target since the new IME target will try to get computed
            // before updating all closing and opening apps, which can cause the ime target to
            // get calculated incorrectly.
            getDisplay().deferUpdateImeTarget();

            // Shift all activities with this task up to the top
            // of the stack, keeping them in the same internal order.
            insertTaskAtTop(tr, null);

            // Don't refocus if invisible to current user
            final ActivityRecord top = tr.getTopActivity();
            if (top == null || !top.okToShowLocked()) {
                if (top != null) {
                    mStackSupervisor.mRecentTasks.add(top.getTask());
                }
                ActivityOptions.abort(options);
                return;
            }

            // Set focus to the top running activity of this stack.
            final ActivityRecord r = topRunningActivityLocked();
            mStackSupervisor.moveFocusableActivityStackToFrontLocked(r, reason);

            if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION, "Prepare to front transition: task=" + tr);
            if (noAnimation) {
                mWindowManager.prepareAppTransition(TRANSIT_NONE, false);
                if (r != null) {
                    mStackSupervisor.mNoAnimActivities.add(r);
                }
                ActivityOptions.abort(options);
            } else {
                updateTransitLocked(TRANSIT_TASK_TO_FRONT, options);
            }
            // If a new task is moved to the front, then mark the existing top activity as
            // supporting

            // picture-in-picture while paused only if the task would not be considered an oerlay
            // on top
            // of the current activity (eg. not fullscreen, or the assistant)
            if (canEnterPipOnTaskSwitch(topActivity, tr, null /* toFrontActivity */,
                    options)) {
                topActivity.supportsEnterPipOnTaskSwitch = true;
            }

            mStackSupervisor.resumeFocusedStackTopActivityLocked();
            EventLog.writeEvent(EventLogTags.AM_TASK_TO_FRONT, tr.userId, tr.taskId);

            mService.mTaskChangeNotificationController.notifyTaskMovedToFront(tr.taskId);
        } finally {
            getDisplay().continueUpdateImeTarget();
        }
    }
```



### ActivityStarter.startActivityUnchecked()-part6

```java
		if (reusedActivity != null) {
			
			......
			
            if (reusedActivity != null) {
                setTaskFromIntentActivity(reusedActivity);

                if (!mAddingToTask && mReuseTask == null) {
                    // We didn't do anything...  but it was needed (a.k.a., client don't use that
                    // intent!)  And for paranoia, make sure we have correctly resumed the top activity.

                    resumeTargetStackIfNeeded();
                    if (outActivity != null && outActivity.length > 0) {
                        outActivity[0] = reusedActivity;
                    }

                    return mMovedToFront ? START_TASK_TO_FRONT : START_DELIVERED_TO_TOP;
                }
            }
        }
```

上述五个部分已经经历了如下操作：
- 对`mLaunchFlags`的调整
- 对远程调用者的来源ActivityStack的处理
- 对可能存在的可以复用的ActivityRecord（TaskRecord和ActivityStack）进行判断
- 当可以复用的ActivityRecord存在时，对`FLAG_ACTIVITY_CLEAR_TOP`标志位或`mLaunchMode`为`LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK`的onNewIntent()的准备处理
- 当可以复用的ActivityRecord存在时，对目标ActivityStack的转至前台

接下来，如果复用的ActivityRecord存在时，还会调用`ActivityStarter.setTaskFromIntentActivity(reusedActivity)`方法来对将要启动的Activity所需要关联的TaskRecord进行判断处理。之前已经处理过了ActivityStack，下一步就是TaskRecord了。这部分也会处理`FLAG_ACTIVITY_SINGLE_TOP`标志位是否需要生成`onNewIntent()`请求。

```java
    private void setTaskFromIntentActivity(ActivityRecord intentActivity) {
        if ((mLaunchFlags & (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))
                == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK)) {
            // The caller has requested to completely replace any existing task with its new
            // activity. Well that should not be too hard...
            // Note: we must persist the {@link TaskRecord} first as intentActivity could be
            // removed from calling performClearTaskLocked (For example, if it is being brought out
            // of history or if it is finished immediately), thus disassociating the task. Also note
            // that mReuseTask is reset as a result of {@link TaskRecord#performClearTaskLocked}
            // launching another activity.
            // TODO(b/36119896):  We shouldn't trigger activity launches in this path since we are
            // already launching one.
            final TaskRecord task = intentActivity.getTask();
            task.performClearTaskLocked();
            mReuseTask = task;
            mReuseTask.setIntent(mStartActivity);
        } else if ((mLaunchFlags & FLAG_ACTIVITY_CLEAR_TOP) != 0
                || isLaunchModeOneOf(LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK)) {
            ActivityRecord top = intentActivity.getTask().performClearTaskLocked(mStartActivity,
                    mLaunchFlags);
            if (top == null) {
                // A special case: we need to start the activity because it is not currently
                // running, and the caller has asked to clear the current task to have this
                // activity at the top.
                mAddingToTask = true;

                // We are no longer placing the activity in the task we previously thought we were.
                mStartActivity.setTask(null);
                // Now pretend like this activity is being started by the top of its task, so it
                // is put in the right place.
                mSourceRecord = intentActivity;
                final TaskRecord task = mSourceRecord.getTask();
                if (task != null && task.getStack() == null) {
                    // Target stack got cleared when we all activities were removed above.
                    // Go ahead and reset it.
                    mTargetStack = computeStackFocus(mSourceRecord, false /* newTask */,
                            mLaunchFlags, mOptions);
                    mTargetStack.addTask(task,
                            !mLaunchTaskBehind /* toTop */, "startActivityUnchecked");
                }
            }
        } else if (mStartActivity.realActivity.equals(intentActivity.getTask().realActivity)) {
            // In this case the top activity on the task is the same as the one being launched,
            // so we take that as a request to bring the task to the foreground. If the top
            // activity in the task is the root activity, deliver this new intent to it if it
            // desires.
            if (((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
                        || LAUNCH_SINGLE_TOP == mLaunchMode)
                    && intentActivity.realActivity.equals(mStartActivity.realActivity)) {
                if (intentActivity.frontOfTask) {
                    intentActivity.getTask().setIntent(mStartActivity);
                }
                deliverNewIntent(intentActivity);
            } else if (!intentActivity.getTask().isSameIntentFilter(mStartActivity)) {
                // In this case we are launching the root activity of the task, but with a
                // different intent. We should start a new instance on top.
                mAddingToTask = true;
                mSourceRecord = intentActivity;
            }
        } else if ((mLaunchFlags & FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) == 0) {
            // In this case an activity is being launched in to an existing task, without
            // resetting that task. This is typically the situation of launching an activity
            // from a notification or shortcut. We want to place the new activity on top of the
            // current task.
            mAddingToTask = true;
            mSourceRecord = intentActivity;
        } else if (!intentActivity.getTask().rootWasReset) {
            // In this case we are launching into an existing task that has not yet been started
            // from its front door. The current task has been brought to the front. Ideally,
            // we'd probably like to place this new task at the bottom of its stack, but that's
            // a little hard to do with the current organization of the code so for now we'll
            // just drop it.
            intentActivity.getTask().setIntent(mStartActivity);
        }
    }
```

在`ActivityStarter.setTaskFromIntentActivity()`方法中，会分为好几部分来完成对ActivityRecord将要放置TaskRecord进行判断，这边都是默认使用reusedActivity所在的TaskRecord：
- 当`mLaunchFlags`中带有`FLAG_ACTIVITY_NEW_TASK`和`FLAG_ACTIVITY_CLEAR_TASK`标志位时，此次启动将清空之前找到的reusedActivity所在的TaskRecord中的所有已存在的ActivityRecord，然后将`mReuseTask`赋值为reusedActivity所在的TaskRecord，同时将此TaskRecord的启动Intent换成将要启动的Activity所对应的Intent；
- 当`mLaunchFlags`中带有`FLAG_ACTIVITY_CLEAR_TOP`或者`mLaunchMode`为`LAUNCH_SINGLE_INSTANCE`或` LAUNCH_SINGLE_TASK`俩者其一，那么这里也会在reusedActivity所在TaskRecord中去放置当前要启动的ActivityRecord还是复用之前存在的ActivityRecord，并触发onNewIntent()方法。这里的处理方式和之前第四步在reusedActivity不为空的时候，处理需要onNewIntent()的情形一致，首先在reusedActivity所在的TaskRecord中找到第一个可以复用的同类名的ActivityRecord，如果存在则说明都不处理，因为之前第四部已经处理过了。如果不存在，则说明当前TaskRecord中还不存在可以复用的ActivityRecord，所以之后需要再reusedActivity所在的TaskRecord中加入当前要创建的Activity的ActivityRecord，这里会用字段`mAddingToTask`标识。
- 若当前要启动的Activity和reusedActivity所在的TaskRecord创建时所启动的Activity是同一个时，此处会判断如果当前`mLaunchFlags`含有`FLAG_ACTIVITY_SINGLE_TOP`标志位，且当前对应reusedActivity（也就是对应TaskRecord的顶部Activity）和要启动的Activity是同一个Activity，这里也会生成一个onNewIntent()请求。若不满足，则会判断reusedActivity所在的TaskRecord启动时的Intent和当前要启动的Activity的Intent是否相等，如果不相等，则说明虽然有相同的Activity，但来自于不同的Intent，需要重新生成一个ActivityRecord然后添加到此TaskRecord之中，这里也是用字段`mAddingToTask`标识。


再完成了`ActivityStarter. setTaskFromIntentActivity(reusedActivity)`方法对TaskRecord的处理后，如果需要添加新的ActivityRecord到TaskRecord中还是指明复用一个已有TaskRecord来放置新的ActivityRecord，这里的`mReuseTask`就会不为null或者`mAddingToTask`字段为`true`，说明Activity的启动还需要继续处理。

### ActivityStarter.startActivityUnchecked()-part7

之前几个步骤已经完成了如下操作：
- 对`mLaunchFlags`的调整
- 对远程调用者的来源ActivityStack的处理
- 对可能存在的可以复用的ActivityRecord（TaskRecord和ActivityStack）进行判断
- 当可以复用的ActivityRecord存在时，对`FLAG_ACTIVITY_CLEAR_TOP`标志位或`mLaunchMode`为`LAUNCH_SINGLE_INSTANCE, LAUNCH_SINGLE_TASK`的onNewIntent()的准备处理
- 当可以复用的ActivityRecord存在时，对目标ActivityStack的转至前台
- 当可以复用的ActivityRecord存在时，对目标TaskRecord的情况进行判断，也处理了`FLAG_ACTIVITY_SINGLE_TOP`标志位的`onNewIntent()`的请求的生成

接下来会继续处理Activity的启动。

```java
        // If the activity being launched is the same as the one currently at the top, then
        // we need to check if it should only be launched once.
        final ActivityStack topStack = mSupervisor.mFocusedStack;
        final ActivityRecord topFocused = topStack.getTopActivity();
        final ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(mNotTop);
        final boolean dontStart = top != null && mStartActivity.resultTo == null
                && top.realActivity.equals(mStartActivity.realActivity)
                && top.userId == mStartActivity.userId
                && top.app != null && top.app.thread != null
                && ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
                || isLaunchModeOneOf(LAUNCH_SINGLE_TOP, LAUNCH_SINGLE_TASK));
        if (dontStart) {
            // For paranoia, make sure we have correctly resumed the top activity.
            topStack.mLastPausedActivity = null;
            if (mDoResume) {
                mSupervisor.resumeFocusedStackTopActivityLocked();
            }
            ActivityOptions.abort(mOptions);
            if ((mStartFlags & START_FLAG_ONLY_IF_NEEDED) != 0) {
                // We don't need to start a new activity, and the client said not to do
                // anything if that is the case, so this is it!
                return START_RETURN_INTENT_TO_CALLER;
            }

            deliverNewIntent(top);

            // Don't use mStartActivity.task to show the toast. We're not starting a new activity
            // but reusing 'top'. Fields in mStartActivity may not be fully initialized.
            mSupervisor.handleNonResizableTaskIfNeeded(top.getTask(), preferredWindowingMode,
                    preferredLaunchDisplayId, topStack);

            return START_DELIVERED_TO_TOP;
        }
```

这里再一次对当前设备的顶部的ActivityStack的顶部ActivityRecord进行了一个判断，如果当前的顶部的ActivityRecord和当前要启动的Activity是同一个Activity，而且`mLaunchFlags`带有`FLAG_ACTIVITY_SINGLE_TOP`或者`mLaunchMode`中是`LAUNCH_SINGLE_INSTANCE`或`LAUNCH_SINGLE_TASK`两者之一，则说明这里是需要onNewIntent()请求的，所以这里也会调用`ActivityStarter.deliverNewIntent()`方法来完成。**但之前处理步骤时，已经有部分情况已经调用过此方法一次了，那么会不会存在重复调用呢？当然是不会。**后续介绍`ActivityStarter.deliverNewIntent()`方法时在做详细说明。

同时，除了这上述几种情况是要生成`onNewIntent()`请求之外，还有`mLaunchFlags`带有`FLAG_ACTIVITY_CLEAR_TOP`标志位时也需要生成，那么这里为什么不处理呢？这是由于这种情况下，会存在要先将当前Top ActivityRecord是对用户不可见的，通过移除其之上的其他ActivityRecord才到TaskRecord顶部的，所以需要先resume之后，在传递`onNewIntent()`请求。这一步也是在之前的就已经调用`ActivityStarter.deliverNewIntent()`方法来将要传递的**Intent保存在本地，然后再后续Activity准备resume时在传递过去**。

#### `ActivityStarter.deliverNewIntent()`传递`onNewIntent()`请求

之前说到可能会要生成`onNewIntent()`请求有如下情况：
- 当`mLaunchFlags`带有`FLAG_ACTIVITY_SINGLE_TOP`标志位，而且当前顶部ActivityRecord对应的Activity就算当前要启动的Activity的同一个Activity类。
- 当`mLaunchFlags`带有`FLAG_ACTIVITY_CLEAR_TOP`标志位，然后同时设置了`mLaunchMode`为`LAUNCH_SINGLE_TOP`或者还带有`FLAG_ACTIVITY_SINGLE_TOP`标志位，同时目标TaskRecord中含有当前要启动的Activity所对应的ActivityRecord。
- 当`mLaunchMode`中是`LAUNCH_SINGLE_INSTANCE`或`LAUNCH_SINGLE_TASK`，同时当前目标TaskRecord的顶部ActivityRecord就是当前要启动的Activity的同一个Activity类。

这里都会借助`ActivityStarter.deliverNewIntent()`来传递`onNewIntent()`请求，下面看到此方法：
```java
    private void deliverNewIntent(ActivityRecord activity) {
        if (mIntentDelivered) {
            return;
        }

        ActivityStack.logStartActivity(AM_NEW_INTENT, activity, activity.getTask());
        activity.deliverNewIntentLocked(mCallingUid, mStartActivity.intent,
                mStartActivity.launchedFromPackage);
        mIntentDelivered = true;
    }
```

在此方法中，会首先去判断当前**ActivityStarter所对应的Intent是否已经传递过**了，如果传递过，则不会再去传递第二次。这也就是之前说的调用了两次`ActivityStarter.deliverNewIntent()`方法，但不会重复发送的原因。

当对应的Intent没有经过传递，则会调用`ActivityRecord.deliverNewIntentLocked()`方法，来传递此Intent。然后将此Intent标记为已经传递过的状态。
```java
    /**
     * Deliver a new Intent to an existing activity, so that its onNewIntent()
     * method will be called at the proper time.
     */
    final void deliverNewIntentLocked(int callingUid, Intent intent, String referrer) {
        // The activity now gets access to the data associated with this Intent.
        service.grantUriPermissionFromIntentLocked(callingUid, packageName,
                intent, getUriPermissionsLocked(), userId);
        final ReferrerIntent rintent = new ReferrerIntent(intent, referrer);
        boolean unsent = true;
        final boolean isTopActivityWhileSleeping = isTopRunningActivity() && isSleeping();

        // We want to immediately deliver the intent to the activity if:
        // - It is currently resumed or paused. i.e. it is currently visible to the user and we want
        //   the user to see the visual effects caused by the intent delivery now.
        // - The device is sleeping and it is the top activity behind the lock screen (b/6700897).
        if ((mState == RESUMED || mState == PAUSED
                || isTopActivityWhileSleeping) && app != null && app.thread != null) {
            try {
                ArrayList<ReferrerIntent> ar = new ArrayList<>(1);
                ar.add(rintent);
                service.getLifecycleManager().scheduleTransaction(app.thread, appToken,
                        NewIntentItem.obtain(ar, mState == PAUSED));
                unsent = false;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception thrown sending new intent to " + this, e);
            } catch (NullPointerException e) {
                Slog.w(TAG, "Exception thrown sending new intent to " + this, e);
            }
        }
        if (unsent) {
            addNewIntentLocked(rintent);
        }
    }
```

可以看到此方法中将要传递的Intent封装成了一个`ReferrerIntent`对象，然后只有再当前接受此封装后的Intent对象的Activity是**对用户可见的状态（resume和pause）**或者当前**Activity运行在顶部且处于睡眠状态**，也就是**锁屏状态**下时才会直接去远程调用到App端，进行`Activity.onNewIntent()`操作。
注意这里的远程回调，和之前的Android 6.0的（Api23）已经有区别了，之前是远程回调对应的方法，此处是`scheduleNewIntent()`，现在是统一的`scheduleTransaction()`方法。

当这里的顶部的ActivityRecord**不满足可以直接远程调用的情况**，这里会将此次生成的封装后的Intent对象**保存在本地的一个列表中**，然后等待后续此Activity在resume过程中，在进行远程回调。

### ActivityStarter.startActivityUnchecked()-part8

上一部分，如果确认了只需要生成onNewIntent()请求的情况下，就已经结束了本次Activity的启动。而对于其他情况，可能需要新建一个ActivityRecord来添加到TaskRecord中并执行resume操作或者将现有的ActivityRecord执行resume操作。

之前已经初步确认过目标TaskRecord的选择，不过这里还是可能存在没有找到目标TaskRecord的情况，同时设置了`FLAG_ACTIVITY_NEW_TASK`标志位，所以这里会尝试创建一个TaskRecord对象，这里是调用`ActivityStarter.setTaskFromReuseOrCreateNewTask()`方法来完成的。
同时目标的TaskRecord也可能来自于SourceRecord的TaskRecord，这里是调用`ActivityStarter.setTaskFromSourceRecord()`来完成的。

```java
        // Should this be considered a new task?
        int result = START_SUCCESS;
        if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
                && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
            newTask = true;
            result = setTaskFromReuseOrCreateNewTask(taskToAffiliate, topStack);
        } else if (mSourceRecord != null) {
            result = setTaskFromSourceRecord();
        } else if (mInTask != null) {
            result = setTaskFromInTask();
        } else {
            // This not being started from an existing activity, and not part of a new task...
            // just put it in the top task, though these days this case should never happen.
            setTaskToCurrentTopOrCreateNewTask();
        }
        if (result != START_SUCCESS) {
            return result;
        }

```

### ActivityStarter.startActivityUnchecked()-part9

在上一步完成了最后的目标的TaskRecord的寻找或者创建后，终于来到了Activity的启动的倒数第二步，这里会调用`ActivityStack.startActivityLocked()`方法来完成要启动的Activity的相关的AppWindowToken的创建和Display中保存相关对应的ActivityRecord的Token和AppWindowToken。

```java
        mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition,
                mOptions);
```

