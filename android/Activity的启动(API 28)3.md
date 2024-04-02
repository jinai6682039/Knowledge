# Activity 的启动（API 28）3

## Android API 28上ActivityManagerService对ActivityThread相关Activity生命周期回调的改动

之前两篇已经有不少地方涉及到了ActivityManagerService对ActivityThread相关Activity生命周期函数的调用，这里**在API 28引入了一个事务的新方式**来完成此操作，而在API 26的时候还是用之前**直接通过ActivityThread的远程代理对象来直接调用**相应方法来完成此操作。

而使用这种直接通过ActivityThread的远程代理对象来直接调用的方法，在处理Activity rusume操作时，会先后**远程调用三次**，分别**传递result数据、onNewIntent数据和远程触发Activity的resume操作**。但对于API 28的事务方式来说，对于Activity resume请求中的所有操作-**传递result数据、onNewIntent数据和远程触发Activity的resume操作**，这里会生成一个完整的事务对象，所以只会**发生一次远程调用**。

## 远程调用事务的几个场景

之前两篇有多处地方涉及到了远程事务调用，分别是如下几个场景：
- 传递onNewIntent求
- 传递上一个resume状态的ActivityRecord的pause请求
- 传递当前要启动的Activity的result数据、onNewIntent数据和resume请求

下面开始分场景来查看这些处理。

### 通过ClientLifecycleManager传递事务

ActivityManagerService会通过ClientLifecycleManager对象来传递事务，而ClientLifecycleManager类中有多个重载的scheduleTransaction()方法，用来传递事务到对应的应用层的主线程所运行的ActivityThread对象中去执行事务。

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

    /**
     * Schedule a single lifecycle request or callback to client activity.
     * @param client Target client.
     * @param activityToken Target activity token.
     * @param stateRequest A request to move target activity to a desired lifecycle state.
     * @throws RemoteException
     *
     * @see ClientTransactionItem
     */
    void scheduleTransaction(@NonNull IApplicationThread client, @NonNull IBinder activityToken,
            @NonNull ActivityLifecycleItem stateRequest) throws RemoteException {
        final ClientTransaction clientTransaction = transactionWithState(client, activityToken,
                stateRequest);
        scheduleTransaction(clientTransaction);
    }

    /**
     * Schedule a single callback delivery to client activity.
     * @param client Target client.
     * @param activityToken Target activity token.
     * @param callback A request to deliver a callback.
     * @throws RemoteException
     *
     * @see ClientTransactionItem
     */
    void scheduleTransaction(@NonNull IApplicationThread client, @NonNull IBinder activityToken,
            @NonNull ClientTransactionItem callback) throws RemoteException {
        final ClientTransaction clientTransaction = transactionWithCallback(client, activityToken,
                callback);
        scheduleTransaction(clientTransaction);
    }

    /**
     * Schedule a single callback delivery to client application.
     * @param client Target client.
     * @param callback A request to deliver a callback.
     * @throws RemoteException
     *
     * @see ClientTransactionItem
     */
    void scheduleTransaction(@NonNull IApplicationThread client,
            @NonNull ClientTransactionItem callback) throws RemoteException {
        final ClientTransaction clientTransaction = transactionWithCallback(client,
                null /* activityToken */, callback);
        scheduleTransaction(clientTransaction);
    }
```

上述几个重载的方法，最后都会调用到第一个直接传递ClientTransaction事务对象的方法。


### 传递onNewIntent请求

会在如下几个情况下产生onNewIntent请求：
-  使用了**singleTop启动模式**或者带有**single_top**标志位，且当前目标TaskRecord的顶部ActivityRecord就是当前要启动的ActivityRecord同一个Activity
-  使用了**singleTop启动模式**或者带有**single_top**标志位外还带有**clear_top**标志位，且当前目标TaskRecord的中含义和将要启动的ActivityRecord描述同一种Activity的ActivityRecord
-  采用了**singleTask启动模式**或者**singleInstance启动模式**，且当前目标TaskRecord的顶部ActivityRecord就是当前要启动的ActivityRecord同一个Activity

这几种情况下都会触发onNewIntent请求，但也只有当要接收onNewIntent的ActivityRecord处于对用户可见的状态或者处于锁屏状态下的最顶层Activity时，才会生成相应的onNewIntent事务来进行远程回调。如果不符合直接远程回调的情况，会将此次要回传的intent对象记录到相应的ActivityRecord中去，等到下一次resume请求时来处理。

```java
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

    private void addNewIntentLocked(ReferrerIntent intent) {
        if (newIntents == null) {
            newIntents = new ArrayList<>();
        }
        newIntents.add(intent);
    }
```

可以看到这里是传递了一个类型为NewIntentItem的事务对象，是通过一个AMS中的一个ClientLifecycleManager类型的字段的一些列`scheduleTransaction()`方法来传递的。

