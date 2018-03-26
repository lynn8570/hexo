---
title: "Android vr ar"
date: "2018-02-25"
categories: 
    - "ANDROID"
---
# Android VR-AR 

如果接下来要做AR眼镜相关的项目，目前需要做一些知识储备。所以本文做为一个项目开始前的准备工作。了解目前已知行业先行者已经实现的技术是已经到了一个什么样的程度后，才知道我们要前进的方向。着重了解Google和目前接触到的一个比较完善的vr-ar系统公司睿悦Niburu。

## VR和AR的概念

VR(Vritual Reality)虚拟现实，最直白的说就是纯虚拟的数字画面。理解起来就是，用户在VR中看到的所有东西都是虚拟的，但是这些虚拟的东西又需要让用户感觉到很真实。

AR(Argumented Reality)增强现实，虚拟数字画面+裸眼现实

此外现在还有一个概念提出来，MR(Mediatd Reality)介导现实，数字化现实+虚拟数字画面



- VR和AR的关系，可以理解为，VR为AR的一种极端情况。即假设我们把人眼的内容分为现实中真实的部分+数字投放的Argumentation（增强）部分。那么如果这种虚拟的部分越来越多以至于一种极端的情况，几乎没有的真实的部分，这种情况下叫：VR。所以这样理解起来，VR是AR的一个真子集。虚拟的东西占了全部，以至于越来越偏离了原来的现实原点。

- MR 数字化的视觉感知。对本身现实的感知能力的大小。例如让你有能力看到更小的东西，例如让佩戴者像有了X射线般的超级视力，看到皮肤下的血管？？？超能力！！或者是在现实的景物傻姑娘加以渲染，置身奇幻世界等等。所以MR是视觉能力维度上的提升。

  ![知乎图示](https://pic3.zhimg.com/80/817549fc583ef153904ef2f324e3fa0f_hd.jpg)

  他们三者的关系像是：在AR之外，增加了数字化的感知能力。

  ![知乎图示](https://pic4.zhimg.com/80/f208e8d885ed7749c9b1c53988f85593_hd.jpg)

> 以上整理自36kr，作者艾韬，易瞳科技CTO，师从Steve Mann。

为什么不是同心圆，而是有偏离呢？全图其实还有：

![](http://a.36krcnd.com/nil_class/6d8fab32-010b-4d77-bf31-20af3eb0358e/7B22.tmp.jpg!heading)

还有个Mixed，混合现实。如果我们把同心圆的原点认为是真实世界的原点。这几个概念给我们的感受是怎样的？

## 市面上的AR-VR眼镜

- 轻量级AR - Google Glass，单眼棱镜光学透视的设计方案，提示型系统
- 中量级AR - Hololens，双目棱镜光学透视方案，还有例如亮风台
- 重量级AR - HTC the Void 动态VR 眼镜，头戴VR显示，背上电脑，将真实的环境渲染成游戏场景。
- 纯VR - VR头盔



## Google VR 做了什么

- DayDream

  > Daydream是由谷歌2016年11月10日发布的一个vr平台，由一个头盔，一个控制杆盒很多兼容的智能手机组成。

  ![](http://7xl98n.com1.z0.glb.clouddn.com/QQ20180225-151253@2x.png)

  我们可以使用google提供的VR SDK来开发Daydream app

  - 硬件

    Daydream 或者 cardboard

    ![](http://7xl98n.com1.z0.glb.clouddn.com/QQ20180225-153056@2x.png)

  - 软件

    android studio，Nougat（API 25）or higher, Google VR SDK

  - VR SDK sample

    sdk-treasurehunt：示例应用，可以展示搜索和搜集物品。

    sdk-controllerclient 	主要演示接收和处理daydream的控制输入

    sdk-simplepanowidget 	可以载入全景视图的组件
    sdk-simplevideowidget 	可以渲染360度视频的视图组件 VRView
    sdk-video360 	可以渲染360度视频的视图组件，在可支持的手机上运行vr效果的播放器时候，**屏幕会闪烁**
    sdk-videoplayer 	通过 Asynchronous Reprojection Video Surface API来播放视频，

  - Treasure Hunt 示例应用演练

    这个应用体现的Google VR SDK的核心功能：**Stereo rendering** 立体渲染，将app的view进行立体渲染，从而创造出3D的体验；**Spatial audio** 空间音效，制造出声音从不同地方出来的效果，增强现实体验；**Head movement tracking**根据头部动作，更新视图；**User input**用户输入响应，接收Daydream controller或者Cardboard 按钮的输入。

    *Todo: [Treasure Hunt walkthrough](https://developers.google.com/vr/android/samples/treasure-hunt) 详细走一遍sample的代码* 

  - 开发GoogleVR项目步骤

    1. 配置项目等级

       ```
       // The Google VR SDK requires version 2.3.3 or higher.
           classpath 'com.android.tools.build:gradle:2.3.3'

           // The Google VR NDK requires experimental version 0.9.3 or higher.
           // classpath 'com.android.tools.build:gradle-experimental:0.9.3'

       ```

    2. 添加依赖库

       ```
       dependencies {
            // Adds Google VR spatial audio support
            compile 'com.google.vr:sdk-audio:1.80.0'

            // Required for all Google VR apps
            compile 'com.google.vr:sdk-base:1.80.0'
         }

       ```

    3. 配置ProGuard

       如果需要使用proguard来减小apk文件大小的话，需要添加

       ```
       buildTypes {
               release {
                   minifyEnabled true
                   proguardFiles.add(file('../../proguard-gvr.txt'))
               }
           }

       ```

    4. 接下来需要哪些知识储备：

       - 浏览一遍谷歌VR示例代码 Treasure Hunt，寻宝游戏

       - GoogleVRSDK视频和全景图片示例应用

       - GoogleVR设计和开发准则

       - android空间音频效果开发向导

       - Google VR API

       - Daydream，学习Daydream用户交互控制器相关操作，可查看Review the controller library in **gvr-android-sdk** > **libraries** >
           **sdk-controller** ，查阅 库api 控制器 API

         > 原文链接如下：

       - [Treasure Hunt sample code walkthrough](https://developers.google.com/vr/android/samples/treasure-hunt)

       - [Google VR SDK video and panoramic image sample app walkthrough](https://developers.google.com/vr/android/samples/vrview)

       - Learn about Google VR design and development principles in [Daydream elements](https://developers.google.com/vr/elements/overview).

       - [Spatial audio for Android tutorial](https://developers.google.com/vr/android/spatial-audio)

       - [Google VR API Reference](https://developers.google.com/vr/android/reference_overview)

       - Daydream:

          Learn about implementing Daydream controller user interactions   in your app:

         - Review the controller library in **gvr-android-sdk** > **libraries** >  **sdk-controller**.
         - See also the [controller library API reference](https://developers.google.com/vr/android/reference/com/google/vr/sdk/controller/package-summary).


- Tilt Brush

  3d空间绘画

  Oculus Rift and HTC VIVE可支持该功能。

  都是vr头戴式设备

  ​

- Earth VR

  vr 视角来查看一些风景名胜

- Cardboard

- Expeditions

  教育类相关，旅行远征，例如体验在火星表面，老师可以带着孩子进行更有沉浸式的体验

- Jump

  Jump is Google's professional VR video solution. Jump makes 3D-360 video
   production at scale possible with best-in-class automated stitching. 
  Jump cameras are designed to work with the Jump Assembler to enable 
  seamless VR video production.

  一系列的摄像头产品，用于拍摄全景视频，自动缝合技术，实现

- Developers

## Google AR 提供了什么

https://developers.google.com/ar/discover/

ARCore 是一个用于在 Android 上构建增强现实应用的平台。ARCore 使用三个主要技术将虚拟内容与通过手机摄像头看到的现实世界整合：

- [**运动跟踪**](https://developers.google.com/ar/discover/concepts#motion_tracking)让手机可以理解和跟踪它相对于现实世界的位置。
- [**环境理解**](https://developers.google.com/ar/discover/concepts#environmental_understanding)让手机可以检测平坦水平表面（例如地面或咖啡桌）的大小和位置。
- [**光估测**](https://developers.google.com/ar/discover/concepts#light_estimation)让手机可以估测环境当前的光照条件。

### ARCore基本工作原理

> ARCore 在做两件事：在移动设备移动时跟踪它的位置和构建自己对现实世界的理解。
>
> ARCore 的运动跟踪技术使用手机摄像头标识兴趣点（称为特征点），并跟踪这些点随着时间变化的移动。将这些点的移动与手机惯性传感器的读数组合，ARCore 可以在手机移动时确定它的位置和屏幕方向。
>
> 除了标识关键点外，ARCore 还会检测平坦的表面（例如桌子或地面），并估测周围区域的平均光照强度。 这些功能共同让 ARCore 可以构建自己对周围世界的理解。
>
> 借助 ARCore 对现实世界的理解，您能够以一种与现实世界无缝整合的方式添加物体、注释或其他信息。 您可以将一只打盹的小猫放在咖啡桌的一角，或者利用艺术家的生平信息为一幅画添加注释。 运动跟踪意味着您可以移动和从任意角度查看这些物体，即使您转身离开房间，当您回来后，小猫或注释还会在您添加的地方。

### Arcore SDK概览

#### Hello AR java  

需要android 7.0以上，需要安装arcore

- 清单文件，声明需要android.hardware.camera.ar，且需要安装Arcore，文件表明应用在Google的应用市场上只能在支持arcore的设备上可见

  ```
  <manifest xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:tools="http://schemas.android.com/tools"
      package="com.google.ar.core.examples.java.helloar">

    <uses-permission android:name="android.permission.CAMERA"/>
    <!-- This tag indicates that this application requires ARCore.  This results in the application
         only being visible in the Google Play Store on devices that support ARCore. -->
    <uses-feature android:name="android.hardware.camera.ar" android:required="true"/>

    <application
        android:allowBackup="false"
        android:icon="@drawable/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/AppTheme"
        android:usesCleartextTraffic="false"
        tools:ignore="GoogleAppIndexingWarning">

      <activity
          android:name=".HelloArActivity"
          android:label="@string/app_name"
          android:configChanges="orientation|screenSize"
          android:exported="true"
          android:theme="@style/Theme.AppCompat.NoActionBar"
          android:screenOrientation="locked">
        <intent-filter>
          <action android:name="android.intent.action.MAIN"/>
          <category android:name="android.intent.category.LAUNCHER"/>
        </intent-filter>
      </activity>
      <!-- 需要安装arcore -->
      <meta-data android:name="com.google.ar.core" android:value="required" />
    </application>
  </manifest>
  ```

#### computevision

- 清单文件

  ```
  <manifest xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:tools="http://schemas.android.com/tools"
      package="com.google.ar.core.examples.java.computervision">

    <uses-permission android:name="android.permission.CAMERA"/>

    <application
        android:allowBackup="false"
        android:icon="@drawable/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/AppTheme"
        android:usesCleartextTraffic="false"
        tools:ignore="GoogleAppIndexingWarning">

      <activity
          android:name=".MainActivity"
          android:label="@string/app_name"
          android:configChanges="orientation|screenSize"
          android:exported="true"
          android:theme="@style/Theme.AppCompat.NoActionBar"
          android:screenOrientation="locked">
        <intent-filter>
          <action android:name="android.intent.action.MAIN"/>
          <category android:name="android.intent.category.LAUNCHER"/>
        </intent-filter>
      </activity>
    </application>
  </manifest>
  ```

> TODO: ARdemo walkthrough

Api 查阅

- [com.google.ar.core](https://developers.google.com/ar/reference/java/com/google/ar/core/package-summary) ARCore API implementation.
- [com.google.ar.core.exceptions](https://developers.google.com/ar/reference/java/com/google/ar/core/exceptions/package-summary) ARCore exceptions.



### 如何启用ARCore

#### 清单文件配置

- AR only: 以下定义表示应用只能在支持android.hardware.camera.ar，且安装了com.google.ar.core的设备上运行

  ```
  <uses-sdk android:minSdkVersion="{24 or higher}" />
  24以上版本
    ...
    <uses-feature android:name="android.hardware.camera.ar" android:required="true" />

    <application>
      ...
          <meta-data android:name="com.google.ar.core" android:value="required" />
      ...
  ```

- AR可选，表示可以在不支持AR Core的设备上运行

  ```
  <uses-sdk android:minSdkVersion="{14 or higher}" />
    ...

    <application>
      ...
          <meta-data android:name="com.google.ar.core" android:value="optional" />
      ...
  ```

#### 添加依赖

确认已经在项目的  `build.gradle` 文件中添加了 Google's Maven 仓库：

```
allprojects {
repositories {
    google()
        ...

```

添加 ARCore 库依赖

```
dependencies {
    // ARCore library
    implementation 'com.google.ar:core:1.0.0'
    ...

```

> Gradle会自动的将ARcore中的标签合并到应用的清单文件中。

#### 执行运行时检查

确认是否已安装ARcore

```
// Set to true ensures requestInstall() triggers installation if necessary.
private boolean mUserRequestedInstall = true;

// in onResume:
try {
  if (mSession == null) {
    switch (ArCoreApk.getInstance().requestInstall(this, mUserRequestedInstall)) {
      case INSTALLED:
        mSession = new Session(this);
        // Success.
        break;
      case INSTALL_REQUESTED:
        // Ensures next invocation of requestInstall() will either return
        // INSTALLED or throw an exception.
        mUserRequestedInstall = false;
        return;
    }
  }
} catch (UnavailableUserDeclinedInstallException e) {
  // Display an appropriate message to the user and return gracefully.
  return;
} catch (...) {  // current catch statements
  ...
  return;  // mSession is still null
}

```

在Activity resume的时候，执行session创建，如果创建不成功，则请求安装，



检查是否支持AR core，可选方案的时候

```
void maybeEnableArButton() {
  // Likely called from Activity.onCreate() of an activity with AR buttons.
  ArAvailability availability = ArCoreApk.getInstance().checkAvailability(this);
  if (availability.isTransient()) {
    // re-query at 5Hz while we check compatibility.
    new Handler().postDelayed(new Runnable() {
      @Override
      public void run() {
        maybeEnableArButton();
      }
    }, 200);
  }
  if (availability.isSupported()) {
    mArButton.setVisibility(VISIBLE);
    mArButton.setEnabled(true);
    // indicator on the button.
  } else { // unsupported or unknown
    mArButton.setVisibility(HIDDEN);
    mArButton.setEnabled(false);
  }
}
```

> checkAvailability可能需要联网检查

## Google VR AR 在现有平台上的运行情况

1. 寻宝游戏，demo 代码根据vr提供的sdk实现了左右分屏的这种效果，也就是说只要运行在普通的安卓手机上，在屏幕上，就可以看到一个左右分屏显示的功能，然后如果配合Google 的cardboard viewer 或者是 Daydream viewer 是可以直接在眼睛上实现3D显示的。那这个demo在我们的820+眼镜上的运行效果就是。在切换到2D模式的时候，我们看到的就是两个左右的图片，切换到3d的时候，确实可以实现看到一个叠加的图片，但是这个叠加后的有些变形。

2. Google vr播放器，需要在7.0以上版本运行，2d的播放器可以在5516上运行，但要切换成3D模式的时候，提示手机与Daydream不兼容；在pixel一些指定的兼容设备上才可运行，但是运行起来感觉屏幕闪烁较为明显，需要配合daydream屏体的控制器运行观看

3. ARcore支持的设备：

   > Asus Zenfone AR
   >
   > LG V30
   >
   > Google Pixel
   >
   > OnePlus 5
   >
   > Samsung Galaxy

   在可支持的设备上，**hello ar**运行的起来就是一个摄像头应用，在识别平坦桌面后，可以放置android玩偶，然后移动相机，可以实现，各种视角查看玩偶。就想真实存在的物体一样。

   5516无法支持ARcore

   **computevision** 纹理阅读，可识别物体纹理


## 如何兼容Google VR AR

- VR、 Daydream

  <uses-feature android:name="android.software.vr.mode" android:required="false"/>和<uses-feature android:name="android.hardware.vr.high_performance" android:required="false"/> 由 Android N加入的新特性，前者表示需要使用Android VR模式，后者表示需要可支持Daydream的设备。

  对于没有Google VR服务的设备，需要安装Google VR 服务，才可以将可支持Daydream的设备与对应的VR viewer配对


- AR

   <uses-feature android:name="android.hardware.camera.ar" android:required="true" />

  另外需要安装ARcore




## Nibiru 系统



## Nibiru VR 

- Nibiru studio

  > 面向VR应用开发者，开发者无需OpenGL基础，无需写
  > shader和计算矩阵，极大地简化了VR应用开发流程。并且
  > 支持Head Tracking和Raycast交互方式，以及具有完善的
  > 事件系统。定义了功能丰富的VR控件与组件集合，在功能点
  > 上涵盖了Image、Label、Skybox、Listview、
  > ProgressBar Gridview等核心控件，同时支持播放、图片、
  > 浏览器等组件。

- demo运行

  新建一个空的项目，选择 file-new-import module，选择模块源码位置，点击finish，完成倒入

  需要睿悦的os。

  demo展示的主要展示了基于睿悦系统的一些基本控件的使用，基于睿悦系统的头部运动操作和响应，以及对系统设置Wi-Fi、蓝牙等系统界面的调用。

  整体体验起来，空间立体的感觉还不够，似乎只是一个2d界面，经过3d左右分屏处理后，融合sensor之后的附加的交互而已。立体渲染部分比较少，给人的空间感体验较差。



## Nibiru AR 

目前就demo而言，AR的特性体现的比较少。

## Unity

游戏与VR/AR开发引擎。

## Unreal

https://www.zhihu.com/question/20116279 知乎上关于unity和unreal两个平台的对比

TODO: 体验一下