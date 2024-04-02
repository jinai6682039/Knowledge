# Activity 的启动（API 28）2

## ActivityStack.startActivityLocked()-part1

```java
        TaskRecord task = null;
        if (!newTask) {
            // If starting in an existing task, find where that is...
            boolean startIt = true;
            for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
                task = mTaskHistory.get(taskNdx);
                if (task.getTopActivity() == null) {
                    // All activities in task are finishing.
                    continue;
                }
                if (task == rTask) {
                    // Here it is!  Now, if this is not yet visible to the
                    // user, then just add it without starting; it will
                    // get started when the user navigates back to it.
                    if (!startIt) {
                        if (DEBUG_ADD_REMOVE) Slog.i(TAG, "Adding activity " + r + " to task "
                                + task, new RuntimeException("here").fillInStackTrace());
                        r.createWindowContainer();
                        ActivityOptions.abort(options);
                        return;
                    }
                    break;
                } else if (task.numFullscreen > 0) {
                    startIt = false;
                }
            }
        }
```

首先根据参数判断当前要启动的Activity是否是在一个现有的TaskRecord中去启动。如果是，则会进入上述的代码块中继续执行。
这里由于已经确认了目标ActivityStack和目标TaskRecord，此时就会去在对应的ActivityStack中的`mTaskHistory`做一个遍历。在`mTaskHistory`描述的TaskRecord列表中越靠在末尾的TaskRecord就表示约拥有更高的展示优先级，此列表的最后一个也就是当前ActivityStack的顶部TaskRecord。
这里会从顶部TaskRecord开始向下遍历，当当前遍历的TaskRecord是将要启动Activity的ActivityRecord所在的TaskRecord时，则会判断其TaskRecord之上是否还有其他拥有全屏Activity的TaskRecord。如果有，则说明此将要启动的Activity所在的TaskRecord不是顶层TaskRecord，是对用户不可见的，所以这里只会调用`ActivityRecord.createWindowContainer()`来创建一个`AppWindowToken对象`，此对象中会包含有`ActivityRecord创建时所新建的唯一的Token对象`，然后添加到对应的`Display对象`中（`在Android 6.0 \API 23时，这里的直接保存在WindowManagerService中`，为之后WindowManagerService为对应的`Activity添加窗口WindowState时`，去查找对应Token对应的ActivityRecord做标识）。在添加完`AppWindowToken对象`之后，这里就会结束此方法的调用，然后等待后续用户将此TaskRecord切换至前台，然后会触发将前台TaskRecord的栈顶Activity的resume操作时，才会去真正的启动此Activity。

## ActivityStack.startActivityLocked()-part2

如果经过上一部分判断的TaskRecord是当前前台的TaskRecord或者即将要启动的Activity是要在一个新的TaskRecord中启动，则会继续往下处理。

```java
		task = activityTask;
        if (r.getWindowContainerController() == null) {
            r.createWindowContainer();
        }
        task.setFrontOfTask();
```

首先这里会和上一部分说到的在现有的非顶部TaskRecord中创建Activity一样，会调用`ActivityRecord.createWindowContainer()`方法来创建一个AppWindowToken对象，同时按ActivityRecord的Token对象为key，AppWindowToken为value，记录到对应的Display对象中去。

然后调用此目标TaskRecord的`TaskRecord.setFrontOfTask()`方法，用来正确设置当前TaskRecord中所有`ActivityRecord.frontOfTask`字段的准确性。

```java
    /** Call after activity movement or finish to make sure that frontOfTask is set correctly */
    final void setFrontOfTask() {
        boolean foundFront = false;
        final int numActivities = mActivities.size();
        for (int activityNdx = 0; activityNdx < numActivities; ++activityNdx) {
            final ActivityRecord r = mActivities.get(activityNdx);
            if (foundFront || r.finishing) {
                r.frontOfTask = false;
            } else {
                r.frontOfTask = true;
                // Set frontOfTask false for every following activity.
                foundFront = true;
            }
        }
        if (!foundFront && numActivities > 0) {
            // All activities of this task are finishing. As we ought to have a frontOfTask
            // activity, make the bottom activity front.
            mActivities.get(0).frontOfTask = true;
        }
    }
```

可以看到这里会将此目标TaskRecord中所包含在`mActivities`中的所有ActivityRecord的`frontOfTask`字段都重新判断。在`mActivities`中的第一位ActivityRecord的`frontOfTask`字段会被设置为true，而其他都将设置为false。

同时此时TaskRecord的顶部ActivityRecord一般都是将要启动ActivityRecord。但也会存在特殊情况，如在使用`FLAG_ACTIVITY_NEW_TASK`标志位去启动一个与栈底ActivityRecord是同一个的Activity，此时不会生成新的实例，而只会将对应的TaskRecord放置在前台。

## ActivityStack.startActivityLocked()-part3

之前两部分已经完成了创建一个AppWindowToken对象，同时按ActivityRecord的Token对象为key，AppWindowToken为value，记录到对应的Display对象中去和完成TaskRecord中`mActivities`中所有ActivityRecord的`frontOfTask`字段的调整。那么下面此部分将会完成Activity启动可能造成的App切换的动画、StartingWindow的展示等。
```java
        if (!isHomeOrRecentsStack() || numActivities() > 0) {
            if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                    "Prepare open transition: starting " + r);
            if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {
                mWindowManager.prepareAppTransition(TRANSIT_NONE, keepCurTransition);
                mStackSupervisor.mNoAnimActivities.add(r);
            } else {
                int transit = TRANSIT_ACTIVITY_OPEN;
                if (newTask) {
                    if (r.mLaunchTaskBehind) {
                        transit = TRANSIT_TASK_OPEN_BEHIND;
                    } else {
                        // If a new task is being launched, then mark the existing top activity as
                        // supporting picture-in-picture while pausing only if the starting activity
                        // would not be considered an overlay on top of the current activity
                        // (eg. not fullscreen, or the assistant)
                        if (canEnterPipOnTaskSwitch(focusedTopActivity,
                                null /* toFrontTask */, r, options)) {
                            focusedTopActivity.supportsEnterPipOnTaskSwitch = true;
                        }
                        transit = TRANSIT_TASK_OPEN;
                    }
                }
                mWindowManager.prepareAppTransition(transit, keepCurTransition);
                mStackSupervisor.mNoAnimActivities.remove(r);
            }
            boolean doShow = true;
            if (newTask) {
                // Even though this activity is starting fresh, we still need
                // to reset it to make sure we apply affinities to move any
                // existing activities from other tasks in to it.
                // If the caller has requested that the target task be
                // reset, then do so.
                if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                    resetTaskIfNeededLocked(r, r);
                    doShow = topRunningNonDelayedActivityLocked(null) == r;
                }
            } else if (options != null && options.getAnimationType()
                    == ActivityOptions.ANIM_SCENE_TRANSITION) {
                doShow = false;
            }
            if (r.mLaunchTaskBehind) {
                // Don't do a starting window for mLaunchTaskBehind. More importantly make sure we
                // tell WindowManager that r is visible even though it is at the back of the stack.
                r.setVisibility(true);
                ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
            } else if (SHOW_APP_STARTING_PREVIEW && doShow) {
                // Figure out if we are transitioning from another activity that is
                // "has the same starting icon" as the next one.  This allows the
                // window manager to keep the previous window it had previously
                // created, if it still had one.
                TaskRecord prevTask = r.getTask();
                ActivityRecord prev = prevTask.topRunningActivityWithStartingWindowLocked();
                if (prev != null) {
                    // We don't want to reuse the previous starting preview if:
                    // (1) The current activity is in a different task.
                    if (prev.getTask() != prevTask) {
                        prev = null;
                    }
                    // (2) The current activity is already displayed.
                    else if (prev.nowVisible) {
                        prev = null;
                    }
                }
                r.showStartingWindow(prev, newTask, isTaskSwitch(r, focusedTopActivity));
            }
        } else {
            // If this is the first activity, don't do any fancy animations,
            // because there is nothing for it to animate on top of.
            ActivityOptions.abort(options);
        }
```

## ActivityStarter.startActivityUnchecked()-part10

在完成了倒数第二步为将要启动的ActivityRecord创建AppWindowToken和调整TaskRecord中所有Activity的`frontOfTask`字段后，如果这里指明接下来要对新创建的ActivityRecord执行resume操作，那么就会开始启动Activity的最后一步。

```java
		if (mDoResume) {
            final ActivityRecord topTaskActivity =
                    mStartActivity.getTask().topRunningActivityLocked();
            if (!mTargetStack.isFocusable()
                    || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                    && mStartActivity != topTaskActivity)) {
                // If the activity is not focusable, we can't resume it, but still would like to
                // make sure it becomes visible as it starts (this will also trigger entry
                // animation). An example of this are PIP activities.
                // Also, we don't want to resume activities in a task that currently has an overlay
                // as the starting activity just needs to be in the visible paused state until the
                // over is removed.
                mTargetStack.ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
                // Go ahead and tell window manager to execute app transition for this activity
                // since the app transition will not be triggered through the resume channel.
                mService.mWindowManager.executeAppTransition();
            } else {
                // If the target stack was not previously focusable (previous top running activity
                // on that stack was not visible) then any prior calls to move the stack to the
                // will not update the focused stack.  If starting the new activity now allows the
                // task stack to be focusable, then ensure that we now update the focused stack
                // accordingly.
                if (mTargetStack.isFocusable() && !mSupervisor.isFocusedStack(mTargetStack)) {
                    mTargetStack.moveToFront("startActivityUnchecked");
                }
                mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                        mOptions);
            }
        } else if (mStartActivity != null) {
            mSupervisor.mRecentTasks.add(mStartActivity.getTask());
        }
```

这里主要是借助`ActivityStackSupervisor.resumeFocusedStackTopActivityLocked()`完成的：
```java
    boolean resumeFocusedStackTopActivityLocked(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

        if (!readyToResume()) {
            return false;
        }

        if (targetStack != null && isFocusedStack(targetStack)) {
            return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }

        final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
        if (r == null || !r.isState(RESUMED)) {
            mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
        } else if (r.isState(RESUMED)) {
            // Kick off any lingering app transitions form the MoveTaskToFront operation.
            mFocusedStack.executeAppTransition(targetOptions);
        }

        return false;
    }
```

在此方法中，可以看到最后还是通过调用对应的顶部`ActivityStack.resumeTopActivityUncheckedLocked()`方法来完成最后Activity的resume操作：

```java
    boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mStackSupervisor.inResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            // Protect against recursion.
            mStackSupervisor.inResumeTopActivity = true;
            result = resumeTopActivityInnerLocked(prev, options);

            // When resuming the top activity, it may be necessary to pause the top activity (for
            // example, returning to the lock screen. We suppress the normal pause logic in
            // {@link #resumeTopActivityUncheckedLocked}, since the top activity is resumed at the
            // end. We call the {@link ActivityStackSupervisor#checkReadyForSleepLocked} again here
            // to ensure any necessary pause logic occurs. In the case where the Activity will be
            // shown regardless of the lock screen, the call to
            // {@link ActivityStackSupervisor#checkReadyForSleepLocked} is skipped.
            final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
            if (next == null || !next.canTurnScreenOn()) {
                checkReadyForSleep();
            }
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }

        return result;
    }
```

在`ActivityStack.resumeTopActivityUncheckedLocked()`方法中，最后是调用`ActivityStack.resumeTopActivityInnerLocked()`方法来完成最后的resume操作。


## ActivityStack.resumeTopActivityInnerLocked()

终于来到了启动新的Activity的地方，这时的新的ActivityRecord已经被加入到目标ActivityStack中的目标TaskRecord中去了，此时可能在ActivityStack中存在不是当前要启动的Activity的但出于resume状态的Activity。所以当存在不是同一个ActivityRecord的，状态已经是rusume状态的Activity时，这里会首先将当前resume状态的Activity先进行处理。

### 第一次ActivityStack.resumeTopActivityInnerLocked()-处理已经存在的resume状态的Activity的pausing操作

在整个Activity启动的过程中，可能会多次调用`ActivityStack.resumeTopActivityInnerLocked()`方法 ,而第一次一般都是处理当前处于resume状态的Activity，先将其pause后再去尝试启动新的Activity。

```java
        // Find the next top-most activity to resume in this stack that is not finishing and is
        // focusable. If it is not focusable, we will fall into the case below to resume the
        // top activity in the next focusable task.
        final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);

        final boolean hasRunningActivity = next != null;

        // TODO: Maybe this entire condition can get removed?
        if (hasRunningActivity && !isAttached()) {
            return false;
        }


		......

        // If the flag RESUME_WHILE_PAUSING is set, then continue to schedule the previous activity
        // to be paused, while at the same time resuming the new resume activity only if the
        // previous activity can't go into Pip since we want to give Pip activities a chance to
        // enter Pip before resuming the next activity.
        final boolean resumeWhilePausing = (next.info.flags & FLAG_RESUME_WHILE_PAUSING) != 0
                && !lastResumedCanPip;

        boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, next, false);
        if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
            pausing |= startPausingLocked(userLeaving, false, next, false);
        }
        if (pausing && !resumeWhilePausing) {
            if (DEBUG_SWITCH || DEBUG_STATES) Slog.v(TAG_STATES,
                    "resumeTopActivityLocked: Skip resume: need to start pausing");
            // At this point we want to put the upcoming activity's process
            // at the top of the LRU list, since we know we will be needing it
            // very soon and it would be a waste to let it get killed if it
            // happens to be sitting towards the end.
            if (next.app != null && next.app.thread != null) {
                mService.updateLruProcessLocked(next.app, true, null);
            }
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            if (lastResumed != null) {
                lastResumed.setWillCloseOrEnterPip(true);
            }
            return true;
        } else if (mResumedActivity == next && next.isState(RESUMED)
                && mStackSupervisor.allResumedActivitiesComplete()) {
            // It is possible for the activity to be resumed when we paused back stacks above if the
            // next activity doesn't have to wait for pause to complete.
            // So, nothing else to-do except:
            // Make sure we have executed any pending transitions, since there
            // should be nothing left to do at this point.
            executeAppTransition(options);
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Top activity resumed (dontWaitForPause) " + next);
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return true;
        }
```

这部分代码就是上面说的，首先将当前ActivityStack中处于resume状态的Activity先pause处理。
此方法中，这里传入的**参数prev**即为当前要启动的新ActivityRecord，然后调用`ActivityStack.topRunningActivityLocked()`方法去获取到当前ActivityStack中最顶层的可以resume的ActivityRecord对象，这里一般都是获取到已经被之前加入到目标ActivityStack和目标TaskRecord中的将要启动的ActivityRecord，也就是**参数prev**对应的ActivityRecord。

然后将会调用`ActivityStackSipervisor.pauseBackStacks()`方法去将非前台ActivityStack的resume的Activity去进行pause操作。这里将一个resume状态的ActivityRecord转变为pause状态是通过调用`ActivityStack.startPausingLocked()`方法来完成。此方法中有一个`boolean pauseImmediately`参数，用来判断是否是要在执行pausing过程中的同时将下一个Activity同步resuming出来。虽然提供了此功能，但在FrameWork层已经写死，都是先等待前一个Activity pause状态完成后，在进行下一个Activity的resuming过程。

```java
    boolean pauseBackStacks(boolean userLeaving, ActivityRecord resuming, boolean dontWait) {
        boolean someActivityPaused = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ActivityDisplay display = mActivityDisplays.valueAt(displayNdx);
            for (int stackNdx = display.getChildCount() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = display.getChildAt(stackNdx);
                if (!isFocusedStack(stack) && stack.getResumedActivity() != null) {
                    if (DEBUG_STATES) Slog.d(TAG_STATES, "pauseBackStacks: stack=" + stack +
                            " mResumedActivity=" + stack.getResumedActivity());
                    someActivityPaused |= stack.startPausingLocked(userLeaving, false, resuming,
                            dontWait);
                }
            }
        }
        return someActivityPaused;
    }
```

在完成了对后台ActivityStack中resume ActivityRecord的pausing操作后，会回到ActivityStack.resumeTopActivityInnerLocked()中去继续执行。接下来会对当前前台ActivityStack是否存在已经处于resume状态的Activity，如果存在，还需要调用`ActivityStack.startPausingLocked()`方法来将其也执行pausing操作。（当然，如果此时将要启动的ActivityRecord和当前resume状态的ActivityRecord是同一个，同时还标记为需要传递onNewInetnt()请求，那么在传递完onNewInetnt()请求后就结束了本次startActivity的启动。）注意这里调用`ActivityStack.startPausingLocked()`的最后一个参数`boolean pauseImmediately`被写死了**false**，之前对后台ActivityStack中的resume状态的ActivityRecord的处理也通过参数`boolean dontWait`写死为**false**。

那么下面看到`ActivityStack.startPausingLocked()`方法的具体实现
```java
    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
            ActivityRecord resuming, boolean pauseImmediately) {
        if (mPausingActivity != null) {
            Slog.wtf(TAG, "Going to pause when pause is already pending for " + mPausingActivity
                    + " state=" + mPausingActivity.getState());
            if (!shouldSleepActivities()) {
                // Avoid recursion among check for sleep and complete pause during sleeping.
                // Because activity will be paused immediately after resume, just let pause
                // be completed by the order of activity paused from clients.
                completePauseLocked(false, resuming);
            }
        }
        ActivityRecord prev = mResumedActivity;

        if (prev == null) {
            if (resuming == null) {
                Slog.wtf(TAG, "Trying to pause when nothing is resumed");
                mStackSupervisor.resumeFocusedStackTopActivityLocked();
            }
            return false;
        }

        if (prev == resuming) {
            Slog.wtf(TAG, "Trying to pause activity that is in process of being resumed");
            return false;
        }

        if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to PAUSING: " + prev);
        else if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Start pausing: " + prev);
        mPausingActivity = prev;
        mLastPausedActivity = prev;
        mLastNoHistoryActivity = (prev.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_HISTORY) != 0
                || (prev.info.flags & ActivityInfo.FLAG_NO_HISTORY) != 0 ? prev : null;
        prev.setState(PAUSING, "startPausingLocked");
        prev.getTask().touchActiveTime();
        clearLaunchTime(prev);

        mStackSupervisor.getLaunchTimeTracker().stopFullyDrawnTraceIfNeeded(getWindowingMode());

        mService.updateCpuStats();

        if (prev.app != null && prev.app.thread != null) {
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Enqueueing pending pause: " + prev);
            try {
                EventLogTags.writeAmPauseActivity(prev.userId, System.identityHashCode(prev),
                        prev.shortComponentName, "userLeaving=" + userLeaving);
                mService.updateUsageStats(prev, false);

                mService.getLifecycleManager().scheduleTransaction(prev.app.thread, prev.appToken,
                        PauseActivityItem.obtain(prev.finishing, userLeaving,
                                prev.configChangeFlags, pauseImmediately));
            } catch (Exception e) {
                // Ignore exception, if process died other code will cleanup.
                Slog.w(TAG, "Exception thrown during pause", e);
                mPausingActivity = null;
                mLastPausedActivity = null;
                mLastNoHistoryActivity = null;
            }
        } else {
            mPausingActivity = null;
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }

        // If we are not going to sleep, we want to ensure the device is
        // awake until the next activity is started.
        if (!uiSleeping && !mService.isSleepingOrShuttingDownLocked()) {
            mStackSupervisor.acquireLaunchWakelock();
        }

        if (mPausingActivity != null) {
            // Have the window manager pause its key dispatching until the new
            // activity has started.  If we're pausing the activity just because
            // the screen is being turned off and the UI is sleeping, don't interrupt
            // key dispatch; the same activity will pick it up again on wakeup.
            if (!uiSleeping) {
                prev.pauseKeyDispatchingLocked();
            } else if (DEBUG_PAUSE) {
                 Slog.v(TAG_PAUSE, "Key dispatch not paused for screen off");
            }

            if (pauseImmediately) {
                // If the caller said they don't want to wait for the pause, then complete
                // the pause now.
                completePauseLocked(false, resuming);
                return false;

            } else {
                schedulePauseTimeout(prev);
                return true;
            }

        } else {
            // This activity failed to schedule the
            // pause, so just treat it as being paused now.
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Activity not running, resuming next.");
            if (resuming == null) {
                mStackSupervisor.resumeFocusedStackTopActivityLocked();
            }
            return false;
        }
    }    final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
            ActivityRecord resuming, boolean pauseImmediately) {
        if (mPausingActivity != null) {
            Slog.wtf(TAG, "Going to pause when pause is already pending for " + mPausingActivity
                    + " state=" + mPausingActivity.getState());
            if (!shouldSleepActivities()) {
                // Avoid recursion among check for sleep and complete pause during sleeping.
                // Because activity will be paused immediately after resume, just let pause
                // be completed by the order of activity paused from clients.
                completePauseLocked(false, resuming);
            }
        }
        ActivityRecord prev = mResumedActivity;

        if (prev == null) {
            if (resuming == null) {
                Slog.wtf(TAG, "Trying to pause when nothing is resumed");
                mStackSupervisor.resumeFocusedStackTopActivityLocked();
            }
            return false;
        }

        if (prev == resuming) {
            Slog.wtf(TAG, "Trying to pause activity that is in process of being resumed");
            return false;
        }

        if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to PAUSING: " + prev);
        else if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Start pausing: " + prev);
        mPausingActivity = prev;
        mLastPausedActivity = prev;
        mLastNoHistoryActivity = (prev.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_HISTORY) != 0
                || (prev.info.flags & ActivityInfo.FLAG_NO_HISTORY) != 0 ? prev : null;
        prev.setState(PAUSING, "startPausingLocked");
        prev.getTask().touchActiveTime();
        clearLaunchTime(prev);

        mStackSupervisor.getLaunchTimeTracker().stopFullyDrawnTraceIfNeeded(getWindowingMode());

        mService.updateCpuStats();

        if (prev.app != null && prev.app.thread != null) {
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Enqueueing pending pause: " + prev);
            try {
                EventLogTags.writeAmPauseActivity(prev.userId, System.identityHashCode(prev),
                        prev.shortComponentName, "userLeaving=" + userLeaving);
                mService.updateUsageStats(prev, false);

                mService.getLifecycleManager().scheduleTransaction(prev.app.thread, prev.appToken,
                        PauseActivityItem.obtain(prev.finishing, userLeaving,
                                prev.configChangeFlags, pauseImmediately));
            } catch (Exception e) {
                // Ignore exception, if process died other code will cleanup.
                Slog.w(TAG, "Exception thrown during pause", e);
                mPausingActivity = null;
                mLastPausedActivity = null;
                mLastNoHistoryActivity = null;
            }
        } else {
            mPausingActivity = null;
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }

        // If we are not going to sleep, we want to ensure the device is
        // awake until the next activity is started.
        if (!uiSleeping && !mService.isSleepingOrShuttingDownLocked()) {
            mStackSupervisor.acquireLaunchWakelock();
        }

        if (mPausingActivity != null) {
            // Have the window manager pause its key dispatching until the new
            // activity has started.  If we're pausing the activity just because
            // the screen is being turned off and the UI is sleeping, don't interrupt
            // key dispatch; the same activity will pick it up again on wakeup.
            if (!uiSleeping) {
                prev.pauseKeyDispatchingLocked();
            } else if (DEBUG_PAUSE) {
                 Slog.v(TAG_PAUSE, "Key dispatch not paused for screen off");
            }

            if (pauseImmediately) {
                // If the caller said they don't want to wait for the pause, then complete
                // the pause now.
                completePauseLocked(false, resuming);
                return false;

            } else {
                schedulePauseTimeout(prev);
                return true;
            }

        } else {
            // This activity failed to schedule the
            // pause, so just treat it as being paused now.
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Activity not running, resuming next.");
            if (resuming == null) {
                mStackSupervisor.resumeFocusedStackTopActivityLocked();
            }
            return false;
        }
    }
```

此方法首先会判断当前ActivityStack当前是否已经存在正在执行pausing过程的Activity，如果是，则直接调用`ActivityStack.completePauseLocked(false, resuming)`方法来结束上一次pausing过程。

同时注意这里传递的第一个参数`boolean resumeNext`，这个参数代表着是否要触发结束pausing操作后的第二次`ActivityStack.resumeTopActivityInnerLocked()`方法来对将要启动的Activity进行resume操作。这里传递了一个false，说明不要在进行对上一次pause完成之后的对将要启动的Activity进行resume操作。因为`即使上一次触发的Activity rusume后，也会因为本次完成pausing操作后，而触发的将要启动的Activity的resume操作而立刻进入pausing状态`。

然后`ActivityStack.startPausingLocked()`方法会去获取当前ActivityStack的处于resume状态的ActivityRecord，如果当前**没有resume状态的ActivityRecord**，则会判断本次**执行pausing操作失败，然后返回false**。

同样，如果**将要resume的ActivityRecord和当前resume状态的ActivityRecord是同一个时，也会返回false，表示此次执行pausing操作失败**。

经过上面的一系列判断后，接下来会将进入执行pausing操作的流程中。

首先将设置当前ActivityStack中的几个`mPausingActivity`、`mLastPausedActivity`和`mLastNoHistoryActivity`字段的值，这里一般都会设置为当**前要执行pausing操作，也就是当前处于resume状态的ActivityRecord**。然后将此ActivityRecord的**状态设置为pausing**。

然后和之前传递`onNewIntent()`请求一样，借助`ActivityManagerService`中的`ClientLifecycleManager`对象的`scheduleTransaction()`方法来通过**ActivityThread的远程Binder代理对象IApplicationThread传递一个Pausing操作事务给ActivityThread。**

在完成了远程事务传递后，此事务最后会转化为一个主线程的Handle Message。然后会回到`ActivityStack.startPausingLocked()`中继续执行。此时如果上面的事务传递没有发生错误，则会根据之前说到过的参数`boolean pauseImmediately`来判断此次新的ActivityRecord resume操作是否等待之前的pausing操作结束。

当为true时，表示不等待pausing的操作，这里会直接调用`ActivityStack.completePauseLocked(false, resuming)`并返回了一个false来结束此次pausing操作。而在`ActivityStack.completePauseLocked(false, resuming)`中，会首先将ActivityStack中的`mPausingActivity`字段设置为null，表示此次pausing操作已经结束，然后由于传递的`boolean resumeNext`为false，所以不会直接在这里执行第二次`ActivityStack.resumeTopActivityInnerLocked()`方法来执行对ActivityRecord的resume操作。

而当为false时，说明需要等待此次pausing操作的完成，这里会调用`ActivityStack.schedulePauseTimeout(prev)`方法来发送一个延时500ms的监听pausing操作是否超时的Message，然后返回值为true。之前已经说过，frameWork层已经默认写死了都为false，所以都是需要等待pausing操作完成的。

无论传入的参数`boolean pauseImmediately`的值为true还是false，这里最后都会返回一个值回到`ActivityStack.resumeTopActivityInnerLocked()`方法中。

当`ActivityStack.startPausingLocked()`返回值为true时，则说明新启动的ActivityRecord的resume操作要等待pausing的完成，所以这里就会在下一个if语句中结束此次`ActivityStack.resumeTopActivityInnerLocked()`方法的调用，等待pausing操作的完成或者pausing操作的超时消息的处理。

当`ActivityStack.startPausingLocked()`返回值为false时，同时将要启动的Activity和当前resume状态的Activity不一致，则会继续往下执行，但一般正常情况下是不会走这个分支的。

#### ActivityStack.completePauseLocked()-结束本次pausing操作并开始接下来的resume操作

之前说到，在执行`ActivityStack.startPausingLocked()`方法中，有两种情况下会调用`ActivityStack.completePauseLocked()`来结束本次pausing操作并开始接下来的resume操作。分别是在发现已经有在pausing操作的ActivityRecord和传入的参数`boolean pauseImmediately`为true时。

当然，传入的参数`boolean pauseImmediately`为false时，当正常走完此次pausing操作，或者是监听到本次pausing操作超时时，也会通过一系列方法调用到ActivityStack.completePauseLocked()方法。

这两种情况下，调用此方法传入的参数是有区别的。
- 前者（已经有在pausing操作的ActivityRecord和传入的参数`boolean pauseImmediately`为true）传入的参数`boolean resumeNext`为false，表示不需要在这里开始进行新的ActivityRecord的resume操作，因为这种情况下在`ActivityStack.startPausingLocked()`方法返回了false后，运行后回到`ActivityStack.resumeTopActivityInnerLocked()`中不会直接结束本次调用，而是继续往下执行。
- 后者（传入的参数`boolean pauseImmediately`为false），会在相应的PauseActivityItem执行完成后远程调用`AMS.activityPaused()`方法，其中会调用到`ActivityStack.activityPausedLocked()`方法或者在ActivityStack处理pausing超时消息时，会调用到`ActivityStack.activityPausedLocked()`方法。而此方法中最后会调用的`ActivityStack.completePauseLocked()`方法，此时传递的参数`boolean resumeNext`为true，表示要在结束此次pausing操作后，才开始第二次`ActivityStack.resumeTopActivityInnerLocked()`调用。

```java
    @Override
    public final void activityPaused(IBinder token) {
        final long origId = Binder.clearCallingIdentity();
        synchronized(this) {
            ActivityStack stack = ActivityRecord.getStackLocked(token);
            if (stack != null) {
                stack.activityPausedLocked(token, false);
            }
        }
        Binder.restoreCallingIdentity(origId);
    }
```

```java
   case PAUSE_TIMEOUT_MSG: {
                    ActivityRecord r = (ActivityRecord)msg.obj;
                    // We don't at this point know if the activity is fullscreen,
                    // so we need to be conservative and assume it isn't.
                    Slog.w(TAG, "Activity pause timeout for " + r);
                    synchronized (mService) {
                        if (r.app != null) {
                            mService.logAppTooSlow(r.app, r.pauseTime, "pausing " + r);
                        }
                        activityPausedLocked(r.appToken, true);
                    }
                } break;
```

```java
    final void activityPausedLocked(IBinder token, boolean timeout) {
        if (DEBUG_PAUSE) Slog.v(TAG_PAUSE,
            "Activity paused: token=" + token + ", timeout=" + timeout);

        final ActivityRecord r = isInStackLocked(token);
        if (r != null) {
            mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
            if (mPausingActivity == r) {
                if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to PAUSED: " + r
                        + (timeout ? " (due to timeout)" : " (pause complete)"));
                mService.mWindowManager.deferSurfaceLayout();
                try {
                    completePauseLocked(true /* resumeNext */, null /* resumingActivity */);
                } finally {
                    mService.mWindowManager.continueSurfaceLayout();
                }
                return;
            } else {
                EventLog.writeEvent(EventLogTags.AM_FAILED_TO_PAUSE,
                        r.userId, System.identityHashCode(r), r.shortComponentName,
                        mPausingActivity != null
                            ? mPausingActivity.shortComponentName : "(none)");
                if (r.isState(PAUSING)) {
                    r.setState(PAUSED, "activityPausedLocked");
                    if (r.finishing) {
                        if (DEBUG_PAUSE) Slog.v(TAG,
                                "Executing finish of failed to pause activity: " + r);
                        finishCurrentActivityLocked(r, FINISH_AFTER_VISIBLE, false,
                                "activityPausedLocked");
                    }
                }
            }
        }
        mStackSupervisor.ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
    }
```

```java
    private void completePauseLocked(boolean resumeNext, ActivityRecord resuming) {
        ActivityRecord prev = mPausingActivity;
        if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Complete pause: " + prev);

        if (prev != null) {
            prev.setWillCloseOrEnterPip(false);
            final boolean wasStopping = prev.isState(STOPPING);
            prev.setState(PAUSED, "completePausedLocked");
	        
	        ......
	        
            mPausingActivity = null;
        }

        if (resumeNext) {
            final ActivityStack topStack = mStackSupervisor.getFocusedStack();
            if (!topStack.shouldSleepOrShutDownActivities()) {
                mStackSupervisor.resumeFocusedStackTopActivityLocked(topStack, prev, null);
            } else {
                checkReadyForSleep();
                ActivityRecord top = topStack.topRunningActivityLocked();
                if (top == null || (prev != null && top != prev)) {
                    // If there are no more activities available to run, do resume anyway to start
                    // something. Also if the top activity on the stack is not the just paused
                    // activity, we need to go ahead and resume it to ensure we complete an
                    // in-flight app switch.
                    mStackSupervisor.resumeFocusedStackTopActivityLocked();
                }
            }
        }

        ......
    }
```

可以看到，在`ActivityStack.completePauseLocked()`方法中的确根据了传入参数`boolean resumeNext`来判断是否要执行第二次`ActivityStack.resumeTopActivityInnerLocked()`的调用，来开始对将要启动的ActivityRecord执行resume操作。


### 第二次`ActivityStack.resumeTopActivityInnerLocked()`-真正启动新ActivityRecord的地方

第二次`ActivityStack.resumeTopActivityInnerLocked()`的调用，是来源于`ActivityStack.completePauseLocked()`方法中的调用。而此次调用传入的参数和第一次也有所区别。

 第一次调用是来自于`ActivityStart.startActivityUnchecked()`方法中，此时传入的参数`ActivityRecord prev`的值其实就是当前要启动的ActivityRecord的值。
而第二次是来源于`ActivityStack.completePauseLocked()`方法中的调用，其中传入的参数`ActivityRecord prev`的值为触发此次`ActivityStack.completePauseLocked()`方法的刚刚经历pausing操作的ActivityRecord。

同时在`ActivityStack.completePauseLocked()`方法中调用`ActivityRecord.setState()方法`来将此ActivityRecord设置为Pause状态时，会同时将目标ActivityStack的`mResumedActivity`置为Null，所以当第二次走到第一次调用的判断是否要进行Activity的pausing操作时，会跳过这一步处理。

```java
        ActivityStack lastStack = mStackSupervisor.getLastStack();
        if (next.app != null && next.app.thread != null) {
            if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Resume running: " + next
                    + " stopped=" + next.stopped + " visible=" + next.visible);

            // If the previous activity is translucent, force a visibility update of
            // the next activity, so that it's added to WM's opening app list, and
            // transition animation can be set up properly.
            // For example, pressing Home button with a translucent activity in focus.
            // Launcher is already visible in this case. If we don't add it to opening
            // apps, maybeUpdateTransitToWallpaper() will fail to identify this as a
            // TRANSIT_WALLPAPER_OPEN animation, and run some funny animation.
            final boolean lastActivityTranslucent = lastStack != null
                    && (lastStack.inMultiWindowMode()
                    || (lastStack.mLastPausedActivity != null
                    && !lastStack.mLastPausedActivity.fullscreen));

            // The contained logic must be synchronized, since we are both changing the visibility
            // and updating the {@link Configuration}. {@link ActivityRecord#setVisibility} will
            // ultimately cause the client code to schedule a layout. Since layouts retrieve the
            // current {@link Configuration}, we must ensure that the below code updates it before
            // the layout can occur.
            synchronized(mWindowManager.getWindowManagerLock()) {
                // This activity is now becoming visible.
                if (!next.visible || next.stopped || lastActivityTranslucent) {
                    next.setVisibility(true);
                }

                // schedule launch ticks to collect information about slow apps.
                next.startLaunchTickingLocked();

                ActivityRecord lastResumedActivity =
                        lastStack == null ? null :lastStack.mResumedActivity;
                final ActivityState lastState = next.getState();

                mService.updateCpuStats();

                if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to RESUMED: " + next
                        + " (in existing)");

                next.setState(RESUMED, "resumeTopActivityInnerLocked");

                mService.updateLruProcessLocked(next.app, true, null);
                updateLRUListLocked(next);
                mService.updateOomAdjLocked();

                // Have the window manager re-evaluate the orientation of
                // the screen based on the new activity order.
                boolean notUpdated = true;

                if (mStackSupervisor.isFocusedStack(this)) {
                    // We have special rotation behavior when here is some active activity that
                    // requests specific orientation or Keyguard is locked. Make sure all activity
                    // visibilities are set correctly as well as the transition is updated if needed
                    // to get the correct rotation behavior. Otherwise the following call to update
                    // the orientation may cause incorrect configurations delivered to client as a
                    // result of invisible window resize.
                    // TODO: Remove this once visibilities are set correctly immediately when
                    // starting an activity.
                    notUpdated = !mStackSupervisor.ensureVisibilityAndConfig(next, mDisplayId,
                            true /* markFrozenIfConfigChanged */, false /* deferResume */);
                }

                if (notUpdated) {
                    // The configuration update wasn't able to keep the existing
                    // instance of the activity, and instead started a new one.
                    // We should be all done, but let's just make sure our activity
                    // is still at the top and schedule another run if something
                    // weird happened.
                    ActivityRecord nextNext = topRunningActivityLocked();
                    if (DEBUG_SWITCH || DEBUG_STATES) Slog.i(TAG_STATES,
                            "Activity config changed during resume: " + next
                                    + ", new next: " + nextNext);
                    if (nextNext != next) {
                        // Do over!
                        mStackSupervisor.scheduleResumeTopActivities();
                    }
                    if (!next.visible || next.stopped) {
                        next.setVisibility(true);
                    }
                    next.completeResumeLocked();
                    if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                    return true;
                }

                try {
                    final ClientTransaction transaction = ClientTransaction.obtain(next.app.thread,
                            next.appToken);
                    // Deliver all pending results.
                    ArrayList<ResultInfo> a = next.results;
                    if (a != null) {
                        final int N = a.size();
                        if (!next.finishing && N > 0) {
                            if (DEBUG_RESULTS) Slog.v(TAG_RESULTS,
                                    "Delivering results to " + next + ": " + a);
                            transaction.addCallback(ActivityResultItem.obtain(a));
                        }
                    }

                    if (next.newIntents != null) {
                        transaction.addCallback(NewIntentItem.obtain(next.newIntents,
                                false /* andPause */));
                    }

                    // Well the app will no longer be stopped.
                    // Clear app token stopped state in window manager if needed.
                    next.notifyAppResumed(next.stopped);

                    EventLog.writeEvent(EventLogTags.AM_RESUME_ACTIVITY, next.userId,
                            System.identityHashCode(next), next.getTask().taskId,
                            next.shortComponentName);

                    next.sleeping = false;
                    mService.getAppWarningsLocked().onResumeActivity(next);
                    mService.showAskCompatModeDialogLocked(next);
                    next.app.pendingUiClean = true;
                    next.app.forceProcessStateUpTo(mService.mTopProcessState);
                    next.clearOptionsLocked();
                    transaction.setLifecycleStateRequest(
                            ResumeActivityItem.obtain(next.app.repProcState,
                                    mService.isNextTransitionForward()));
                    mService.getLifecycleManager().scheduleTransaction(transaction);

                    if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Resumed "
                            + next);
                } catch (Exception e) {
                    // Whoops, need to restart this activity!
                    if (DEBUG_STATES) Slog.v(TAG_STATES, "Resume failed; resetting state to "
                            + lastState + ": " + next);
                    next.setState(lastState, "resumeTopActivityInnerLocked");

                    // lastResumedActivity being non-null implies there is a lastStack present.
                    if (lastResumedActivity != null) {
                        lastResumedActivity.setState(RESUMED, "resumeTopActivityInnerLocked");
                    }

                    Slog.i(TAG, "Restarting because process died: " + next);
                    if (!next.hasBeenLaunched) {
                        next.hasBeenLaunched = true;
                    } else  if (SHOW_APP_STARTING_PREVIEW && lastStack != null
                            && lastStack.isTopStackOnDisplay()) {
                        next.showStartingWindow(null /* prev */, false /* newTask */,
                                false /* taskSwitch */);
                    }
                    mStackSupervisor.startSpecificActivityLocked(next, true, false);
                    if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                    return true;
                }
            }

            // From this point on, if something goes wrong there is no way
            // to recover the activity.
            try {
                next.completeResumeLocked();
            } catch (Exception e) {
                // If any exception gets thrown, toss away this
                // activity and try the next one.
                Slog.w(TAG, "Exception thrown during resume of " + next, e);
                requestFinishActivityLocked(next.appToken, Activity.RESULT_CANCELED, null,
                        "resume-exception", true);
                if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                return true;
            }
        } else {
            // Whoops, need to restart this activity!
            if (!next.hasBeenLaunched) {
                next.hasBeenLaunched = true;
            } else {
                if (SHOW_APP_STARTING_PREVIEW) {
                    next.showStartingWindow(null /* prev */, false /* newTask */,
                            false /* taskSwich */);
                }
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Restarting: " + next);
            }
            if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Restarting " + next);
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
        }
```

这里会分两种情况来对ActivityRecord的resume操作进行处理。这里的区分规则是当前要**启动的ActivityRecord所运行的进程是否存在**，同时运行在**主线程的ActivityThread对象的远程代理对象是否初始化**了。

如果运行所在的进程存在，同时运行在主线程中的ActivityThread对象也已经经过初始化，那么才会继续去往下启动ActivityRecord，执行resume操作。否则会调用`ActivityStackSupervisor.startSpecificActivityLocked()`方法来初始化一个新的App进程，同时初始化ActivityThread对象。

那么接下来先看到**进程存在而且ActivityThread对象也进行过初始化的情况**，这里主要会执行一下几步：
- 首先将要启动的ActivityRecord设置为可见，这里调用`ActivityRecord.setVisibility()`方法来完成。
- 将将要启动的ActivityRecord的状态设置为resumed状态，这里是调用`ActivityRecord.setState()`方法来完成。同时此方法中会触发目标TaskRecord和目标ActivityStack的`onActivityStateChanged()`方法，从而将目标`ActivityStack.mResumedActivity`赋值为当前要启动的ActivityRecord。
- 开始构造Resume操作的`ClientTransaction`事务对象。
- 首先会将通过之前有通过`Activitty.setResult()`回传的result数据添加到事务中。对于刚刚启动的ActivityRecord，这些数据都是为空的。
- 然后又将之前启动流程中可能存在的，通过`deliverNewIntent()`方法向处于对**用户不可见状态的ActivityRecord**传递的`onNewIntent`的请求Intent数据也添加到事务中去。而对于向那些**对用户可见的ActivityRecord传递onNewIntent()请求时**，会**直接生成事务，进行远程调用**。
- 最后将上面填充过result数据和newIntent数据的事务，和之前执行onNewIntent请求或者pause请求一样，添加一个ResumeActivityItem的生命周期的请求，表示要进行resume请求，然后通过`ActivityManagerService.ClientLifecycleManager`对象继续远程调用。


#### ActivityStackSupervisor.startSpecificActivityLocked() 

当将要启动的ActivityRecord所要运行的进程不存在时，在`ActivityStack.resumeTopActivityInnerLocked()`方法中的最后，就会调用`ActivityStackSupervisor.startSpecificActivityLocked()`方法来完成此种情况的ActivityRecord的启动。

```java
    void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        getLaunchTimeTracker().setLaunchTime(r);

        if (app != null && app.thread != null) {
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
                    // Don't add this if it is a platform component that is marked
                    // to run in multiple processes, because this is actually
                    // part of the framework so doesn't make sense to track as a
                    // separate apk in the process.
                    app.addPackage(r.info.packageName, r.info.applicationInfo.longVersionCode,
                            mService.mProcessStats);
                }
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
 
在此方法中，首先会再次借助**ActivityManagerService**去查找一次当前要启动的**ActivityRecord所运行的进程ProcessRecord是否存在**。
如果**存在**，则会调用`ActivityStackSupervisor.realStartActivityLocked()`方法来启动Activity。

如果**不存在**，则会调用`ActivityManagerService.startProcessLocked()`方法来为创建一个新的进程来启动将要启动的ActivityRecord。这里最后会借助`ZygoteProcess`来初始化一个新的进程，同时调用`ActivityThread.main()`方法。 
在`ActivityThread.main()`中，除了会开始主线程Looper的初始化和开启主线程的Looper消息循环外，还会在其中远程调用`ActivityManagerService.attachApplicationLocked()`方法，在其中主要会完成将ActivityThread对象的远程代理对象传递到AMS中，然后构建相应新创建的进程ProcessRecord对象。在此过程中，AMS会借助刚刚接受到的ActivityThread的远程代理对象来远程回调`ActivityThread.handleBindApplication()`方法，在此方法中会去为ActivityThread对象初始化好Instrumentation对象、ContextImpl对象以及Application对象等，然后通过借助Instrumentation对象来调用`Application.onCreate()`方法。当此次远程调用`ActivityThread.handleBindApplication()`方法结束后，会继续回到`ActivityManagerService.attachApplicationLocked()`方法中继续执行，此时新创建的进程（主线程）已经完成了初始化工作。接下来就会去查找是否有ActivityRecord、ServiceRecord等要启动，这里也就会调用`ActivityStackSupervisor.realStartActivityLocked()`方法来启动Activity。

#### ActivityStackSupervisor.realStartActivityLocked()

之前说的，在将要启动的ActivityRecord所运行的进程还不存在时，会借助AMS来创建一个新的进程（主线程），然后再初始化完成新进程的ProcessRecord、ActivityThread后，还会去判断是否有ActivityRecord、ServiceRecord等要启动。这里就会调用`ActivityStackSupervisor.realStartActivityLocked()`方法来启动Activity。

```java
    final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {

        if (!allPausedActivitiesComplete()) {
            // While there are activities pausing we skipping starting any new activities until
            // pauses are complete. NOTE: that we also do this for activities that are starting in
            // the paused state because they will first be resumed then paused on the client side.
            if (DEBUG_SWITCH || DEBUG_PAUSE || DEBUG_STATES) Slog.v(TAG_PAUSE,
                    "realStartActivityLocked: Skipping start of r=" + r
                    + " some activities pausing...");
            return false;
        }

        final TaskRecord task = r.getTask();
        final ActivityStack stack = task.getStack();

        beginDeferResume();

        try {
            r.startFreezingScreenLocked(app, 0);

            // schedule launch ticks to collect information about slow apps.
            r.startLaunchTickingLocked();

            r.setProcess(app);

            if (getKeyguardController().isKeyguardLocked()) {
                r.notifyUnknownVisibilityLaunched();
            }

            // Have the window manager re-evaluate the orientation of the screen based on the new
            // activity order.  Note that as a result of this, it can call back into the activity
            // manager with a new orientation.  We don't care about that, because the activity is
            // not currently running so we are just restarting it anyway.
            if (checkConfig) {
                // Deferring resume here because we're going to launch new activity shortly.
                // We don't want to perform a redundant launch of the same record while ensuring
                // configurations and trying to resume top activity of focused stack.
                ensureVisibilityAndConfig(r, r.getDisplayId(),
                        false /* markFrozenIfConfigChanged */, true /* deferResume */);
            }

            if (r.getStack().checkKeyguardVisibility(r, true /* shouldBeVisible */,
                    true /* isTop */)) {
                // We only set the visibility to true if the activity is allowed to be visible
                // based on
                // keyguard state. This avoids setting this into motion in window manager that is
                // later cancelled due to later calls to ensure visible activities that set
                // visibility back to false.
                r.setVisibility(true);
            }

            final int applicationInfoUid =
                    (r.info.applicationInfo != null) ? r.info.applicationInfo.uid : -1;
            if ((r.userId != app.userId) || (r.appInfo.uid != applicationInfoUid)) {
                Slog.wtf(TAG,
                        "User ID for activity changing for " + r
                                + " appInfo.uid=" + r.appInfo.uid
                                + " info.ai.uid=" + applicationInfoUid
                                + " old=" + r.app + " new=" + app);
            }

            app.waitingToKill = null;
            r.launchCount++;
            r.lastLaunchTime = SystemClock.uptimeMillis();

            if (DEBUG_ALL) Slog.v(TAG, "Launching: " + r);

            int idx = app.activities.indexOf(r);
            if (idx < 0) {
                app.activities.add(r);
            }
            mService.updateLruProcessLocked(app, true, null);
            mService.updateOomAdjLocked();

            final LockTaskController lockTaskController = mService.getLockTaskController();
            if (task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE
                    || task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE_PRIV
                    || (task.mLockTaskAuth == LOCK_TASK_AUTH_WHITELISTED
                            && lockTaskController.getLockTaskModeState()
                                    == LOCK_TASK_MODE_LOCKED)) {
                lockTaskController.startLockTaskMode(task, false, 0 /* blank UID */);
            }

            try {
                if (app.thread == null) {
                    throw new RemoteException();
                }
                List<ResultInfo> results = null;
                List<ReferrerIntent> newIntents = null;
                if (andResume) {
                    // We don't need to deliver new intents and/or set results if activity is going
                    // to pause immediately after launch.
                    results = r.results;
                    newIntents = r.newIntents;
                }
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                        "Launching: " + r + " icicle=" + r.icicle + " with results=" + results
                                + " newIntents=" + newIntents + " andResume=" + andResume);
                EventLog.writeEvent(EventLogTags.AM_RESTART_ACTIVITY, r.userId,
                        System.identityHashCode(r), task.taskId, r.shortComponentName);
                if (r.isActivityTypeHome()) {
                    // Home process is the root process of the task.
                    mService.mHomeProcess = task.mActivities.get(0).app;
                }
                mService.notifyPackageUse(r.intent.getComponent().getPackageName(),
                        PackageManager.NOTIFY_PACKAGE_USE_ACTIVITY);
                r.sleeping = false;
                r.forceNewConfig = false;
                mService.getAppWarningsLocked().onStartActivity(r);
                mService.showAskCompatModeDialogLocked(r);
                r.compat = mService.compatibilityInfoForPackageLocked(r.info.applicationInfo);
                ProfilerInfo profilerInfo = null;
                if (mService.mProfileApp != null && mService.mProfileApp.equals(app.processName)) {
                    if (mService.mProfileProc == null || mService.mProfileProc == app) {
                        mService.mProfileProc = app;
                        ProfilerInfo profilerInfoSvc = mService.mProfilerInfo;
                        if (profilerInfoSvc != null && profilerInfoSvc.profileFile != null) {
                            if (profilerInfoSvc.profileFd != null) {
                                try {
                                    profilerInfoSvc.profileFd = profilerInfoSvc.profileFd.dup();
                                } catch (IOException e) {
                                    profilerInfoSvc.closeFd();
                                }
                            }

                            profilerInfo = new ProfilerInfo(profilerInfoSvc);
                        }
                    }
                }

                app.hasShownUi = true;
                app.pendingUiClean = true;
                app.forceProcessStateUpTo(mService.mTopProcessState);
                // Because we could be starting an Activity in the system process this may not go
                // across a Binder interface which would create a new Configuration. Consequently
                // we have to always create a new Configuration here.

                final MergedConfiguration mergedConfiguration = new MergedConfiguration(
                        mService.getGlobalConfiguration(), r.getMergedOverrideConfiguration());
                r.setLastReportedConfiguration(mergedConfiguration);

                logIfTransactionTooLarge(r.intent, r.icicle);


                // Create activity launch transaction.
                final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
                        r.appToken);
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                        r.persistentState, results, newIntents, mService.isNextTransitionForward(),
                        profilerInfo));

                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                // Schedule transaction.
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);


                if ((app.info.privateFlags & ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0
                        && mService.mHasHeavyWeightFeature) {
                    // This may be a heavy-weight process!  Note that the package
                    // manager will ensure that only activity can run in the main
                    // process of the .apk, which is the only thing that will be
                    // considered heavy-weight.
                    if (app.processName.equals(app.info.packageName)) {
                        if (mService.mHeavyWeightProcess != null
                                && mService.mHeavyWeightProcess != app) {
                            Slog.w(TAG, "Starting new heavy weight process " + app
                                    + " when already running "
                                    + mService.mHeavyWeightProcess);
                        }
                        mService.mHeavyWeightProcess = app;
                        Message msg = mService.mHandler.obtainMessage(
                                ActivityManagerService.POST_HEAVY_NOTIFICATION_MSG);
                        msg.obj = r;
                        mService.mHandler.sendMessage(msg);
                    }
                }

            } catch (RemoteException e) {
                if (r.launchFailed) {
                    // This is the second time we failed -- finish activity
                    // and give up.
                    Slog.e(TAG, "Second failure launching "
                            + r.intent.getComponent().flattenToShortString()
                            + ", giving up", e);
                    mService.appDiedLocked(app);
                    stack.requestFinishActivityLocked(r.appToken, Activity.RESULT_CANCELED, null,
                            "2nd-crash", false);
                    return false;
                }

                // This is the first time we failed -- restart process and
                // retry.
                r.launchFailed = true;
                app.activities.remove(r);
                throw e;
            }
        } finally {
            endDeferResume();
        }

        ......

        return true;
    }
```

在此方法中，会和之前第二次调用`ActivityStack.resumeTopActivityInnerLocked()`来启动Activity，并执行resume操作类似，最主要的也是创建了`ClientTransaction`事务对象，然后向其填充了一些列包括result数据、onNewIntent数据等参数，生成一个Callback，最后在填充一个`ResumeActivityItem`对象作为事务的生命周期请求，也就是resume请求。最后通过`ActivityManagerService.ClientLifecycleManager`对象来远程调用ActivityThread去执行相应的事务。










