## Logger

### 1 将Throwable转为String

```java
  static String getStackTraceString(Throwable tr) {
    if (tr == null) {
      return "";
    }

    // This is to reduce the amount of log spew that apps do in the non-error
    // condition of the network being unavailable.
    Throwable t = tr;
    while (t != null) {
      if (t instanceof UnknownHostException) {
        return "";
      }
      t = t.getCause();
    }

    StringWriter sw = new StringWriter();
    PrintWriter pw = new PrintWriter(sw);
    tr.printStackTrace(pw);
    pw.flush();
    return sw.toString();
  }

```

### 2 Log日志链式调用，设置Tag生成日志信息

```java
 /**
 *每次调用会切换tag 调用示例 logger.t("tag").d("message");
 */
  public static Printer t(String tag) {
    return printer.t(tag);
  }
```

### 3 思想：如何实现只使用一次的tag? 

```java
  /**
   * Provides one-time used tag for the log message
   */
  private final ThreadLocal<String> localTag = new ThreadLocal<>();
  
  public Printer t(String tag) {
    if (tag != null) {
      localTag.set(tag);
    }
    return this;
  }
  
  private String getTag() {
    String tag = localTag.get();
    if (tag != null) {
      localTag.remove();//每次getTag后都清空
      return tag;
    }
    return null;
  }
```

