---
title: ActivityManagerService
tags: 
 - android
 - framework
categories:
 - 笔记
---
## ActivityManagerService

AMS是通过ActivityStack及其他数据结构来记录，管理系统中的Activity及其他组件状态的，并提供查询功能的一个系统服务。

- 组件状态管理：包括四大组件的开启，关闭等一系列操作

  如startActivity,activityPaused,startService,stopService,removeContentProvider等

- 组件状态查询：查询组件当前运行等情况。如getCallingActivity,getService等

- Task相关：包括removeTask,removeSubTask,moveTaskBackwards,moveTaskToFront等

### AMS启动流程

AMS寄存与system server中，它会在系统启动时，创建一个线程来循环处理客户端的请求

```java
/frameworks/base/services/java/com/android/server/SystemServer.java
private void startBootstrapServices() {
    ...
    mActivityManagerService = ActivityManagerService.Lifecycle.startService(
                  mSystemServiceManager, atm);//启动AMS
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);//向ServiceManager注册AMS
    ...
}
```

```java
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
    public static final class Lifecycle extends SystemService {
        private final ActivityManagerService mService;
        private static ActivityTaskManagerService sAtm;

        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityManagerService(context, sAtm);
        }

        public static ActivityManagerService startService(
                SystemServiceManager ssm, ActivityTaskManagerService atm) {
            sAtm = atm;
            return ssm.startService(ActivityManagerService.Lifecycle.class).getService();//创建要启动的AMS
        }

        @Override
        public void onStart() {
            mService.start();//会被SystemServiceManager调用，启动自己，也就是AMS
        }

        @Override
        public void onBootPhase(int phase) {
            mService.mBootPhase = phase;
            if (phase == PHASE_SYSTEM_SERVICES_READY) {
                mService.mBatteryStatsService.systemServicesReady();
                mService.mServices.systemServicesReady();
            } else if (phase == PHASE_ACTIVITY_MANAGER_READY) {
                mService.startBroadcastObservers();
            } else if (phase == PHASE_THIRD_PARTY_APPS_CAN_START) {
                mService.mPackageWatchdog.onPackagesReady();
            }
        }

        @Override
        public void onCleanupUser(int userId) {
            mService.mBatteryStatsService.onCleanupUser(userId);
        }

        public ActivityManagerService getService() {
            return mService;//返回自己——AMS
        }
    }
```

```java
frameworks/base/services/core/java/com/android/server/SystemServiceManager.java
	public <T extends SystemService> T startService(Class<T> serviceClass) {
    	try {
            final String name = serviceClass.getName();
            Slog.i(TAG, "Starting " + name);
            Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartService " + name);

            // Create the service.
            if (!SystemService.class.isAssignableFrom(serviceClass)) {
                throw new RuntimeException("Failed to create " + name
                        + ": service must extend " + SystemService.class.getName());
            }
            final T service;
            try {
                Constructor<T> constructor = serviceClass.getConstructor(Context.class);
                service = constructor.newInstance(mContext); //通过反射的方式创建service
            } catch (InstantiationException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service could not be instantiated", ex);
            } catch (IllegalAccessException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service must have a public constructor with a Context argument", ex);
            } catch (NoSuchMethodException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service must have a public constructor with a Context argument", ex);
            } catch (InvocationTargetException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service constructor threw an exception", ex);
            }

            startService(service);//调用startService(@NonNull final SystemService service)
            return service;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        }
    }

    public void startService(@NonNull final SystemService service) {
        // Register it.
        mServices.add(service);//注册AMS到mServices中
        // Start it.
        long time = SystemClock.elapsedRealtime();
        try {
            service.onStart();//调用ActivityManagerService.Lifecycle.onStart()
        } catch (RuntimeException ex) {
            throw new RuntimeException("Failed to start service " + service.getClass().getName()
                    + ": onStart threw an exception", ex);
        }
        warnIfTooLong(SystemClock.elapsedRealtime() - time, service, "onStart");
    }
```

至此，AMS启动流程结束

### AMS的功能

#### ActivityTaskManagerService

上节AMS在启动过程中，在AMS的构造方法中就已经完成了ActivityTaskManagerService的初始化工作了

AMS的构造方法只有两个参数，Context systemContext和ActivityTaskManagerService atm，ActivityTaskManagerService的重要性不言而喻

```java
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
public ActivityTaskManagerService mActivityTaskManager;//声明mActivityTaskManager来管理ActivityStack

public ActivityManagerService(Context systemContext, ActivityTaskManagerService atm) {
    ...
    //省略了很多启动其他服务的代码，目前仅关注mActivityTaskManager
        mActivityTaskManager = atm;//atm是system server启动AMS时所传进来的参数
        mActivityTaskManager.initialize(mIntentFirewall, mPendingIntentController,
                DisplayThread.get().getLooper());//初始化mActivityTaskManager
    ...
}
```

接下来看一下ActivityTaskManagerService初始化工作

```java
    public void initialize(IntentFirewall intentFirewall, PendingIntentController intentController,
            Looper looper) {
        mH = new H(looper);//looper是DisplayThread.get().getLooper()
        mUiHandler = new UiHandler();
        mIntentFirewall = intentFirewall;
        final File systemDir = SystemServiceManager.ensureSystemDir();
        mAppWarnings = new AppWarnings(this, mUiContext, mH, mUiHandler, systemDir);
        mCompatModePackages = new CompatModePackages(this, systemDir, mH);
        mPendingIntentController = intentController;

        mTempConfig.setToDefaults();
        mTempConfig.setLocales(LocaleList.getDefault());
        mConfigurationSeq = mTempConfig.seq = 1;
        mStackSupervisor = createStackSupervisor();//创建ActivityStackSupervisor
        mRootActivityContainer = new RootActivityContainer(this);//创建RootActivityContainer
        mRootActivityContainer.onConfigurationChanged(mTempConfig);

        mTaskChangeNotificationController =
                new TaskChangeNotificationController(mGlobalLock, mStackSupervisor, mH);
        mLockTaskController = new LockTaskController(mContext, mStackSupervisor, mH);
        mActivityStartController = new ActivityStartController(this);
        mRecentTasks = createRecentTasks();//创建RecentTasks
        mStackSupervisor.setRecentTasks(mRecentTasks);
        mVrController = new VrController(mGlobalLock);
        mKeyguardController = mStackSupervisor.getKeyguardController();
    }
```

> 可以看出ActivityTaskManagerService是一个用于管理Activity及其Container（Task，Stack，displays）的系统服务，相比AMS，它的职责更专一

#### ActivityTask

ActivityTask是管理一个堆栈中所有的Activity的数据结构，下列内容是从ActivityTask提取出来的

```java
	frameworks/base/services/core/java/com/android/server/wm/ActivityStack.java    
	ActivityStack(ActivityDisplay display, int stackId, ActivityStackSupervisor supervisor,
            int windowingMode, int activityType, boolean onTop) {
        mStackSupervisor = supervisor;
        mService = supervisor.mService;//mService是在ActivityTaskManagerService创建ActivityStackSupervisor的时候传递来的
        mRootActivityContainer = mService.mRootActivityContainer;
        mHandler = new ActivityStackHandler(supervisor.mLooper);
        mWindowManager = mService.mWindowManager;
        mStackId = stackId;
        mCurrentUser = mService.mAmInternal.getCurrentUserId();
        // Set display id before setting activity and window type to make sure it won't affect
        // stacks on a wrong display.
        mDisplayId = display.mDisplayId;
        setActivityType(activityType);
        createTaskStack(display.mDisplayId, onTop, mTmpRect2);
        setWindowingMode(windowingMode, false /* animate */, false /* showRecents */,
                false /* enteringSplitScreenMode */, false /* deferEnsuringVisibility */,
                true /* creating */);
        display.addChild(this, onTop ? POSITION_TOP : POSITION_BOTTOM);
    }
```

**维护Activity状态**

```java
   enum ActivityState {
        INITIALIZING,
        RESUMED,
        PAUSING,
        PAUSED,
        STOPPING,
        STOPPED,
        FINISHING,
        DESTROYING,
        DESTROYED,
        RESTARTING_PROCESS
    }
```

**一些ArrayList**

```java
	/**
     * The back history of all previous (and possibly still
     * running) activities.  It contains #TaskRecord objects.
     */
    private final ArrayList<TaskRecord> mTaskHistory = new ArrayList<>();

    /**
     * List of running activities, sorted by recent usage.
     * The first entry in the list is the least recently used.
     * It contains HistoryRecord objects.
     */
    private final ArrayList<ActivityRecord> mLRUActivities = new ArrayList<>();
```

**特殊状态的Activity**

```java
    /**
     * When we are in the process of pausing an activity, before starting the
     * next one, this variable holds the activity that is currently being paused.
     */
    ActivityRecord mPausingActivity = null;

    /**
     * This is the last activity that we put into the paused state.  This is
     * used to determine if we need to do an activity transition while sleeping,
     * when we normally hold the top activity paused.
     */
    ActivityRecord mLastPausedActivity = null;

    /**
     * Activities that specify No History must be removed once the user navigates away from them.
     * If the device goes to sleep with such an activity in the paused state then we save it here
     * and finish it later if another activity replaces it on wakeup.
     */
    ActivityRecord mLastNoHistoryActivity = null;

    /**
     * Current activity that is resumed, or null if there is none.
     */
    ActivityRecord mResumedActivity = null;
```

> AMS是通过ActivityStack来记录、管理系统中的Activity（和其他组件）状态，并提供查询功能的一个系统服务，而且只是负责保管Activity（和其他组件）状态信息，而像Activity中描述的UI界面如何在屏幕上显示等工作是有WMS和SF来完成的

#### StartActivity流程

本节希望通过StartActivity理清AMS工作的流程，方法会很多，这里只总结重要的流程（基于androidR版本代码）

先从Activity开始吧

##### Activity#startActivityForResult

 frameworks/base/core/java/android/app/Activity.java

```java
	public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }

    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (mIntent != null && mIntent.hasExtra(AutofillManager.EXTRA_RESTORE_SESSION_TOKEN)
                  && mIntent.hasExtra(AutofillManager.EXTRA_RESTORE_CROSS_ACTIVITY)) {
            if (TextUtils.equals(getPackageName(),
                    intent.resolveActivity(getPackageManager()).getPackageName())) {
                // Apply Autofill restore mechanism on the started activity by startActivity()
                final IBinder token =
                mIntent.getIBinderExtra(AutofillManager.EXTRA_RESTORE_SESSION_TOKEN);
                // Remove restore ability from current activity
                mIntent.removeExtra(AutofillManager.EXTRA_RESTORE_SESSION_TOKEN);
                mIntent.removeExtra(AutofillManager.EXTRA_RESTORE_CROSS_ACTIVITY);
                // Put restore token
                intent.putExtra(AutofillManager.EXTRA_RESTORE_SESSION_TOKEN, token);
                intent.putExtra(AutofillManager.EXTRA_RESTORE_CROSS_ACTIVITY, true);
            }
        }
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
        if (mParent == null) { //第一次启动会执行该分支的代码
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

##### Instrumentation#execStartActivity

 frameworks/base/core/java/android/app/Instrumentation.java

```java
	public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        //省略部分代码
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            int result = ActivityTaskManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options); //进入到ActivityTaskManagerService
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

##### ActivityTaskManagerService#startActivityAsUser

frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java

接下来就进入到系统进程了

```java
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());//需要加上调用者的用户ID，后续用来进行权限检查
    }

	public int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
                true /*validateIncomingUser*/);//需要验证调用的用户
    }

	private int startActivityAsUser(IApplicationThread caller, String callingPackage,
            @Nullable String callingFeatureId, Intent intent, String resolvedType,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
        assertPackageMatchesCallingUid(callingPackage);
        enforceNotIsolatedCaller("startActivityAsUser");

        userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        // TODO: Switch to user app stacks here.
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setCallingFeatureId(callingFeatureId)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setUserId(userId) //该配置是R版本新增的
                .execute();
        //返回ActivityStart.execute()
    }
```

getActivityStartController().obtainStarter(intent, "startActivityAsUser")返回的是ActivityStart

##### ActivityStarter#executeRequest

frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java

该部分代码相较与之前变动还是比较大的,核心代码移植到了executeRequest(mRequest)中,篇幅有限,后面只贴关键代码

```java
	int execute() {
        try {
            int res;
            synchronized (mService.mGlobalLock) {
				......                
				//判断目标Activity是不是重量级的，如果系统中已经存在的重量级进程不是即将要启动的这个，那么重新给intent赋值
                res = resolveToHeavyWeightSwitcherIfNeeded();
                
                if (res != START_SUCCESS) {
                    return res;
                }
                res = executeRequest(mRequest);//这是启动activity的关键方法
                ......
            }
        } finally {
            onExecutionComplete();
        }
    }
```

```java
    private int executeRequest(Request request) {
		......
		//首先要确定调用者必须存在,否则返回START_PERMISSION_DENIED
        WindowProcessController callerApp = null;
        if (caller != null) {
            callerApp = mService.getProcessController(caller);
            if (callerApp != null) {
                callingPid = callerApp.getPid();
                callingUid = callerApp.mInfo.uid;
            } else {
                Slog.w(TAG, "Unable to find app for caller " + caller + " (pid=" + callingPid
                        + ") when starting: " + intent.toString());
                err = ActivityManager.START_PERMISSION_DENIED;
            }
        }
		......
		//FLAG_ACTIVITY_FORWARD_RESULT具有跨越式传递作用,针对此情况改变调用者
        final int launchFlags = intent.getFlags();
        if ((launchFlags & Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0 && sourceRecord != null) {
            // Transfer the result target from the source activity to the new one being started,
            // including any failures.
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
                // The new activity is being launched from the same uid as the previous activity
                // in the flow, and asking to forward its result back to the previous.  In this
                // case the activity is serving as a trampoline between the two, so we also want
                // to update its launchedFromPackage to be the same as the previous activity.
                // Note that this is safe, since we know these two packages come from the same
                // uid; the caller could just as well have supplied that same package name itself
                // . This specifially deals with the case of an intent picker/chooser being
                // launched in the app flow to redirect to an activity picked by the user, where
                // we want the final activity to consider it to have been launched by the
                // previous app activity.
                callingPackage = sourceRecord.launchedFromPackage;
                callingFeatureId = sourceRecord.launchedFromFeatureId;
            }
        }
		
        //新建一个ActivityRecord,记录当前的判断结果,接下来调用startActivityUnchecked
        final ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
                callingPackage, callingFeatureId, intent, resolvedType, aInfo,
                mService.getGlobalConfiguration(), resultRecord, resultWho, requestCode,
                request.componentSpecified, voiceSession != null, mSupervisor, checkedOptions,
                sourceRecord);
        mLastStartActivityRecord = r;

        mService.onStartActivitySetDidAppSwitch();
        mController.doPendingActivityLaunches(false);

        mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
                request.voiceInteractor, startFlags, true /* doResume */, checkedOptions, inTask,
                restrictedBgActivity, intentGrants);

        if (request.outActivity != null) {
            request.outActivity[0] = mLastStartActivityRecord;
        }

        return mLastStartActivityResult;
    }
```

```java
    private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                int startFlags, boolean doResume, ActivityOptions options, Task inTask,
                boolean restrictedBgActivity, NeededUriGrants intentGrants) {
        int result = START_CANCELED;
        final ActivityStack startedActivityStack;
        try {
            mService.deferWindowLayout();
            Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "startActivityInner");
            result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, restrictedBgActivity, intentGrants);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
            startedActivityStack = handleStartResult(r, result);
            mService.continueWindowLayout();
        }

        postStartActivityProcessing(r, result, startedActivityStack);

        return result;
    }
```

##### ActivityStarter#startActivityInner

```java
    int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, Task inTask,
            boolean restrictedBgActivity, NeededUriGrants intentGrants) {
        setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
                voiceInteractor, restrictedBgActivity);

        computeLaunchingTaskFlags();

        computeSourceStack();

        mIntent.setFlags(mLaunchFlags);

        final Task reusedTask = getReusableTask();

        // If requested, freeze the task list
        if (mOptions != null && mOptions.freezeRecentTasksReordering()
                && mSupervisor.mRecentTasks.isCallerRecents(r.launchedFromUid)
                && !mSupervisor.mRecentTasks.isFreezeTaskListReorderingSet()) {
            mFrozeTaskList = true;
            mSupervisor.mRecentTasks.setFreezeTaskListReordering();
        }

        // Compute if there is an existing task that should be used for.
        final Task targetTask = reusedTask != null ? reusedTask : computeTargetTask();
        final boolean newTask = targetTask == null;
        mTargetTask = targetTask;

        computeLaunchParams(r, sourceRecord, targetTask);

        // Check if starting activity on given task or on a new task is allowed.
        int startResult = isAllowedToStart(r, newTask, targetTask);
        if (startResult != START_SUCCESS) {
            return startResult;
        }

        final ActivityRecord targetTaskTop = newTask
                ? null : targetTask.getTopNonFinishingActivity();
        if (targetTaskTop != null) {
            // Recycle the target task for this launch.
            startResult = recycleTask(targetTask, targetTaskTop, reusedTask, intentGrants);
            if (startResult != START_SUCCESS) {
                return startResult;
            }
        } else {
            mAddingToTask = true;
        }

        // If the activity being launched is the same as the one currently at the top, then
        // we need to check if it should only be launched once.
        final ActivityStack topStack = mRootWindowContainer.getTopDisplayFocusedStack();
        if (topStack != null) {
            startResult = deliverToCurrentTopIfNeeded(topStack, intentGrants);
            if (startResult != START_SUCCESS) {
                return startResult;
            }
        }

        if (mTargetStack == null) {
            mTargetStack = getLaunchStack(mStartActivity, mLaunchFlags, targetTask, mOptions);
        }
        if (newTask) {
            final Task taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
                    ? mSourceRecord.getTask() : null;
            setNewTask(taskToAffiliate);
            if (mService.getLockTaskController().isLockTaskModeViolation(
                    mStartActivity.getTask())) {
                Slog.e(TAG, "Attempted Lock Task Mode violation mStartActivity=" + mStartActivity);
                return START_RETURN_LOCK_TASK_MODE_VIOLATION;
            }
        } else if (mAddingToTask) {
            addOrReparentStartingActivity(targetTask, "adding to task");
        }

        if (!mAvoidMoveToFront && mDoResume) {
            mTargetStack.getStack().moveToFront("reuseOrNewTask", targetTask);
            if (mOptions != null) {
                if (mOptions.getTaskAlwaysOnTop()) {
                    mTargetStack.setAlwaysOnTop(true);
                }
            }
            if (!mTargetStack.isTopStackInDisplayArea() && mService.mInternal.isDreaming()) {
                // Launching underneath dream activity (fullscreen, always-on-top). Run the launch-
                // -behind transition so the Activity gets created and starts in visible state.
                mLaunchTaskBehind = true;
                r.mLaunchTaskBehind = true;
            }
        }

        mService.mUgmInternal.grantUriPermissionUncheckedFromIntent(intentGrants,
                mStartActivity.getUriPermissionsLocked());
        if (mStartActivity.resultTo != null && mStartActivity.resultTo.info != null) {
            // we need to resolve resultTo to a uid as grantImplicitAccess deals explicitly in UIDs
            final PackageManagerInternal pmInternal =
                    mService.getPackageManagerInternalLocked();
            final int resultToUid = pmInternal.getPackageUidInternal(
                            mStartActivity.resultTo.info.packageName, 0, mStartActivity.mUserId);
            pmInternal.grantImplicitAccess(mStartActivity.mUserId, mIntent,
                    UserHandle.getAppId(mStartActivity.info.applicationInfo.uid) /*recipient*/,
                    resultToUid /*visible*/, true /*direct*/);
        }
        if (newTask) {
            EventLogTags.writeWmCreateTask(mStartActivity.mUserId,
                    mStartActivity.getTask().mTaskId);
        }
        mStartActivity.logStartActivity(
                EventLogTags.WM_CREATE_ACTIVITY, mStartActivity.getTask());

        mTargetStack.mLastPausedActivity = null;

        mRootWindowContainer.sendPowerHintForLaunchStartIfNeeded(
                false /* forceSend */, mStartActivity);

        mTargetStack.startActivityLocked(mStartActivity, topStack.getTopNonFinishingActivity(),
                newTask, mKeepCurTransition, mOptions);
        if (mDoResume) {
            final ActivityRecord topTaskActivity =
                    mStartActivity.getTask().topRunningActivityLocked();
            if (!mTargetStack.isTopActivityFocusable()
                    || (topTaskActivity != null && topTaskActivity.isTaskOverlay()
                    && mStartActivity != topTaskActivity)) {
                // If the activity is not focusable, we can't resume it, but still would like to
                // make sure it becomes visible as it starts (this will also trigger entry
                // animation). An example of this are PIP activities.
                // Also, we don't want to resume activities in a task that currently has an overlay
                // as the starting activity just needs to be in the visible paused state until the
                // over is removed.
                // Passing {@code null} as the start parameter ensures all activities are made
                // visible.
                mTargetStack.ensureActivitiesVisible(null /* starting */,
                        0 /* configChanges */, !PRESERVE_WINDOWS);
                // Go ahead and tell window manager to execute app transition for this activity
                // since the app transition will not be triggered through the resume channel.
                mTargetStack.getDisplay().mDisplayContent.executeAppTransition();
            } else {
                // If the target stack was not previously focusable (previous top running activity
                // on that stack was not visible) then any prior calls to move the stack to the
                // will not update the focused stack.  If starting the new activity now allows the
                // task stack to be focusable, then ensure that we now update the focused stack
                // accordingly.
                if (mTargetStack.isTopActivityFocusable()
                        && !mRootWindowContainer.isTopDisplayFocusedStack(mTargetStack)) {
                    mTargetStack.moveToFront("startActivityInner");
                }
                mRootWindowContainer.resumeFocusedStacksTopActivities(
                        mTargetStack, mStartActivity, mOptions);
            }
        }
        mRootWindowContainer.updateUserStack(mStartActivity.mUserId, mTargetStack);

        // Update the recent tasks list immediately when the activity starts
        mSupervisor.mRecentTasks.add(mStartActivity.getTask());
        mSupervisor.handleNonResizableTaskIfNeeded(mStartActivity.getTask(),
                mPreferredWindowingMode, mPreferredTaskDisplayArea, mTargetStack);

        return START_SUCCESS;
    }
```

##### RootWindowContainer#resumeFocusedStacksTopActivities

frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java

```java
    boolean resumeFocusedStacksTopActivities(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

        if (!mStackSupervisor.readyToResume()) {
            return false;
        }

        boolean result = false;
        if (targetStack != null && (targetStack.isTopStackInDisplayArea()
                || getTopDisplayFocusedStack() == targetStack)) {
            result = targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }

        for (int displayNdx = getChildCount() - 1; displayNdx >= 0; --displayNdx) {
            boolean resumedOnDisplay = false;
            final DisplayContent display = getChildAt(displayNdx);
            for (int tdaNdx = display.getTaskDisplayAreaCount() - 1; tdaNdx >= 0; --tdaNdx) {
                final TaskDisplayArea taskDisplayArea = display.getTaskDisplayAreaAt(tdaNdx);
                for (int sNdx = taskDisplayArea.getStackCount() - 1; sNdx >= 0; --sNdx) {
                    final ActivityStack stack = taskDisplayArea.getStackAt(sNdx);
                    final ActivityRecord topRunningActivity = stack.topRunningActivity();
                    if (!stack.isFocusableAndVisible() || topRunningActivity == null) {
                        continue;
                    }
                    if (stack == targetStack) {
                        // Simply update the result for targetStack because the targetStack had
                        // already resumed in above. We don't want to resume it again, especially in
                        // some cases, it would cause a second launch failure if app process was
                        // dead.
                        resumedOnDisplay |= result;
                        continue;
                    }
                    if (taskDisplayArea.isTopStack(stack) && topRunningActivity.isState(RESUMED)) {
                        // Kick off any lingering app transitions form the MoveTaskToFront
                        // operation, but only consider the top task and stack on that display.
                        stack.executeAppTransition(targetOptions);
                    } else {
                        resumedOnDisplay |= topRunningActivity.makeActiveIfNeeded(target);
                    }
                }
            }
            if (!resumedOnDisplay) {
                // In cases when there are no valid activities (e.g. device just booted or launcher
                // crashed) it's possible that nothing was resumed on a display. Requesting resume
                // of top activity in focused stack explicitly will make sure that at least home
                // activity is started and resumed, and no recursion occurs.
                final ActivityStack focusedStack = display.getFocusedStack();
                if (focusedStack != null) {
                    result |= focusedStack.resumeTopActivityUncheckedLocked(target, targetOptions);
                } else if (targetStack == null) {
                    result |= resumeHomeActivity(null /* prev */, "no-focusable-task",
                            display.getDefaultTaskDisplayArea());
                }
            }
        }

        return result;
    }
```

##### ActivityStack#resumeTopActivityUncheckedLocked

```java
    @GuardedBy("mService")
    boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mInResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            // Protect against recursion.
            mInResumeTopActivity = true;
            result = resumeTopActivityInnerLocked(prev, options);

            // When resuming the top activity, it may be necessary to pause the top activity (for
            // example, returning to the lock screen. We suppress the normal pause logic in
            // {@link #resumeTopActivityUncheckedLocked}, since the top activity is resumed at the
            // end. We call the {@link ActivityStackSupervisor#checkReadyForSleepLocked} again here
            // to ensure any necessary pause logic occurs. In the case where the Activity will be
            // shown regardless of the lock screen, the call to
            // {@link ActivityStackSupervisor#checkReadyForSleepLocked} is skipped.
            final ActivityRecord next = topRunningActivity(true /* focusableOnly */);
            if (next == null || !next.canTurnScreenOn()) {
                checkReadyForSleep();
            }
        } finally {
            mInResumeTopActivity = false;
        }

        return result;
    }

```

**resumeTopActivityInnerLocked**

```java
    @GuardedBy("mService")
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        ......
        mStackSupervisor.startSpecificActivity(next, true, true);
        ......
	}
```

##### ActivityStackSupervisor#startSpecificActivity

```java
    void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        final WindowProcessController wpc =
                mService.getProcessController(r.processName, r.info.applicationInfo.uid);

        boolean knownToBeDead = false;
        if (wpc != null && wpc.hasThread()) {
            try {
                realStartActivityLocked(r, wpc, andResume, checkConfig); //wpc.hasThread()存在,这个app已经被启动
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
            knownToBeDead = true;
        }

        r.notifyUnknownVisibilityLaunchedForKeyguardTransition();

        final boolean isTop = andResume && r.isTopRunningActivity();
        mService.startProcessAsync(r, knownToBeDead, isTop, isTop ? "top-activity" : "activity");//app没有启动,创建app进程
    }
```

**realStartActivityLocked**

```java
    boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {
            // Create activity launch transaction.
        final ClientTransaction clientTransaction = ClientTransaction.obtain(
                        proc.getThread(), r.appToken);
        final DisplayContent dc = r.getDisplay().mDisplayContent;
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                        r.getSavedState(), r.getPersistentSavedState(), results, newIntents,
                        dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                        r.assistToken, r.createFixedRotationAdjustmentsIfNeeded()));//此处标记一下LaunchActivityItem
        		// Schedule transaction.
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);//事务被执行
	}
```

**ClientLifecycleManager#scheduleTransaction(ClientTransaction)**

```java
    void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        transaction.schedule();
        if (!(client instanceof Binder)) {
            // If client is not an instance of Binder - it's a remote call and at this point it is
            // safe to recycle the object. All objects used for local calls will be recycled after
            // the transaction is executed on client in ActivityThread.
            transaction.recycle();
        }
    }
```

```java
android.app.servertransaction.ClientTransaction#schedule
    public void schedule() throws RemoteException {
        mClient.scheduleTransaction(this);
    }
```

mClient是IApplicationThread,其实现类是ActivityThread中的一个内部类ApplicationThread

```java
android.app.ActivityThread.ApplicationThread#scheduleTransaction

        @Override
        public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
            ActivityThread.this.scheduleTransaction(transaction);
        }
```

ctrl跳转会进入到ClientTransactionHandler

android.app.ClientTransactionHandler#scheduleTransaction

```java
	/** Prepare and schedule transaction for execution. */
    void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }
```

还是回到ActivityThread

```java
case EXECUTE_TRANSACTION:
                    final ClientTransaction transaction = (ClientTransaction) msg.obj;
                    mTransactionExecutor.execute(transaction);
                    if (isSystem()) {
                        // Client transactions inside system process are recycled on the client side
                        // instead of ClientLifecycleManager to avoid being cleared before this
                        // message is handled.
                        transaction.recycle();
                    }
                    // TODO(lifecycler): Recycle locally scheduled transactions.
                    break;
```

mTransactionExecutor.execute(transaction)

private final TransactionExecutor mTransactionExecutor = new TransactionExecutor(this)


进入到TransactionExecutor#execute

```java
      public void execute(ClientTransaction transaction) {
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Start resolving transaction");

        final IBinder token = transaction.getActivityToken();
        if (token != null) {
            final Map<IBinder, ClientTransactionItem> activitiesToBeDestroyed =
                    mTransactionHandler.getActivitiesToBeDestroyed();
            final ClientTransactionItem destroyItem = activitiesToBeDestroyed.get(token);
            if (destroyItem != null) {
                if (transaction.getLifecycleStateRequest() == destroyItem) {
                    // It is going to execute the transaction that will destroy activity with the
                    // token, so the corresponding to-be-destroyed record can be removed.
                    activitiesToBeDestroyed.remove(token);
                }
                if (mTransactionHandler.getActivityClient(token) == null) {
                    // The activity has not been created but has been requested to destroy, so all
                    // transactions for the token are just like being cancelled.
                    Slog.w(TAG, tId(transaction) + "Skip pre-destroyed transaction:\n"
                            + transactionToString(transaction, mTransactionHandler));
                    return;
                }
            }
        }

        if (DEBUG_RESOLVER) Slog.d(TAG, transactionToString(transaction, mTransactionHandler));

        executeCallbacks(transaction);

        executeLifecycleState(transaction);
        mPendingActions.clear();
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "End resolving transaction");
    }
```

具体看一下executeCallbacks(transaction)

```java
    /** Cycle through all states requested by callbacks and execute them at proper times. */
    @VisibleForTesting
    public void executeCallbacks(ClientTransaction transaction) {
        final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
        if (callbacks == null || callbacks.isEmpty()) {
            // No callbacks to execute, return early.
            return;
        }
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Resolving callbacks in transaction");

        final IBinder token = transaction.getActivityToken();
        ActivityClientRecord r = mTransactionHandler.getActivityClient(token);

        // In case when post-execution state of the last callback matches the final state requested
        // for the activity in this transaction, we won't do the last transition here and do it when
        // moving to final state instead (because it may contain additional parameters from server).
        final ActivityLifecycleItem finalStateRequest = transaction.getLifecycleStateRequest();
        final int finalState = finalStateRequest != null ? finalStateRequest.getTargetState()
                : UNDEFINED;
        // Index of the last callback that requests some post-execution state.
        final int lastCallbackRequestingState = lastCallbackRequestingState(transaction);

        final int size = callbacks.size();
        for (int i = 0; i < size; ++i) {
            final ClientTransactionItem item = callbacks.get(i);
            if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Resolving callback: " + item);
            final int postExecutionState = item.getPostExecutionState();
            final int closestPreExecutionState = mHelper.getClosestPreExecutionState(r,
                    item.getPostExecutionState());
            if (closestPreExecutionState != UNDEFINED) {
                cycleToPath(r, closestPreExecutionState, transaction);
            }
			//*********************************
            item.execute(mTransactionHandler, token, mPendingActions);
            item.postExecute(mTransactionHandler, token, mPendingActions);
            if (r == null) {
                // Launch activity request will create an activity record.
                r = mTransactionHandler.getActivityClient(token);
            }

            if (postExecutionState != UNDEFINED && r != null) {
                // Skip the very last transition and perform it by explicit state request instead.
                final boolean shouldExcludeLastTransition =
                        i == lastCallbackRequestingState && finalState == postExecutionState;
                cycleToPath(r, postExecutionState, shouldExcludeLastTransition, transaction);
            }
        }
    }
```

item.execute(mTransactionHandler, token, mPendingActions)这个item是什么呢?

上述流程中clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),好像添加了回调,看一下LaunchActivityItem

```java
public class LaunchActivityItem extends ClientTransactionItem {
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
        ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
                mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
                mPendingResults, mPendingNewIntents, mIsForward,
                mProfilerInfo, client, mAssistToken, mFixedRotationAdjustments);
        client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
}
```

果然LaunchActivityItem继承了ClientTransactionItem

execute方法里调用ClientTransactionHandler执行handleLaunchActivity,但是handleLaunchActivity是一个抽象方法,所以最终是由ClientTransactionHandler的实现类ActivityThread来实现

##### ActivityThread#handleLaunchActivity

```java
    /**
     * Extended implementation of activity launch. Used when server requests a launch or relaunch.
     */
    @Override
    public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
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

        if (localLOGV) Slog.v(
            TAG, "Handling launch of " + r);

        // Initialize before creating the activity
        if (!ThreadedRenderer.sRendererDisabled
                && (r.activityInfo.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            HardwareRenderer.preload();
        }
        WindowManagerGlobal.initialize();

        // Hint the GraphicsEnvironment that an activity is launching on the process.
        GraphicsEnvironment.hintActivityLaunch();

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
                ActivityTaskManager.getService()
                        .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                                Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }

        return a;
    }
```

这个方法里做了初始化的工作,比如处理配置改变,初始化渲染进程,提示GPUactivity已经在进程中启动了

下一步处理performLaunchActivity(r, customIntent);

```java
   /**  Core implementation of activity launch. */
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }

        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
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

                // Activity resources must be initialized with the same loaders as the
                // application context.
                appContext.getResources().addLoaders(
                        app.getResources().getLoaders().toArray(new ResourcesLoader[0]));

                appContext.setOuterContext(activity);
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
                mLastReportedWindowingMode.put(activity.getActivityToken(),
                        config.windowConfiguration.getWindowingMode());
            }
            r.setState(ON_CREATE);

            // updatePendingActivityConfiguration() reads from mActivities to update
            // ActivityClientRecord which runs in a different thread. Protect modifications to
            // mActivities to avoid race.
            synchronized (mResourcesManager) {
                mActivities.put(r.token, r);
            }

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

如注释所说,该方法是启动activity的核心实现,主要做了以下工作

1. 根据ActivityClientRecord信息通过Instrumentation新建了一个activity
2. 创建Application
3. 设置app和activity的resources loader
4. 绑定activity

下面看到了mInstrumentation.callActivityOnCreate,饶了一大圈终于又回来了

##### Instrumentation#callActivityOnCreate

```java
    public void callActivityOnCreate(Activity activity, Bundle icicle) {
        prePerformCreate(activity);
        activity.performCreate(icicle);
        postPerformCreate(activity);
    }
```

回调到

##### android.app.Activity#performCreate

```java
   @UnsupportedAppUsage
    final void performCreate(Bundle icicle, PersistableBundle persistentState) {
        dispatchActivityPreCreated(icicle);
        mCanEnterPictureInPicture = true;
        // initialize mIsInMultiWindowMode and mIsInPictureInPictureMode before onCreate
        final int windowingMode = getResources().getConfiguration().windowConfiguration
                .getWindowingMode();
        mIsInMultiWindowMode = inMultiWindowMode(windowingMode);
        mIsInPictureInPictureMode = windowingMode == WINDOWING_MODE_PINNED;
        restoreHasCurrentPermissionRequest(icicle);
        if (persistentState != null) {
            onCreate(icicle, persistentState);
        } else {
            onCreate(icicle);
        }
        EventLogTags.writeWmOnCreateCalled(mIdent, getComponentName().getClassName(),
                "performCreate");
        mActivityTransitionState.readState(icicle);

        mVisibleFromClient = !mWindow.getWindowStyle().getBoolean(
                com.android.internal.R.styleable.Window_windowNoDisplay, false);
        mFragments.dispatchActivityCreated();
        mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
        dispatchActivityPostCreated(icicle);
    }
```

可以看到onCreate(icicle)被调用了,activity启动了

##### StartActivity流程小结

设计到的主要类:

主要分部在两个路径下

- frameworks/base/core/java/android/app/
- frameworks/base/services/core/java/com/android/server/wm/

至此就先暂时告一段落吧,整个流程的很多细节都没有涉及到

还有一种需要新建app的进程的流程还没有分析