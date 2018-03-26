---
title: "使用第三方库后proguard报错问题"
date: "2016-06-25"
categories: 
    - "android"
---
# 添加jar包或者aar库 #
在mk源码编译下，当aar做普通的jar包lib库用就可以了
Android.mk

	LOCAL_PROGUARD_FLAG_FILES := proguard.flags
	LOCAL_STATIC_JAVA_LIBRARIES := \
	    android-ex-variablespeed \
	    lmlibrary
	
	.......
	
	include $(CLEAR_VARS)
	LOCAL_PREBUILT_STATIC_JAVA_LIBRARIES := \
	    lmlibrary:libs/lmlibrary-release.aar
	
	include $(BUILD_MULTI_PREBUILT)

由于编译后会进行混淆，所以一般如果有proguard报错，需要在proguard文件中添加如下内容：


	-libraryjars libs/lmlibrary-release.aar
	-keep class  com.alibaba.fastjson.** { *;}
	-dontwarn  com.alibaba.fastjson.**


