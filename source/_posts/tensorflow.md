---
title: "TensorFlow Android walkthrough"
date: "2018-05-31"
categories: 
    - "TensorFlow"
---
【链接】tensorflow/tensorflow
https://github.com/tensorflow/tensorflow/tree/master/tensorflow/examples/android



# 用pip 安装tensor flow

```
https://storage.googleapis.com/tensorflow/mac/cpu/tensorflow-1.0.0-py3-none-any.whl
```

```
pip3 install --upgrade https://storage.googleapis.com/tensorflow/mac/tensorflow-0.6.0-py3-none-any.whl

 Downloading https://storage.googleapis.com/tensorflow/mac/tensorflow-0.6.0-py3-none-any.whl (10.2MB)
    100% |████████████████████████████████| 10.3MB 1.9MB/s 
Collecting numpy>=1.8.2 (from tensorflow==0.6.0)
  Downloading https://files.pythonhosted.org/packages/8e/75/7a8b7e3c073562563473f2a61bd53e75d0a1f5e2047e576ee61d44113c22/numpy-1.14.3-cp36-cp36m-macosx_10_6_intel.macosx_10_9_intel.macosx_10_9_x86_64.macosx_10_10_intel.macosx_10_10_x86_64.whl (4.7MB)
    100% |████████████████████████████████| 4.7MB 832kB/s 
Collecting protobuf==3.0.0a3 (from tensorflow==0.6.0)
  Downloading https://files.pythonhosted.org/packages/d7/92/34c5810fa05e98082d141048110db97d2f98d318fa96f8202bf146ab79de/protobuf-3.0.0a3.tar.gz (88kB)
    100% |████████████████████████████████| 92kB 18.6MB/s 
Requirement not upgraded as not directly required: wheel>=0.26 in ./venv/lib/python3.6/site-packages (from tensorflow==0.6.0) (0.31.1)
Collecting six>=1.10.0 (from tensorflow==0.6.0)
  Downloading https://files.pythonhosted.org/packages/67/4b/141a581104b1f6397bfa78ac9d43d8ad29a7ca43ea90a2d863fe3056e86a/six-1.11.0-py2.py3-none-any.whl
Requirement not upgraded as not directly required: setuptools in ./venv/lib/python3.6/site-packages (from protobuf==3.0.0a3->tensorflow==0.6.0) (39.1.0)
Building wheels for collected packages: protobuf
  Running setup.py bdist_wheel for protobuf ... done
  Stored in directory: /Users/zowee-laisc/Library/Caches/pip/wheels/07/0a/98/ca8fbec7368a85849700304bf0cf40d2d8e183f9a5dd136795
Successfully built protobuf
Installing collected packages: numpy, protobuf, six, tensorflow
Successfully installed numpy-1.14.3 protobuf-3.0.0a3 six-1.11.0 tensorflow-0.6.0
(venv) zowee-laiscdeMacBook-Pro:tensorflow zowee-laisc$ 
```



就这么简单，然后测试环境ok

```
(venv) zowee-laiscdeMacBook-Pro:tensorflow zowee-laisc$ python
Python 3.6.4 (v3.6.4:d48ecebad5, Dec 18 2017, 21:07:28) 
[GCC 4.2.1 (Apple Inc. build 5666) (dot 3)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow as tf
>>> hello = tf.constant('hello tensorflow')
>>> sess = tf.Session()
I tensorflow/core/common_runtime/local_device.cc:40] Local device intra op parallelism threads: 4
I tensorflow/core/common_runtime/direct_session.cc:58] Direct session inter op parallelism threads: 4
>>> print(sess.run(hello))
b'hello tensorflow'
>>> a = tf.constant(10)
>>> b = tf.constant(32)
>>> print(sess.run(a+b))
42
>>> exit();
```

python3运行起来有些问题，后面把环境换成python2.7了，

Pycharm 配置python版本为 2.7，然后激活虚拟环境即可。





# android 的环境



## 下载GitHub 源码

```
git clone --recurse-submodules https://github.com/tensorflow/tensorflow.git
```

将example下的Android目录作为项目根目录，用android studio直接打开就可以了

将buildsystem改为 cmake，编译。





## Android demo walkthrough

### 归类识别classifier



ClassifierActivity 用于识别分类的，继承了基础类CameraActivity

```
public class ClassifierActivity extends CameraActivity implements OnImageAvailableListener {
```



其中CameraActivity封装了一些camera的操作

先查看下CameraActivity的代码

在onCreate的时候

```
  @Override
  protected void onCreate(final Bundle savedInstanceState) {
    LOGGER.d("onCreate " + this);
    super.onCreate(null);
    //设置屏幕常亮，只要给该window对用户可见的时候，保持屏幕亮屏
    getWindow().addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON);

    setContentView(R.layout.activity_camera);
    if (hasPermission()) {
      setFragment();
    } else {
      requestPermission();
    }
  }
```



其中setFragment()

```
protected void setFragment() {
    String cameraId = chooseCamera();//选择合适的camera
    if (cameraId == null) {
      Toast.makeText(this, "No Camera Detected", Toast.LENGTH_SHORT).show();
      finish();
    }

    Fragment fragment;
    if (useCamera2API) {//true 初始化fragment
      CameraConnectionFragment camera2Fragment =
          CameraConnectionFragment.newInstance(
              new CameraConnectionFragment.ConnectionCallback() {
                @Override
                public void onPreviewSizeChosen(final Size size, final int rotation) {
                  Log.i("linlian","useCamera2API onPreviewSizeChosen=");
                  previewHeight = size.getHeight();
                  previewWidth = size.getWidth();
                  CameraActivity.this.onPreviewSizeChosen(size, rotation);
                }
              },
              this,
              getLayoutId(),
              getDesiredPreviewFrameSize());

      camera2Fragment.setCamera(cameraId);
      fragment = camera2Fragment;
    } else {
      fragment =
          new LegacyCameraConnectionFragment(this, getLayoutId(), getDesiredPreviewFrameSize());
    }

   //将合适的fragment添加到Activity中
    getFragmentManager()
        .beginTransaction()
        .replace(R.id.container, fragment)
        .commit();
  }
```



其中CameraConnectionFragment，对应布局为`protected int getLayoutId() {  return R.layout.camera_connection_fragment;}`



```
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <org.tensorflow.demo.AutoFitTextureView
        android:id="@+id/texture"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentBottom="true" /> 

    <org.tensorflow.demo.RecognitionScoreView
        android:id="@+id/results"
        android:layout_width="match_parent"
        android:layout_height="112dp"
        android:layout_alignParentTop="true" /> 用于显示类别 在顶部

    <org.tensorflow.demo.OverlayView
        android:id="@+id/debug_overlay"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_alignParentBottom="true" />

</RelativeLayout>

```

RecognitionScoreView 自定义控件用于显示识别结果`canvas.drawText(recog.getTitle() + ": " + recog.getConfidence(), x, y, fgPaint);`

OverlayView 简单的提供回调的view，例如调试的时候，可用于显示调试中的图像等。

AutoFitTextureView 继承 TextureView

TextureView可用于显示流内容，可以是视频或者OpenGL场景。

surfaceview 窗口的工作方式是创建一个置于应用窗口之后的新窗口，效率高，因为刷新新窗口的时候，不需要重新绘制应用程序的窗口，但是surfaceview不在应用窗口上，所以不能使用view.setAlpha()之类的变换。也很难放在list view或者scrollview中。

Textureview在Android4.0引入，来解决上述问题，textureview必须在硬件加速器中开启。

AutoFitTextureView 在Textureview的基础上添加的长宽适应的功能。

Textureview主要用法设置 `TextureView.SurfaceTextureListener` 

```
/**
   * {@link android.view.TextureView.SurfaceTextureListener} handles several lifecycle events on a
   * {@link TextureView}.
   */
  private final TextureView.SurfaceTextureListener surfaceTextureListener =
      new TextureView.SurfaceTextureListener() {
        @Override
        public void onSurfaceTextureAvailable(//初始化
            final SurfaceTexture texture, final int width, final int height) {
          openCamera(width, height);
        }

        @Override
        public void onSurfaceTextureSizeChanged(//size 变化时候
            final SurfaceTexture texture, final int width, final int height) {
          configureTransform(width, height);
        }

        @Override
        public boolean onSurfaceTextureDestroyed(final SurfaceTexture texture) {
          return true;
        }

        @Override
        public void onSurfaceTextureUpdated(final SurfaceTexture texture) {}
      };
```



打开摄像头，配置最合适的preview size

```
 /**
   * Opens the camera specified by {@link CameraConnectionFragment#cameraId}.
   */
  private void openCamera(final int width, final int height) {
    setUpCameraOutputs();//设置预览大小
    configureTransform(width, height);//旋转偏移量
    final Activity activity = getActivity();
    final CameraManager manager = (CameraManager) activity.getSystemService(Context.CAMERA_SERVICE);
    try {
      if (!cameraOpenCloseLock.tryAcquire(2500, TimeUnit.MILLISECONDS)) {
        throw new RuntimeException("Time out waiting to lock camera opening.");
      }
      manager.openCamera(cameraId, stateCallback, backgroundHandler);//打开camera
    } catch (final CameraAccessException e) {
      LOGGER.e(e, "Exception!");
    } catch (final InterruptedException e) {
      throw new RuntimeException("Interrupted while trying to lock camera opening.", e);
    }
  }
```

stateCallback

```
 /**
   * {@link android.hardware.camera2.CameraDevice.StateCallback}
   * is called when {@link CameraDevice} changes its state.
   */
  private final CameraDevice.StateCallback stateCallback =
      new CameraDevice.StateCallback() {
        @Override
        public void onOpened(final CameraDevice cd) {
          // This method is called when the camera is opened.  We start camera preview here.
          cameraOpenCloseLock.release();
          cameraDevice = cd;
          createCameraPreviewSession();
        }

        @Override
        public void onDisconnected(final CameraDevice cd) {
          cameraOpenCloseLock.release();
          cd.close();
          cameraDevice = null;
        }

        @Override
        public void onError(final CameraDevice cd, final int error) {
          cameraOpenCloseLock.release();
          cd.close();
          cameraDevice = null;
          final Activity activity = getActivity();
          if (null != activity) {
            activity.finish();
          }
        }
      };

```

backgroundThread

```
 /**
   * Starts a background thread and its {@link Handler}.
   */
  private void startBackgroundThread() {//在onresume的时候被调用
    backgroundThread = new HandlerThread("ImageListener");
    backgroundThread.start();
    backgroundHandler = new Handler(backgroundThread.getLooper());
  }
```



其中 在camera onOpened() 的时候调用 createCameraPreviewSession

```
 /**
   * Creates a new {@link CameraCaptureSession} for camera preview.
   */
  private void createCameraPreviewSession() {
    try {
      final SurfaceTexture texture = textureView.getSurfaceTexture();
      assert texture != null;

      // We configure the size of default buffer to be the size of camera preview we want.
      texture.setDefaultBufferSize(previewSize.getWidth(), previewSize.getHeight());

      // This is the output Surface we need to start preview.
      final Surface surface = new Surface(texture);

      // We set up a CaptureRequest.Builder with the output Surface.
      previewRequestBuilder = cameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW);
      previewRequestBuilder.addTarget(surface);

      LOGGER.i("Opening camera preview: " + previewSize.getWidth() + "x" + previewSize.getHeight());

      // Create the reader for the preview frames.
      previewReader =
          ImageReader.newInstance(
              previewSize.getWidth(), previewSize.getHeight(), ImageFormat.YUV_420_888, 2);

      previewReader.setOnImageAvailableListener(imageListener, backgroundHandler);
      previewRequestBuilder.addTarget(previewReader.getSurface());

      // Here, we create a CameraCaptureSession for camera preview.
      cameraDevice.createCaptureSession(
          Arrays.asList(surface, previewReader.getSurface()),
          new CameraCaptureSession.StateCallback() {

            @Override
            public void onConfigured(final CameraCaptureSession cameraCaptureSession) {
              // The camera is already closed
              if (null == cameraDevice) {
                return;
              }

              // When the session is ready, we start displaying the preview.
              captureSession = cameraCaptureSession;
              try {
                // Auto focus should be continuous for camera preview.
                previewRequestBuilder.set(
                    CaptureRequest.CONTROL_AF_MODE,
                    CaptureRequest.CONTROL_AF_MODE_CONTINUOUS_PICTURE);
                // Flash is automatically enabled when necessary.
                previewRequestBuilder.set(
                    CaptureRequest.CONTROL_AE_MODE, CaptureRequest.CONTROL_AE_MODE_ON_AUTO_FLASH);

                // Finally, we start displaying the camera preview.
                previewRequest = previewRequestBuilder.build();
                captureSession.setRepeatingRequest(
                    previewRequest, captureCallback, backgroundHandler);
              } catch (final CameraAccessException e) {
                LOGGER.e(e, "Exception!");
              }
            }

            @Override
            public void onConfigureFailed(final CameraCaptureSession cameraCaptureSession) {
              showToast("Failed");
            }
          },
          null);
    } catch (final CameraAccessException e) {
      LOGGER.e(e, "Exception!");
    }
  }
```



以上只是一些camera的操作和预览大小的设置

实际与识别相关的

```
/**
   * Callback for android.hardware.Camera API
   */
  @Override
  public void onPreviewFrame(final byte[] bytes, final Camera camera) {
    Log.i("linlian","CameraActivity.onPreviewFrame()");
    if (isProcessingFrame) {
      LOGGER.w("Dropping frame!");//如果正在处理，则丢掉这一frame
      return;
    }

    try {
      // Initialize the storage bitmaps once when the resolution is known.
      if (rgbBytes == null) {
        Camera.Size previewSize = camera.getParameters().getPreviewSize();
        previewHeight = previewSize.height;
        previewWidth = previewSize.width;
        rgbBytes = new int[previewWidth * previewHeight];//初始化 rgbBytes
        onPreviewSizeChosen(new Size(previewSize.width, previewSize.height), 90);
      }
    } catch (final Exception e) {
      LOGGER.e(e, "Exception!");
      return;
    }

    isProcessingFrame = true;
    lastPreviewFrame = bytes;
    yuvBytes[0] = bytes;
    yRowStride = previewWidth;

    imageConverter =
        new Runnable() {
          @Override
          public void run() {//最终调用native方法实现转化
            ImageUtils.convertYUV420SPToARGB8888(bytes, previewWidth, previewHeight, rgbBytes);
          }
        };

    postInferenceCallback =
        new Runnable() {
          @Override
          public void run() {
            camera.addCallbackBuffer(bytes);
            isProcessingFrame = false;
          }
        };
    processImage();//在子类实现图片处理
  }

```



ClassifierActivity 

```
 @Override
  protected void processImage() {
    rgbFrameBitmap.setPixels(getRgbBytes(), 0, previewWidth, 0, 0, previewWidth, previewHeight);
    final Canvas canvas = new Canvas(croppedBitmap);

    canvas.drawBitmap(rgbFrameBitmap, frameToCropTransform, null);

    // For examining the actual TF input.
    if (SAVE_PREVIEW_BITMAP) {
      ImageUtils.saveBitmap(croppedBitmap);
    }
    
    //将原始图片进行剪切处理成需要的尺寸croppedBitmap
    
    runInBackground(
        new Runnable() {
          @Override
          public void run() {
            final long startTime = SystemClock.uptimeMillis();
            //进行识别
            final List<Classifier.Recognition> results = classifier.recognizeImage(croppedBitmap);

            lastProcessingTimeMs = SystemClock.uptimeMillis() - startTime;
            LOGGER.i("Detect: %s", results);//[[838] pot (42.3%), [322] pineapple (12.4%)]
            cropCopyBitmap = Bitmap.createBitmap(croppedBitmap);
            if (resultsView == null) {
              resultsView = (ResultsView) findViewById(R.id.results);
            }
            resultsView.setResults(results);
            requestRender();
            readyForNextImage();
          }
        });
  } 
```



` final List<Classifier.Recognition> results = classifier.recognizeImage(croppedBitmap);`

详细i 看下TensorFlowImageClassifier

```

    classifier =
        TensorFlowImageClassifier.create(
            getAssets(),
            MODEL_FILE,
            LABEL_FILE,
            INPUT_SIZE,
            IMAGE_MEAN,
            IMAGE_STD,
            INPUT_NAME,
            OUTPUT_NAME);
```



其中的几个参赛和所用的模型文件有关

```
  private static final int INPUT_SIZE = 224;
  private static final int IMAGE_MEAN = 117;
  private static final float IMAGE_STD = 1;
  private static final String INPUT_NAME = "input";
  private static final String OUTPUT_NAME = "output";
```



主要是在 TensorFlowImageClassifier

我们有两个文件，一个是模型文件pb结尾的，一个是标签文件，txt结尾的，TensorFlowImageClassifier需要读取模型和标签文件，然后在使用模型去识别处理新的文件

```
private static final String MODEL_FILE = "file:///android_asset/tensorflow_inception_graph.pb";
private static final String LABEL_FILE =
    "file:///android_asset/imagenet_comp_graph_label_strings.txt";
```



读取标签文件并且添加到列表中标签列表 `private Vector<String> labels = new Vector<String>();`

```
String actualFilename = labelFilename.split("file:///android_asset/")[1];
    Log.i(TAG, "Reading labels from: " + actualFilename);
    BufferedReader br = null;
    try {
      br = new BufferedReader(new InputStreamReader(assetManager.open(actualFilename)));
      String line;
      while ((line = br.readLine()) != null) {
        c.labels.add(line);
      }
      br.close();
    } catch (IOException e) {
      throw new RuntimeException("Problem reading label file!" , e);
    }
```

加载模型，查看 TensorFlowInferenceInterface内部代码，大概就是加载文件后，该文件是一个byte[] graphDef ，图标定义的学习模型，通过this.loadGraph(graphDef, this.g);得到Graph对象

```
c.inferenceInterface = new TensorFlowInferenceInterface(assetManager, modelFilename);
```



完成以上两步后，只要对图片进行合理的处理后，作为输入，就可以得到tensor flow模型给出的识别结果了

输入输出的一些数据定义则需要深刻理解模型的定义。The shape of the output is [N, NUM_CLASSES], where N is the batch size.

```
    c.outputNames = new String[] {outputName};//输出
    c.intValues = new int[inputSize * inputSize];//输入是一组大小为inputSize * inputSize的int数据，需要将图片信息转化为这种数据
    c.floatValues = new float[inputSize * inputSize * 3];
    c.outputs = new float[numClasses];
```



识别处理

```
 @Override
  public List<Recognition> recognizeImage(final Bitmap bitmap) {
      Log.i("linlian","recognizeImage");
    // Log this method so that it can be analyzed with systrace.
    Trace.beginSection("recognizeImage");

    Trace.beginSection("preprocessBitmap");
    // Preprocess the image data from 0-255 int to normalized float based
    // on the provided parameters.
    bitmap.getPixels(intValues, 0, bitmap.getWidth(), 0, 0, bitmap.getWidth(), bitmap.getHeight());
      Log.i("linlian","recognizeImage intValues.length="+intValues.length);
    for (int i = 0; i < intValues.length; ++i) {
      final int val = intValues[i];
      floatValues[i * 3 + 0] = (((val >> 16) & 0xFF) - imageMean) / imageStd;
      floatValues[i * 3 + 1] = (((val >> 8) & 0xFF) - imageMean) / imageStd;
      floatValues[i * 3 + 2] = ((val & 0xFF) - imageMean) / imageStd;
      //Log.i("linlian"," i="+i+"  "+floatValues[i * 3 + 0]+" "+floatValues[i * 3 + 0]+"  "+floatValues[i * 3 + 0]);
    }
    Trace.endSection();

    // Copy the input data into TensorFlow.	输入
    Trace.beginSection("feed");
    inferenceInterface.feed(inputName, floatValues, 1, inputSize, inputSize, 3);
    Trace.endSection();

    // Run the inference call.运行
    Trace.beginSection("run");
    inferenceInterface.run(outputNames, logStats);
    Trace.endSection();

    // Copy the output Tensor back into the output array.
    Trace.beginSection("fetch");输出
    inferenceInterface.fetch(outputName, outputs);
    Trace.endSection();

    // Find the best classifications.
    PriorityQueue<Recognition> pq =
        new PriorityQueue<Recognition>(
            3,
            new Comparator<Recognition>() {
              @Override
              public int compare(Recognition lhs, Recognition rhs) {
                // Intentionally reversed to put high confidence at the head of the queue.
                return Float.compare(rhs.getConfidence(), lhs.getConfidence());
              }
            });
    for (int i = 0; i < outputs.length; ++i) {
      if (outputs[i] > THRESHOLD) {
        pq.add(
            new Recognition(
                "" + i, labels.size() > i ? labels.get(i) : "unknown", outputs[i], null));
      }
    }
    final ArrayList<Recognition> recognitions = new ArrayList<Recognition>();
    int recognitionsSize = Math.min(pq.size(), MAX_RESULTS);
    for (int i = 0; i < recognitionsSize; ++i) {
      recognitions.add(pq.poll());
    }
    Trace.endSection(); // "recognizeImage"
    return recognitions;
  }
```



### 检测 detector

关于检测

加载的模型是由三种 可选 multi box，使用旧的API训练的模型 

```
  private static final String MB_INPUT_NAME = "ResizeBilinear";
  private static final String MB_OUTPUT_LOCATIONS_NAME = "output_locations/Reshape";
  private static final String MB_OUTPUT_SCORES_NAME = "output_scores/Reshape";
  private static final String MB_MODEL_FILE = "file:///android_asset/multibox_model.pb";
  private static final String MB_LOCATION_FILE =
      "file:///android_asset/multibox_location_priors.txt";
```

另外一部分 tensor flow object detect

```
 private static final int TF_OD_API_INPUT_SIZE = 300;
  private static final String TF_OD_API_MODEL_FILE =
      "file:///android_asset/ssd_mobilenet_v1_android_export.pb";
  private static final String TF_OD_API_LABELS_FILE = "file:///android_asset/coco_labels_list.txt";

```

还有yolo??

```

  // Configuration values for tiny-yolo-voc. Note that the graph is not included with TensorFlow and
  // must be manually placed in the assets/ directory by the user.
  // Graphs and models downloaded from http://pjreddie.com/darknet/yolo/ may be converted e.g. via
  // DarkFlow (https://github.com/thtrieu/darkflow). Sample command:
  // ./flow --model cfg/tiny-yolo-voc.cfg --load bin/tiny-yolo-voc.weights --savepb --verbalise
  private static final String YOLO_MODEL_FILE = "file:///android_asset/graph-tiny-yolo-voc.pb";
  private static final int YOLO_INPUT_SIZE = 416;
  private static final String YOLO_INPUT_NAME = "input";
  private static final String YOLO_OUTPUT_NAMES = "output";
  private static final int YOLO_BLOCK_SIZE = 32;
  
```

> yolo 是一个实时物体识别的
>
> You only look once (YOLO) is a state-of-the-art, real-time object detection system. On a Pascal Titan X it processes images at 30 FPS and has a mAP of 57.9% on COCO test-dev.



主要看下tensorflow TF_OD 相关的

coco_labels_list.txt 标签文件，人啊，自行车啊，识别

ssd_mobilenet_v1_android_export.pb 模型文件

```
tracker = new MultiBoxTracker(this);
detector = TensorFlowObjectDetectionAPIModel.create(
            getAssets(), TF_OD_API_MODEL_FILE, TF_OD_API_LABELS_FILE, TF_OD_API_INPUT_SIZE);
        cropSize = TF_OD_API_INPUT_SIZE;
```



模型的读取和输入输出定义 TensorFlowObjectDetectionAPIModel

```
public static Classifier create(
      final AssetManager assetManager,
      final String modelFilename,
      final String labelFilename,
      final int inputSize) throws IOException {
    final TensorFlowObjectDetectionAPIModel d = new TensorFlowObjectDetectionAPIModel();

    InputStream labelsInput = null;
    String actualFilename = labelFilename.split("file:///android_asset/")[1];
    labelsInput = assetManager.open(actualFilename);
    BufferedReader br = null;
    br = new BufferedReader(new InputStreamReader(labelsInput));
    String line;
    while ((line = br.readLine()) != null) {
      LOGGER.w(line);
      d.labels.add(line);//逐行读取标签文件
    }
    br.close();


    d.inferenceInterface = new TensorFlowInferenceInterface(assetManager, modelFilename);

    final Graph g = d.inferenceInterface.graph();

    d.inputName = "image_tensor";//输入的shap定义
    // The inputName node has a shape of [N, H, W, C], where
    // N is the batch size
    // H = W are the height and width
    // C is the number of channels (3 for our purposes - RGB)
    final Operation inputOp = g.operation(d.inputName);
    if (inputOp == null) {
      throw new RuntimeException("Failed to find input Node '" + d.inputName + "'");
    }
    d.inputSize = inputSize;
    // The outputScoresName node has a shape of [N, NumLocations], where N
    // is the batch size.  三个输出
    final Operation outputOp1 = g.operation("detection_scores");
    if (outputOp1 == null) {
      throw new RuntimeException("Failed to find output Node 'detection_scores'");
    }
    final Operation outputOp2 = g.operation("detection_boxes");
    if (outputOp2 == null) {
      throw new RuntimeException("Failed to find output Node 'detection_boxes'");
    }
    final Operation outputOp3 = g.operation("detection_classes");
    if (outputOp3 == null) {
      throw new RuntimeException("Failed to find output Node 'detection_classes'");
    }

    // Pre-allocate buffers.
    d.outputNames = new String[] {"detection_boxes", "detection_scores",
                                  "detection_classes", "num_detections"};
    d.intValues = new int[d.inputSize * d.inputSize];
    d.byteValues = new byte[d.inputSize * d.inputSize * 3];
    d.outputScores = new float[MAX_RESULTS];
    d.outputLocations = new float[MAX_RESULTS * 4];
    d.outputClasses = new float[MAX_RESULTS];
    d.outputNumDetections = new float[1];
    return d;
  }
```



识别

```
@Override
  public List<Recognition> recognizeImage(final Bitmap bitmap) {
    // Log this method so that it can be analyzed with systrace.
    Trace.beginSection("recognizeImage");

    Trace.beginSection("preprocessBitmap");
    // Preprocess the image data from 0-255 int to normalized float based
    // on the provided parameters.
    bitmap.getPixels(intValues, 0, bitmap.getWidth(), 0, 0, bitmap.getWidth(), bitmap.getHeight());

    for (int i = 0; i < intValues.length; ++i) {//图片数据处理
      byteValues[i * 3 + 2] = (byte) (intValues[i] & 0xFF);
      byteValues[i * 3 + 1] = (byte) ((intValues[i] >> 8) & 0xFF);
      byteValues[i * 3 + 0] = (byte) ((intValues[i] >> 16) & 0xFF);
    }
    Trace.endSection(); // preprocessBitmap

    // Copy the input data into TensorFlow.输入
    Trace.beginSection("feed");
    inferenceInterface.feed(inputName, byteValues, 1, inputSize, inputSize, 3);
    Trace.endSection();

    // Run the inference call.运行
    Trace.beginSection("run");
    inferenceInterface.run(outputNames, logStats);
    Trace.endSection();

    // Copy the output Tensor back into the output array.结果输出
    Trace.beginSection("fetch");
    outputLocations = new float[MAX_RESULTS * 4];
    outputScores = new float[MAX_RESULTS];
    outputClasses = new float[MAX_RESULTS];
    outputNumDetections = new float[1];
    inferenceInterface.fetch(outputNames[0], outputLocations);
    inferenceInterface.fetch(outputNames[1], outputScores);
    inferenceInterface.fetch(outputNames[2], outputClasses);
    inferenceInterface.fetch(outputNames[3], outputNumDetections);
    Trace.endSection();

    // Find the best detections.
    final PriorityQueue<Recognition> pq =
        new PriorityQueue<Recognition>(
            1,
            new Comparator<Recognition>() {
              @Override
              public int compare(final Recognition lhs, final Recognition rhs) {
                // Intentionally reversed to put high confidence at the head of the queue.
                return Float.compare(rhs.getConfidence(), lhs.getConfidence());
              }
            });

    // Scale them back to the input size.
    for (int i = 0; i < outputScores.length; ++i) {
      final RectF detection =
          new RectF(
              outputLocations[4 * i + 1] * inputSize,
              outputLocations[4 * i] * inputSize,
              outputLocations[4 * i + 3] * inputSize,
              outputLocations[4 * i + 2] * inputSize);
      pq.add(
          new Recognition("" + i, labels.get((int) outputClasses[i]), outputScores[i], detection));
    }

    final ArrayList<Recognition> recognitions = new ArrayList<Recognition>();
    for (int i = 0; i < Math.min(pq.size(), MAX_RESULTS); ++i) {
      recognitions.add(pq.poll());
    }
    Trace.endSection(); // "recognizeImage"
    return recognitions;
  }
```



# 训练模型

运行手写识别

只需要直接运行`fully_connected_feed.py`文件，就可以开始训练：

`python fully_connected_feed.py`



```
Traceback (most recent call last):

  File "fully_connected_feed.py", line 279, in <module>

    tf.app.run(main=main, argv=[sys.argv[0]] + unparsed)

TypeError: run() got an unexpected keyword argument 'main'


```

版本太低？

Pip3 install tensorflow==1.4.0   

```
(venv) zowee-laiscdeMacBook-Pro:tensorflow zowee-laisc$ pip3 install --upgrade tensorflow==1.6.0 
```



**TensorFlow Python API 依赖 Python 2.7 版本.**



yong pycharm 新建2.7环境的python项目

激活 

```
 source venv/bin/activate 
```

安装tensor flow

```
pip install --upgrade tensorflow==1.6.0
```



到下载的tensorflow

```
tensorflow/tensorflow/tensorflow/examples/tutorials/mnist
```

运行

```
(venv) zowee-laiscdeMacBook-Pro:mnist zowee-laisc$ python fully_connected_feed.py 
Successfully downloaded train-images-idx3-ubyte.gz 9912422 bytes.
Extracting /tmp/tensorflow/mnist/input_data/train-images-idx3-ubyte.gz
Successfully downloaded train-labels-idx1-ubyte.gz 28881 bytes.
Extracting /tmp/tensorflow/mnist/input_data/train-labels-idx1-ubyte.gz
Successfully downloaded t10k-images-idx3-ubyte.gz 1648877 bytes.
Extracting /tmp/tensorflow/mnist/input_data/t10k-images-idx3-ubyte.gz
Successfully downloaded t10k-labels-idx1-ubyte.gz 4542 bytes.
Extracting /tmp/tensorflow/mnist/input_data/t10k-labels-idx1-ubyte.gz
2018-05-22 16:15:26.976082: I tensorflow/core/platform/cpu_feature_guard.cc:140] Your CPU supports instructions that this TensorFlow binary was not compiled to use: AVX2 FMA
Step 0: loss = 2.32 (0.215 sec)
Step 100: loss = 2.13 (0.001 sec)
Step 200: loss = 1.96 (0.001 sec)
Step 300: loss = 1.70 (0.001 sec)
Step 400: loss = 1.31 (0.001 sec)
Step 500: loss = 1.11 (0.001 sec)
Step 600: loss = 0.89 (0.001 sec)
Step 700: loss = 0.87 (0.001 sec)
Step 800: loss = 0.70 (0.001 sec)
Step 900: loss = 0.64 (0.001 sec)
Training Data Eval:
Num examples: 55000  Num correct: 46676  Precision @ 1: 0.8487
Validation Data Eval:
Num examples: 5000  Num correct: 4264  Precision @ 1: 0.8528
Test Data Eval:
Num examples: 10000  Num correct: 8526  Precision @ 1: 0.8526
Step 1000: loss = 0.55 (0.012 sec)
Step 1100: loss = 0.58 (0.123 sec)
Step 1200: loss = 0.40 (0.001 sec)
Step 1300: loss = 0.49 (0.001 sec)
Step 1400: loss = 0.37 (0.001 sec)
Step 1500: loss = 0.70 (0.001 sec)
Step 1600: loss = 0.40 (0.001 sec)
Step 1700: loss = 0.24 (0.001 sec)
Step 1800: loss = 0.31 (0.001 sec)
Step 1900: loss = 0.39 (0.001 sec)
Training Data Eval:
Num examples: 55000  Num correct: 49053  Precision @ 1: 0.8919
Validation Data Eval:
Num examples: 5000  Num correct: 4509  Precision @ 1: 0.9018
Test Data Eval:
Num examples: 10000  Num correct: 8971  Precision @ 1: 0.8971
(venv) zowee-laiscdeMacBook-Pro:mnist zowee-laisc$ 
```



启动tensorboard

```
(venv) zowee-laiscdeMacBook-Pro:mnist zowee-laisc$ tensorboard --logdir=logs/
TensorBoard 1.6.0 at http://zowee-laiscdeMacBook-Pro.local:6006 (Press CTRL+C to quit)
```











