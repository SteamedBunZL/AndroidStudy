## LeakCanary



### 1 判断组件所在进程

```java
    PackageManager packageManager = context.getPackageManager();
    PackageInfo packageInfo;
    try {
      packageInfo = packageManager.getPackageInfo(context.getPackageName(), GET_SERVICES);
    } catch (Exception e) {
      CanaryLog.d(e, "Could not get package info for %s", context.getPackageName());
      return false;
    }
    String mainProcess = packageInfo.applicationInfo.processName;

    ComponentName component = new ComponentName(context, serviceClass);
    ServiceInfo serviceInfo;
    try {
      serviceInfo = packageManager.getServiceInfo(component, 0);
    } catch (PackageManager.NameNotFoundException ignored) {
      // Service is disabled.
      return false;
    }
	serviceInfo.processName
```



### 2 判断对象不为Null

```java
/**
   * Returns instance unless it's null.
   *
   * @throws NullPointerException if instance is null
   */
  static <T> T checkNotNull(T instance, String name) {
    if (instance == null) {
      throw new NullPointerException(name + " must not be null");
    }
    return instance;
  }
```



### 3 DumpHprof

```java
Debug.dumpHprofData(heapDumpFile.getAbsolutePath());
```



### 4 如何判断当前Activity发生了泄漏

```java
 private void removeWeaklyReachableReferences() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    KeyedWeakReference ref;
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
      retainedKeys.remove(ref.key);
    }
  }


  private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
  }
```

每次初始化传入了一个`ReferenceQueue`这个队列是用来存放`每当前的弱引用被GC回收了，那么当前这个弱引用对象就会被存入到这个Queue中去`，所以每次只要能从`retainedKeys`把当前的`KeyedWeakReference`弱引用对应的key移除那么就证明没有发生泄漏，而当泄漏的话`queue`就没有供`retainedKeys`移除的key值