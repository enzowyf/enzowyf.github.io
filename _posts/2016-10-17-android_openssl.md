---
layout: post
title:  "Android Openssl交叉编译"
date:   2016-10-17 10:43:42 +0800
categories: Android开发
tags: android openssl
---


官网下载`openssl-1.0.2j` 源码：[https://www.openssl.org/source/openssl-1.0.2j.tar.gz](https://www.openssl.org/source/openssl-1.0.2j.tar.gz)

首先要编译`tool chain`，这里用的是`android-ndk-r9`，命令行执行：

```bash
export DEV_PREFIX=$HOME
export ANDROID_NDK=/Users/enzo/AndroidDev/android-ndk/
export NDK=${ANDROID_NDK}
export TOOLCHAIN_INSTALL_DIR=/Users/enzo/AndroidDev/android-ndk/arm-linux-androideabi-4.8/arm-linux-androideabi

$NDK/build/tools/make-standalone-toolchain.sh --platform=android-9 --toolchain=arm-linux-androideabi-4.8 --install-dir=$TOOLCHAIN_INSTALL_DIR
```

编译好`toolchain`继续设置环境变量：

```bash
export TOOLCHAIN_PATH=$TOOLCHAIN_INSTALL_DIR/bin
export TOOL=arm-linux-androideabi
export NDK_TOOLCHAIN_BASENAME=${TOOLCHAIN_PATH}/${TOOL}
export CC=$NDK_TOOLCHAIN_BASENAME-gcc
export CXX=$NDK_TOOLCHAIN_BASENAME-g++
export CPP=${NDK_TOOLCHAIN_BASENAME}-cpp
export LINK=${CXX}
export LD=$NDK_TOOLCHAIN_BASENAME-ld
export AR=$NDK_TOOLCHAIN_BASENAME-ar
export AS=${NDK_TOOLCHAIN_BASENAME}-as
export RANLIB=$NDK_TOOLCHAIN_BASENAME-ranlib
export STRIP=$NDK_TOOLCHAIN_BASENAME-strip
export NM=${NDK_TOOLCHAIN_BASENAME}-nm
export OBJDUMP=${NDK_TOOLCHAIN_BASENAME}-objdump
export ARCH_FLAGS="-mthumb"
export ARCH_LINK=
export CPPFLAGS=" ${ARCH_FLAGS} -fpic -ffunction-sections -funwind-tables -fstack-protector -fno-strict-aliasing -finline-limit=64 "
export CXXFLAGS=" ${ARCH_FLAGS} -fpic -ffunction-sections -funwind-tables -fstack-protector -fno-strict-aliasing -finline-limit=64 -frtti -fexceptions "
export CFLAGS=" ${ARCH_FLAGS} -fpic -ffunction-sections -funwind-tables -fstack-protector -fno-strict-aliasing -finline-limit=64 "
export LDFLAGS=" ${ARCH_LINK} "
```

然后执行：

```bash
./Configure android shared --prefix=~/openssl-1.0.2j/libs/armeabi --openssldir=openssl
make
make install
```

编译好的`so`在`~/openssl-1.0.2j/libs/armeabi/lib`

将`libcrypto.so`, `libcrypto.so.1.0.0`, `libssl.so,libssl.so.1.0.0`放到手机中的`/system/lib/`目录替换掉旧的`so`文件，大功告成。
