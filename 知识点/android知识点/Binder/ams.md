https://www.androidos.net.cn/android/10.0.0_r6/xref/frameworks/base/core/java/android/app/IActivityTaskManager.aidl

https://www.androidos.net.cn/android/10.0.0_r6/xref/frameworks/base/core/java/android/app/IActivityManager.aidl

https://www.androidos.net.cn/android/10.0.0_r6/xref/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

https://www.androidos.net.cn/android/10.0.0_r6/xref/frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java

```java
    private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
            new Singleton<IActivityTaskManager>() {
                @Override
                protected IActivityTaskManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                    return IActivityTaskManager.Stub.asInterface(b);
                }
            };
```

这里返回的b 就是远端的ActivityTaskManagerService的引用，

 IActivityTaskManager.Stub是生成的代码，调用asInterface把b保存在内部类

```java
   private static class Proxy implements com.young.server.IBuyBookAIDL
    {
      private android.os.IBinder mRemote;
      Proxy(android.os.IBinder remote)
      {
        mRemote = remote;
      }
```

然后返回给外面的就是这个Proxy对象，调用proxy的方法也就间接调用mReote的方法，也就是atms

获取servicemanager的远端引用 getService