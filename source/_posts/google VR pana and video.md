---
title: "Google VR SDK 示例 360 media"
date: "2018-02-27"
categories: 
    - "ANDROID"
---
# Google VR SDK 示例代码全景和视频

Google的SDK示例代码中有两个应用 simplepanowidget和simlevrvideo展示了如何内嵌一个360度的图片或者是视频。

## 什么是360 media

360度媒介，包括360度全景图片和360度全景视频。通过他们，可以加强传统应用的沉浸式体验。例如可以在旅游类应用中添加这种视图查看器作为海底世界游览体验，在家装应用中，让消费者可以先体验一下虚拟的效果。

这些360度媒介支持立体变化并与Google cardboard等vr平台兼容，也支持在浏览器中或者移动应用中查看而不需要特定的vr硬件支持。

- 常见格式

  360度媒体的格式可以是 mono或者是stereo。图片和视频通常需要存储在equirectangular-panoramic (equirect-pano)格式中，可以理解为柱体全景格式，这种格式为大部分拍摄解决方案所支持的格式。

  Mono 360使用一张全景，stero 360使用两张堆积的全景图片。stero在全景之上，还有一个景深的这样一个概念，我们对前景和背景的距离深度有了一个感官的体验。

- 360度图片

  可以以png,jpeg,gif的格式进行存储，建议使用jpeg来提高压缩率

  为了最大限度的提高兼容性和性能，需要的尺寸是2次幂，2024，4096这样

  mono图片，单张的方案，需要2:1的宽高比 例如4096*4096

  stereo图片，左右两张堆积方案的，需要1:1的宽高比 4096*4096

- 360度视频

  以h2642编码的mp4s格式存储

  mono方案提供2:1宽高比

  stereo提供1:1宽高比

  一些比较老的设备无法对大雨1080p(1920*1080)的视频进行解码，如果要进行兼容，建议提供提供monoscopic 1920 * 1080的视频和stereo 2048 * 2048的的视频

- 如何内嵌和显示这些媒体

  - [VR View on Android](https://developers.google.com/vr/android/samples/vrview)
  - [VR View on the Web](https://developers.google.com/vr/concepts/vrview-web)

  另外也有ios平台等

## 如何拍摄到全景的媒体

- 实景拍摄

  - [Cardboard Camera App](https://play.google.com/store/apps/details?id=com.google.vr.cyclops):    使用该应用可以快速的拍摄360全景图片，然后可以下载  [download](https://support.google.com/cardboard/answer/6333620)    这些图片然后使用    [conversion tool](https://storage.googleapis.com/cardboard-camera-converter/index.html)    来生成 stereo 360° 图片，就可以符合 Cardboard's 图片要求了。
  - [Ricoh Theta](https://theta360.com/): 可用于拍摄 mono 360° 图片和视频 

- CG capture

  cgi软件可根据生成全景图片和视频，一些热门的拍摄方案如下

  - [360 Panorama Capture for Unity](https://assetstore.unity.com/packages/tools/camera/360-panorama-capture-38755):    A free, easy-to-use 360° capture plugin for **Unity**.
  - [Unreal](https://docs.unrealengine.com/latest/INT/Platforms/VR/StereoPanoramicCapture/): Stero panoramic capture in **Unreal**.
  - (Unsupported) [Domemaster3D](https://github.com/zicher3d-org/domemaster-stereo-shader/):    A free solution for capturing mono and stereo 360° images from Maya / Autodesk 3ds Max.
  - [Renderman](https://drive.google.com/drive/folders/0B0lYKpimsJgWfjFwa2xLQlVGNnJXcGxvbmNJbzIwUkRWQ2YtOHB5blNMNjlXSUlxbmJNVVU):    Open-source library for capturing 360° content.开源库
  - [Rendering Omnidirectional Stereo Content](https://developers.google.com/vr/jump/rendering-ods-content.pdf):    A whitepaper for anyone interested in writing their own 360° capture    solutions.

- YouTube下载，在YouTube上搜索monoscopic的视频，然后通过 http://youtubemp4.to/ 来下载mp4格式的文件

## 全景图片walkthrough

根据Gsensor的数据展示不同的方位景物；

全屏模式 和 cardboard 模式切换

- 代码概览

  在layout XML文件中，嵌入，该组件自身就集成了全屏，立体模式，退出等模式的切换按钮

  ```
    <com.google.vr.sdk.widgets.pano.VrPanoramaView
              android:id="@+id/pano_view"
              android:layout_margin="5dip"
              android:layout_width="match_parent"
              android:scrollbars="@null"
              android:layout_height="250dip"/>
  ```

- activitie 代码

  初始化

  ```
    @Override
    protected void onCreate(Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
      setContentView(R.layout.main_layout);
  .......
      panoWidgetView = (VrPanoramaView) findViewById(R.id.pano_view);
      panoWidgetView.setEventListener(new ActivityEventListener());

      // Initial launch of the app or an Activity recreation due to rotation.
      handleIntent(getIntent());
    }
  ```

  其中ActivityEventListener继承自VrPanoramaEventListener，用于监听来自widget的事件，例如当widget加载完图片后的onLoadSuccess回调。或者加载失败后的onLoadError回调。

  ```
  /**
     * Listen to the important events from widget.
     */
    private class ActivityEventListener extends VrPanoramaEventListener {
      /**
       * Called by pano widget on the UI thread when it's done loading the image.
       */
      @Override
      public void onLoadSuccess() {
        loadImageSuccessful = true;
      }

      /**
       * Called by pano widget on the UI thread on any asynchronous error.
       */
      @Override
      public void onLoadError(String errorMessage) {
        loadImageSuccessful = false;
        Toast.makeText(
            SimpleVrPanoramaActivity.this, "Error loading pano: " + errorMessage, Toast.LENGTH_LONG)
            .show();
        Log.e(TAG, "Error loading pano: " + errorMessage);
      }
  ```

  在线程中加载图片

  ```
  /**
     * Helper class to manage threading.
     */
    class ImageLoaderTask extends AsyncTask<Pair<Uri, Options>, Void, Boolean> {

      /**
       * Reads the bitmap from disk in the background and waits until it's loaded by pano widget.
       */
      @Override
      protected Boolean doInBackground(Pair<Uri, Options>... fileInformation) {
        Options panoOptions = null;  // It's safe to use null VrPanoramaView.Options.
        InputStream istr = null;
        if (fileInformation == null || fileInformation.length < 1
            || fileInformation[0] == null || fileInformation[0].first == null) {
          AssetManager assetManager = getAssets();
          try {
            istr = assetManager.open("andes.jpg");／／打开assert文件
            panoOptions = new Options();
            panoOptions.inputType = Options.TYPE_STEREO_OVER_UNDER;
          } catch (IOException e) {
            Log.e(TAG, "Could not decode default bitmap: " + e);
            return false;
          }
        } else {
          try {
            istr = new FileInputStream(new File(fileInformation[0].first.getPath()));
            panoOptions = fileInformation[0].second;
          } catch (IOException e) {
            Log.e(TAG, "Could not load file: " + e);
            return false;
          }
        }

        panoWidgetView.loadImageFromBitmap(BitmapFactory.decodeStream(istr), panoOptions);／／将读取的iostream进行加载
        try {
          istr.close();
        } catch (IOException e) {
          Log.e(TAG, "Could not close input stream: " + e);
        }

        return true;
      }
    }
  ```

  ​

## 全景视频walkthrough

全景视频的视图和全景图片的非常相似，使用上也差不多，展现出来的，就是一个是视频画面生动的效果

XML集成：

```
<com.google.vr.sdk.widgets.video.VrVideoView
          android:id="@+id/video_view"
          android:layout_width="match_parent"
          android:scrollbars="@null"
          android:layout_height="250dip"/>
```



初始化

```
 // Bind input and output objects for the view.
    videoWidgetView = (VrVideoView) findViewById(R.id.video_view);
    videoWidgetView.setEventListener(new ActivityEventListener());
```



其中ActivityEventListener

```
/**
   * Listen to the important events from widget.
   */
  private class ActivityEventListener extends VrVideoEventListener  {
    /**
     * Called by video widget on the UI thread when it's done loading the video.
     */
    @Override
    public void onLoadSuccess() {
    ／／加载成功的时候，根据视频的长度，设置进度条最大值
      Log.i(TAG, "Successfully loaded video " + videoWidgetView.getDuration());
      loadVideoStatus = LOAD_VIDEO_STATUS_SUCCESS;
      seekBar.setMax((int) videoWidgetView.getDuration());
      updateStatusText();
    }

    /**
     * Called by video widget on the UI thread on any asynchronous error.
     */
    @Override
    public void onLoadError(String errorMessage) {
      // An error here is normally due to being unable to decode the video format.
      loadVideoStatus = LOAD_VIDEO_STATUS_ERROR;
      Toast.makeText(
          SimpleVrVideoActivity.this, "Error loading video: " + errorMessage, Toast.LENGTH_LONG)
          .show();
      Log.e(TAG, "Error loading video: " + errorMessage);
    }

    @Override
    public void onClick() {
      togglePause();
    }

    /**
     * Update the UI every frame.每一帧更新进度条
     */
    @Override
    public void onNewFrame() {
      updateStatusText();
      seekBar.setProgress((int) videoWidgetView.getCurrentPosition());
    }

    /**
     * Make the video play in a loop. This method could also be used to move to the next video in
     * a playlist.
     */
    @Override
    public void onCompletion() {
      videoWidgetView.seekTo(0);
    }
  }
```



加载视频的线程

```
/**
   * Helper class to manage threading.
   */
  class VideoLoaderTask extends AsyncTask<Pair<Uri, Options>, Void, Boolean> {
    @Override
    protected Boolean doInBackground(Pair<Uri, Options>... fileInformation) {
      try {
         if (fileInformation == null || fileInformation.length < 1
          || fileInformation[0] == null || fileInformation[0].first == null) {
          // No intent was specified, so we default to playing the local stereo-over-under video.
          Options options = new Options();
          options.inputType = Options.TYPE_STEREO_OVER_UNDER;
          videoWidgetView.loadVideoFromAsset("congo.mp4", options);直接根据文件名加载
         } else {
          videoWidgetView.loadVideo(fileInformation[0].first, fileInformation[0].second);
        }
      } catch (IOException e) {
        // An error here is normally due to being unable to locate the file.
        loadVideoStatus = LOAD_VIDEO_STATUS_ERROR;
        // Since this is a background thread, we need to switch to the main thread to show a toast.
        videoWidgetView.post(new Runnable() {
          @Override
          public void run() {
            Toast
                .makeText(SimpleVrVideoActivity.this, "Error opening file. ", Toast.LENGTH_LONG)
                .show();
          }
        });
        Log.e(TAG, "Could not open video: " + e);
      }

      return true;
    }
  }
```



## 总结

这两个应用主要展示 VrPanoramaView 和VrVideoView的基本用法



## 番外示例video360和videoplayer





### sdk-video360 	

video360要求设备最低版本为android 24，android N 7.0以上

可以渲染全景视频的播放器。 根据intent中的文件uri来进行进行视频播放。应用中定义了两个activity，一个只能在2d的launcher中显示图标，另外一个只能在专属的Daydream Home中显示图标

```
 <!-- This is a 2D Version of the video player. -->
    <activity
        android:name=".VideoActivity"
        android:icon="@drawable/icon"
        android:label="@string/app_name"
        android:launchMode="singleTask"
        android:screenOrientation="portrait">
      <!-- This Activity only shows up in the 2D launcher.  -->
      <intent-filter>
          <action android:name="android.intent.action.MAIN" />
          <category android:name="android.intent.category.LAUNCHER" />
      </intent-filter>
    </activity>

    <!-- This is a standard Daydream Activity.  -->
    <activity
        android:name=".VrVideoActivity"
        android:configChanges="orientation|keyboardHidden|screenSize|uiMode|navigation"
        android:enableVrMode="@string/gvr_vr_mode_component"
        android:label="@string/app_name"
        android:launchMode="singleTask"
        android:resizeableActivity="false"
        android:screenOrientation="landscape"
        android:theme="@style/VrActivityTheme">

      <!-- The VR icon to be used in Daydream Home comes in two parts: a foreground icon and a
           background icon. The foreground icon is also used by the 2D Activity. -->
      <meta-data
          android:name="com.google.android.vr.icon"
          android:resource="@drawable/icon" />
      <meta-data
          android:name="com.google.android.vr.icon_background"
          android:resource="@drawable/vr_icon_background" />

      <!-- This Activity only shows up in Daydream Home and not the 2D Launcher. -->
      <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="com.google.intent.category.DAYDREAM"/>
      </intent-filter>
    </activity>
```

其中activity的大部分代码都是和权限相关的，真正的视频解析加载在MediaLoader中，从类的说明中可以看到，可以通过adb命令来传递intent来打开视频：

```
 <p>Example intents compatible with adb are:
 *   <ul>
 *     <li>
 *       A top-bottom stereo image in the VR Activity.
 *       <b>adb shell am start -a android.intent.action.VIEW  \
 *          -n com.google.vr.sdk.samples.video360/.VrVideoActivity \
 *          -d "file:///sdcard/IMAGE.JPG" \
 *          --ei stereoFormat 2
 *       </b>
 *     </li>
 *     <li>
 *       A monoscopic video in the 2D Activity.
 *       <b>adb shell am start -a android.intent.action.VIEW  \
 *          -n com.google.vr.sdk.samples.video360/.VideoActivity \
 *          -d "file:///sdcard/VIDEO.MP4" \
 *          --ei stereoFormat 0
 *       </b>
 *     </li>
 *   </ul>
 *
 * <p>This sample does not validiate that a given file is readable by the Android media decoders.
 * You should validate that the file plays on your target devices via
 * <b>adb shell am start -a android.intent.action.VIEW -t video/mpeg -d "file:///VIDEO.MP4"</b>
```

我们从YouTube上下载了一个monoscopic的视频，放在sdcard的目录下，通过运行命令可播放对应的全景视频

```
F:\09androidsdk>adb shell am start -a android.intent.action.VIEW -n com.google.v
r.sdk.samples.video360/.VideoActivity -d "file://storage/sdcard/monoscopicVideo.
mp4" --ei stereoFormat 0
Starting: Intent { act=android.intent.action.VIEW dat=file://storage/sdcard/mono
scopicVideo.mp4 cmp=com.google.vr.sdk.samples.video360/.VideoActivity (has extra
s) }
```

该命令启用的是VideoActivity来播放全景视频，参数stereoFormat 0表示，视频类型为单个摄像头帧拍摄的全景视频。1表示用左右眼的方式渲染的，如果在非vr的显示中显示的话，则只显示左眼部分。2表示上下部分分割的左右眼渲染方式，如果在非vr显示中显示的话，只显示上部分内容

```
/** Standard media where a single camera frame takes up the entire media frame. */
  public static final int MEDIA_MONOSCOPIC = 0;
  /**
   * Stereo media where the left & right halves of the frame are rendered for the left & right eyes,
   * respectively. If the stereo media is rendered in a non-VR display, only the left half is used.
   */
  public static final int MEDIA_STEREO_LEFT_RIGHT = 1;
  /**
   * Stereo media where the top & bottom halves of the frame are rendered for the left & right eyes,
   * respectively. If the stereo media is rendered in a non-VR display, only the top half is used.
   */
  public static final int MEDIA_STEREO_TOP_BOTTOM = 2;
```

播放器demo在8996上 android6.0无法运行。

**在8953上可以运行播放3d stereo视频，视频播放不正常，叠加不正确。**

但是切换成cardboard模式的点击 眼镜 标示的图标按钮后，切换到VrVideoActivity提示：

> 手机不兼容：手机必须与Daydream兼容，请访问Daydream帮助中心了解详情。

在moto 	z和pixel等兼容手机上可正常运行，实现左右眼形变播放，中间会弹出和daydream控制器的配对的等。



对应配合Daydream viewer应该就可以看到立体视频效果。

*TODO: 如何与Daydream平台进行兼容，需要做哪些工作，目前官网上列出的课兼容手机有华为mate，三星galaxy note8，pixel，moto z等手机。*

### sdk-videoplayer 	

通过 Asynchronous Reprojection Video Surface API来播放视频，也是一个播放器。

在5516上，运行出错，暂时不研究了



