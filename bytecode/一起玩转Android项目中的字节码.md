# 一起玩转Android项目中的字节码



作为Android开发，日常写Java代码之余，是否想过，玩玩class文件？直接对class文件的字节码下手，我们可以做很多好玩的事情，比如：

- 对全局所有class插桩，做UI，内存，网络等等方面的性能监控
- 发现某个第三方依赖，用起来不爽，但是不想拿它的源码修改再重新编译，而想对它的class直接做点手脚
- 每次写打log时，想让TAG自动生成，让它默认就是当前类的名称，甚至你想让log里自动加上当前代码所在的行数，更方便定位日志位置
- Java自带的动态代理太弱了，只能对接口类做动态代理，而我们想对任何类做动态代理

为了实现上面这些想法，可能我们最开始的第一反应，都是能否通过代码生成技术、APT，抑或反射、抑或动态代理来实现，但是想来想去，貌似这些方案都不能很好满足上面的需求，而且，有些问题不能从Java文件入手，而应该从class文件寻找突破。而从class文件入手，我们就不得不来近距离接触一下字节码！

JVM平台上，修改、生成字节码无处不在，从ORM框架（如Hibernate, MyBatis）到Mock框架（如Mockio），再到Java Web中的常青树Spring框架，再到新兴的JVM语言[Kotlin的编译器](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2FJetBrains%2Fkotlin%2Ftree%2Fv1.2.30%2Fcompiler%2Fbackend%2Fsrc%2Forg%2Fjetbrains%2Fkotlin%2Fcodegen)，还有大名鼎鼎的[cglib](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fcglib%2Fcglib)项目，都有字节码的身影。

字节码相关技术的强大之处自然不用多说，而且在Android开发中，无论是使用Java开发和Kotlin开发，都是JVM平台的语言，所以如果我们在Android开发中，使用字节码技术做一下hack，还可以天然地兼容Java和Kotlin语言。

 

阅读这篇文章，读者最好有Android开发以及编写简单Gradle插件的背景知识。

话不多说，让我们开始吧。

# 一、Transform

## 引入Transform

[Transform](https://link.juejin.im/?target=http%3A%2F%2Fgoogle.github.io%2Fandroid-gradle-dsl%2Fjavadoc%2F3.2%2F)是Android gradle plugin 1.5开始引入的概念。

我们先从如何引入Transform依赖说起，首先我们需要编写一个自定义插件，然后在插件中注册一个自定义Transform。这其中我们需要先通过gradle引入Transform的依赖，这里有一个坑，Transform的库最开始是独立的，后来从2.0.0版本开始，被归入了Android编译系统依赖的gradle-api中，让我们看看Transform在[jcenter](https://link.juejin.im/?target=https%3A%2F%2Fdl.bintray.com%2Fandroid%2Fandroid-tools%2Fcom%2Fandroid%2Ftools%2Fbuild%2Ftransform-api%2F)上的历个版本。

[![img](http://5b0988e595225.cdn.sohucs.com/images/20181215/43796b6bc51847ffad3fb24bf580e588.jpeg)](https://link.juejin.im/?target=http%3A%2F%2Fquinnchen.me%2Fimages%2Ftransform_2.png)

所以，很久很久以前我引入transform依赖是这样

```groovy
compile 'com.android.tools.build:transform-api:1.5.0'
```

现在是这样

```groovy
//从2.0.0版本开始就是在gradle-api中了
implementation 'com.android.tools.build:gradle-api:3.1.4'
```

然后，让我们在自定义插件中注册一个自定义Transform，gradle插件可以使用java，groovy，kotlin编写，我这里选择使用java。

```groovy
public class CustomPlugin implements Plugin<Project> {
    @SuppressWarnings("NullableProblems")
    @Override
    public void apply(Project project) {
        AppExtension appExtension = (AppExtension)project.getProperties().get("android");
        appExtension.registerTransform(new CustomTransform(), Collections.EMPTY_LIST);
    }
}
```

那么如何写一个自定义Transform呢？

## Transform的原理与应用

介绍如何应用Transform之前，我们先介绍Transform的原理，一图胜千言

[![img](http://5b0988e595225.cdn.sohucs.com/images/20181215/13273522fd1e4ae6bf21fb44249d20b9.jpeg)](https://link.juejin.im/?target=http%3A%2F%2Fquinnchen.me%2Fimages%2Ftransformconsume_transform.png)

每个Transform其实都是一个gradle task，Android编译器中的TaskManager将每个Transform串连起来，第一个Transform接收来自javac编译的结果，以及已经拉取到在本地的第三方依赖（jar. aar），还有resource资源，注意，这里的resource并非android项目中的res资源，而是asset目录下的资源。这些编译的中间产物，在Transform组成的链条上流动，每个Transform节点可以对class进行处理再传递给下一个Transform。我们常见的混淆，Desugar等逻辑，它们的实现如今都是封装在一个个Transform中，而我们自定义的Transform，会插入到这个Transform链条的最前面。

但其实，上面这幅图，只是展示Transform的其中一种情况。而Transform其实可以有两种输入，一种是消费型的，当前Transform需要将消费型型输出给下一个Transform，另一种是引用型的，当前Transform可以读取这些输入，而不需要输出给下一个Transform，比如Instant Run就是通过这种方式，检查两次编译之间的diff的。至于怎么在一个Transform中声明两种输入，以及怎么处理两种输入，后面将有示例代码。

为了印证Transform的工作原理和应用方式，我们也可以从Android gradle plugin源码入手找出证据，在TaskManager中，有一个方法`createPostCompilationTasks`.为了避免贴篇幅太长的源码，这里附上链接

[TaskManager#createPostCompilationTasks](https://link.juejin.im/?target=https%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Ftools%2Fbase%2F%2B%2Fstudio-master-dev%2Fbuild-system%2Fgradle-core%2Fsrc%2Fmain%2Fjava%2Fcom%2Fandroid%2Fbuild%2Fgradle%2Finternal%2FTaskManager.java%232154)

这个方法的脉络很清晰，我们可以看到，Jacoco，Desugar，MergeJavaRes，AdvancedProfiling，Shrinker，Proguard, JarMergeTransform, MultiDex, Dex都是通过Transform的形式一个个串联起来。其中也有将我们自定义的Transform插进去。

讲完了Transform的数据流动的原理，我们再来介绍一下Transform的输入数据的过滤机制，Transform的数据输入，可以通过Scope和ContentType两个维度进行过滤。

[![img](http://5b0988e595225.cdn.sohucs.com/images/20181215/02dced3c38364f2fb0a92cc1d9d62d60.png)](https://link.juejin.im/?target=http%3A%2F%2Fquinnchen.me%2Fimages%2Ftransformscope%26contenttype.png)

ContentType，顾名思义，就是数据类型，在插件开发中，我们一般只能使用CLASSES和RESOURCES两种类型，注意，其中的CLASSES已经包含了class文件和jar文件

[![img](http://5b0988e595225.cdn.sohucs.com/images/20181215/21a5a2e1c2024fa4aa292c8a83690308.jpeg)](https://link.juejin.im/?target=http%3A%2F%2Fquinnchen.me%2Fimages%2FtransformContentType.png)

从图中可以看到，除了CLASSES和RESOURCES，还有一些我们开发过程无法使用的类型，比如DEX文件，这些隐藏类型在一个独立的枚举类[ExtendedContentType](https://link.juejin.im/?target=https%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Ftools%2Fbase%2F%2B%2Fstudio-master-dev%2Fbuild-system%2Fgradle-core%2Fsrc%2Fmain%2Fjava%2Fcom%2Fandroid%2Fbuild%2Fgradle%2Finternal%2Fpipeline%2FExtendedContentType.java%3Fautodive%3D0%2F%2F%2F)中，这些类型只能给Android编译器使用。另外，我们一般使用 [TransformManager](https://link.juejin.im/?target=https%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Ftools%2Fbase%2F%2B%2Fgradle_2.0.0%2Fbuild-system%2Fgradle-core%2Fsrc%2Fmain%2Fgroovy%2Fcom%2Fandroid%2Fbuild%2Fgradle%2Finternal%2Ftransforms%2FShrinkResourcesTransform.java)中提供的几个常用的ContentType集合和Scope集合，如果是要处理所有class和jar的字节码，ContentType我们一般使用`TransformManager.CONTENT_CLASS`。

Scope相比ContentType则是另一个维度的过滤规则，

[![img](http://5b0988e595225.cdn.sohucs.com/images/20181215/00ee5e92672740b2977373573b5695e3.jpeg)](https://link.juejin.im/?target=http%3A%2F%2Fquinnchen.me%2Fimages%2FtransformScope.png)

我们可以发现，左边几个类型可供我们使用，而我们一般都是组合使用这几个类型，[TransformManager](https://link.juejin.im/?target=https%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Ftools%2Fbase%2F%2B%2Fgradle_2.0.0%2Fbuild-system%2Fgradle-core%2Fsrc%2Fmain%2Fgroovy%2Fcom%2Fandroid%2Fbuild%2Fgradle%2Finternal%2Ftransforms%2FShrinkResourcesTransform.java)有几个常用的Scope集合方便开发者使用。
如果是要处理所有class字节码，Scope我们一般使用`TransformManager.SCOPE_FULL_PROJECT`。

好，目前为止，我们介绍了Transform的数据流动的原理，输入的类型和过滤机制，我们再写一个简单的自定义Transform，让我们对Transform可以有一个更具体的认识

```groovy
public class CustomTransform extends Transform {
    public static final String TAG = "CustomTransform";
    public CustomTransform() {
        super();
    }
   @Override
   public String getName() {
 			  return "CustomTransform";
    }
   @Override
    public void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        super.transform(transformInvocation);
		    //当前是否是增量编译
        boolean isIncremental = transformInvocation.isIncremental();
        //消费型输入，可以从中获取jar包和class文件夹路径。需要输出给下一个任务
        Collection<TransformInput> inputs = transformInvocation.getInputs();
        //引用型输入，无需输出。
        Collection<TransformInput> referencedInputs = transformInvocation.getReferencedInputs();
        //OutputProvider管理输出路径，如果消费型输入为空，你会发现OutputProvider == null
        TransformOutputProvider outputProvider = transformInvocation.getOutputProvider();
       	for(TransformInput input : inputs) {
            for(JarInput jarInput : input.getJarInputs()) {
                  File dest = outputProvider.getContentLocation(
                      jarInput.getFile().getAbsolutePath(),
                      jarInput.getContentTypes(),
                      jarInput.getScopes(),
                          Format.JAR);
                  //将修改过的字节码copy到dest，就可以实现编译期间干预字节码的目的了        
                  FileUtils.copyFile(jarInput.getFile(), dest);
             }
            for(DirectoryInput directoryInput : input.getDirectoryInputs()) {
                File dest = outputProvider.getContentLocation(directoryInput.getName()
                        directoryInput.getContentTypes(), directoryInput.getScopes(),
                        Format.DIRECTORY);
                //将修改过的字节码copy到dest，就可以实现编译期间干预字节码的目的了        
                FileUtils.copyDirectory(directoryInput.getFile(), dest);
            }
        }
    }
    @Override
    public Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS;
    }
    @Override
    public Set<? super QualifiedContent.Scope> getScopes() {
        return TransformManager.SCOPE_FULL_PROJECT;
    }

    @Override
    public Set<QualifiedContent.ContentType> getOutputTypes() {
        return super.getOutputTypes();
    }


    @Override
    public Set<? super QualifiedContent.Scope> getReferencedScopes() {
        return TransformManager.EMPTY_SCOPES;
    }


    @Override
    public Map<String, Object> getParameterInputs() {
        return super.getParameterInputs();
    }

    @Override
    public boolean isCacheable() {
        return true;
    }

    @Override 
    public boolean isIncremental() {
        return true; //是否开启增量编译
    }
}
```

可以看到，在transform方法中，我们将每个jar包和class文件复制到dest路径，这个dest路径就是下一个Transform的输入数据，而在复制时，我们就可以做一些狸猫换太子，偷天换日的事情了，先将jar包和class文件的字节码做一些修改，再进行复制即可，至于怎么修改字节码，就要借助我们后面介绍的ASM了。而如果开发过程要看你当前transform处理之后的class/jar包，可以到
/build/intermediates/transforms/CustomTransform/下查看，你会发现所有jar包命名都是123456递增，这是正常的，这里的命名规则可以在OutputProvider.getContentLocation的具体实现中找到

```groovy
public synchronized File getContentLocation(
        @NonNull String name,
        @NonNull Set<ContentType> types,
        @NonNull Set<? super Scope> scopes,
        @NonNull Format format) {
    // runtime check these since it's (indirectly) called by 3rd party transforms.
    checkNotNull(name);
    checkNotNull(types);
    checkNotNull(scopes);
    checkNotNull(format);
    checkState(!name.isEmpty());
    checkState(!types.isEmpty());
    checkState(!scopes.isEmpty());
    // search for an existing matching substream.
    for (SubStream subStream : subStreams) {
        // look for an existing match. This means same name, types, scopes, and format.
        if (name.equals(subStream.getName())
                && types.equals(subStream.getTypes())
                && scopes.equals(subStream.getScopes())
                && format == subStream.getFormat()) {
            return new File(rootFolder, subStream.getFilename());
        }
    }



    //按位置递增！！	
    // didn't find a matching output. create the new output
    SubStream newSubStream = new SubStream(name, nextIndex++, scopes, types, format, true);
    subStreams.add(newSubStream);
    return new File(rootFolder, newSubStream.getFilename());
}
```

## Transform的优化：增量与并发

到此为止，看起来Transform用起来也不难，但是，如果直接这样使用，会大大拖慢编译时间，为了解决这个问题，摸索了一段时间后，也借鉴了Android编译器中Desugar等几个Transform的实现，发现我们可以使用增量编译，并且上面transform方法遍历处理每个jar/class的流程，其实可以并发处理，加上一般编译流程都是在PC上，所以我们可以尽量敲诈机器的资源。

想要开启增量编译，我们需要重写Transform的这个接口，返回true。

```groovy
@Override 
public boolean isIncremental() {
    return true;
}
```

虽然开启了增量编译，但也并非每次编译过程都是支持增量的，毕竟一次clean build完全没有增量的基础，所以，我们需要检查当前编译是否是增量编译。

如果不是增量编译，则清空output目录，然后按照前面的方式，逐个class/jar处理
如果是增量编译，则要检查每个文件的Status，Status分四种，并且对这四种文件的操作也不尽相同

[![img](http://5b0988e595225.cdn.sohucs.com/images/20181215/cceb01be444249ac83dd18ce76789eb6.png)](https://link.juejin.im/?target=http%3A%2F%2Fquinnchen.me%2Fimages%2Ftransform_Status.png)

- NOTCHANGED: 当前文件不需处理，甚至复制操作都不用；
- ADDED、CHANGED: 正常处理，输出给下一个任务；
- REMOVED: 移除outputProvider获取路径对应的文件。

大概实现可以一起看看下面的代码

```groovy
@Override
public void transform(TransformInvocation transformInvocation){
    Collection<TransformInput> inputs = transformInvocation.getInputs();
    TransformOutputProvider outputProvider = transformInvocation.getOutputProvider();
    boolean isIncremental = transformInvocation.isIncremental();
    //如果非增量，则清空旧的输出内容
    if(!isIncremental) {
        outputProvider.deleteAll();
    }	
    for(TransformInput input : inputs) {
        for(JarInput jarInput : input.getJarInputs()) {
            Status status = jarInput.getStatus();
            File dest = outputProvider.getContentLocation(
                    jarInput.getName(),
                    jarInput.getContentTypes(),
                    jarInput.getScopes(),
                    Format.JAR);
            if(isIncremental && !emptyRun) {
                switch(status) {
                    case NOTCHANGED:
                        continue;
                    case ADDED:
                    case CHANGED:
                        transformJar(jarInput.getFile(), dest, status);
                        break;
                    case REMOVED:
                        if (dest.exists()) {
                            FileUtils.forceDelete(dest);
                        }
                        break;
                }
            } else {
                transformJar(jarInput.getFile(), dest, status);
            }
        }

        for(DirectoryInput directoryInput : input.getDirectoryInputs()) {
            File dest = outputProvider.getContentLocation(directoryInput.getName(),
                    directoryInput.getContentTypes(), directoryInput.getScopes(),
                    Format.DIRECTORY);
            FileUtils.forceMkdir(dest);
            if(isIncremental && !emptyRun) {
                String srcDirPath = directoryInput.getFile().getAbsolutePath();
                String destDirPath = dest.getAbsolutePath();
                Map<File, Status> fileStatusMap = directoryInput.getChangedFiles();
                for (Map.Entry<File, Status> changedFile : fileStatusMap.entrySet()) {
                    Status status = changedFile.getValue();
                    File inputFile = changedFile.getKey();
                    String destFilePath = inputFile.getAbsolutePath().replace(srcDirPath, destDirPath);
                    File destFile = new File(destFilePath);
                    switch (status) {
                        case NOTCHANGED:
                            break;
                        case REMOVED:
                            if(destFile.exists()) {
                                FileUtils.forceDelete(destFile);
                            }
                            break;
                        case ADDED:
                        case CHANGED:
                            FileUtils.touch(destFile);
                            transformSingleFile(inputFile, destFile, srcDirPath);
                            break;
                    }
                }
            } else {
                transformDir(directoryInput.getFile(), dest);
            }
        }
    }
}
```

这就能为我们的编译插件提供增量的特性。

实现了增量编译后，我们最好也支持并发编译，并发编译的实现并不复杂，只需要将上面处理单个jar/class的逻辑，并发处理，最后阻塞等待所有任务结束即可。

```groovy
private WaitableExecutor waitableExecutor = WaitableExecutor.useGlobalSharedThreadPool();
//异步并发处理jar/class
waitableExecutor.execute(() -> {
    bytecodeWeaver.weaveJar(srcJar, destJar);
    return null;
});

waitableExecutor.execute(() -> {
    bytecodeWeaver.weaveSingleClassToFile(file, outputFile, inputDirPath);
    return null;
});  
//等待所有任务结束
waitableExecutor.waitForTasksWithQuickFail(true);
```

接下来我们对编译速度做一个对比，每个实验都是5次同种条件下编译10次，去除最大大小值，取平均时间

首先，在QQ邮箱Android客户端工程中，我们先做一次cleanbuild

```groovy
./gradlew clean assembleDebug --profile
```

给项目中添加UI耗时统计，全局每个方法（包括普通class文件和第三方jar包中的所有class）的第一行和最后一行都进行插桩，实现方式就是Transform+ASM，对比一下并发Transform和非并发Transform下，Tranform这一步的耗时

[![img](http://5b0988e595225.cdn.sohucs.com/images/20181215/3674c726aa354123aaefd4dc4965c6c3.png)](https://link.juejin.im/?target=http%3A%2F%2Fquinnchen.me%2Fimages%2Ftransform_time_1.png)

可以发现，并发编译，基本比非并发编译速度提高了80%。效果很显著。

然后，让我们再做另一个试验，我们在项目中模拟日常修改某个class文件的一行代码，这时是符合增量编译的环境的。然后在刚才基础上还是做同样的插桩逻辑，对比增量Transform和全量Transform的差异。

```groovy
./gradlew assembleDebug --profile
```

[![img](http://5b0988e595225.cdn.sohucs.com/images/20181215/fdd3c5cdf303486a8a301f076a870a3e.png)](https://link.juejin.im/?target=http%3A%2F%2Fquinnchen.me%2Fimages%2Ftransform_time_2.png)

可以发现，增量的速度比全量的速度提升了3倍多，而且这个速度优化会随着工程的变大而更加显著。

数据表明，增量和并发对编译速度的影响是很大的。而我在查看Android gradle plugin自身的十几个Transform时，发现它们实现方式也有一些区别，有些用kotlin写，有些用java写，有些支持增量，有些不支持，而且是代码注释写了一个大大的FIXME, To support incremental build。所以，讲道理，现阶段的Android编译速度，还是有提升空间的。

上面我们介绍了Transform，以及如何高效地在编译期间处理所有字节码，那么具体怎么处理字节码呢？接下来让我们一起看看JVM平台上的处理字节码神兵利器，ASM!

# 二、ASM

ASM的官网在这里[asm.ow2.io/](https://link.juejin.im/?target=https%3A%2F%2Fasm.ow2.io%2F)，贴一下它的主页介绍，一起感受下它的强大

[![img](https://user-gold-cdn.xitu.io/2018/12/9/16791ec7aee6736f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)](https://link.juejin.im/?target=http%3A%2F%2Fquinnchen.me%2Fimages%2Ftransform_asm_introduce.png)

JVM平台上，处理字节码的框架最常见的就三个，ASM，Javasist，AspectJ。我尝试过Javasist，而AspectJ也稍有了解，最终选择ASM，因为使用它可以更底层地处理字节码的每条命令，处理速度、内存占用，也优于其他两个框架。

我们可以来做一个对比，上面我们所做的计算编译时间实验的基础上，做如下试验，分别用ASM和Javasist全量处理工程所有class，并且都不开启并发处理的情况下，一次clean build中，transform的耗时对比如下

[![img](https://user-gold-cdn.xitu.io/2018/12/9/16791ec7d5c49fa1?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)](https://link.juejin.im/?target=http%3A%2F%2Fquinnchen.me%2Fimages%2Ftransform_asm_javasist.png)

ASM相比Javasist的优势非常显著，ASM相比其他字节码操作库的效率和性能优势应该毋庸置疑的，毕竟是诸多JVM语言钦定的字节码生成库。

我们这部分将来介绍ASM，但是由于篇幅问题，不会从字节码的基础展开介绍，会通过几个实例的实现介绍一些字节码的相关知识，另外还会介绍ASM的使用，以及ASM解析class文件结构的原理，还有应用于Android插件开发时，遇到的问题，及其解决方案。

## ASM的引入

下面是一份完整的gradle自定义plugin + transform + asm所需依赖，注意一下，此处两个gradleApi的区别

```groovy
dependencies {
    //使用项目中指定的gradle wrapper版本，插件中使用的Project对象等等就来自这里
    implementation gradleApi()     
    //使用本地的groovy
    implementation localGroovy()   
    //Android编译的大部分gradle源码，比如上面讲到的TaskManager
    implementation 'com.android.tools.build:gradle:3.1.4'    
    //这个依赖里其实主要存了transform的依赖，注意，这个依赖不同于上面的gradleApi()
    implementation 'com.android.tools.build:gradle-api:3.1.4'   
    //ASM相关
    implementation 'org.ow2.asm:asm:5.1'                        
    implementation 'org.ow2.asm:asm-util:5.1'                    
    implementation 'org.ow2.asm:asm-commons:5.1'   
}
```

### ASM的应用

ASM设计了两种API类型，一种是Tree API, 一种是基于Visitor API(visitor pattern)，

Tree API将class的结构读取到内存，构建一个树形结构，然后需要处理Method、Field等元素时，到树形结构中定位到某个元素，进行操作，然后把操作再写入新的class文件。

Visitor API则将通过接口的方式，分离读class和写class的逻辑，一般通过一个ClassReader负责读取class字节码，然后ClassReader通过一个ClassVisitor接口，将字节码的每个细节按顺序通过接口的方式，传递给ClassVisitor（你会发现ClassVisitor中有多个visitXXXX接口），这个过程就像ClassReader带着ClassVisitor游览了class字节码的每一个指令。

上面这两种解析文件结构的方式在很多处理结构化数据时都常见，一般得看需求背景选择合适的方案，而我们的需求是这样的，出于某个目的，寻找class文件中的一个hook点，进行字节码修改，这种背景下，我们选择Visitor API的方式比较合适。

让我们来写一个简单的demo，这段代码很简单，通过Visitor API读取一个class的内容，保存到另一个文件

```groovy
private void copy(String inputPath, String outputPath) {
    try {
        FileInputStream is = new FileInputStream(inputPath);
        ClassReader cr = new ClassReader(is);
        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
        cr.accept(cw, 0);
        FileOutputStream fos = new FileOutputStream(outputPath);
        fos.write(cw.toByteArray());
        fos.close();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

首先，我们通过ClassReader读取某个class文件，然后定义一个ClassWriter，这个ClassWriter我们可以看它源码，其实就是一个ClassVisitor的实现，负责将ClassReader传递过来的数据写到一个字节流中，而真正触发这个逻辑就是通过ClassWriter的accept方式。

```java
public void accept(ClassVisitor classVisitor, Attribute[] attributePrototypes, int parsingOptions) {

    // 读取当前class的字节码信息
    int accessFlags = this.readUnsignedShort(currentOffset);
    String thisClass = this.readClass(currentOffset + 2, charBuffer);
    String superClass = this.readClass(currentOffset + 4, charBuffer);
    String[] interfaces = new String[this.readUnsignedShort(currentOffset + 6)];
    
  	//classVisitor就是刚才accept方法传进来的ClassWriter，每次visitXXX都负责将字节码的信息存储起来
    classVisitor.visit(this.readInt(this.cpInfoOffsets[1] - 7), accessFlags, thisClass, signature, superClass, interfaces);


    /**
        略去很多visit逻辑
    */

    //visit Attribute
    while(attributes != null) {
        Attribute nextAttribute = attributes.nextAttribute;
        attributes.nextAttribute = null;
        classVisitor.visitAttribute(attributes);
        attributes = nextAttribute;
    }


    /**
        略去很多visit逻辑
    */
    classVisitor.visitEnd();
}
```

最后，我们通过ClassWriter的toByteArray()，将从ClassReader传递到ClassWriter的字节码导出，写入新的文件即可。这就完成了class文件的复制，这个demo虽然很简单，但是涵盖了ASM使用Visitor API修改字节码最底层的原理，大致流程如图

[![img](https://user-gold-cdn.xitu.io/2018/12/9/16791ec7d70efc38?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)](https://link.juejin.im/?target=http%3A%2F%2Fquinnchen.me%2Fimages%2Ftransform_ASM-1.png)

我们来分析一下，不难发现，如果我们要修改字节码，就是要从ClassWriter入手，上面我们提到ClassWriter中每个visitXXX（这些接口实现自ClassVisitor）都会保存字节码信息并最终可以导出，那么如果我们可以代理ClassWriter的接口，就可以干预最终字节码的生成了。

那么上面的图就应该是这样

[![img](https://user-gold-cdn.xitu.io/2018/12/9/16791ec7dac33a28?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)](https://link.juejin.im/?target=http%3A%2F%2Fquinnchen.me%2Fimages%2Ftransform_ASM-2.png)

我们只要稍微看一下ClassVisitor的代码，发现它的构造函数，是可以接收另一个ClassVisitor的，从而通过这个ClassVisitor代理所有的方法。让我们来看一个例子，为class中的每个方法调用语句的开头和结尾插入一行代码

修改前的方法是这样

```java
private static void printTwo() {
    printOne();
    printOne();
}
```

被修改后的方法是这样

```java
private static void printTwo() {
    System.out.println("CALL printOne");
    printOne();
    System.out.println("RETURN printOne");
    System.out.println("CALL printOne");
    printOne();
    System.out.println("RETURN printOne");
}
```

让我们来看一下如何用ASM实现

```java
private static void weave(String inputPath, String outputPath) {
    try {
        FileInputStream is = new FileInputStream(inputPath);
        ClassReader cr = new ClassReader(is);
        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
        CallClassAdapter adapter = new CallClassAdapter(cw);
        cr.accept(adapter, 0);
        FileOutputStream fos = new FileOutputStream(outputPath);
        fos.write(cw.toByteArray());
        fos.close();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

这段代码和上面的实现复制class的代码唯一区别就是，使用了CallClassAdapter，它是一个自定义的ClassVisitor，我们将ClassWriter传递给CallClassAdapter的构造函数。来看看它的实现

```java
//CallClassAdapter.java
public class CallClassAdapter extends ClassVisitor implements Opcodes {
    public CallClassAdapter(final ClassVisitor cv) {
        super(ASM5, cv);
    }


    @Override
    public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
        super.visit(version, access, name, signature, superName, interfaces);
    }

    @Override
    public MethodVisitor visitMethod(final int access, final String name,
                                     final String desc, final String signature, final String[] exceptions) {

        MethodVisitor mv = cv.visitMethod(access, name, desc, signature, exceptions);
        return mv == null ? null : new CallMethodAdapter(name, mv);
    }
}



//CallMethodAdapter.java
class CallMethodAdapter extends MethodVisitor implements Opcodes {
    public CallMethodAdapter(final MethodVisitor mv) {
        super(ASM5, mv);
    }

    @Override
    public void visitMethodInsn(int opcode, String owner, String name, String desc, boolean itf) {
        mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
        mv.visitLdcInsn("CALL " + name);
        mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
        mv.visitMethodInsn(opcode, owner, name, desc, itf);
        mv.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
        mv.visitLdcInsn("RETURN " + name);
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);

    }

}
```

CallClassAdapter中的visitMethod使用了一个自定义的MethodVisitor—–CallMethodAdapter，它也是代理了原来的MethodVisitor，原理和ClassVisitor的代理一样。

看到这里，貌似使用ASM修改字节码的大概套路都走完了，那么如何写出上面`visitMethodInsn`方法中插入打印方法名的逻辑，这就需要一些字节码的基础知识了，我们说过这里不会展开介绍字节码，但是我们可以介绍一些快速学习字节码的方式，同时也是开发字节码相关工程一些实用的工具。