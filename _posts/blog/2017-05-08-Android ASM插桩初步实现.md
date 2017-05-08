---
layout: post
title: Android ASM插桩初步实现
description: what's your main focus today?
category: blog
---



## 背景：
最近在对几个项目做一个启动速度的优化，发现使用Traceview的获取的时间和Runtime运行时的时间值其实是有比较大的差异，于是刚开始打算用手打的方式对每个方法进行时间戳的打印，后来发现，工作量巨大且重复，不适用这样的场景；于是想通过一些hook的方法来实现这样的功能； 


## ASM

什么ASM，他是一款可以操作class文件字节码的工具库，和javassist一样，可以对class文件的字节码进行操作；

一般来说，对一个普通的java文件字节码操作流程是这样的：

```java
	1、javac Test.java 生成Test.class文件
	
	2、使用ClassWriter和ClassReader修改Test.class文件
	
	3、将修改后的文件保存到新的目录即可
```

那么对于我们要对安卓代码进行插桩，就只能在编译时插入代码，然后在运行时判断条件进行是否执行我们想要的逻辑。
那么接下来有几个问题：

```java
	1、我们从哪里获取所有编译后的class文件，如何拦截，并替换他们。
	2、如何修改某个class文件
	3、修改完成之后，如何替换到原有的class,达到hook的目的
```

## android gralde Transform 拦截class文件

对groovy不熟悉的同学可以先了解一下[groovy语法](http://blog.csdn.net/kmyhy/article/details/4200563)

对gradle不熟悉的同学可以先了解一下[gradle](http://wiki.jikexueyuan.com/project/deep-android-gradle/four-four.html)

我们知道，在Android apk编译流程里，有一步是将编译后的class文件转换为dex，android plugin给我们提供了一个接口，让我们可以在生成dex之前做
一些我们自己的工作；它就是Transfrom;
我们来看看我们是怎么获得所有编译后的class文件的；

```java

package com.meiyou.meetyoucostplugin

import com.android.build.api.transform.*
import com.android.build.gradle.AppExtension
import com.android.build.gradle.internal.pipeline.TransformManager
import org.apache.commons.codec.digest.DigestUtils
import org.apache.commons.io.FileUtils
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.objectweb.asm.ClassReader
import org.objectweb.asm.ClassVisitor
import org.objectweb.asm.ClassWriter

import static org.objectweb.asm.ClassReader.EXPAND_FRAMES

public class PluginImpl extends Transform implements Plugin<Project> {
    void apply(Project project) {
        //遍历class文件和jar文件，在这里可以进行class文件asm文件替换
        def android = project.extensions.getByType(AppExtension);
        android.registerTransform(this)
    }


    @Override
    public String getName() {
        return "PluginImpl";
    }

    @Override
    public Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS;
    }

    @Override
    public Set<QualifiedContent.Scope> getScopes() {
        return TransformManager.SCOPE_FULL_PROJECT;
    }

    @Override
    public boolean isIncremental() {
        return false;
    }
    @Override
    void transform(Context context, Collection<TransformInput> inputs, Collection<TransformInput> referencedInputs,
                   TransformOutputProvider outputProvider, boolean isIncremental) throws IOException, TransformException, InterruptedException {
        println '//===============asm visit start===============//'
        //遍历inputs里的TransformInput
        inputs.each { TransformInput input ->
            //遍历input里边的DirectoryInput
            input.directoryInputs.each {
                DirectoryInput directoryInput ->
                    //是否是目录
                    if (directoryInput.file.isDirectory()) {
                        //遍历目录
                        directoryInput.file.eachFileRecurse {
                            File file ->
                                def filename = file.name;
                                def name = file.name
                                //这里进行我们的处理 TODO
                                if (name.endsWith(".class") && !name.startsWith("R\$") &&
                                        !"R.class".equals(name) && !"BuildConfig.class".equals(name)) {
                                    ClassReader classReader = new ClassReader(file.bytes)
                                    ClassWriter classWriter = new ClassWriter(classReader,ClassWriter.COMPUTE_MAXS)
                                    ClassVisitor cv = new CostMethodClassVisitor(classWriter)
                                    classReader.accept(cv, EXPAND_FRAMES)
                                    byte[] code = classWriter.toByteArray()
                                    FileOutputStream fos = new FileOutputStream(
                                            file.parentFile.absolutePath + File.separator + name)
                                    fos.write(code)
                                    fos.close()
                                    CostMethodClassVisitor
                                }
                                println '//PluginImpl find file:' + file.getAbsolutePath()
                                //project.logger.
                        }
                    }
                    //处理完输入文件之后，要把输出给下一个任务
                    def dest = outputProvider.getContentLocation(directoryInput.name,
                            directoryInput.contentTypes, directoryInput.scopes,
                            Format.DIRECTORY)
                    FileUtils.copyDirectory(directoryInput.file, dest)
            }


            input.jarInputs.each { JarInput jarInput ->
                /**
                 * 重名名输出文件,因为可能同名,会覆盖
                 */
                def jarName = jarInput.name
                def md5Name = DigestUtils.md5Hex(jarInput.file.getAbsolutePath())
                if (jarName.endsWith(".jar")) {
                    jarName = jarName.substring(0, jarName.length() - 4)
                }
                println '//PluginImpl find Jar:' + jarInput.getFile().getAbsolutePath()

                //处理jar进行字节码注入处理 TODO

                def dest = outputProvider.getContentLocation(jarName + md5Name,
                        jarInput.contentTypes, jarInput.scopes, Format.JAR)

                FileUtils.copyFile(jarInput.file, dest)
            }
        }
        println '//===============asm visit end===============//'

    }
}

```

以上注释写的很清楚了，我们定义了个PluginImpl的插件，然后进行了android.registerTransform；
这样之后，当插件编译的时候就会执行transform方法，这里有所有的输入文件，我们就对其进行遍历，并使用ClassReader和ClassReader进行处理之后，
将处理后的文件作为下个task的输出；


## 使用ASM操作class文件

我们看到在transform里步骤里有一处处理class文件的地方

```java
ClassReader classReader = new ClassReader(file.bytes)
                                    ClassWriter classWriter = new ClassWriter(classReader,ClassWriter.COMPUTE_MAXS)
                                    ClassVisitor cv = new CostMethodClassVisitor(classWriter)
                                    classReader.accept(cv, EXPAND_FRAMES)
                                    byte[] code = classWriter.toByteArray()
                                    FileOutputStream fos = new FileOutputStream(
                                            file.parentFile.absolutePath + File.separator + name)
                                    fos.write(code)
                                    fos.close()
```

使用ClassReader和ClassWriter需要引入库

```java
    compile 'org.ow2.asm:asm:5.0.3'
    compile 'org.ow2.asm:asm-commons:5.0.3'
```

首先我们使用ClassReader读取class文件为asm code，然后定义一个Visitor用来处理字节码；当classReader调用accept的时候,Visitor就会按一定的顺序调用
里边重载的方法

```java
package com.meiyou.meetyoucostplugin;

import org.objectweb.asm.Opcodes;
import org.objectweb.asm.Type;

import org.objectweb.asm.AnnotationVisitor;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.Type;
import org.objectweb.asm.commons.AdviceAdapter;
import org.objectweb.asm.signature.SignatureVisitor;

/**
 * 方法耗时 visitor
 * Author: lwh
 * Date: 4/28/17 15:37.
 */

public class CostMethodClassVisitor extends ClassVisitor {

    public CostMethodClassVisitor(ClassVisitor classVisitor) {
        super(Opcodes.ASM5,classVisitor);
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String desc, String signature,
                                     String[] exceptions) {

        MethodVisitor methodVisitor = cv.visitMethod(access, name, desc, signature, exceptions);
        methodVisitor = new AdviceAdapter(Opcodes.ASM5, methodVisitor, access, name, desc) {

            boolean inject = false;
            private boolean isInject(){
               /* if(name.equals("setStartTime") || name.equals("setEndTime") || name.equals("getCostTime")){
                   return false;
                }
                return true;*/
               return inject;
            }
            @Override
            public void visitCode() {
                super.visitCode();

            }

            @Override
            public org.objectweb.asm.AnnotationVisitor visitAnnotation(String desc, boolean visible) {
                if (Type.getDescriptor(Cost.class).equals(desc)) {
                    inject = true;
                }

                return super.visitAnnotation(desc, visible);
            }

            @Override
            public void visitFieldInsn(int opcode, String owner, String name, String desc) {
                super.visitFieldInsn(opcode, owner, name, desc);
            }



            @Override
            protected void onMethodEnter() {
                //super.onMethodEnter();
                if(isInject()){

                    mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
                    mv.visitLdcInsn("========start========="+name+"==>des:"+desc);
                    mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println",
                            "(Ljava/lang/String;)V", false);

                    mv.visitLdcInsn(name);
                    mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "nanoTime", "()J", false);
                    mv.visitMethodInsn(INVOKESTATIC, "com/meiyou/meetyoucost/TimeCache", "setStartTime",
                            "(Ljava/lang/String;J)V", false);
                }
            }

            @Override
            protected void onMethodExit(int i) {
                //super.onMethodExit(i);
                if(isInject()){
                    mv.visitLdcInsn(name);
                    mv.visitMethodInsn(INVOKESTATIC, "java/lang/System", "nanoTime", "()J", false);
                    mv.visitMethodInsn(INVOKESTATIC, "com/meiyou/meetyoucost/TimeCache", "setEndTime",
                            "(Ljava/lang/String;J)V", false);

                    mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
                    mv.visitLdcInsn(name);
                    mv.visitMethodInsn(INVOKESTATIC, "com/meiyou/meetyoucost/TimeCache", "getCostTime",
                            "(Ljava/lang/String;)Ljava/lang/String;", false);
                    mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println",
                            "(Ljava/lang/String;)V", false);

                    mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
                    mv.visitLdcInsn("========end=========");
                    mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println",
                            "(Ljava/lang/String;)V", false);
                }
            }
        };
        return methodVisitor;
        //return super.visitMethod(i, s, s1, s2, strings);

    }
}

```
我们重点分析三个方法

```java
	visitAnnotation
	onMethodEnter
	onMethodExit
```

visitAnnotation

当我们在某个方法写上注解的时候，它会执行这个方法，我们可以在这个方法判断是否是我们注解做一些事情，比如重置一个标志位；

onMethodEnter

方法进入调用

onMethodExit

方法结束时调用

就这样，我们完成了打点。那么我们在onMethodEnter，onMethodExit里的ASM代码是如何生成的呢。

一般情况下，如果对ASM很熟悉的话，可以手动编写，但是其实有很多其他的工具可以代替我们生成这样的代码；

首先我们先写好java文件，然后执行javac 生成.class文件，然后执行
java -classpath "/Users/mu/Downloads/asm-5.2/lib/all/asm-all-5.2.jar" org.objectweb.asm.util.ASMifier TestAsm.class
就可以生成asm code，然后把需要的代码拷贝过来即可。



## 插件完成后，如何使用呢

在主app最外层build.gradle里配置插件路径 classpath "com.meiyou:meetyoucostplugin:${DEPLOY_VERSION}"

然后在app的build.gradle里添加apply plugin: 'meetyoucost'以及 compile "com.meiyou:meetyoucost:${DEPLOY_VERSION}"

在需要统计的方法前面加上

```java
 @Cost
    public void show() {
        for (int i = 0; i < 100; i++) {

        }
    }
```
然后我们运行app，可以在logcat看到：

```java
05-08 11:10:25.620 32123-32123/? I/System.out: ========start=========show==>des:()V
05-08 11:10:25.621 32123-32123/? I/System.out: method: show cost 0  ms
05-08 11:10:25.621 32123-32123/? I/System.out: ========end========= }
```   

至此，埋点工程初步完成，还有其他很多细节可以完善的地方，这里只做演示使用。

[项目地址](https://github.com/iceAnson/MeetyouCost)点此获取

----------
QQ:452825089

mail:452825089@qq.com

wechat:ice3897315

blog:http://iceAnson.github.io




