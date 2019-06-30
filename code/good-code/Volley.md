## Volley

### 1 文件缓存

#### 1.1 缓存接口设计

```java
public interface Cache {
  
    Entry get(String key);
  
    void put(String key, Entry entry);

    void initialize();

    void invalidate(String key, boolean fullExpire);

    void remove(String key);

    void clear();
  
  static class Entry {
   	String name; 
  }
}
```



#### 1.2 基于访问顺序的双链表

```java
private final Map<String, CacheHeader> mEntries =
            new LinkedHashMap<String, CacheHeader>(16, .75f, true);//访问顺序
```



#### 1.3 设计思想 

文件缓存，但是在内存中存有一份映射表，可以快速的查询访问，**这样就不用每次都遍历一遍文件**，会大大加快缓存读取的速度，同步于本地文件列表，而且每个文件，都有缓存头部信息（key,size,data）



#### 1.4 初始化initialize

```java
    public synchronized void initialize() {//初始化，每次阻塞执行，刷新内存中关于文件缓存的记录
        if (!mRootDirectory.exists()) {//不存在
            if (!mRootDirectory.mkdirs()) {//创建文件，如果不成功，输出日志
                VolleyLog.e("Unable to create cache dir %s", mRootDirectory.getAbsolutePath());
            }
            return;
        }

        //遍历文件，更新内存中的Map
        File[] files = mRootDirectory.listFiles();
        if (files == null) {
            return;
        }
        for (File file : files) {
            BufferedInputStream fis = null;
            try {
                fis = new BufferedInputStream(new FileInputStream(file));//使用BufferedInputStream 包装FileInputStream
                CacheHeader entry = CacheHeader.readHeader(fis);
                entry.size = file.length();
                putEntry(entry.key, entry);//从文件中更新当前内存中的缓存记录
            } catch (IOException e) {
                if (file != null) {
                   file.delete();
                }
            } finally {
                try {
                    if (fis != null) {
                        fis.close();
                    }
                } catch (IOException ignored) { }
            }
        }
    }
```



```java
    private void putEntry(String key, CacheHeader entry) {
        if (!mEntries.containsKey(key)) {//不包含，新增大小
            mTotalSize += entry.size;
        } else {//包含，更新大小
            CacheHeader oldEntry = mEntries.get(key);
            mTotalSize += (entry.size - oldEntry.size);
        }
        mEntries.put(key, entry);
    }
```



#### 1.5 put新增

```java
   public synchronized void put(String key, Entry entry) {
        pruneIfNeeded(entry.data.length);//每次新增缓存，先判断是否需要进行调整，以防文件缓存超过最大缓存限制
        File file = getFileForKey(key);
        try {
            BufferedOutputStream fos = new BufferedOutputStream(new FileOutputStream(file));//BufferedOutputStream包装FileOutputStream
            CacheHeader e = new CacheHeader(key, entry);
            boolean success = e.writeHeader(fos);
            if (!success) {
                fos.close();
                VolleyLog.d("Failed to write header for %s", file.getAbsolutePath());
                throw new IOException();
            }
            fos.write(entry.data);
            fos.close();
            putEntry(key, e);
            return;
        } catch (IOException e) {
        }
        boolean deleted = file.delete();//如果异常会走到这里，记得把文件删除
        if (!deleted) {//删除失败，输出日志
            VolleyLog.d("Could not clean up file %s", file.getAbsolutePath());
        }
    }
```



#### 1.6 !!!调整大小方法!!!

```java
    private void pruneIfNeeded(int neededSpace) {
        if ((mTotalSize + neededSpace) < mMaxCacheSizeInBytes) {//小于最大缓存，不做调整
            return;
        }
        if (VolleyLog.DEBUG) {
            VolleyLog.v("Pruning old cache entries.");
        }

        long before = mTotalSize;
        int prunedFiles = 0;
        long startTime = SystemClock.elapsedRealtime();

        //科学的遍历方式
        Iterator<Map.Entry<String, CacheHeader>> iterator = mEntries.entrySet().iterator();
        while (iterator.hasNext()) {
            Map.Entry<String, CacheHeader> entry = iterator.next();
            CacheHeader e = entry.getValue();
            boolean deleted = getFileForKey(e.key).delete();
            if (deleted) {//严谨，文件删除时都做了判断
                mTotalSize -= e.size;
            } else {//删除不了的文件，做了日志输出
               VolleyLog.d("Could not delete cache entry for key=%s, filename=%s",
                       e.key, getFilenameForKey(e.key));
            }
            iterator.remove();
            prunedFiles++;//用于日志输出

            if ((mTotalSize + neededSpace) < mMaxCacheSizeInBytes * HYSTERESIS_FACTOR) {
                //一轮调整后，如果满足条件，跳出循环
                //如果不满足，继续调整
                break;
            }
        }

        if (VolleyLog.DEBUG) {
            VolleyLog.v("pruned %d files, %d bytes, %d ms",
                    prunedFiles, (mTotalSize - before), SystemClock.elapsedRealtime() - startTime);
        }
    }
```



#### 1.7 获取get

```java
    public synchronized Entry get(String key) {
        CacheHeader entry = mEntries.get(key);
        // if the entry does not exist, return.
        if (entry == null) {
            return null;
        }

        File file = getFileForKey(key);
        CountingInputStream cis = null;
        try {
            cis = new CountingInputStream(new BufferedInputStream(new FileInputStream(file)));
            CacheHeader.readHeader(cis); // eat header //为了跳过header，得到下面的data
            byte[] data = streamToBytes(cis, (int) (file.length() - cis.bytesRead));
            return entry.toCacheEntry(data);
        } catch (IOException e) {
            VolleyLog.d("%s: %s", file.getAbsolutePath(), e.toString());
            remove(key);
            return null;
        }  catch (NegativeArraySizeException e) {
            VolleyLog.d("%s: %s", file.getAbsolutePath(), e.toString());
            remove(key);
            return null;
        } finally {
            if (cis != null) {
                try {
                    cis.close();
                } catch (IOException ioe) {
                    return null;
                }
            }
        }
    }
```

