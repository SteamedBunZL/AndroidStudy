## TrayPreferences



### 1.Manifest里注册的Provider authorities如何跟随包名而改变

```java
<provider
  android:name=".provider.TrayContentProvider"
  android:authorities="${applicationId}.tray"
  android:exported="false"
  android:multiprocess="false" />
```



### 2.设计思想

![Tray类关系图](/Users/zhanglong/Desktop/UML/Tray%E7%B1%BB%E5%85%B3%E7%B3%BB%E5%9B%BE.png)

核心接口PreferenceAccessor 提供用户可访问控制接口，具体的功能业务实现使用的是PreferenceStorage接口，通过抽象类Preferences把它们耦合在一起，耦合的方式值得学习，这里使用了泛型，可以做到不同层次之间的分别耦合

```java
public abstract class Preferences<T, S extends PreferenceStorage<T>>
        implements PreferenceAccessor<T>{
          	@NonNull
    		private S mStorage;
    		
    		public Preferences(@NonNull final S storage, final int version) {
        		mStorage = storage;
        		mVersion = version;
        		mChangeVersionSucceeded = false;

        		isVersionChangeChecked();
    		}
        }
```

存储的type限制为T Storage 限制为 S必须继承自PreferenceStorage接口，通过构造函数，由子类具体实现类传入PreferenceStorage的具体实现类



### 3.动态获取Authority方法

```java
    static String sAuthority;
	@NonNull
    private static synchronized String getAuthority(@NonNull final Context context) {
        if (sAuthority != null) {
            return sAuthority;
        }

        checkOldWayToSetAuthority(context);

        // read all providers of the app and find the TrayContentProvider to read the authority
        final List<ProviderInfo> providers = context.getPackageManager()
                .queryContentProviders(context.getPackageName(), Process.myUid(), 0);
        if (providers != null) {
            for (ProviderInfo provider : providers) {
                if (provider.name.equals(TrayContentProvider.class.getName())) {
                    sAuthority = provider.authority;
                    TrayLog.v("found authority: " + sAuthority);
                    return sAuthority;
                }
            }
        }

        // Should never happen. Otherwise we implemented tray in a wrong way!
        throw new TrayRuntimeException("Internal tray error. "
                + "Could not find the provider authority. "
                + "Please fill an issue at https://github.com/grandcentrix/tray/issues");
    }
```

这里使用动态获取Authority方式，结合上面的manifest里注册的方式，这样可以在最少的地方，最简单的方式去修改authority，这里的写法比较适合解决多个渠道包，大小号共存的问题。

