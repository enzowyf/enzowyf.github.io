---
layout: post
title:  "AGP 7.0 + Gradle 7.0 升级记录"
date:   2022-3-1 17:49:42 +0800
categories: Android 
tags: agp
---

随着`Jetpack Compose`正式版的发布，打算尝鲜一下，`Jetpack Compose`强制绑定了`AGP 7.0` 和 `kotlin 1.5.31`，`kotlin 1.5.31`好办，但`AGP 7.0`又强制搭配`Gradle 7.0` 和 `JDK 11`，正好春节期间没什么大事，可以尝试一下升级。

当前项目的组合是`Gradle 6.9` + `AGP 4.2.2`，为了避免同时升级`AGP`和`Gradle`引入过多的变量，可以一步一步来，先升级`Gradle 7.0`，最后是升级`AGP 7.0` + `JDK 11`。

# 一、Gradle 6.9 -> 7.3.2

## 1. Cannot run Project.afterEvaluate(Closure) when the project is already evaluated


第一个报错：
> Cannot run Project.afterEvaluate(Closure) when the project is already evaluated.

定位到下面这一段：

```groovy
def installCustomLintTask = tasks.findByPath(":lint:install")
subprojects {
    afterEvaluate { Project project ->
    	...
    }
} 
```   

`stackoverflow`给到的答案是把`afterEvaluate`里面的内容移到`plugin`中去，尝试了一下确实可以，但疑惑的是，项目中也挺多`afterEvaluate`，怎么只有这一处报错，原来问题其实出在`def installCustomLintTask = tasks.findByPath(":lint:install")`，这一行出现了在`afterEvaluate`外面，而`findByPath`会引发evaluate，所以才会出现`is already evaluated`的情况
    
       
## 2. property 'xxx' is missing an input or output annotation

报错例子如下：

```
> In plugin 'com.google.protobuf' type 'com.google.protobuf.gradle.ProtobufExtract' property 'destDir' is missing an input or output annotation.
    
 Reason: A property without annotation isn't considered during up-to-date checking.
    
 Possible solutions:
      1. Add an input or output annotation.
      2. Mark it as @Internal.
    
 Please refer to https://docs.gradle.org/7.2/userguide/validation_problems.html#missing_annotation for more details about this problem.
```

参考报错中给的解法，第一加上`@input`或者`@output`的注解来提高构建缓存命中率；或者直接加上`@Internal`。两种方法都行，就算具体的属性是不是需要纳入缓存了。



# 一、AGP 4.2.2 -> 7.0.2

这一步比想象中要顺利，基本没有报错，只有一个`JDK`的要求：
```
* What went wrong:
A problem occurred evaluating script.
> Failed to apply plugin 'com.android.internal.application'.
   > Android Gradle plugin requires Java 11 to run. You are currently using Java 1.8.
     You can try some of the following options:
       - changing the IDE settings.
       - changing the JAVA_HOME environment variable.
       - changing `org.gradle.java.home` in `gradle.properties`.
```

解决方法就是设置`JAVA_HOME`到`JDK 11`即可。

问题少的原因主要是在升级`AGP 4.2.2`过程中把提示预计在`7.0`要废弃的借口都给提前修改了。但`AGP 7.0`改动其实还是比较大的，可以参考[https://developer.android.google.cn/studio/releases/gradle-plugin#7-0-0](https://developer.android.google.cn/studio/releases/gradle-plugin#7-0-0)，具体来说：

## 新的`variant api`成为了稳定版本

这个改动涉及老的写法被废除：

```groovy
android.applicationVariants.all {
}
```

改成：

```
androidComponents {
        onVariants(selector().withName("all"), { variant ->
        })
}
```
如果只针对`debug varaint`，就把上面的`all`改成`debug`即可。

其他的还有简化`task`之间关系的`Artifacts`，再也不用`dependsOn`、`finalizedBy`了，具体可以参考[官方demo](https://github.com/android/gradle-recipes)

由于老版本`variant api`相关接口要在`AGP 8`中移除，所以现在还不着急适配新`variant api`。（但其实也不远了，官方蓝图中`AGP 8`将会在2022年上半年到来）