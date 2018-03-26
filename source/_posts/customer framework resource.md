---
title: "自定义framework资源apk"
date: "2015-12-10"
categories: 
    - "ANDROID"
---

# 前言 #
去年在高通平台上的加过一次这种自定义framework级别的资源包，今年在MTK平台又部署了一次，当然第二次就比较轻车熟路了；趁现在编译时间，把这次的部署，完整记录一下吧。

# 自定义framework资源的 packing&accessing #

首先，我们先明确这么做的目的是，将公司自定义的一些图片文字样式等资源独立与平台；这样可以与平台代码区分；这么做的好处就是，方便以后每个os的升级，在移植上的工作量相对减少。以下为具体的操作步骤：

1. 建立自定义资源目录结构
	 在vendor目录下，建立一个与framework res类似路径的资源文件夹'vendor\zowee\proprietary\framework\base\res'；目录结构如下图
	
	![](http://7xl98n.com1.z0.glb.clouddn.com/QQ截图20151210141422.jpg)

2. 编写Android.mk文件，模块名称`LOCAL_PACKAGE_NAME := zowee-res`，资源flag 3~9可用`LOCAL_AAPT_FLAGS := -x3`；`LOCAL_NO_ZOWEE := true`

	```
	LOCAL_PATH := $(call my-dir)
	include $(CLEAR_VARS)
	# include apicheck.mk later, we need the build pass to prepare the first version
	# include $(LOCAL_PATH)/apicheck.mk
	LOCAL_PACKAGE_NAME := zowee-res
	LOCAL_CERTIFICATE := platform
	LOCAL_AAPT_FLAGS := -x3
	
	# Tell aapt to build resource in utf16(the ROM will be enlarged),
	# in order to save RAM size for string cache table
	ifeq (yes,strip$(MTK_GMO_RAM_OPTIMIZE))
	LOCAL_AAPT_FLAGS += --utf16
	endif
	
	#LOCAL_NO_MTKRES := true
	LOCAL_NO_ZOWEE := true
	LOCAL_MODULE_TAGS := optional
	# Install this alongside the libraries.
	LOCAL_MODULE_PATH := $(TARGET_OUT_JAVA_LIBRARIES)
	# Create package-export.apk, which other packages can use to get
	# PRODUCT-agnostic resource data like IDs and type definitions.
	LOCAL_EXPORT_PACKAGE_RESOURCES := true	
	include $(BUILD_PACKAGE)
	
	# define a global intermediate target that other module may depend on.
	.PHONY: zowee-res-package-target
	zowee-res-package-target: $(LOCAL_BUILT_MODULE)
```

3. AndroidManifest.xml文件，代码可参考：

	```
	<?xml version="1.0" encoding="utf-8"?>
	
	<manifest xmlns:android="http://schemas.android.com/apk/res/android"
	    package="com.zowee" android:sharedUserId="android.uid.system"
	</manifest>
	
	```


4. 修改\device\mediatek\common下device.mk，将新的资源模块添加到system img中

	```
		# for mediatek-res
		PRODUCT_PACKAGES += mediatek-res
	
		# for zowee-ress
		PRODUCT_PACKAGES += zowee-res
	```
5. 修改Z:\MT5505\alps\frameworks\base\libs\androidfw下的Assetmanager.cpp，将新的资源模块添加的默认的assets路径中

	```
	///M:add the resource path
	static const char* kMediatekAssets = "framework/mediatek-res/mediatek-res.apk";
	static const char* kZoweeAssets = "framework/zowee-res/zowee-res.apk";
	....
	 ///M:add the new resource path into default path,so all the app can reference,@{	
	bool isOK1 =addAssetPath(path, NULL);
	String8 path2(root);
	path2.appendPath(kMediatekAssets);
	bool isOK2 =addAssetPath(path2, NULL);
	if(!isOK2){
	  ALOGW("AssetManager-->addDefaultAssets isok2 is false");
	}
	
	String8 path3(root);
	path3.appendPath(kZoweeAssets);
	bool isOK3 =addAssetPath(path3, NULL);
	if(!isOK3){
	  ALOGW("AssetManager-->addDefaultAssets isok3 is false");
	}else{
		 ALOGW("AssetManager-->addDefaultAssets isok3 is true");
	}
	return isOK1;
	
	```

6. 修改alps\build\core下的package_internal.mk，添加编译依赖关系等
	
	```
	resource_export_package := $(intermediates.COMMON)/package-export.apk
	$(R_file_stamp): $(resource_export_package)
	$(call intermediates-dir-for,APPS,mediatek-res,,COMMON)/package-export.apk:$(call intermediates-dir-for,APPS,framework-res,,COMMON)/src/R.stamp
	$(call intermediates-dir-for,APPS,zowee-res,,COMMON)/package-export.apk:$(call intermediates-dir-for,APPS,mediatek-res,,COMMON)/src/R.stamp
	........
	mediatek_res_package_export :=
	mediatek_res_package_export_deps :=
	  ifneq ($(LOCAL_NO_MTKRES), true)
	    mediatek_res_package_export += \
	      $(call intermediates-dir-for,APPS,mediatek-res,,COMMON)/package-export.apk
	      mediatek_res_package_export_deps += \
	      $(call intermediates-dir-for,APPS,mediatek-res,,COMMON)/src/R.stamp
	      
	      ifneq ($(LOCAL_NO_ZOWEE), true)
	       mediatek_res_package_export += \
	          $(call intermediates-dir-for,APPS,zowee-res,,COMMON)/package-export.apk
	       mediatek_res_package_export_deps += \
	          $(call intermediates-dir-for,APPS,zowee-res,,COMMON)/src/R.stamp
	      endif
	  endif
	......
	$(filter-out $(mediatek_res_package_export),$(resource_export_package)): $(mediatek_res_package_export_deps)
	$(R_file_stamp): $(mediatek_res_package_export_deps)
	$(resource_export_package) $(R_file_stamp) $(LOCAL_BUILT_MODULE): $(all_library_res_package_export_deps)
	$(LOCAL_INTERMEDIATE_TARGETS): \
	    PRIVATE_AAPT_INCLUDES := $(all_library_res_package_exports) $(mediatek_res_package_export)
	endif # LOCAL_NO_STANDARD_LIBRARIES
	
	```
7. 修改alps\frameworks\base下Android.mk将资源路径添加的编译路径中
	
	```
	framework_res_source_path := APPS/framework-res_intermediates/src
	# M:add mediatek resource path
	mediatek-res-source-path := APPS/mediatek-res_intermediates/src
	zowee-res-source-path := APPS/zowee-res_intermediates/src
	.......
	LOCAL_INTERMEDIATE_SOURCES += \
	      $(zowee-res-source-path)/com/zowee/R.java \
	      $(zowee-res-source-path)/com/zowee/Manifest.java \
	      $(zowee-res-source-path)/com/zowee/internal/R.java \
				$(mediatek-res-source-path)/com/mediatek/internal/R.java \
				$(mediatek-res-source-path)/com/mediatek/R.java \
				$(mediatek-res-source-path)/com/mediatek/Manifest.java 
	# @}
	......
	zowee_res_R_stamp := \
		$(call intermediates-dir-for,APPS,zowee-res,,COMMON)/src/R.stamp
	$(full_classes_compiled_jar): $(zowee_res_R_stamp)
	```
8. Z:\MT5505\alps\build\core clear_vars.mk添加`LOCAL_NO_ZOWEE :=`
9. 修改alps\frameworks\base\services\core\java\com\android\server\pm
PackageManagerService.java，不对新的资源apk进行 dexopted

	```
	alreadyDexOpted.add(frameworkDir.getPath() + "/framework-res.apk");
	
	/// M: This file doesn't contain any code,just like framework-res,so don't dexopt it
	alreadyDexOpted.add(frameworkDir.getPath() + "/mediatek-res/mediatek-res.apk");
	alreadyDexOpted.add(frameworkDir.getPath() + "/zowee-res/zowee-res.apk");
	
	/** M: [CIP] Add CIP scanning path variable @{ */
	File customFrameworkDir = new File("/custom/framework");
	/// M: [CIP] Add custom resources to libFiles to avoid dex opt.
	alreadyDexOpted.add(customFrameworkDir.getPath() + "/framework-res.apk");
	alreadyDexOpted.add(customFrameworkDir.getPath() + "/mediatek-res.apk");
	alreadyDexOpted.add(customFrameworkDir.getPath() + "/zowee-res.apk");
	/** @} */
	```
10. 针对L版本，还需要修个alps\frameworks\base\libs\androidfw    for L 版本
`ResourceTypes.cpp` 主要是对资源id的查找映射表

	```
	#define APP_PACKAGE_ID      0x7f
	#define SYS_MTK_PACKAGE_ID  0x08 /// M: 0x08 is mediatek-res.apk resource ID
	#define SYS_ZOWEE_PACKAGE_ID  0x03 /// M: 0x03 is zowee-res.apk resource ID
	#define SYS_PACKAGE_ID      0x01
	.......
	 if (packageId != APP_PACKAGE_ID && packageId != SYS_PACKAGE_ID && packageId != SYS_MTK_PACKAGE_ID
	                	&& packageId != SYS_ZOWEE_PACKAGE_ID) {
	                    outValue->dataType = Res_value::TYPE_DYNAMIC_REFERENCE;
	                }
	......
	else if (packageId == APP_PACKAGE_ID || packageId == SYS_PACKAGE_ID || packageId == SYS_MTK_PACKAGE_ID
	                    	|| packageId == SYS_ZOWEE_PACKAGE_ID) {
	
	......
	mLookupTable[APP_PACKAGE_ID] = APP_PACKAGE_ID;
	    mLookupTable[SYS_PACKAGE_ID] = SYS_PACKAGE_ID;
	    mLookupTable[SYS_ZOWEE_PACKAGE_ID] = SYS_ZOWEE_PACKAGE_ID;
	    mLookupTable[SYS_MTK_PACKAGE_ID] = SYS_MTK_PACKAGE_ID;
	
	```

# 自定义资源的添加和访问 #

1. 在自定义资源目录下，像普通的应用一样，添加`values/strings.xml`等字符串，颜色等资源，编译之后，在out目录下的APPS intermediate文件中，将生成的`public_resource.xml`文件中，需要开放权限，作为外部访问的添加到资源目录下的`values/public.xml`中


	```
	<resources>
	  <!-- Declared at vendor/zowee/proprietary/framework/base/res/res/values/strings.xml:23 -->
	  <public type="string" name="zowee" id="0x03020000" />
	  <!-- Declared at vendor/zowee/proprietary/framework/base/res/res/values/attrs.xml:5 -->
	  <public type="attr" name="bracketColor" id="0x03010000" />
	
	  <!-- Declared at vendor/zowee/proprietary/framework/base/res/res/values/colors.xml:4 -->
	  <public type="color" name="zui_common_default_hint" id="0x03030000" />
	  <!-- Declared at vendor/zowee/proprietary/framework/base/res/res/values/colors.xml:6 -->
	  <public type="color" name="zui_common_default_textcolor" id="0x03030001" />
	  <!-- Declared at vendor/zowee/proprietary/framework/base/res/res/values/colors.xml:9 -->
	  <public type="color" name="zui_common_activited_textcolor" id="0x03030002" />
	</resources>
	```

2. 另外如果想作为私有的ID资源则可以放在`values/symbols.xml`中

	```
	<resources>
	  <!-- We don't want to publish private symbols in com.mediatek.R as part of the
	       SDK.  Instead, put them here. -->
	  <private-symbols package="com.zowee.internal" />
	  <java-symbol type="string" name="zowee_internal" />
	
	</resources>
	```

# 总结 #
经过以上步骤之后，可在所有的应用中通过`android:text="@com.zowee:string/zowee" `访问自定义的资源，在java代码中通过`int z =com.zowee.R.string.zowee`直接访问资源ID。
另附mtk提供的diff包，如果有什么不对的，可以对比查证
[mtk提供的diff包](/media/mtkcustomer-res-L.zip)

