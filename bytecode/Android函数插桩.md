# 【Android】函数插桩（Gradle + ASM）



## 前言

第一次看到插桩，是在[Android开发高手课](https://time.geekbang.org/column/142)中。看完去查了一下：“咦！**还有这东西，有点意思**”。

本着不断学习和探索的精神，便走上学习函数插桩的“**不归路**”。



![img](https://upload-images.jianshu.io/upload_images/1638147-7f0daaea81f0b27c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/176)





## 函数插桩

### 是什么函数插桩

**插桩**：目标程序代码中某些位置**插入或修改**成一些代码，从而在目标程序运行过程中获取某些程序状态并加以分析。简单来说就是**在代码中插入代码**。
那么**函数插桩**，便是在函数中插入或修改代码。

本文将介绍**在Android编译过程中，往字节码里插入自定义的字节码**，所以也可以称为**字节码插桩**。

### 作用

函数插桩可以帮助我们实现很多手术刀式的代码设计，如**无埋点统计上报、轻量级AOP**等。
应用到在**Android**中，可以用来做用行为统计、方法耗时统计等功能。

## 技术点

在动手之前，需要掌握以下相关知识：

- **Android打包流程**
  相关资料：[Apk 打包流程梳理](https://juejin.im/entry/58b78d1b61ff4b006cd47e5b)、[Android APK打包流程](http://shinelw.com/2016/04/27/android-make-apk/)
- **Java字节码**
  相关资料：[一文让你明白Java字节码](https://www.jianshu.com/p/252f381a6bc4)、[Java字节码(维基百科)](https://zh.wikipedia.org/zh-hans/Java字节码)、[如何阅读JAVA 字节码（一）](https://juejin.im/post/58ac086d2f301e006c3d6600)、《深入理解Java虚拟机》第6章（有条件的话，推荐看书）
- **自定义Gradle插件、Transform API**
  相关资料：[在AndroidStudio中自定义Gradle插件](https://blog.csdn.net/huachao1001/article/details/51810328)、[深入理解Android之Gradle](https://blog.csdn.net/innost/article/details/48228651)、[打包Apk过程中的Transform API](https://www.jianshu.com/p/811b0d0975ef)、[Transform官方文档](https://link.juejin.im/?target=http%3A%2F%2Ftools.android.com%2Ftech-docs%2Fnew-build-system%2Ftransform-api)
- **ASM**
  相关资料：[AOP 的利器：ASM 3.0 介绍](https://www.ibm.com/developerworks/cn/java/j-lo-asm30/)、[Class文件格式实战：使用ASM动态生成class文件](https://blog.csdn.net/zhangjg_blog/article/details/22976929)

**一定要先熟悉上面的知识**
**一定要先熟悉上面的知识**
**一定要先熟悉上面的知识**

以下内容**涉及知识过多**，需熟练掌握以上知识。否则，可能会引起**头大、目眩、烦躁**等一系列不良反应。**请在大人的陪同下阅读**

## 实战

### 需求

你可能会遇到一个这样需求：**在Android应用中，记录每个页面的打开\关闭**。

### 开工前的思考

记录页面被打开\关闭，一般来说就是记录**Activity的创建和销毁**（这里以`Activity`区分页面）。所以，我们只要在`Activity`的`onCreate()`和`onDestroy()`中插入对应的代码即可。

这时候就会遇到一个问题：**如何为Activity插入代码？**
**一个个写？**不可能！毕竟我们是**高（懒）效（惰）**的程序员；
**写在BaseActivity中？**好像可以，不过项目中如果有第三方的页面就显得有些无力了，而且不通用；

我们希望实现一个可以自动在`Activity`的`onCreate()`和`onDestroy()`中插入代码的工具，可以在**任意工程中使用**。

于是，**自定义Gradle插件 + ASM**便成了一个不错的选择



![img](https://upload-images.jianshu.io/upload_images/1638147-c9b7865197e1fc2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/176)





### 实现思路

对**Android打包过程**和**自定义Gradle插件**了解后发现，**java**文件会先转化为`class`文件，然后在转化为`dex`文件。而通过`Gradle`插件提供的`Transform API`，可以在编译成`dex`文件之前得到`class`文件。
得到`class`文件之后，便可以通过**ASM**对字节码进行修改，即可完成**字节码插桩**。

**步骤如下：**

- 了解**Android打包过程**，在过程中找**插入点**（`class`转换成 `.dex`过程）；

  

  ![img](https://upload-images.jianshu.io/upload_images/1638147-2181231c2143222a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/496)

  插入点（部分打包过程）

  

- 了解**自定义Gradle插件、Transform API**，在`Transform#transform()`中得到`class`文件;

- 找到`FragmentActivity`的`class`文件，通过**ASM**库，在`onCreate()`中**插入代码**;（为什么是`FragmentActivity`而不是`Activity`后面会说到）

- 将原文件**替换为**修改后的`class`文件。

如下图：





![img](https://upload-images.jianshu.io/upload_images/1638147-4f4b6463869dac87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

实现思路

> **class文件**：java源文件经过`javac`后生成一种紧凑的8位字节的二进制流文件。
> **插入点**：“dex”节点，表示将`class`文件打包到`dex`文件的过程，其输入包括`class`文件以及第三方依赖的`class`文件。

> 关于Transform API：从`1.5.0-beta1`开始，Gradle插件包含一个Transform API，允许第三方插件在将编译后的类文件转换为`dex`文件之前对其进行操作。

> 关于**混淆**：关于混淆可以不用当心。混淆其实是个`ProguardTransform`，在自定义的**Transform**之后执行。

### 动手实现

主要实现以下功能：

- **自定义Gradle插件**
- **处理class文件**
- **替换**

（以下为部分关键代码，完整源码点击[这里](https://github.com/Gavin-ZYX/asmDemo)）

#### 自定义Gradle插件

如何自定义插件这里就不详细介绍了，具体参考[在AndroidStudio中自定义Gradle插件](https://blog.csdn.net/huachao1001/article/details/51810328)、[打包Apk过程中的Transform API](https://www.jianshu.com/p/811b0d0975ef)。

##### 目录结构

目录结构分为两部分：**插件部分**（`src/main/groovy`中）、**ASM部分**（`src/main/java`中）



![img](https://upload-images.jianshu.io/upload_images/1638147-71993bf1679b6e7b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/410)

目录结构



##### LifecyclePlugin.groovy

继承`Transform`，实现`Plugin`接口，通过`Transform#transform()`得到`Collection<TransformInput> inputs`，里面有我们想要的`class`文件。

```groovy
class LifecyclePlugin extends Transform implements Plugin<Project> {

    @Override
    void apply(Project project) {
        //registerTransform
        def android = project.extensions.getByType(AppExtension)
        android.registerTransform(this)
    }

    @Override
    String getName() {
        return "LifecyclePlugin"
    }

    @Override
    Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS
    }

    @Override
    Set<? super QualifiedContent.Scope> getScopes() {
        return TransformManager.SCOPE_FULL_PROJECT
    }

    @Override
    boolean isIncremental() {
        return false
    }

    @Override
    void transform(@NonNull TransformInvocation transformInvocation) {
        ...
        ...
        ...
    }
}
```

主要看方法`transform()`

```groovy
@Override
void transform(@NonNull TransformInvocation transformInvocation) {
    println '--------------- LifecyclePlugin visit start --------------- '
    def startTime = System.currentTimeMillis()
    Collection<TransformInput> inputs = transformInvocation.inputs
    TransformOutputProvider outputProvider = transformInvocation.outputProvider
    //删除之前的输出
    if (outputProvider != null)
        outputProvider.deleteAll()
    //遍历inputs
    inputs.each { TransformInput input ->
        //遍历directoryInputs
        input.directoryInputs.each { DirectoryInput directoryInput ->
            //处理directoryInputs
            handleDirectoryInput(directoryInput, outputProvider)
        }

        //遍历jarInputs
        input.jarInputs.each { JarInput jarInput ->
            //处理jarInputs
            handleJarInputs(jarInput, outputProvider)
        }
    }
    def cost = (System.currentTimeMillis() - startTime) / 1000
    println '--------------- LifecyclePlugin visit end --------------- '
    println "LifecyclePlugin cost ： $cost s"
}
```

通过参数`inputs`可以拿到所有的`class`文件。`inputs`中包括`directoryInputs`和`jarInputs`，`directoryInputs`为文件夹中的`class`文件，而`jarInputs`为jar包中的`class`文件。

对应两个处理方法`handleDirectoryInput`、`handleJarInputs`

**LifecyclePlugin#handleDirectoryInput()**

```groovy
/**
 * 处理文件目录下的class文件
 */
static void handleDirectoryInput(DirectoryInput directoryInput, TransformOutputProvider outputProvider) {
    //是否是目录
    if (directoryInput.file.isDirectory()) {
        //列出目录所有文件（包含子文件夹，子文件夹内文件）
        directoryInput.file.eachFileRecurse { File file ->
            def name = file.name
            if (name.endsWith(".class") && !name.startsWith("R\$")
                    && !"R.class".equals(name) && !"BuildConfig.class".equals(name)
                    && "android/support/v4/app/FragmentActivity.class".equals(name)) {
                println '----------- deal with "class" file <' + name + '> -----------'
                ClassReader classReader = new ClassReader(file.bytes)
                ClassWriter classWriter = new ClassWriter(classReader, ClassWriter.COMPUTE_MAXS)
                ClassVisitor cv = new LifecycleClassVisitor(classWriter)
                classReader.accept(cv, EXPAND_FRAMES)
                byte[] code = classWriter.toByteArray()
                FileOutputStream fos = new FileOutputStream(
                        file.parentFile.absolutePath + File.separator + name)
                fos.write(code)
                fos.close()
            }
        }
    }
    //处理完输入文件之后，要把输出给下一个任务
    def dest = outputProvider.getContentLocation(directoryInput.name,
            directoryInput.contentTypes, directoryInput.scopes,
            Format.DIRECTORY)
    FileUtils.copyDirectory(directoryInput.file, dest)
}
```

**LifecyclePlugin#handleJarInputs()**

```groovy
/**
 * 处理Jar中的class文件
 */
static void handleJarInputs(JarInput jarInput, TransformOutputProvider outputProvider) {
    if (jarInput.file.getAbsolutePath().endsWith(".jar")) {
        //重名名输出文件,因为可能同名,会覆盖
        def jarName = jarInput.name
        def md5Name = DigestUtils.md5Hex(jarInput.file.getAbsolutePath())
        if (jarName.endsWith(".jar")) {
            jarName = jarName.substring(0, jarName.length() - 4)
        }
        JarFile jarFile = new JarFile(jarInput.file)
        Enumeration enumeration = jarFile.entries()
        File tmpFile = new File(jarInput.file.getParent() + File.separator + "classes_temp.jar")
        //避免上次的缓存被重复插入
        if (tmpFile.exists()) {
            tmpFile.delete()
        }
        JarOutputStream jarOutputStream = new JarOutputStream(new FileOutputStream(tmpFile))
        //用于保存
        while (enumeration.hasMoreElements()) {
            JarEntry jarEntry = (JarEntry) enumeration.nextElement()
            String entryName = jarEntry.getName()
            ZipEntry zipEntry = new ZipEntry(entryName)
            InputStream inputStream = jarFile.getInputStream(jarEntry)
            //插桩class
            if (entryName.endsWith(".class") && !entryName.startsWith("R\$")
                    && !"R.class".equals(entryName) && !"BuildConfig.class".equals(entryName)
                    && "android/support/v4/app/FragmentActivity.class".equals(entryName)) {
                //class文件处理
                println '----------- deal with "jar" class file <' + entryName + '> -----------'
                jarOutputStream.putNextEntry(zipEntry)
                ClassReader classReader = new ClassReader(IOUtils.toByteArray(inputStream))
                ClassWriter classWriter = new ClassWriter(classReader, ClassWriter.COMPUTE_MAXS)
                ClassVisitor cv = new LifecycleClassVisitor(classWriter)
                classReader.accept(cv, EXPAND_FRAMES)
                byte[] code = classWriter.toByteArray()
                jarOutputStream.write(code)
            } else {
                jarOutputStream.putNextEntry(zipEntry)
                jarOutputStream.write(IOUtils.toByteArray(inputStream))
            }
            jarOutputStream.closeEntry()
        }
        //结束
        jarOutputStream.close()
        jarFile.close()
        def dest = outputProvider.getContentLocation(jarName + md5Name,
                jarInput.contentTypes, jarInput.scopes, Format.JAR)
        FileUtils.copyFile(tmpFile, dest)
        tmpFile.delete()
    }
}
```

这两个方法都在做同一件事，就是遍历`directoryInputs`、`jarInputs`，得到对应的`class`文件，然后交给**ASM**处理，最后覆盖原文件。

> 发现：在`input.jarInputs`中并没有`android.jar`。本想在`Activity`中做处理，因为找不到`android.jar`，只好退而求其次选择`android.support.v4.app`中的`FragmentActivity`。
> **那么，所以如何的到android.jar ？请指教**

#### 处理class文件

在`handleDirectoryInput`和`handleJarInputs`中，可以看到**ASM**的部分代码了。这里以`handleDirectoryInput`为例。

`handleDirectoryInput`中**ASM**代码：

```
ClassReader classReader = new ClassReader(file.bytes)
ClassWriter classWriter = new ClassWriter(classReader, ClassWriter.COMPUTE_MAXS)
ClassVisitor cv = new LifecycleClassVisitor(classWriter)
classReader.accept(cv, EXPAND_FRAMES)
```

其中，关键处理类`LifecycleClassVisitor`

##### `LifecycleClassVisitor`

用于访问`class`的工具，在`visitMethod()`里对**类名**和**方法名**进行判断是否需要处理。若需要，则交给`MethodVisitor`。

```java
public class LifecycleClassVisitor extends ClassVisitor implements Opcodes {

    private String mClassName;

    public LifecycleClassVisitor(ClassVisitor cv) {
        super(Opcodes.ASM5, cv);
    }

    @Override
    public void visit(int version, int access, String name, String signature, 
        String superName, String[] interfaces) {
        System.out.println("LifecycleClassVisitor : visit -----> started ：" + name);
        this.mClassName = name;
        super.visit(version, access, name, signature, superName, interfaces);
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String desc, 
        String signature, String[] exceptions) {
        System.out.println("LifecycleClassVisitor : visitMethod : " + name);
        MethodVisitor mv = cv.visitMethod(access, name, desc, signature, exceptions);
        //匹配FragmentActivity
        if ("android/support/v4/app/FragmentActivity".equals(this.mClassName)) {
            if ("onCreate".equals(name) ) {
                //处理onCreate
                return new LifecycleOnCreateMethodVisitor(mv);
            } else if ("onDestroy".equals(name)) {
                //处理onDestroy
                return new LifecycleOnDestroyMethodVisitor(mv);
            }
        }
        return mv;
    }

    @Override
    public void visitEnd() {
        System.out.println("LifecycleClassVisitor : visit -----> end");
        super.visitEnd();
    }
}
```

在`visitMethod()`中判断是否为`FragmentActivity`，且为方法`onCreate`或`onDestroy`，然后交给`LifecycleOnDestroyMethodVisitor`或`LifecycleOnCreateMethodVisitor`处理。

**回到需求**，我们希望在`onCreate()`中插入对应的代码，来记录页面被打开。（这里通过Log代替）

```java
Log.i("TAG", "-------> onCreate : " + this.getClass().getSimpleName());
```

于是，在`LifecycleOnCreateMethodVisitor`中如下处理
（*LifecycleOnDestroyMethodVisitor与LifecycleOnCreateMethodVisitor相似*，完整代码点击[这里](https://github.com/Gavin-ZYX/asmDemo)
）

##### `LifecycleOnCreateMethodVisitor`

```java
public class LifecycleOnCreateMethodVisitor extends MethodVisitor {

    public LifecycleOnCreateMethodVisitor(MethodVisitor mv) {
        super(Opcodes.ASM4, mv);
    }

    @Override
    public void visitCode() {
        //方法执行前插入
        mv.visitLdcInsn("TAG");
        mv.visitTypeInsn(Opcodes.NEW, "java/lang/StringBuilder");
        mv.visitInsn(Opcodes.DUP);
        mv.visitMethodInsn(Opcodes.INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false);
        mv.visitLdcInsn("-------> onCreate : ");
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
        mv.visitVarInsn(Opcodes.ALOAD, 0);
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/Object", "getClass", "()Ljava/lang/Class;", false);
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/Class", "getSimpleName", "()Ljava/lang/String;", false);
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "append", "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
        mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/lang/StringBuilder", "toString", "()Ljava/lang/String;", false);
        mv.visitMethodInsn(Opcodes.INVOKESTATIC, "android/util/Log", "i", "(Ljava/lang/String;Ljava/lang/String;)I", false);
        mv.visitInsn(Opcodes.POP);

        super.visitCode();
        //方法执行后插入
    }
    
    @Override
    public void visitInsn(int opcode) {
        super.visitInsn(opcode);
    }
}
```

只需要在`visitCode()`中插入上面的代码，即可实现`onCreate()`内容执行之前，先执行我们插入的代码。

**如果想在onCreate()内容执行之后插入代码，该怎么做？**
和上面相似，只要在`visitInsn()`方法中插入对应的代码即可。代码如下：

```java
@Override
public void visitInsn(int opcode) {
    //判断RETURN
    if (opcode == Opcodes.RETURN) {
        //在这里插入代码
        ...
    }
    super.visitInsn(opcode);
}
```

如果对字节码不是很了解，看到上面`visitCode()`中的代码可能会觉得既熟悉又陌生，那是**ASM插入字节码**的用法。
**如果你写不来**，没关系，这里介绍一个插件——**ASM Bytecode Outline**，包教包会。

> **通过ASM Bytecode Outline插件生成代码**
> 1、在Android Studio中安装**ASM Bytecode Outline**插件；
> 2、安装后，在编译器中，点击右键，选择**Show Bytecode outLine**；
>
> 
>
> ![img](https://upload-images.jianshu.io/upload_images/1638147-9a55eb3269c1530f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/318)
>
> 
>
> 3、在
>
> ASM标签
>
> 中选择
>
> ASMified
>
> ，即可在右侧看到当前类对应的
>
> ASM
>
> 代码。（可以忽略
>
> Label
>
> 相关的代码，以下选框的内容为对应的代码）
>
> 
>
> ![img](https://upload-images.jianshu.io/upload_images/1638147-03d4fee1f974866d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

> **提示**：`ClassVisitor#visitMethod()`只能访问当前类定义的`method`（一开始想访问父类的方法，陷入误区）。
> 如，在`MainActivity`中只重写了`onCreate()`，没有重写`onDestroy()`。那么在`visitMethod()`中只会出现`onCreate()`，不会有`onDestroy()`。

#### 替换

`class`文件的插桩已经说完，剩下最后一步——**替换**。眼尖的同学应该发现，代码上面已经出现过了。还是以`LifecyclePlugin#handleDirectoryInput()`中的代码为例：

```groovy
byte[] code = classWriter.toByteArray()
FileOutputStream fos = new FileOutputStream(
      file.parentFile.absolutePath + File.separator + name)
fos.write(code)
fos.close()
```

从`classWriter`得到`class`修改后的`byte`流，然后通过流的写入覆盖原来的`class`文件。
（Jar包的覆盖会稍微复杂一点，这里就不细说了）

> ```
> File.separator`：文件的分隔符。不同系统分隔符可能不一样。
> 如：同样一个文件，**Windows**下是`C:\tmp\test.txt`；**Linux**下却是`/tmp/test.txt
> ```

#### 使用

插件写完，便可以投入使用了。

创建一个**Android**项目`app`，在`app.gradle`中引用插件。（完整代码点击[这里](https://github.com/Gavin-ZYX/asmDemo)）

```shell
apply plugin: 'com.gavin.gradle'
```

运行后，按步骤操作：
打开`MainActivity`——>打开`SecondActivity`——>返回`MainActivity`。

查看效果：

```shell
com.gavin.asmdemo I/TAG: -------> onCreate : MainActivity
com.gavin.asmdemo I/TAG: -------> onCreate : SecondActivity
com.gavin.asmdemo I/TAG: -------> onDestroy : SecondActivity
```

可以发现，页面**打开\关闭**都会打印对应的log。说明我们插入的代码被执行了，**而且，使用时对项目没有任何“入侵”**。

## 结语

本文内容涉及知识较多，在熟悉**Android打包过程**、**字节码**、**Gradle Transform API**、**ASM**等之前，阅读起来会很困难。不过，在了解并学习这些知识的之后，相信你对**Android**会有新的认识。