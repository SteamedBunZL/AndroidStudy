## Picasso



### 1 单例+Builder设计模式

```java
 public static Picasso with(@NonNull Context context) {
    if (context == null) {
      throw new IllegalArgumentException("context == null");
    }
    if (singleton == null) {
      synchronized (Picasso.class) {
        if (singleton == null) {
          singleton = new Builder(context).build();
        }
      }
    }
    return singleton;
  }
```

```java
public Picasso build() {
      Context context = this.context;

      if (downloader == null) {
        downloader = Utils.createDefaultDownloader(context);
      }
      if (cache == null) {
        cache = new LruCache(context);
      }
      if (service == null) {
        service = new PicassoExecutorService();
      }
      if (transformer == null) {
        transformer = RequestTransformer.IDENTITY;
      }

      Stats stats = new Stats(cache);

      Dispatcher dispatcher = new Dispatcher(context, service, HANDLER, downloader, cache, stats);

      return new Picasso(context, dispatcher, cache, listener, transformer, requestHandlers, stats,
          defaultBitmapConfig, indicatorsEnabled, loggingEnabled);
    }
```



### 2 检查主线程

可以收入自己的工具类

```java
static boolean isMain() {
    return Looper.getMainLooper().getThread() == Thread.currentThread();
  }
```



### 3 StringBuilder.ensureCapacity

这个不常使用，但是对于性能提升是有帮助的

```java
public void ensureCapacity(int minimumCapacity)
```



### 4 子线程HandlerThread + Handler 模式

#### 4.1 继承HandlerThread

```java
static class DispatcherThread extends HandlerThread {
    DispatcherThread() {
      super(Utils.THREAD_PREFIX + DISPATCHER_THREAD_NAME, THREAD_PRIORITY_BACKGROUND);
    }
  }
```

#### 4.2 继承Handler

```java
private static class DispatcherHandler extends Handler {
    private final Dispatcher dispatcher;

    public DispatcherHandler(Looper looper, Dispatcher dispatcher) {
      super(looper);
      this.dispatcher = dispatcher;
    }

    @Override public void handleMessage(final Message msg) {
      
    }
  }

```

#### 4.3 使用

```java
this.dispatcherThread = new DispatcherThread();
this.dispatcherThread.start();
Utils.flushStackLocalLeaks(dispatcherThread.getLooper());
this.handler = new DispatcherHandler(dispatcherThread.getLooper(), this);
```

其实看picasso的源码，这种设计思想是值得借鉴的，使用一个全局的子线程去做事件分发，这里使用HandlerThread，利用Android的方式而不是纯Java线程去做，方便事件和数据在线程间传输，点个赞。

picasso清晰明确了两个过程，1事件分发阶段 2事件执行阶段

事件的执行使用可控大小的线程池去执行，这种思路是值得借鉴的

#### 4.4 处理5.0之前Handler Bug

```java
static void flushStackLocalLeaks(Looper looper) {
    Handler handler = new Handler(looper) {
      @Override public void handleMessage(Message msg) {
        sendMessageDelayed(obtainMessage(), THREAD_LEAK_CLEANING_MS);
      }
    };
    handler.sendMessageDelayed(handler.obtainMessage(), THREAD_LEAK_CLEANING_MS);
  }
```

### 5 自定义线程池PicassoExecutorService

#### 5.1 构造

```java
PicassoExecutorService() {
    super(DEFAULT_THREAD_COUNT, DEFAULT_THREAD_COUNT, 0, TimeUnit.MILLISECONDS,
        new PriorityBlockingQueue<Runnable>(), new Utils.PicassoThreadFactory());
  }

```

#### 5.2 可以根据网络状况调节线程池大小

```java
 void adjustThreadCount(NetworkInfo info) {
    if (info == null || !info.isConnectedOrConnecting()) {
      setThreadCount(DEFAULT_THREAD_COUNT);
      return;
    }
    switch (info.getType()) {
      case ConnectivityManager.TYPE_WIFI:
      case ConnectivityManager.TYPE_WIMAX:
      case ConnectivityManager.TYPE_ETHERNET:
        setThreadCount(4);
        break;
      case ConnectivityManager.TYPE_MOBILE:
        switch (info.getSubtype()) {
          case TelephonyManager.NETWORK_TYPE_LTE:  // 4G
          case TelephonyManager.NETWORK_TYPE_HSPAP:
          case TelephonyManager.NETWORK_TYPE_EHRPD:
            setThreadCount(3);
            break;
          case TelephonyManager.NETWORK_TYPE_UMTS: // 3G
          case TelephonyManager.NETWORK_TYPE_CDMA:
          case TelephonyManager.NETWORK_TYPE_EVDO_0:
          case TelephonyManager.NETWORK_TYPE_EVDO_A:
          case TelephonyManager.NETWORK_TYPE_EVDO_B:
            setThreadCount(2);
            break;
          case TelephonyManager.NETWORK_TYPE_GPRS: // 2G
          case TelephonyManager.NETWORK_TYPE_EDGE:
            setThreadCount(1);
            break;
          default:
            setThreadCount(DEFAULT_THREAD_COUNT);
        }
        break;
      default:
        setThreadCount(DEFAULT_THREAD_COUNT);
    }
  }

 private void setThreadCount(int threadCount) {
    setCorePoolSize(threadCount);
    setMaximumPoolSize(threadCount);
  }

```

### 6 不受检getService

```java
static <T> T getService(Context context, String service) {
    return (T) context.getSystemService(service);
  }

```

### 7 检查某种权限

```java
static boolean hasPermission(Context context, String permission) {
    return context.checkCallingOrSelfPermission(permission) == PackageManager.PERMISSION_GRANTED;
  }

```

#### 7.1 检查网络权限

```java
hasPermission(context, Manifest.permission.ACCESS_NETWORK_STATE);

```

看picasso可以学习更加规范和标准的写法，也有很多设计思想是我们可以借鉴的



### 8 计算可用的RAM大小的15%

```java
  static int calculateMemoryCacheSize(Context context) {
    ActivityManager am = getService(context, ACTIVITY_SERVICE);
    boolean largeHeap = (context.getApplicationInfo().flags & FLAG_LARGE_HEAP) != 0;
    int memoryClass = am.getMemoryClass();
    if (largeHeap && SDK_INT >= HONEYCOMB) {
      memoryClass = ActivityManagerHoneycomb.getLargeMemoryClass(am);
    }
    // Target ~15% of the available heap.
    return (int) (1024L * 1024L * memoryClass / 7);
  }

  @TargetApi(HONEYCOMB)
  private static class ActivityManagerHoneycomb {//这种写法赞
    static int getLargeMemoryClass(ActivityManager activityManager) {
      return activityManager.getLargeMemoryClass();
    }
  }

```



### 9 计算ROM空间大小，设置最小和最大的上限

```java
@TargetApi(JELLY_BEAN_MR2)
  static long calculateDiskCacheSize(File dir) {
    long size = MIN_DISK_CACHE_SIZE;

    try {
      StatFs statFs = new StatFs(dir.getAbsolutePath());
      //noinspection deprecation
      long blockCount =
          SDK_INT < JELLY_BEAN_MR2 ? (long) statFs.getBlockCount() : statFs.getBlockCountLong();
      //noinspection deprecation
      long blockSize =
          SDK_INT < JELLY_BEAN_MR2 ? (long) statFs.getBlockSize() : statFs.getBlockSizeLong();
      long available = blockCount * blockSize;
      // Target 2% of the total space.
      size = available / 50;
    } catch (IllegalArgumentException ignored) {
    }

    // Bound inside min/max size for disk cache.
    return Math.max(Math.min(size, MAX_DISK_CACHE_SIZE), MIN_DISK_CACHE_SIZE);
  }

```

