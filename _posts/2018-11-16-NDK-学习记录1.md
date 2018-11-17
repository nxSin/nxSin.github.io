---

layout: post
title: NDK学习记录1
date: 2018-11-16
categories: [Android NDK]
author: 小鬼
tags: NDK基础配置
description: 利用cmake简单的配置NDK环境，为后面学习整一个开始

---

### 一. Android Studio配置

#### (一) 组件下载

* 要使用和调试，先下载NDK组件：
    * NDK包：这套工具集允许您为 Android 使用 C 和 C++ 代码，并提供众多平台库，让您可以管理原生 Activity 和访问物理设备组件，例如传感器和触摸输入。
    * cmake：一款外部构建工具，可与 Gradle 搭配使用来构建原生库。如果您只计划使用 ndk-build，则不需要此组件。
    * LLDB：一种调试程序，Android Studio 使用它来调试原生代码。


![](https://github.com/nxSin/BlogPic/blob/master/Android/ndk/AndroidSDK%E9%85%8D%E7%BD%AE%E5%9B%BE.png?raw=true)

勾选如上图所示三项即可。**这个不用经常更新，特别是NDK，一般有新版本出来了会提示你，但是不用随意更新，因为有可能更新到比较新的版本，你之前的版本编译就不行了哦，改起来还挺麻烦的，嘿嘿，特别是针对含有第三方库编译的情况。**

#### (二) 配置

Android新版本推荐使用CMake，这里讲的都是CMake，使用的脚本为CMakeLists.txt

[CMake概述](http://www.hahack.com/codes/cmake/)：

```
Make 工具有挺多的：GNU Make ，QT 的 qmake ，微软的 MS nmake，BSD Make（pmake），Makepp，等等。这些 Make 工具遵循着不同的规范和标准，所执行的 Makefile 格式也千差万别。这样就带来了一个严峻的问题：如果软件想跨平台，必须要保证能够在不同平台编译。而如果使用上面的 Make 工具，就得为每一种标准写一次 Makefile ，这将是一件让人抓狂的工作。

CMake就是针对上面问题所设计的工具：它首先允许开发者编写一种平台无关的 CMakeList.txt 文件来定制整个编译流程，然后再根据目标用户的平台进一步生成所需的本地化 Makefile 和工程文件，如 Unix 的 Makefile 或 Windows 的 Visual Studio 工程。从而做到“Write once, run everywhere”。显然，CMake 是一个比上述几种 make 更高级的编译配置工具。一些使用 CMake 作为项目架构系统的知名开源项目有 VTK、ITK、KDE、OpenCV、OSG 等.

在 linux 平台下使用 CMake 生成 Makefile 并编译的流程如下：

编写 CMake 配置文件 CMakeLists.txt
执行命令 cmake PATH 或者 cmake PATH 生成 Makefile。其中， PATH 是 CMakeLists.txt 所在的目录。
使用 make 命令进行编译。
```


1. 如果是新建项目，则勾选C++支持即可完成创建

2. 如果是老项目：
    1. 在main目录下新建cpp的原生代码目录，然后再创建一个native-lib.cpp的c++文件
    2. 在module下新建文件:CMakeLists.txt
    3. 链接到gradle中，目的通过gradle来识别编译脚本，在运行的时候以依赖的方式来运行CMake

* gradle配置

```
android {
    defaultConfig {
        externalNativeBuild {
            cmake {
            		//打包x86的so
                arguments "-DANDROID_ABI=x86"
                cppFlags ""
            }
        }
    }
    externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
        }
    }
}
```

* CMakeLists.txt基本内容如下：
    * add_library：配置我们库的名称、类型、和编译文件
    * find_library:找寻ndk中的库
    * target_link_libraries:链接指定的库到我们库中

```
# For more information about using CMake with Android Studio, read the
# documentation: https://d.android.com/studio/projects/add-native-code.html

# Sets the minimum version of CMake required to build the native library.

cmake_minimum_required(VERSION 3.4.1)

# Creates and names a library, sets it as either STATIC
# or SHARED, and provides the relative paths to its source code.
# You can define multiple libraries, and CMake builds them for you.
# Gradle automatically packages shared libraries with your APK.

#指定需传入三个参数（准备打包生成so函数库名称，库类型，依赖源文件相对路径即代码路径）
add_library( # Sets the name of the library.
             my_native_lib

            # SHARED 动态链接库  STATIC 静态链接库
             SHARED

             # Provides a relative path to your source file(s).
             src/main/cpp/native-lib.cpp )

# Searches for a specified prebuilt library and stores the path as a
# variable. Because CMake includes system libraries in the search path by
# default, you only need to specify the name of the public NDK library
# you want to add. CMake verifies that the library exists before
# completing its build.

# 用于定位NDK中的库，通过find来找到，并取个path变量
# 需要传入两个参数（path变量、ndk库名称）
find_library( # Sets the name of the path variable.
                #变量名取为log-lib
              log-lib

              # Specifies the name of the NDK library that
              # you want CMake to locate.
            #找ndk中的log库，即找ndk中的包为liblog.so
              log )

# Specifies libraries CMake should link to your target library. You
# can link multiple libraries, such as libraries you define in this
# build script, prebuilt third-party libraries, or system libraries.

# 指定关联到原生库的库
target_link_libraries( # Specifies the target library.
                        #我们的库,与前面的name保持一致
                       my_native_lib

                       # Links the target library to the log library
                       # included in the NDK.
                        #要链接进来的库，如上面指定的path
                       ${log-lib} )
```

* 最后的项目结构如下：

![](https://github.com/nxSin/BlogPic/blob/master/Android/ndk/%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png?raw=true)

到此，基础配置完成。下一篇开始进行使用