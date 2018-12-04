---

layout: post
title: NDK学习记录8-jni调用java错误处理
date: 2018-12-04
categories: [Android-NDK]
author: 小鬼
tags: [NDK,反编译]
description: jni调用java错误处理

---

咱们jni中异常了，前面说到了崩溃，除了jni中异常，前面文章说到了jni调用java，那么调用java代码执行异常了又是什么情况，怎么处理，这一篇记录学习了。

### 一. 异常产生情况

Java的异常处理我想大家都很清晰了,有编译时的异常，比如操作File的时候会有FileNotFoundException,运行时异常，比如IllegalArgumentException等，然而在jni中依旧有该这些异常。所以主要分的也是编译时异常和运行时异常

### 二. jni调用java异常与java调用java异常的区别

1. java代码:logInfo提供给后续jni调用

方法中第一行打印日志，参数为null，会崩溃。

```
    public void logInfo() {
    //参数为null
        Log.i(TAG, null);
        Log.i(TAG, "age:" + age + ",name:" + name);
    }
```

2. jni代码

在执行方法前和执行方法后都打印出Log

```
//···省略前面的寻找class与methid等步骤

//执行前打印
LOGE("准备执行loginfo");
    //4. 调用方法,CallVoidMethod,参数:实例、要调用的方法id
    env->CallVoidMethod(object_, meth_id_logInfo);
    //执行完打印
    LOGE("执行完loginfo");
```

3. 输出

```
I/Native-lib: 准备执行loginfo
I/Native-lib: 执行完loginfo
D/AndroidRuntime: Shutting down VM
E/AndroidRuntime: FATAL EXCEPTION: main
    Process: shixin.ndkdemo, PID: 22022
    java.lang.RuntimeException: Unable to start activity ComponentInfo{shixin.ndkdemo/shixin.ndkdemo.MainActivity}: java.lang.NullPointerException: println needs a message
        at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2817)
        at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2892)
        at android.app.ActivityThread.-wrap11(Unknown Source:0)
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1593)
        at android.os.Handler.dispatchMessage(Handler.java:105)
        at android.os.Looper.loop(Looper.java:164)
        at android.app.ActivityThread.main(ActivityThread.java:6541)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.Zygote$MethodAndArgsCaller.run(Zygote.java:240)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:767)
     Caused by: java.lang.NullPointerException: println needs a message
        at android.util.Log.println_native(Native Method)
        at android.util.Log.i(Log.java:165)
        at shixin.ndkdemo.obj_test.ObjectTest.logInfo(ObjectTest.java:46)
        at shixin.ndkdemo.MainActivity.testObjectMethodUse(Native Method)
        at shixin.ndkdemo.MainActivity.onCreate(MainActivity.java:33)
        at android.app.Activity.performCreate(Activity.java:6975)
        at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1213)
        at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2770)
        at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2892) 
        at android.app.ActivityThread.-wrap11(Unknown Source:0) 
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1593) 
        at android.os.Handler.dispatchMessage(Handler.java:105) 
        at android.os.Looper.loop(Looper.java:164) 
        at android.app.ActivityThread.main(ActivityThread.java:6541) 
        at java.lang.reflect.Method.invoke(Native Method) 
        at com.android.internal.os.Zygote$MethodAndArgsCaller.run(Zygote.java:240) 
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:767) 
W/ActivityManager:   Force finishing activity shixin.ndkdemo/.MainActivity
```

在前两行，看到了，执行前和执行后的打印都出来了。对于java调用就不用多说明了，抛异常后，java代码就不会继续往下执行了，这就是和java直接调用不同的地方：

**区别：jni调用java异常后，jni代码还会继续执行下去。**

### 三. 处理方式

为了让出现异常后不继续往下执行，避免之后的代码有依赖的地方造成错误，则可在jni处进行检查和做出对后面方法的return

1. 检测异常方法：ExceptionCheck()或者ExceptionOccurred()
2. 打印异常:ExceptionDescribe()
3. 为了让程序不崩溃，可执行clear：ExceptionClear();
4. 测试从jni层手动抛出一个异常：利用Exception类，env的ThrowNew方法

部分代码如下：

```
//执行方法
if (env->ExceptionOccurred()) {
        //打印异常
        env->ExceptionDescribe();
        //clear方法执行了后，应用就不会因为这个异常而崩溃
        env->ExceptionClear();
        LOGE("出现异常");
        jclass clss_excep=env->FindClass("java/lang/Exception");
        char *errorMessage = "这是一个异常";
        //咱们试试手动抛出一个异常
        env->ThrowNew(clss_excep, errorMessage);
        return;
    }
    LOGE("执行完loginfo");
```

### 总结

1. jni调用java方法崩溃后，jni代码会继续往下执行
2. jni代码可以ExceptionCheck检测到java异常(前提是java代码没有自行捕获异常)，还可以clear就像捕获异常似的，可以使得程序不崩溃
3. 可以通过jni抛出异常到java端，这个的使用主要不是今天讲这里，不然抛出去java也捕获不到，程序必然崩溃。比如可以在jni运行异常了，可以抛出异常给java端。