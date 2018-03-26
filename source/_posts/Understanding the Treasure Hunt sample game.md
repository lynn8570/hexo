---
title: "Understanding the Treasure Hunt sample game"
date: "2018-02-26"
categories: 
    - "ANDROID"
---
# Google VR SDK 寻宝游戏code overview



## 开发环境

- 硬件要求：
  - Daydream平台：可支持Daydream的手机：Galaxy note8，MotoZ force，pixel，Axon7 ZTE，Moto Z，Mate 9，LG v30等 和  Daydream Viewer
  - Cardboard平台，运行android 4.4 api19以上的手机和一个 Cardboard viewer
- 软件要求：
  - android sdk 25以上
  - Google VR SDK

## 运行Google VR SDK

运行**samples-sdk-treasurehunt** 

在android 4.4以上就可以运行，但是不一定能用，例如如果没有Gsensor数据等无法进行运动捕捉。虽然可以正常运行，但是无法进行体验。

## Manifest 文件

- VR应用属性

  ```
  <manifest ...>
      <uses-sdk android:minSdkVersion="19" android:targetSdkVersion="24"/>
      ...
      <uses-feature android:glEsVersion="0x00020000" android:required="true" />
      <uses-feature android:name="android.software.vr.mode" android:required="false"/>
      <uses-feature android:name="android.hardware.vr.high_performance" android:required="false"/>
      ...
      <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
      ...
  </manifest>
  ```

  Google VR SDK 支持的最低版本为api 19, 另外Daydream 应用的target API 要24或者以上级别。需 OpenGL ES 2.0 支持，可正确渲染VR内容。`<uses-feature android:name="android.software.vr.mode" android:required="false"/>`和`<uses-feature android:name="android.hardware.vr.high_performance" android:required="false"/>` 由 Android N加入的新特性，前者表示需要使用Android VR模式，后者表示需要可支持Daydream的设备。

  对于没有Google VR服务的设备，需要安装Google VR 服务，才可以将可支持Daydream的设备与对应的VR viewer配对。因此还需要添加`READ_EXTERNAL_STORAGE`权限。


- VR Activity 属性

  Activity也需要添加一些属性声明，才可兼容Daydream

  ```
  <activity
      android:name=".MyActivity"
      android:screenOrientation="landscape"
      android:enableVrMode="@string/gvr_vr_mode_component"
      android:theme="@style/VrActivityTheme"
      android:resizeableActivity="false"
      android:configChanges="density|keyboardHidden|navigation|orientation|screenSize|uiMode"
      ...>

  ```

  `enableVrMode`属性用于启用Android VR 模式，Android 7.0 N 引入的功能特性，用于支持高性能的移动 VR 应用。该属性表示，当启动Activity的时候，系统需要自动启动VR模式。

  > 启动 VR activity的时候，屏幕会有些轻微的闪烁，这是因为当启动VR模式的时候，Android需要切换到低持久性模式显示。

  `android:theme="@style/VrActivityTheme"`确保Activity在VR转换期间正常运行，`android:resizeableActivity="false"`表明Activity不可调整显示大小，不与其他的Activity分屏。landscape表明屏幕需要横屏显示。

  `configChanges`可避免activity因为一些配置变化而造成activity的重新创建。

  Google VR应用基本和要求：

  https://developers.google.com/vr/distribute/daydream/app-quality

  - 设计要求：

    1. 物体对象需要放置在一定的距离，以便用户可以看清楚而不是看到两张图片。如果需要辅助文字，需要保证文字的可读性。建议物体对象需要放置在0.5米距离以上
    2. 需要维护头部动作位置追踪
    3. 地平线维持，不至于倾斜
    4. 相机操作需要用户启动，但可以不是一直直接的控制操作
    5. app不干涉系统级别的复位行为
    6. …….

  - 功能要求

    1. 使用支持的VR SDK版本

    2. 使用Daydream API进行activity切换`daydreamApi.launchVrHomescreen();`  `daydreamApi.launchInVr(componentName);`

    3. 不请求nfc权限，应用不能使用nfc权限，而是Daydream平台需要使用nfc功能来进行配对操作。

    4. 应用的manifest清单文件中，需要设置正确的VR activity样式：禁止android默认的windowmanager的切换动画；隐藏系统级的ui元素，例如系统状态栏，导航栏和键盘等；采用landscape，禁止方向转化；禁用多窗口支持。

       `android:theme="@style/VrActivityTheme"`可用于禁止切换动画和隐藏系统ui;

       `android:enableVrMode="@string/gvr_vr_mode_component"`可用于设置仅从Daydream Home或者另外一个VR activities来启动Activity，以便得到更快的切换。

       示例的activity清单代码可查看前面的示例。

       需要显示在VR Home中的Daydream activity需要设置`com.google.intent.category.DAYDREAM`，如果不想显示的VR Home中，只要设置`android.intent.action.VIEW`

       ```
       <activity
           ... >
           ...
           <intent-filter>
               <action android:name="android.intent.action.MAIN" />
               <category android:name="com.google.intent.category.DAYDREAM"/>
           </intent-filter>
       </activity>

       ```

  - 根据平台设置硬件特性，对于Daydream的应用，以下硬件特性都是必须的。对于某些具有cardboard模式的VR则为可选的。

        android:glEsVersion="0x00020000"
        android.hardware.sensor.gyroscope
        android.hardware.sensor.accelerometer
        android.hardware.vr.high_performance
        android.software.vr.mode
        
        
        Examples
        
        Application only supports Daydream:
        
        <uses-feature android:glEsVersion="0x00020000" android:required="true" />
        <uses-feature android:name="android.hardware.sensor.accelerometer" android:required="true" />
        <uses-feature android:name="android.hardware.sensor.gyroscope" android:required="true" />
        <uses-feature android:name="android.hardware.vr.high_performance" android:required="true" />
        <uses-feature android:name="android.software.vr.mode" android:required="true" />
        
        Application supports both Daydream and Cardboard devices:
        
        <uses-feature android:glEsVersion="0x00020000" android:required="true" />
        <uses-feature android:name="android.hardware.sensor.accelerometer" android:required="true" />
        <uses-feature android:name="android.hardware.sensor.gyroscope" android:required="true" />
        <uses-feature android:name="android.hardware.vr.high_performance" android:required="false" />
        <uses-feature android:name="android.software.vr.mode" android:required="false" />
        
        Application's VR mode is optional:
        
        <uses-feature android:glEsVersion="0x00020000" android:required="false" />
        <uses-feature android:name="android.hardware.sensor.accelerometer" android:required="false" />
        <uses-feature android:name="android.hardware.sensor.gyroscope" android:required="false" />
        <uses-feature android:name="android.hardware.vr.high_performance" android:required="false" />
        <uses-feature android:name="android.software.vr.mode" android:required="false" />
        

  - 应用可以正常的暂停

    这包括很多常见的操作场景暂停和恢复。例如用户通过控制器或者手机按键按下的Home按钮来立即暂停应用还有音频播放等。数据的保存和恢复。

  - 应用可正常关闭

- 平台兼容性

  每个activity都必须声明好支持的VR平台；Daydream还是cardboard。两个可兼容

  ```
  <!-- Example of a Daydream/Cardboard Activity -->
  <activity ... >
      <intent-filter>
          <action android:name="android.intent.action.MAIN" />

          <!-- This marks the Activity as a Daydream Activity and allows it
               to be launched from the Daydream Home. -->
          <category android:name="com.google.intent.category.DAYDREAM" />

          <!-- This marks the Activity as a Cardboard Activity and allows it
               to be launched from the Cardboard app. -->
          <category android:name="com.google.intent.category.CARDBOARD" />

          <!-- This allows this Activity to be launched from the traditional
               Android 2D launcher as well. Remove it if you do not want
               this Activity to be launched directly from the 2D launcher. -->
          <category android:name="android.intent.category.LAUNCHER" />
      </intent-filter>
  </activity>

  ```



## 继承 GvrActivity

GvrActivity作为基类可对GoogleVR设备进行简单的集成，可以发布一些事件来与VR环境进行交互并且可以处理创建用于VR渲染的activity的一些通用的细节实现。GvrAcitivity使用的是sticky的沉浸式模式，因为GvrView只能在全屏模式下才可以进行渲染。封装的东西还是比较少的

```
package com.google.vr.sdk.base;

import android.app.Activity;
import android.os.Bundle;
import android.view.KeyEvent;
import android.view.View;
import android.view.ViewGroup.LayoutParams;
import com.google.vr.cardboard.AndroidNCompat;
import com.google.vr.cardboard.FullscreenMode;
import com.google.vrtoolkit.cardboard.ScreenOnFlagHelper;

public class GvrActivity extends Activity {//直接集成的普通的activity
    private FullscreenMode fullscreenMode;//全屏的flag设置和兼容模式
    private final ScreenOnFlagHelper screenOnFlagHelper = new ScreenOnFlagHelper(this);
    //在resume的启动sensor相关的监听
    private GvrView cardboardView;
    private boolean androidVrModeEnabled;

    public GvrActivity() {
    }

    public void setGvrView(GvrView gvrView) {
        this.setGvrView(gvrView, true);
    }

    public void setGvrView(GvrView gvrView, boolean enableVrModeFallbacks) {
        if (this.cardboardView != gvrView) {
            if (this.cardboardView != null) {
                this.cardboardView.setOnCardboardTriggerListener((Runnable)null);
            }

            this.cardboardView = gvrView;
            boolean enableAndroidVrMode = gvrView != null;
            this.androidVrModeEnabled = AndroidNCompat.setVrModeEnabled(this, enableAndroidVrMode, enableVrModeFallbacks ? 1 : 0) && enableAndroidVrMode;
            if (gvrView != null) {
                gvrView.setOnCardboardTriggerListener(new Runnable() {
                    public void run() {
                        GvrActivity.this.onCardboardTrigger();
                    }
                });
            }
        }
    }

    public GvrView getGvrView() {
        return this.cardboardView;
    }

    public void onCardboardTrigger() {//可重写
    }

    protected void updateGvrViewerParams(GvrViewerParams newParams) {
        if (this.cardboardView != null) {
            this.cardboardView.updateGvrViewerParams(newParams);
        }

    }

    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        this.requestWindowFeature(1);
        this.fullscreenMode = new FullscreenMode(this.getWindow());
    }

    protected void onResume() {
        super.onResume();
        if (this.cardboardView != null) {
            this.cardboardView.onResume();
        }

        this.fullscreenMode.goFullscreen();
        this.screenOnFlagHelper.start();
    }

    protected void onPause() {
        super.onPause();
        if (this.cardboardView != null) {
            this.cardboardView.onPause();
        }

        this.screenOnFlagHelper.stop();
    }

    protected void onDestroy() {
        if (this.cardboardView != null) {
            this.cardboardView.setOnCardboardTriggerListener((Runnable)null);
            this.cardboardView.shutdown();
            this.cardboardView = null;
        }

        super.onDestroy();
    }

    public void setContentView(View view) {
        if (view instanceof GvrView) {
            this.setGvrView((GvrView)view);
        }

        super.setContentView(view);
    }

    public void setContentView(View view, LayoutParams params) {
        if (view instanceof GvrView) {
            this.setGvrView((GvrView)view);
        }

        super.setContentView(view, params);
    }

    public void onBackPressed() {
        super.onBackPressed();
        this.cardboardView.onBackPressed();
    }

    public boolean onKeyDown(int keyCode, KeyEvent event) {
        return this.shouldSuppressKey(keyCode) || super.onKeyDown(keyCode, event);
    }

    public boolean onKeyUp(int keyCode, KeyEvent event) {
        return this.shouldSuppressKey(keyCode) || super.onKeyUp(keyCode, event);
    }

    public void onWindowFocusChanged(boolean hasFocus) {
        super.onWindowFocusChanged(hasFocus);
        this.fullscreenMode.onWindowFocusChanged(hasFocus);
    }

    public void setScreenAlwaysOn(boolean enabled) {
        this.screenOnFlagHelper.setScreenAlwaysOn(enabled);
    }

    private boolean shouldSuppressKey(int keyCode) {
        if (!this.androidVrModeEnabled) {
            return false;
        } else {
            return keyCode == 24 || keyCode == 25;//处理了音量加减键
        }
    }
}
```



## 定义GvrView

GvrView主要用于VR渲染

- common_ui.xml

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/ui_layout"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >


    <com.google.vr.sdk.base.GvrView
        android:id="@+id/gvr_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_alignParentTop="true"
        android:layout_alignParentLeft="true" />

</RelativeLayout>
```

- 初始化

```
  public void initializeGvrView() {
    setContentView(R.layout.common_ui);//设置xml文件

    GvrView gvrView = (GvrView) findViewById(R.id.gvr_view);
    gvrView.setEGLConfigChooser(8, 8, 8, 8, 16, 8);
    // Android设备往往支持多种EGL配置，可以使用不同数目的通道(channel)，也可以指定每个通道具有不同数目的位(bits)深度。因此， 在渲染器工作之前就应该指定EGL的配置。如果没有调用的话，那么默认的配置是 RGB_888 surface with a depth buffer depth of at least 16 bits. 

    gvrView.setRenderer(this);
    gvrView.setTransitionViewEnabled(true);//提示页面

    // Enable Cardboard-trigger feedback with Daydream headsets. This is a simple way of supporting
    // Daydream controller input for basic interactions using the existing Cardboard trigger API.
    //即使用现有的trigger API实现与daydream 头戴式设备的交互。
    gvrView.enableCardboardTriggerEmulation();

    if (gvrView.setAsyncReprojectionEnabled(true)) {
      // Async reprojection decouples the app framerate from the display framerate,
      // allowing immersive interaction even at the throttled clockrates set by
      // sustained performance mode.
      AndroidCompat.setSustainedPerformanceMode(this, true);
    }

    setGvrView(gvrView);
  }
```

## 渲染视图

Google VR 支持两种renderers，GvrView.StereoRendere和GvrView.Renderer。demo中采用的是前者。

```
 public interface StereoRenderer {
        @UsedByNative
        void onNewFrame(HeadTransform var1);

        @UsedByNative
        void onDrawEye(Eye var1);

        @UsedByNative
        void onFinishFrame(Viewport var1);

        void onSurfaceChanged(int var1, int var2);

        void onSurfaceCreated(EGLConfig var1);

        void onRendererShutdown();
    }

    public interface Renderer {
        @UsedByNative
        void onDrawFrame(HeadTransform var1, Eye var2, Eye var3);

        @UsedByNative
        void onFinishFrame(Viewport var1);

        void onSurfaceChanged(int var1, int var2);

        void onSurfaceCreated(EGLConfig var1);

        void onRendererShutdown();
    }
```

实现GvrView.StereoRendere接口饼设置gvrView.setRenderer(this);

```
public class TreasureHuntActivity extends GvrActivity implements GvrView.StereoRendere{
    
    
    
    .....
    private void init(){
        gvrView.setRenderer(this);
    }
}
```

- void onNewFrame(HeadTransform var1);app 渲染的时候都会调用
- void onDrawEye(Eye var1);eye参数不同的时候调用

需要扩展相关OpenGL知识

## Render实现

- onSurfaceCreated

  ```
   /**
     * Creates the buffers we use to store information about the 3D world.
     *
     * <p>OpenGL doesn't use Java arrays, but rather needs data in a format it can understand.
     * Hence we use ByteBuffers.
     *
     * @param config The EGL configuration used when creating the surface.
     */
    @Override
    public void onSurfaceCreated(EGLConfig config) {
       GLES20.glClearColor(0.1f, 0.1f, 0.1f, 0.5f); // 黑色背景.

      ／／初始化一些buffer
      ByteBuffer bbVertices = ByteBuffer.allocateDirect(WorldLayoutData.CUBE_COORDS.length * 4);
      bbVertices.order(ByteOrder.nativeOrder());
      cubeVertices = bbVertices.asFloatBuffer();
      cubeVertices.put(WorldLayoutData.CUBE_COORDS);
      cubeVertices.position(0);
      。。。。。。
      int vertexShader = loadGLShader(GLES20.GL_VERTEX_SHADER, R.raw.light_vertex);
      int gridShader = loadGLShader(GLES20.GL_FRAGMENT_SHADER, R.raw.grid_fragment);
      int passthroughShader = loadGLShader(GLES20.GL_FRAGMENT_SHADER, R.raw.passthrough_fragment);／／从文件中加载
      
      更多需要扩展下GLES20的相关api
      
     // 另外起一个线程，来对音频文件进行解码
      new Thread(
              new Runnable() {
                @Override
                public void run() {
                  
                  gvrAudioEngine.preloadSoundFile(OBJECT_SOUND_FILE);//立体音频文件加载
                  sourceId = gvrAudioEngine.createSoundObject(OBJECT_SOUND_FILE);
                  gvrAudioEngine.setSoundObjectPosition(／／可设置声音发出的位置
                      sourceId, modelPosition[0], modelPosition[1], modelPosition[2]);
                  gvrAudioEngine.playSound(sourceId, true /* looped playback */);
                  
                  gvrAudioEngine.preloadSoundFile(SUCCESS_SOUND_FILE);
                }
              })
          .start();
    	
    }
  ```

  ​


- onNewFrame

  ```
  /**
     * Prepares OpenGL ES before we draw a frame.
     *
     * @param headTransform The head transformation in the new frame.
     */
    @Override
    public void onNewFrame(HeadTransform headTransform) {
      setCubeRotation();

      // Build the camera matrix and apply it to the ModelView.
      Matrix.setLookAtM(camera, 0, 0.0f, 0.0f, CAMERA_Z, 0.0f, 0.0f, 0.0f, 0.0f, 1.0f, 0.0f);

      headTransform.getHeadView(headView, 0);

      // Update the 3d audio engine with the most recent head rotation.
      headTransform.getQuaternion(headRotation, 0);
      gvrAudioEngine.setHeadRotation(
          headRotation[0], headRotation[1], headRotation[2], headRotation[3]);
      // Regular update call to GVR audio engine.
      gvrAudioEngine.update();

      checkGLError("onReadyToDraw");
    }
  ```

  public void onNewFrame(HeadTransform headTransform) 其中HeadTransform封装了头部位置动作信息，可获取**四元数，坐标体系**扩张（扩展知识）

- onDrawEye

  ```

    /**
     * Draws a frame for an eye.
     *
     * @param eye The eye to render. Includes all required transformations.
     */
    @Override
    public void onDrawEye(Eye eye) {
      GLES20.glEnable(GLES20.GL_DEPTH_TEST);
      GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT | GLES20.GL_DEPTH_BUFFER_BIT);

      checkGLError("colorParam");

      // Apply the eye transformation to the camera.
      Matrix.multiplyMM(view, 0, eye.getEyeView(), 0, camera, 0);

      // Set the position of the light
      Matrix.multiplyMV(lightPosInEyeSpace, 0, view, 0, LIGHT_POS_IN_WORLD_SPACE, 0);

      // Build the ModelView and ModelViewProjection matrices
      // for calculating cube position and light.
      float[] perspective = eye.getPerspective(Z_NEAR, Z_FAR);
      Matrix.multiplyMM(modelView, 0, view, 0, modelCube, 0);
      Matrix.multiplyMM(modelViewProjection, 0, perspective, 0, modelView, 0);
      drawCube();

      // Set modelView for the floor, so we draw floor in the correct location
      Matrix.multiplyMM(modelView, 0, view, 0, modelFloor, 0);
      Matrix.multiplyMM(modelViewProjection, 0, perspective, 0, modelView, 0);
      drawFloor();
    }
  ```

  Google VR 实现左右眼视觉变形操作。

- 响应

  ```
   /**
     * Called when the Cardboard trigger is pulled.cardboard上的按钮，或者是屏幕点击事件都可以触发
     */
    @Override
    public void onCardboardTrigger() {
      Log.i(TAG, "onCardboardTrigger");

      if (isLookingAtObject()) {如果目标物体在视觉范围内，则
        successSourceId = gvrAudioEngine.createStereoSound(SUCCESS_SOUND_FILE);
        gvrAudioEngine.playSound(successSourceId, false /* looping disabled */);
        hideObject();
      }

      // Always give user feedback.
      vibrator.vibrate(50);
    }
  ```

- 音频

  ```
  gvrAudioEngine =
      new GvrAudioEngine(this, GvrAudioEngine.RenderingMode.BINAURAL_HIGH_QUALITY);

  ```

## 总结

demo主要展示了GvrActivity，GvrView, GvrAudioEngine d的几个基本用法，SDK主要提供了左右眼形变后产生立体效果的这部分功能，已经声音引擎对应的使用，以及头部位置信息动作的封装。