# 在AndroidStudio中自定义Gradle插件

一直都想好好学习AndroidStudio中的gradle，总感觉不懂如何在AndroidStudio中自定义gradle插件的程序员就不是个好程序员，这次上网查了一下相关资料，做了一个总结~

# 1 创建Gradle Module

`AndroidStudio`中是没有新建类似`Gradle Plugin`这样的选项的，那我们如何在`AndroidStudio`中编写`Gradle`插件，并打包出来呢？

> - (1) 首先，你得新建一个`Android Project`
> - (2) 然后再新建一个`Module`，这个`Module`用于开发`Gradle`插件，同样，`Module`里面没有`gradle plugin`给你选，但是我们只是需要一个“容器”来容纳我们写的插件，因此，你可以随便选择一个`Module`类型（如`Phone&Tablet Module`或`Android Librarty`）,因为接下来一步我们是将里面的大部分内容删除，所以选择哪个类型的`Module`不重要。
> - (3) 将`Module`里面的内容删除，只保留`build.gradle`文件和`src/main`目录。
> - 由于`gradle`是基于`groovy`，因此，我们开发的`gradle`插件相当于一个`groovy`项目。所以需要在`main`目录下新建`groovy`目录
> - (4) `groovy`又是基于`Java`，因此，接下来创建`groovy`的过程跟创建`java`很类似。在`groovy`新建包名，如：`com.hc.plugin`，然后在该包下新建`groovy`文件，通过`new->file->MyPlugin.groovy`来新建名为`MyPlugin`的`groovy`文件。
> - (5) 为了让我们的groovy类申明为gradle的插件，新建的groovy需要实现`org.gradle.api.Plugin`接口。如下所示：

```java
package  com.hc.plugin

import org.gradle.api.Plugin
import org.gradle.api.Project

public class MyPlugin implements Plugin<Project> {

    void apply(Project project) {
        System.out.println("========================");
        System.out.println("hello gradle plugin!");
        System.out.println("========================");
    }
}
```

因为我本人对`groovy`也不是特别熟悉，所以我尽可能的用`Java`语言，使用`System.out.println`而不是用`groovy的pintln ""`，我们的代码里面啥也没做，就打印信息。

> - (6) 现在，我们已经定义好了自己的`gradle`插件类，接下来就是告诉`gradle`，哪一个是我们自定义的插件类，因此，需要在`main`目录下新建`resources`目录，然后在`resources`目录里面再新建`META-INF`目录，再在`META-INF`里面新建`gradle-plugins`目录。最后在`gradle-plugins`目录里面新建properties文件，注意这个文件的命名，你可以随意取名，但是后面使用这个插件的时候，会用到这个名字。比如，你取名为`com.hc.gradle.properties`，而在其他build.gradle文件中使用自定义的插件时候则需写成：

```java
apply plugin: 'com.hc.gradle'
```

然后在`com.hc.gradle.properties`文件里面指明你自定义的类

```properties
implementation-class=com.hc.plugin.MyPlugin
```

现在，你的目录应该如下：

![自定义插件目录结构](https://img-blog.csdn.net/20160702123302689)

> - (7) 因为我们要用到`groovy`以及后面打包要用到`maven`,所以在我们自定义的`Module`下的`build.gradle`需要添加如下代码：

```java
apply plugin: 'groovy'
apply plugin: 'maven'

dependencies {
    //gradle sdk
    compile gradleApi()
    //groovy sdk
    compile localGroovy()
}

repositories {
    mavenCentral()
}
```

# 2 打包到本地Maven

前面我们已经自定义好了插件，接下来就是要打包到Maven库里面去了，你可以选择打包到本地，或者是远程服务器中。在我们自定义`Module`目录下的build.gradle添加如下代码：

```groovy
//group和version在后面使用自定义插件的时候会用到
group='com.hc.plugin'
version='1.0.0'

uploadArchives {
    repositories {
        mavenDeployer {
            //提交到远程服务器：
           // repository(url: "http://www.xxx.com/repos") {
            //    authentication(userName: "admin", password: "admin")
           // }
           //本地的Maven地址设置为D:/repos
            repository(url: uri('D:/repos'))
        }
    }
}
```

其中，`group`和`version`后面会用到，我们后面再讲。虽然我们已经定义好了打包地址以及打包相关配置，但是还需要我们让这个打包task执行。点击`AndroidStudio`右侧的`gradle工具`，如下图所示：

![上传Task](https://img-blog.csdn.net/20160702130539639)

可以看到有`uploadArchives`这个`Task`,双击`uploadArchives`就会执行打包上传啦！执行完成后，去我们的`Maven`本地仓库查看一下：

![打包上传后](https://img-blog.csdn.net/20160702130836877)

其中，`com/hc/plugin`这几层目录是由我们的`group`指定，`myplugin`是模块的名称，`1.0.0`是版本号（`version`指定）。

# 3 使用自定义的插件

接下来就是使用自定义的插件了，一般就是在`app`这个模块中使用自定义插件，因此在`app`这个`Module`的build.gradle文件中，需要指定本地`Maven`地址、自定义插件的名称以及依赖包名。简而言之，就是在`app`这个`Module`的build.gradle文件中后面附s加如下代码：

```groovy
buildscript {
    repositories {
        maven {//本地Maven仓库地址
            url uri('D:/repos')
        }
    }
    dependencies {
        //格式为-->group:module:version
        classpath 'com.hc.plugin:myplugin:1.0.0'
    }
}
//com.hc.gradle为resources/META-INF/gradle-plugins
//下的properties文件名称
apply plugin: 'com.hc.gradle'
```

好啦，接下来就是看看效果啦！先`clean project`(很重要！),然后再`make project`.从`messages`窗口打印如下信息：

![使用自定义插件](https://img-blog.csdn.net/20160702131957936)

好啦，现在终于运行了自定义的gradle插件啦！

# 4 开发只针对当前项目的Gradle插件

前面我们讲了如何自定义gradle插件并且打包出去，可能步骤比较多。有时候，你可能并不需要打包出去，只是在这一个项目中使用而已，那么你无需打包这个过程。

只是针对当前项目开发的Gradle插件相对较简单。步骤之前所提到的很类似，只是有几点需要注意：

> 1. 新建的Module名称必须为`BuildSrc`
> 2. 无需resources目录

目录结构如下所示：

![针对当前项目的gradle插件目录](https://img-blog.csdn.net/20160702135323958)

其中，`build.gradle`内容为：

```groovy
apply plugin: 'groovy'

dependencies {
    compile gradleApi()//gradle sdk
    compile localGroovy()//groovy sdk
}

repositories {
    jcenter()
}
```

SecondPlugin.groovy内容为：

```java
package  com.hc.second

import org.gradle.api.Plugin
import org.gradle.api.Project

public class SecondPlugin implements Plugin<Project> {

    void apply(Project project) {
        System.out.println("========================");
        System.out.println("这是第二个插件!");
        System.out.println("========================");
    }
}
```

在`app`这个`Module`中如何使用呢？直接在app的build.gradle下加入

```groovy
apply plugin: com.hc.second.SecondPlugin
```

`clean`一下，再`make project`，`messages`窗口信息如下：

![打印信息](https://img-blog.csdn.net/20160702135750329)