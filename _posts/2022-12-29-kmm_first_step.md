---
layout: post
title:  "KMM 初探"
date:   2022-12-29 19:56:42 +0800
categories: Android iOs KMM
tags: Android iOS KMM
---


## 环境搭建

按照官方文档[Get started with Kotlin Multiplatform Mobile \| Kotlin Documentation](https://kotlinlang.org/docs/multiplatform-mobile-getting-started.html)一步一步来即可：

安装检查工具**kdoctor**并运行之：

> brew install kdoctor
> 
> kdoctor


工具检查下面几个环境：

1. mac os
2. java 环境
3. Adnroid studio
4. Xcode
5. Cocoapods

需要注意的有:

#### a. Xcode requires to perform the First Launch tasks
要求**Launch Xcode or execute 'xcodebuild -runFirstLaunch' in terminal**

在命令行运行**xcodebuild -runFirstLaunch**可能会报错：

```
xcode-select: error: tool 'xcodebuild' requires Xcode, but active developer directory '/Library/Developer/CommandLineTools' is a command line tools instance
```

这个时候需要：

> sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
> 
> xcodebuild -runFirstLaunch

即可

#### b. cocoapods not found

需要安装**cocoapods**：

> sudo gem install cocoapods


### Kotlin Multiplatform Mobile plugin
需要在**Android Studio**中安装**KMM**插件：
**Settings/Preferences | Plugins | Marketplace**中搜索**Kotlin Multiplatform Mobile**即可

另外**kdoctor**会提示：
```
Android Studio 2021.3 has the issue with running shared unit tests via run gutters
```
忽略即可，这个时候：

> ✓ Your system is ready for Kotlin Multiplatform Mobile Development!
 
 
## 创建KMM App
 
 在**Android Studio**中：
 **File | New | New Project.**选择**Kotlin Multiplatform App**
 
* 第一步: 跟以前一样，起app名，包名，选择最小sdk等等：
![457f992e.png](./../assets/img/2022-12-29-kmm_first_step/457f992e.png)

* 第二步: 多了个KMM的选项：
![04be95c9.png](./../assets/img/2022-12-29-kmm_first_step/04be95c9.png)
前面是指定android app、ios app、共享模块的名称，默认即可

主要是区别是**iOS framework distribution**：

* Regular framework: 整合了KMM框架，配置简单
* CocoaPods Dependency Manager: 需要自己配置KMM的CocoaPods依赖

一般成熟的项目都用到了CocoaPods，这里就一步到位吧

可以看下创建好的工程目录，有三个gradle module，名字都是上一步指定的：

### shared

从**build.gradle**文件看：

```
plugins {
    kotlin("multiplatform")
    kotlin("native.cocoapods")
    id("com.android.library")
}
```
首先它是一个**android library**模块，同时应用了**kmm**插件和**cocoapods**插件

然后是配置**cocoapods**：

```kotlin
kotlin {
    cocoapods {
        summary = "Some description for the Shared Module"
        homepage = "Link to the Shared Module homepage"
        version = "1.0"
        ios.deploymentTarget = "14.1"
        podfile = project.file("../iosApp/Podfile")
        framework {
            baseName = "shared"
        }
    }
}
```

最后再设置各个依赖
```kotlin
kotlin {
    sourceSets {
        val commonMain by getting
        val commonTest by getting {
            dependencies {
                implementation(kotlin("test"))
            }
        }
        val androidMain by getting
        val androidTest by getting
        val iosX64Main by getting
        val iosArm64Main by getting
        val iosSimulatorArm64Main by getting
        val iosMain by creating {
            dependsOn(commonMain)
            iosX64Main.dependsOn(this)
            iosArm64Main.dependsOn(this)
            iosSimulatorArm64Main.dependsOn(this)
        }
        val iosX64Test by getting
        val iosArm64Test by getting
        val iosSimulatorArm64Test by getting
        val iosTest by creating {
            dependsOn(commonTest)
            iosX64Test.dependsOn(this)
            iosArm64Test.dependsOn(this)
            iosSimulatorArm64Test.dependsOn(this)
        }
    }
}
```

其实上面大多数**sourceSet**都不需要，只需要保留三个即可：

  * commonMain，跨平台的共享代码，包括公共的接口、数据类、通用实现和公共的依赖
  
  * androidMain，需要由android实现的扩展
  
  * iosMain，需要由ios实现的扩展

通过这种方式可以给不同的**sourceSet**提供不同的依赖，例如有些库只提供了**android**版本，而有些kt库可以运行在**KMM**下

通过上述配置，也可以在老工程中直接加入**KMM**模块

### androidApp

即**android**端的工程文件，从**build.gradle**看跟普通的**android**工程模块没有区别，依赖了**shared**模块

### iosApp

即**ios**端的工程文件，标准的**ios**工程，通过**Pods**依赖了**shared**模块

```
target 'iosApp' do
  use_frameworks!
  platform :ios, '14.1'
  pod 'shared', :path => '../shared'
end
```

## 编译和运行

androidApp 和 iosApp都可以在**android studio**里面运行，结果如下：

![e5718082.png](./../assets/img/2022-12-29-kmm_first_step/e5718082.png)

![e3317a98.png](./../assets/img/2022-12-29-kmm_first_step/e3317a98.png)

对于iosApp，也可以在**xcode**单独打开，如果是通过**android studio**创建的工程，则都配置好了，直接运行即可

如果是从xcode创建的工程，则需要根据这个链接进行配置：
[Make your Android application work on iOS – tutorial \| Kotlin Documentation](https://kotlinlang.org/docs/multiplatform-mobile-integrate-in-existing-app.html#make-your-cross-platform-application-work-on-ios)