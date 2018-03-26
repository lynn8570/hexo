---
title: "Android N  无法调用私有库问题"
date: "2017-12-11"
categories: 
    - "ANDROID"
---

# 问题背景 #

因项目需要，需要将一个第三方的应用集成到系统平台上，这个第三方应用带有自己的私有so库和jni代码等。按照以前的集成方式，

    
    LOCAL_PATH:= $(call my-dir)
    include $(CLEAR_VARS)
    
    LOCAL_MODULE_TAGS := optional
    
    LOCAL_TARGET_CPU_ABI := $(TARGET_CPU_ABI)
    LOCAL_MULTILIB := 32
    
    ifeq ($(TARGET_CPU_ABI), arm64-v8a)
    	LOCAL_MULTILIB := 64
    else
    	ifeq ($(TARGET_CPU_ABI), armeabi-v7a)
    		LOCAL_TARGET_CPU_ABI := armeabi
    	endif
    endif
    
    LOCAL_PREBUILT_LIBS  :=	\
    	libs/$(LOCAL_TARGET_CPU_ABI)/xxx.so 
    
    include $(BUILD_MULTI_PREBUILT)

按照以上MK方式集成后，发现如下错：


    01-01 09:18:59.348  4092  4092 E AndroidRuntime: FATAL EXCEPTION: main
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: Process: com.olc.magicuhf, PID: 4092
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: java.lang.UnsatisfiedLinkError: dlopen failed: library "/system/lib64/libuhf-tools.so" needed or dlopened by "/system/lib64/libnativeloader.so" is not accessible for the namespace "classloader-namespace"
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: 	at java.lang.Runtime.loadLibrary0(Runtime.java:989)
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: 	at java.lang.System.loadLibrary(System.java:1562)
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: 	at android.hardware.uhf.magic.Reader.<clinit>(Reader.java:1279)
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: 	at android.hardware.uhf.magic.Reader.init(Native Method)
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: 	at com.olc.magicuhf.App.InitUHF(App.java:22)
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: 	at com.olc.magicuhf.App.onCreate(App.java:18)
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: 	at android.app.Instrumentation.callApplicationOnCreate(Instrumentation.java:1025)
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: 	at android.app.ActivityThread.handleBindApplication(ActivityThread.java:5405)
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: 	at android.app.ActivityThread.-wrap2(ActivityThread.java)
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1546)
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: 	at android.os.Handler.dispatchMessage(Handler.java:102)
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: 	at android.os.Looper.loop(Looper.java:154)
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: 	at android.app.ActivityThread.main(ActivityThread.java:6121)
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: 	at java.lang.reflect.Method.invoke(Native Method)
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: 	at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:889)
    01-01 09:18:59.348  4092  4092 E AndroidRuntime: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:779)
    01-01 09:18:59.351  1342  1816 W ActivityManager:   Force finishing activity com.olc.magicuhf/.MainActivity


# 方式一：NO #

查了一些资料说：ndroid N 不能直接调用系统的一些私有库了，公用的库都定义在public.libraries.txt里面系统应用刚刷机是能够调用system/lib64下的库，但通过install升级该应用时，应用打开会挂。因为升级后permitted_paths就不再包含system/lib64了。所以我们可以将apk要用到的库名称写到public.libraries.txt中去解决快速调试问题。

尝试了这种方法，引发更多的编译错误，惹不起，惹不起。



# 方式二：yes#

之前都是讲这些SO库集成编译到系统里面，现在需要so库编译到apk中，这样就不是系统的私有库了，是apk自带的库。重点如下

    LOCAL_PATH:= $(call my-dir)
    include $(CLEAR_VARS)
    LOCAL_STATIC_JAVA_LIBRARIES := android-support-v4
    LOCAL_MODULE_TAGS := optional
    LOCAL_SRC_FILES := \
    $(call all-java-files-under, src)
    LOCAL_PACKAGE_NAME := Demo
    LOCAL_CERTIFICATE := platform
    LOCAL_PROGUARD_FLAG_FILES := proguard.flags
    #LOCAL_PROGUARD_ENABLED :=full
    LOCAL_PRIVILEGED_MODULE := false
    LOCAL_JNI_SHARED_LIBRARIES := libfingerprint
    LOCAL_MODULE_INCLUDE_LIBRARY  := true
    LOCAL_PREBUILT_JNI_LIBS := jni/xxx.so
    $(shell cp -rf $(LOCAL_PATH)/jni/xx.so $(TARGET_OUT)/lib64/xxx.so)
    
    # := proguard.flagLOCAL_PROGUARD_FLAG_FILESs
    
    LOCAL_AAPT_FLAGS += -c zz_ZZ
    
    include $(BUILD_PACKAGE)
    
    include $(call all-makefiles-under, jni)
    
    # Use the folloing include to make our test apk.
    include $(call all-makefiles-under,$(LOCAL_PATH))


#  题外话 #

1.如果需要访问系统文件节点，则需要系统权限，且uid要为android:sharedUserId="android.uid.system"才可以访问。
具体和SELinux的权限设置有关，除非改对应的te文件，不然，根据权限设置，必须系统应用而且UID也需要是系统的才可以访问。

2.so库需要打包到应用中才可以调用，Android N之后，非系统应用无法调用系统私有库。

