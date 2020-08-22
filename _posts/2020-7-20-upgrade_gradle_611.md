---
layout: post
title:  "Gradle 6.1.1 升级踩坑记录"
date:   2020-07-20 17:49:42 +0800
categories: Android 
tags: gradle
---

最近Android Studio更新到了4.0，随之更新的有Gradle 6.1.1 和 Android Gradle Plugin 4.0，这里记录一下Gradle 从5.6.4升级到6.1.1的过程


### 1. 升级greendao至3.3.0

如果是老版本的greendao，会遇到类似这样的问题：

```
Caused by: java.lang.NoSuchMethodError: org.gradle.api.tasks.TaskInputs.property(Ljava/lang/String;Ljava/lang/Object;)Lorg/gradle/api/tasks/TaskInputs;
        at org.greenrobot.greendao.gradle.Greendao3GradlePlugin.createGreendaoTask(Greendao3GradlePlugin.kt:60)
        at org.greenrobot.greendao.gradle.Greendao3GradlePlugin.access$createGreendaoTask(Greendao3GradlePlugin.kt:14)
        at org.greenrobot.greendao.gradle.Greendao3GradlePlugin$apply$1.execute(Greendao3GradlePlugin.kt:47)
        at org.greenrobot.greendao.gradle.Greendao3GradlePlugin$apply$1.execute(Greendao3GradlePlugin.kt:14)
        
```
只要升级至3.3.0即可：[https://github.com/greenrobot/greenDAO/issues/1002](https://github.com/greenrobot/greenDAO/issues/1002)




### 2. 升级android gradle plugin 至3.4.0以上

如果是3.3.x以下版本的agp，会遇到类似这样的问题：


```
> Failed to notify project evaluation listener.
   > org.gradle.api.file.ProjectLayout.fileProperty(Lorg/gradle/api/provider/Provider;)Lorg/gradle/api/file/RegularFileProperty;
```
至少要升级到3.4.0：[https://discuss.pytorch.org/t/android-build-failed/60873
](https://discuss.pytorch.org/t/android-build-failed/60873
)


### 3. ndk相关task的变化
由于我的工程中有自定义task，且dependsOn `transformNativeLibsWithStripDebugSymbolForXXX`和`transformNativeLibsWithMergeJniLibsForXXX`，因此会报错：

```
* What went wrong:
A problem occurred configuring project ':app'.
> Cannot invoke method dependsOn() on null object

```

仔细查看新的task列表发现这两个task换了名字，只要改成新的task即可：

* transformNativeLibsWithStripDebugSymbolForXXX -> stripXXXDebugSymbols
* transformNativeLibsWithMergeJniLibsForXXX -> mergeXXXJniLibFolders

> XXX表示Debug或者Release

### 4. maven库必须提供pom
这个问题刚开始还有点疑惑，看报错信息是一个maven库下载失败，找不到这个库，但是回退gradle版本却是没问题的，翻了半天官方手册，发现这么一段：

[Maven or Ivy repositories are no longer queried for artifacts without metadata by default
If Gradle fails to locate the metadata file (.pom or ivy.xml) of a module in a repository defined in the repositories { } section, it now assumes that the module does not exist in that repository.](https://docs.gradle.org/current/userguide/upgrading_version_5.html#maven_or_ivy_repositories_are_no_longer_queried_for_artifacts_without_metadata_by_default)

maven库缺少了pom文件？比较好奇这个库是怎么上传上去的。。。
幸好这个库是自己的，重新上传一下就好，万一是第三方的，就蛋疼了。。。



### 5.Property 'xxx' is not annotated with an input or output annotation

在某个task中报了这样一个错误：

```
Property 'xxx' is not annotated with an input or output annotation
```

官方手册说，gradle 6 所有默认访问权限的属性、方法都要打上input/output注解，所以要么乖乖打上input/output注解，要么把不需要开放的改成public/private：

[Task dependencies are honored for task @Input properties whose value is a Property](https://docs.gradle.org/current/userguide/upgrading_version_5.html#task_dependencies_are_honored_for_task_input_properties_whose_value_is_a_property)

### 6.Warning: Type 'xxx': field 'xxx' without corresponding getter has been annotated with @Input.

跟前面第5点相反，gradle 6 所有input/output注解都不能打在public/private上，所以只需要去掉public/private即可😓

### 7.D8: Unknown option "-ignorewarning"
这个问题不是gradle 6带来的，而是agp 3.3.x之后默认开启了新的R8混淆工具，根据提示，把`-ignorewarning`替换成`-ignorewarnings`就可以了。

### 8. R8 混淆的问题
前面都解决了之后，总算可以正常编译了，但发现编译的debug包一切正常，release包运行crash，由于是release包，基本可以确定跟混淆有关。

第一个办法，试试关闭R8，`android.enableR8 = false`，

ok, release运行正常，但同时编译速度降低了一些，安装包也有所增大，大致跟android gradle plugin 3.2.x一个水平

其实crash原因是反射调用了一个被混淆的方法，keep一下就好了，但比较坑的是，这个方法一直都没有keep，但一直正常，解开老的包发现也确实没有混淆，有点违反常识。。。
