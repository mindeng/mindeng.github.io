---
layout: post
title:  "Android + CMake 踩坑记"
date:   2016-12-22
tags:   [android, cmake, ndk, c++, c]
---

**更新：**

发现结合 android.toolchain.cmake 使用 cmake 是更好的办法，推荐使用。示例如下：

```bash
#!/bin/sh
cd `dirname $0`/..

mkdir -p build_android_arm
cd build_android_arm

cmake -DCMAKE_BUILD_WITH_INSTALL_RPATH=ON -DCMAKE_TOOLCHAIN_FILE=scripts/android.toolchain.cmake \
    -DANDROID_ABI="armeabi-v7a with NEON" -DANDROID_NATIVE_API_LEVEL=android-16 \
    -DANDROID_STL=gnustl_static -DANDROID_TOOLCHAIN_NAME=arm-linux-androideabi-4.9 \
    $@ ..
```

将该文件保存在工程顶级目录的某个子目录中，例如：`$PROJECT_DIR/scripts/cmake_android_arm.sh`，然后执行：

```
sh ./scripts/cmake_android_arm.sh && cd build_android_arm && make -j8
```

关于 `android.toolchain.cmake` 文件的选择，我尝试使用了 NDK 自带的
`$NDK/build/cmake/android.toolchain.cmake` 会出错，说找不到
`armv7-none-linux-androideabi` 。最后选择用 OpenCV 带的
`android.toolchain.cmake` ，好用。

另外，如果项目需要 C++11/RTTI/exception 支持，记得在每个 CMakeLists.txt 中加上：

```
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -frtti -fexceptions")
```

-----------------------

近期在移植一些开源库到 Android 平台，由于目标库本身大多数是基于 CMake
构建的，为图方便和避免后期维护的麻烦，理所当然希望沿用 CMake 构建系统了。

最开始尝试阶段是在 Android Studio 里面搞的，新版 Android Studio
还比较方便，直接可以创建带 C++ 支持的工程，甚至可以直接添加 jni
方法，同时帮你生成对应的符合 jni 规范的 C++ 函数模版，感觉棒极了。

不过需要注意的是，该功能只能在工程自动生成的 C++ 文件
`src/main/cpp/native-lib.cpp`
中生成函数模版，如果你删掉了该文件，就爱莫能助了。

尝鲜之后，虽然感觉方便，但是局限太多，而且修改 C/C++ 代码或编辑
CMakeLists.txt/Makefile 等文件还是 Vim 比较方便，况且每次构建都得开个 Android
Studio 也挺麻烦，所以最后还是打算切换到 [Android standalone
toolchain](https://developer.android.com/ndk/guides/standalone_toolchain.html)。

坑从这里开始。

坑1: `$NDK/build/tools/make-standalone-toolchain.sh` 有bug
-----------------------------------------------------------

尝试多次该命令，发现我装的这个版本(Mac 系统最新的 NDK 版本)带的这个 sh 脚本有
bug，执行 n
遍安装路径里死活都是空的，也没有任何错误提示信息。也有可能是我漏了必需的参数，
但好歹也给个错误提示吧。

后来发现这个脚本其实就是封装了另一个 python 脚本，直接运行
`$NDK/build/tools/make_standalone_toolchain.py`，好使。

坑2: stl 的选择问题
-----------------------------------------------------------
用 Android Studio 构建时没发现这个问题，gradle 都配好了，设置好 `cppFlags
"-std=c++11 -frtti -fexceptions"` 基本就没问题了。

切换成 standalone toolchain 之后，却一直报错 <tr1/functional> 找不到，<tr1/unordered_map>
找不到之类的。后来去 $NDK 目录下 `find $NDK -name functional`，才发现 tr1 目录在 gnu-libstdc++
下面才有，其他几个 stl 下面都没有带 tr1 的。

因为当初安装 standalone toolchain 时，看到这篇文档
[C++ Library Support](https://developer.android.com/ndk/guides/cpp-support.html)
里面有提到libc++ 默认支持 C++11，没仔细看误以为要支持 C++11 只能选 libc++，于是参数用的
`make_standalone_toolchain.py --arch arm --stl=libc++`, 导致找不到 tr1
下面的头文件。 

找到了问题，重新执行 `make_standalone_toolchain.py --arch arm --stl=gnustl
--install-dir=/install/path --force`，问题解决。

坑3: CMake Configuring 死循环的问题
-------------------------------------
这个不算 Android 的问题，是由于我对 cmake 的这个特性还不熟悉导致的。

编写 CMakeLists.txt 时，为图方便，我在 CMakeLists.txt 里直接配置好 C/C++ flags
以及 compiler 信息，配置如下:

```cmake
project("My_Project")

OPTION(ANDROID "Build for Android" ON)

if(ANDROID)
    message("Configuring for Android ...")

    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -frtti -fexceptions")

    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIE")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIE")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fPIE -pie")

    set(NDK_STANDALONE_TOOLCHAIN /usr/local/ndk-toolchain/arm)
    set(CMAKE_SYSTEM_NAME Android)
    set(CMAKE_SYSTEM_VERSION 24)
    set(CMAKE_C_COMPILER ${NDK_STANDALONE_TOOLCHAIN}/bin/clang)
    set(CMAKE_CXX_COMPILER ${NDK_STANDALONE_TOOLCHAIN}/bin/clang++)
    set(CMAKE_FIND_ROOT_PATH ${NDK_STANDALONE_TOOLCHAIN})

    add_definitions("--sysroot=${NDK_STANDALONE_TOOLCHAIN}/sysroot")

    set(ARCH "armv7-a")

    # TODO: add more abi support?
    # For OpenCV
    set(ANDROID_NDK_ABI_NAME "armeabi-v7a")
    #set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}  -Wall  -O3 -march=armv7-a ")
    #set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall   -O3 -march=armv7-a")

    add_definitions(-DANDROID)
    add_definitions(-D__ANDROID__)
endif()
```

结果执行 cmake 时就悲剧的死循环了。一遍又一遍的 Configuring。

后来分析出原因是：如果把设置 compiler 的指令放在 project() 指令后面，cmake
会认为 Configuration 启动后，compiler 信息
发生了改变，因此需要再重新配置一次，从而造成死循环。 

解决方案：把 project() 指令放到 endif() 后面就好了。
