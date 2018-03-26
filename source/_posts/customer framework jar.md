---
title: "自定义framework动态链接库jar"
date: "2015-12-11"
categories: 
    - "ANDROID"
---
# 前言 #
上一篇总结的是自定义framework资源包的打包和访问，而这一篇，相对要简单一些，是android平台的自定义动态链接库。目的在于定义一个framework级别的jar，供相关应用使用。之前一个项目的common组件采用的是BUILD_STATIC_JAVA_LIBRARY方式，调试起来相对简单，不需要涉及到framework的资源，framework的jar包这样，但是会造成每个应用内都引入一个common的jar包，各自为营，不知不觉加大了整体软件的大小。因此这种common级别的jar包，还是作成动态jar，加入到系统中。

# common library #

其实参考android的sample代码就可以完成基本的jar包编译，这里只是记录一下，其实，没那么难，啦啦。jar库的基文件目录如下：

![](http://7xl98n.com1.z0.glb.clouddn.com/QQ截图20151216094557.jpg)

java下存放纯java代码，一些需要的资源，我们就直接使用上一篇中说的自定义framework资源。
## jar包的com.zowee.widget.xml ##

	<?xml version="1.0" encoding="utf-8"?>
	<permissions>
	    <library name="com.zowee.widget"
	            file="/system/framework/com.zowee.widget.jar"/>
	</permissions>


## 项目编译的mk ##

 项目编译的mk代码如下


	```
	ifneq ($(TARGET_BUILD_JAVA_SUPPORT_LEVEL),)
	# This makefile shows how to build your own shared library that can be
	# shipped on the system of a phone, and included additional examples of
	# including JNI code with the library and writing client applications against it.
	LOCAL_PATH := $(call my-dir)
	# the library
	# ============================================================
	include $(CLEAR_VARS)
	LOCAL_SRC_FILES := \
	        $(call all-subdir-java-files)
	LOCAL_MODULE_TAGS := optional
	
	# 我们要编译的库名就是com.zowee.widget.jar
	LOCAL_MODULE:= com.zowee.widget
	include $(BUILD_JAVA_LIBRARY)
	# 下面我们需要将com.zowee.widget.xml编译会生出到
	# system/etc/permission下，用于作权限声明的
	include $(CLEAR_VARS)
	LOCAL_MODULE := com.zowee.widget.xml
	LOCAL_MODULE_TAGS := optional
	LOCAL_MODULE_CLASS :=ETC
	LOCAL_MODULE_PATH :=$(TARGET_OUT_ETC)/permissions
	LOCAL_SRC_FILES :=$(LOCAL_MODULE)
	include $(BUILD_PREBUILT)
	
	# the documentation
	# ============================================================
	include $(CLEAR_VARS)
	LOCAL_SRC_FILES := $(call all-subdir-java-files) $(call all-subdir-html-files)
	LOCAL_MODULE:= com.zowee.widget
	LOCAL_DROIDDOC_OPTIONS := com.zowee.widget
	LOCAL_MODULE_CLASS := JAVA_LIBRARIES
	LOCAL_DROIDDOC_USE_STANDARD_DOCLET := true
	include $(BUILD_DROIDDOC)
	# The JNI component
	# ============================================================
	# Also build all of the sub-targets under this one: the library's
	# associated JNI code, and a sample client of the library.
	include $(CLEAR_VARS)
	include $(call all-makefiles-under,$(LOCAL_PATH))
	endif # JAVA_SUPPORT
```
    

以上大部分为参考sample代码，编译之后，会生成 

`Install:out/target/product/.../system/framework/com.zowee.widget.jar` 
和
 `Copy xml: out/target/product/.../system/etc/permissions/com.zowee.widget.xml`;

## 修改device.mk文件 ##

将这两个添加到项目编译中，这样生成的system.image才会包含所需要的jar和xml文件；
拷贝xml文件：
`PRODUCT_COPY_FILES += vendor/zowee/proprietary/framework/WidgetLibrary/com.zowee.widget.xml:system/etc/permissions/com.zowee.widget.xml
`
添加jar模块：`PRODUCT_PACKAGES+=com.zowee.widget`

# app如何访问引入动态链接库 #

作为一个普通的app，如何访问系统的framework res资源，如何引入动态的jar包，并在xml和java代码中访问公共资源中的内容.

## app的mk文件 ##

	```
	LOCAL_PATH:= $(call my-dir)
	include $(CLEAR_VARS)
	LOCAL_MODULE_TAGS := optional
	
	# 要编译的app模块名称PlatformLibraryClient.apk
	LOCAL_PACKAGE_NAME := PlatformLibraryClient
	# Only compile source java files in this apk.
	LOCAL_SRC_FILES := $(call all-java-files-under, src)
	# 不要用current！，不然无法访问到自定义的framework组件
	#LOCAL_SDK_VERSION := current
	# 将动态库导入编译
	LOCAL_JAVA_LIBRARIES := com.zowee.widget
	LOCAL_PROGUARD_ENABLED := disabled
	include $(BUILD_PACKAGE)
	```
## AndroidManifest.xml文件 ##

	```
	<manifest xmlns:android="http://schemas.android.com/apk/res/android"
	    package="com.example.android.platform_library.client">
	    
	    <application android:label="Platform Lib Client">
	    
	        <!--将 com.zowee.widget 引入-->
	        
	        <uses-library android:name="com.zowee.widget" />
	        
	        <activity android:name="Client">
	            <intent-filter>
	                <action android:name="android.intent.action.MAIN"/>
	                <category android:name="android.intent.category.LAUNCHER"/>
	            </intent-filter>
	        </activity>
	    </application>
	</manifest>
	```


# app如何访问自定义framework资源 #

## java代码中 ##

只要导入编译正常，基本上java代码中就可以直接对自定义framework的资源进行访问，例如：
对jar包中的自定义控件，自定义资源的访问

`import com.zowee.widget.ToolbarImageTextButton;`

`b2.setText(getString(com.zowee.R.string.zowee),1);`

## xml中自定义控件的访问 ##
xml中对自定义控件的访问，自定义属性的访问，自定义资源的访问，可参考如下：

	<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:tools="http://schemas.android.com/tools"
	    xmlns:zowee="http://schemas.android.com/apk/res/com.zowee" 
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    >
	    <com.zowee.widget.ToolbarImageTextButton
	    android:layout_width="wrap_content"
	    android:layout_height="wrap_content"
	    android:text="@com.zowee:string/zowee"
	    zowee:bracketColor="#057852dd" 
	    android:id="@+id/toolbutton"
	    />
	</RelativeLayout>

请注意 ` xmlns:zowee="http://schemas.android.com/apk/res/com.zowee"` 和 `zowee:bracketColor="#057852dd" `
以及
`android:text="@com.zowee:string/zowee"`

# 总结 #

自此，针对自定义的一套资源和jar的框架已经完成了，这样做的目的是，当整个平台升级的时候，我们可以单独将自定义的framework res 和 framework jar 先进行移植，接着依赖于这些的自定义app,也可以顺利的在新的平台上运行了