# Android Gradle Plugin打包Apk过程中的Transform API



**Transform API 是在1.5.0-beta1版开始使用，利用Transform API，第三方的插件可以在.class文件转为dex文件之前，对一些.class 文件进行处理。Transform API 简化了这个处理过程，而且使用起来很灵活。**

## 使用Transform API

使用Transform API主要是写一个类继承Transform，并把该Transform注入到打包过程中。
注入Transform很简单，先获取com.android.build.gradle.AppExtension对象，然后调用它的registerTransform()方法。
这个方法实际上是属于BaseExtension的，AppExtension继承自BaseExtension。

```java
#com.android.build.gradle.BaseExtension
public void registerTransform(@NonNull Transform transform, Object... dependencies) {
    transforms.add(transform);
    transformDependencies.add(Arrays.asList(dependencies));
}
```

注入Transform对象：

```java
AppExtension android = project.extensions.getByType(AppExtension)
android.registerTransform(new AJXTransform(project))
```

AJXTransform是自定义的Transform类：

```java
class AJXTransform extends Transform {
    Project project
    AJXTransform(Project project) {
        this.project = project
    }

    @Override
    String getName() {
        return "ajx"
    }

    @Override
    Set<QualifiedContent.ContentType> getInputTypes() {
        // 输入类型，可以使class文件，也可以是源码文件 ，这是表示输入的class文件
        return ImmutableSet.<QualifiedContent.ContentType> of(QualifiedContent.DefaultContentType.CLASSES)
    }

    @Override
    Set<? super QualifiedContent.Scope> getScopes() {
        // 作用范围
        return TransformManager.SCOPE_FULL_PROJECT
    }

    @Override
    boolean isIncremental() {
        //是否支持增量编译
        return false
    }

    @Override
    void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        //在这里对输入输出的class进行处理
    }
}
```

ContentType是一个接口，默认有一个枚举类型DefaultContentType实现了ContentType，包含有CLASSES和RESOURCES类型。

- CLASSES类型表示的是在jar包或者文件夹中的.class文件。
- RESOURCES类型表示的是标准的Java源文件。

Scope 作用范围

| Scope类型               | 说明                                                         |
| ----------------------- | ------------------------------------------------------------ |
| PROJECT                 | 只处理当前的项目                                             |
| SUB_PROJECTS            | 只处理子项目                                                 |
| EXTERNAL_LIBRARIES      | 只处理外部的依赖库                                           |
| TESTED_CODE             | 只处理测试代码                                               |
| PROVIDED_ONLY           | 只处理provided-only的依赖库                                  |
| PROJECT_LOCAL_DEPS      | 只处理当前项目的本地依赖,例如jar, aar（过期，被EXTERNAL_LIBRARIES替代） |
| SUB_PROJECTS_LOCAL_DEPS | 只处理子项目的本地依赖,例如jar, aar（过期，被EXTERNAL_LIBRARIES替代） |

Transform中的getInputTypes()方法和getScopes() 方法返回的是Set集合，因此这些类型是可以进行组合的。在`com.android.build.gradle.internal.pipeline.TransformManager`中就包含了多种Set集合。



![img](https://upload-images.jianshu.io/upload_images/2083810-09a2feedc88f88b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/687)

TransformManager.png

Transform 的 `isIncremental()` 方法表示是否支持增量编译，返回true的话表示支持，这个时候可以根据 `com.android.build.api.transform.TransformInput` 来获得更改、移除或者添加的文件目录或者jar包。

```java
public interface TransformInput {

    /**
     * Returns a collection of {@link JarInput}.
     */
    @NonNull
    Collection<JarInput> getJarInputs();

    /**
     * Returns a collection of {@link DirectoryInput}.
     */
    @NonNull
    Collection<DirectoryInput> getDirectoryInputs();
}
```

JarInput有一个方法是`getStatus()`来获取 `com.android.build.api.transform.Status`。Status是一个枚举类，包含了NOTCHANGED、ADDED、CHANGED、REMOVED，所以可以根据JarInput的status来对它进行相应的处理，比如添加或者移除。

DirectoryInput有一个方法`getChangedFiles()`开获取一个Map<File, Status>集合，所以可以遍历这个Map集合，然后根据File对应的Status来对File进行处理。

如果不支持增量编译，就在处理.class之前把之前的输出目录中的文件删除。

获取TransformInput对象是根据 `com.android.build.api.transform.TransformInvocation` 。

```java
public interface TransformInvocation {
// 返回transform运行的上下文，在android gradle plugin中有唯一的实现类TransformTask
    @NonNull
    Context getContext();

// 获取transform的输入
    @NonNull
    Collection<TransformInput> getInputs();

    /**
     * Returns the referenced-only inputs which are not consumed by this transformation.
     * @return the referenced-only inputs.
     */
    @NonNull Collection<TransformInput> getReferencedInputs();
    /**
     * Returns the list of secondary file changes since last. Only secondary files that this
     * transform can handle incrementally will be part of this change set.
     * @return the list of changes impacting a {@link SecondaryInput}
     */
    @NonNull Collection<SecondaryInput> getSecondaryInputs();

//TransformOutputProvider用于删除输出目录或者创建文件对应的生成目录
    @Nullable
    TransformOutputProvider getOutputProvider();

// transform过程是否支持增量编译
    boolean isIncremental();
}
```

TransformInvocation包含了输入、输出相关信息。其输出相关内容是由TransformOutputProvider来做处理。TransformOutputProvider的getContentLocation()方法可以获取文件的输出目录，如果目录存在的话直接返回，如果不存在就会重新创建一个。例如：

```java
// getContentLocation方法相当于创建一个对应名称表示的目录
// 是从0 、1、2开始递增。如果是目录，名称就是对应的数字，如果是jar包就类似0.jar
File outputDir = transformInvocation.outputProvider.getContentLocation("include", 
         dirInput.contentTypes, dirInput.scopes, Format.DIRECTORY)

File outputJar = transformInvocation.outputProvider.getContentLocation(jarInput.name
        , jarInput.contentTypes
        , jarInput.scopes
        , Format.JAR)
```

在执行编译过程中会生成对应的目录，例如在/app/build/intermediates/transforms目录下生成了一个名为`ajx`的目录，这个名称就是根据自定义的Transform类`getName()`方法返回的字符串来的。



![img](https://upload-images.jianshu.io/upload_images/2083810-c5cb7d8fcabd333d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/263)

transforms下的ajx目录.png





![img](https://upload-images.jianshu.io/upload_images/2083810-7fa9504bbe5ff237.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/330)

transforms下的ajx中内容.png

ajx目录下还会有一个名为`__content__`的.json文件。该文件中展示了ajx中文件目录下的内容



![img](https://upload-images.jianshu.io/upload_images/2083810-7e4d4360b0ffb331.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/316)

image.png



其实当你注入一个自定义的Transform的时候还会生成对应的Task，即TransformTask，该Task还会有一个对应的名称，例如：



![img](https://upload-images.jianshu.io/upload_images/2083810-a422dfd4590b410f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/404)

TransformTask名称.png

这个名称生成过程是在 TransformManager 的 `addTransform()` 中：

```java
public <T extends Transform> Optional<TransformTask> addTransform(
            @NonNull TaskFactory taskFactory,
            @NonNull TransformVariantScope scope,
            @NonNull T transform,
            @Nullable TransformTask.ConfigActionCallback<T> callback) {
...
        String taskName = scope.getTaskName(getTaskNamePrefix(transform));
...
}

static String getTaskNamePrefix(@NonNull Transform transform) {
    StringBuilder sb = new StringBuilder(100);
    sb.append("transform");

    sb.append(
            transform
                    .getInputTypes()
                    .stream()
                    .map(
                            inputType ->
                                    CaseFormat.UPPER_UNDERSCORE.to(
                                            CaseFormat.UPPER_CAMEL, inputType.name()))
                    .sorted() // Keep the order stable.
                    .collect(Collectors.joining("And")));
    sb.append("With");
    StringHelper.appendCapitalized(sb, transform.getName());
    sb.append("For");

    return sb.toString();
}
```

在Transform中，其`transform()` 方法是重头戏，需要对输入的文件进行处理，然后放到输出目录中。

例如，我在[aop-tech](https://github.com/sososeen09/android-blog-demos/tree/master/aop-tech)中就是自定义一个Transform子类，然后在transform过程把输入的目录中的.class文件或者jar包中的.class 文件使用aspectj进行处理。如果对aspectj不太了解的，可以查看我之前写的一篇文章：[AOP开发——AspectJ的使用](https://www.jianshu.com/p/c66f4e3113b3)。

```java
@Override
void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
    TransformTask transformTask = (TransformTask) transformInvocation.context
    //VariantCache 就是保存一些跟当前variant相关的一些缓存，以及在支持增量编译的情况下存储一些信息
    VariantCache variantCache = new VariantCache(ajxProcedure.project, ajxProcedure.ajxCache, transformTask.variantName)

    if (transformInvocation.isIncremental()) {
        //TODO 增量
        print("====================增量编译=================")
    }else {
        print("====================非增量编译=================")
        //非增量,需要删除输出目录
        transformInvocation.outputProvider.deleteAll()
        variantCache.reset()

        AJXFileProcess ajxFileProcess = new AJXFileProcess(project, variantCache, transformInvocation)
        ajxFileProcess.proceed()
        AJXTaskProcess ajxTaskProcess = new AJXTaskProcess(project, variantCache, transformInvocation)
        ajxTaskProcess.proceed()
    }
}
```

## Transform中的transform()方法是如何被执行的

我们在前面讲到的调用`android.registerTransform(transform)`注册方法，实际上只是把Transform对象放到了一个List集合中。那么什么时候用到这个集合呢？也就是说Transform的transform()方法是什么时候被执行的呢？

在这里可以先告诉你答案，Transform的transform()方法是在TransformTask的transform()方法中执行的。

```java
# com.android.build.gradle.internal.pipeline.TransformTask
@TaskAction
void transform(final IncrementalTaskInputs incrementalTaskInputs)
        throws IOException, TransformException, InterruptedException {
    ...
    recorder.record(
            ExecutionType.TASK_TRANSFORM,
            executionInfo,
            getProject().getPath(),
            getVariantName(),
            new Recorder.Block<Void>() {
                @Override
                public Void call() throws Exception {

                    transform.transform(
                            new TransformInvocationBuilder(TransformTask.this)
                                    .addInputs(consumedInputs.getValue())
                                    .addReferencedInputs(referencedInputs.getValue())
                                    .addSecondaryInputs(changedSecondaryInputs.getValue())
                                    .addOutputProvider(
                                            outputStream != null
                                                    ? outputStream.asOutput()
                                                    : null)
                                    .setIncrementalMode(isIncremental.getValue())
                                    .build());

                    if (outputStream != null) {
                        outputStream.save();
                    }
                    return null;
                }
            });
}
```

实际上每个Transform都会有一个对应的TransformTask，TransformTask本质上就是表示Gradle中的一个Task，那么一个Task在执行的时候其`@TaskAction`注解的方法会被执行，也就是`com.android.build.gradle.internal.pipeline.TransformTask#transform()`方法会被执行，在该方法中会调用该TransformTask对应的Transform对象的transform()方法。

写过Gradle插件的都知道，在build.gradle中apply 插件后其apply(project)方法就会调用。例如我们在一个app中应用的是`apply plugin: 'com.andorid.application'` ，这个实际上引入的就是AppPlugin，其apply()方法会被调用。关于这个源码过程我们就不多说了，我们只讲一个这个过程，感兴趣的可以查看文章末尾的相关文章。

->com.android.build.gradle.BasePlugin#apply()
->com.android.build.gradle.BasePlugin#createTasks()
->com.android.build.gradle.BasePlugin#createAndroidTasks()
->com.android.build.gradle.internal.VariantManager#createAndroidTasks()
->com.android.build.gradle.internal.VariantManager#createTasksForVariantData()
->com.android.build.gradle.internal.ApplicationTaskManager#createTasksForVariantScope() **//该方法中干的事很多，可以重点关注**
->com.android.build.gradle.internal.ApplicationTaskManager#addCompileTask()
->com.android.build.gradle.internal.TaskManager#createPostCompilationTasks() **//在这个方法中会调用AppExtension的getTransforms()方法，也就是我们之前注册的transform**
-> com.android.build.gradle.internal.pipeline.TransformManager#addTransform() **//在这个方法中创建了TransformTask**

在上面的这些流程中TaskManager的**createPostCompilationTasks()**需要重点关注，在该方法中对Apk打包过程中的各种Transform进行处理，创建对应了TransformTask并构建隐式的依赖关系。

到此TransformTask是创建完毕，TransformTask相当于是对我们自定义的Transform进行的包装。

那么这个TransformTask是什么时候执行呢？
我们可以显式地使用Gradle命令去执行一个Task，这是没问题的。但为什么我们执行`gradle assemble`命令的时候，TransformTask也会执行呢？这个就牵涉到Task的依赖关系了。假设TaskA依赖TaskB，那么如果我们要执行TaskA，那么在TaskA执行之前TaskB就会执行。但是要注意一点就是，假设TaskA依赖TaskB和TaskC，那么只能保证TaskA执行之前TaskB和TaskC都执行了，并不能保证TaskB和TaskC的执行顺序。

构建Task的依赖关系可以显式的调用其 `dependsOn()` 方法。 但是我在查看Transform API的过程中发现这个TransformTask之间并没有显式地调用`dependsOn()` 方法来保证依赖关系，难道这个TransformTask的执行顺序是任意的吗？如果是任意的，比如一个task是dexMerger是要把每个class编译成的dex合成为一个dex，而此时输入目录中还没有dex存在，该任务不就失败了吗？android gradle plugin 的开发者当然不会让这种情况存在了。

实际情况是Task除了显式地通过dependsOn来指定Task依赖，其实还可以**使用Task依赖推断来判断依赖关系，Gradle通过使用一个Task的输出作为另一个Task的输入，就可以推断出依赖关系。**
在Transform API中，使用的是 TransformStream 来连接TransformTask的依赖关系，进行可以控制Transform的执行顺序。
计算TransformTask的输入输出是在 TransformManager 的 addTransform() 方法中。
通过输入输出已经隐式地确定了TransformTask的依赖关系。

```java
# com.android.build.gradle.internal.pipeline.TransformManager
public <T extends Transform> Optional<TransformTask> addTransform(
        @NonNull TaskFactory taskFactory,
        @NonNull TransformVariantScope scope,
        @NonNull T transform,
        @Nullable TransformTask.ConfigActionCallback<T> callback) {
...
    List<TransformStream> inputStreams = Lists.newArrayList();
    String taskName = scope.getTaskName(getTaskNamePrefix(transform));

    // get referenced-only streams
    List<TransformStream> referencedStreams = grabReferencedStreams(transform);

    // find input streams, and compute output streams for the transform.
    // 通过之前添加的Transform来计算输入，并计算输出
    IntermediateStream outputStream = findTransformStreams(
            transform,
            scope,
            inputStreams,
            taskName,
            scope.getGlobalScope().getBuildDir());
   ...
    transforms.add(transform);

    // create the task...
    // 在创建Task的过程中传入了输入和输出，上一个Task的输出是该Task的输入，这就保证了Task的一个隐式的依赖关系
    TransformTask task =
            taskFactory.create(
                    new TransformTask.ConfigAction<>(
                            scope.getFullVariantName(),
                            taskName,
                            transform,
                            inputStreams,
                            referencedStreams,
                            outputStream,
                            recorder,
                            callback));

    return Optional.ofNullable(task);
}
```

所以通过以上分析也可以当我们自定义Plugin要注入多个Transform的时候，按照添加顺序来保证依赖关系，先添加的Transform先执行。对于下面的例子，transformA的`transform()`方法会先于transformB执行。

```java
android.registerTransform(transformA);
android.registerTransform(transformB);
```

## Android Gradle Plugin中的Transform子类

Android Gradle Plugin中有多个Transform子类，在编写自己的Transform类的时候可以作为一个参考。





![img](https://upload-images.jianshu.io/upload_images/2083810-f076900e17578724.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/534)

Transform子类.png

可以在图中看到，Google已经在使用kotlin写Gradle插件了。

## 总结

自定义Transform并注入：

```java
class TransformPlugin implements Plugin<Project> {
    @Override
    void apply(Project project) {
        project.android.registerTransform(new ClassTransform(project))
    }
}

class ClassTransform extends Transform {
```

Transform API的执行有顺序，每一个Transform都对应一个TransformTask和TransformStream，通过输入输出隐式的构成Task依赖关系。开发者自定义的Transform按照注册顺序执行。

## 补充——源码调试

在之前的一篇文章[启用Gradle远程调试](https://www.jianshu.com/p/b86330a877db)中介绍了如何调试Gradle插件，实际上使用该方式也可以用来调试查看Android Gradle Plugin的执行流程。这个时候最好要找到有能下载到源码的版本，比如3.1.2。

```java
buildscript {
    repositories {
        google()
        jcenter()
    }
    dependencies {
 ...
        classpath 'com.android.tools.build:gradle:3.1.2'
...
    }
}
```

以下是App或者Library插件对应的类的继承关系图





![img](https://upload-images.jianshu.io/upload_images/2083810-96bd7e87a558f504.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/369)

如果你想要调试app打包过程，可以在 AppPlugin 的 `apply()` 方法中打断点。它直接调用了其父类 BasePlugin 的 `apply()` 方法。

```java
#com.android.build.gradle.AppPlugin
@Override
public void apply(@NonNull Project project) {
    super.apply(project);
}
```