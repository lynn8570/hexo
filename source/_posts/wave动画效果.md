---
title: "wave动画效果"
date: "2018-04-13"
categories: 
    - "animation"
---
# Wave动画效果

项目地址：[Github](https://github.com/tangqi92/WaveLoadingView)

动画效果：

![image](https://camo.githubusercontent.com/8f83a9935a2b79ed9c63749a1b1b907e2cae850a/687474703a2f2f3778696b66632e636f6d312e7a302e676c622e636c6f7564646e2e636f6d2f776176656c6f6164696e67766965772e706e67)

实现波浪wave效果，以及水位位置调整。

## 代码实现

### 外部形状

外部形状可以是圆形，三角形，整个图形的绘制区域由canvas决定

初始化paint

```

    private void initBoarderPaint() {
        //默认的style是fill，填充的
        mBoarderPain = new Paint();
        mBoarderPain.setAntiAlias(true);
        mBoarderPain.setStyle(Paint.Style.STROKE);//描边
        mBoarderPain.setStrokeWidth(DEFAULT_BOARD_WIDTH);
        mBoarderPain.setColor(Color.RED);
    }

    private void initBgPaint() {
        //默认的style是fill，填充的
        mBgPaint = new Paint();
        mBgPaint.setAntiAlias(true);
        mBgPaint.setColor(getResources().getColor(R.color.voicebg));
    }
```

在onDraw中绘制

```
 @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        float boaderwidth = DEFAULT_BOARD_WIDTH;

        canvas.drawCircle(getWidth() / 2f, getHeight() / 2f, (getWidth() - boaderwidth) / 2f - 1f, mBoarderPain);//画边框
        canvas.drawCircle(getWidth() / 2f, getHeight() / 2f, getWidth() / 2f - boaderwidth, mBgPaint);//画圆形背景

    }
```



怎么画波浪呢？

初始化paint

```
 private void initWavePaint() {
        mWavePain = new Paint();
        mWavePain.setAntiAlias(true);
        mWavePain.setColor(getResources().getColor(R.color.voicefr));
        updateWaveShader();
    }
```

其中updateWaveShader，使用的着色器是BitmapShader。

```
 private void updateWaveShader() {
        if (getWaveBitmap() != null) {
            mWaveShader = new BitmapShader(getWaveBitmap(), Shader.TileMode.REPEAT, Shader.TileMode.CLAMP);//x坐标repeat模式，y方向上最后一个像素重复
            mWavePain.setShader(mWaveShader);
        } else {
            mWavePain.setShader(null);
        }
    
```

而getWaveBitmap()的任务是获得一个画有两个波浪的图形

```
    private Bitmap getWaveBitmap() {//返回一个波形的图案，两条线

        int wavecount = 1;//容纳多少个完整波形


        int width = getMeasuredWidth();
        int height = getMeasuredHeight();
        Log.i("linlian", "width=" + width + " height=" + height);
        if (width > 0 && height > 0) {

            Bitmap bitmap = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888);
            Canvas waveLineCanvas = new Canvas(bitmap);


            //坐标点数组，x从0-width，y=Asin(Wx+Q)+H
            // W = 2*PI/width
            // A = height
            // h = height
            // Q = 0
            final int endX = width + 1;
            final int endY = height + 1;
            float W = (float) (2f * Math.PI * wavecount / width);
            float A = height / 10f;//波浪幅度在整体的10分之一
            float H = height / 2;//默认水位在一半的位置
            float[] waveY = new float[endX];

            for (int x = 0; x < endX; x++) {
                waveY[x] = (float) (A * Math.sin(W * x)) + H;
            }

            int xShift = width / 4;
            mWavePaint.setColor(getResources().getColor(R.color.wavebg));
            for (int x = 0; x < endX; x++) {
                waveLineCanvas.drawLine(x, waveY[(x + xShift) % endX], x, endY, mWavePaint);// .:|:. 像这样画线
            }
            mWavePaint.setColor(getResources().getColor(R.color.wavefr));
            for (int x = 0; x < endX; x++) {

                waveLineCanvas.drawLine(x, waveY[x], x, endY, mWavePaint);// .:|:. 像这样画线
            }

            return bitmap;
        }
        return null;
    }
```



设置好图形着色器之后，在view的onDraw方法上，添加

```
 @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        float boaderwidth = DEFAULT_BOARD_WIDTH;

        canvas.drawCircle(getWidth() / 2f, getHeight() / 2f, (getWidth() - boaderwidth) / 2f - 1f, mBoarderPain);//画边框
        canvas.drawCircle(getWidth() / 2f, getHeight() / 2f, getWidth() / 2f - boaderwidth, mBgPaint);//画圆形背景


        canvas.drawCircle(getWidth() / 2f, getHeight() / 2f, getWidth() / 2f - boaderwidth, mWavePain);//画波浪图形
    }
```



### 如何实现动画效果呢

通过属性动画来实现，改变waveXshift的值，从0到1变化，两秒，重复变化

```
 private void initAnimation() {
        mShaderMatrix = new Matrix();
        ObjectAnimator waveXshiftAnimator = ObjectAnimator.ofFloat(this, "waveXshift", 0f, 1f); 
        waveXshiftAnimator.setRepeatCount(ValueAnimator.INFINITE);
        waveXshiftAnimator.setDuration(2000);
        waveXshiftAnimator.setInterpolator(new LinearInterpolator());
        mAnimatorset = new AnimatorSet();
        mAnimatorset.play(waveXshiftAnimator);

    }
```

属性变化的时候，重新绘制

```
    public void setWaveXshift(float mWaveXshift) {
        if (this.mWaveXshift != mWaveXshift) {
            this.mWaveXshift = mWaveXshift;
            //变化的是重新绘制view，实现动画效果
            invalidate();
        }
    }
```



在onDraw中，实现图像的水平移动 mWaveXshift * getWidth();

```
 if (mWaveShader != null) {
            if (mWavePaint.getShader() == null) {
                mWavePaint.setShader(mWaveShader);
            }
            float dx = mWaveXshift * getWidth();
            mShaderMatrix.setTranslate(dx, 0);//平移波浪，实现推进效果
            mWaveShader.setLocalMatrix(mShaderMatrix);
        } else {
            mWavePaint.setShader(null);
        }
        canvas.drawCircle(getWidth() / 2f, getHeight() / 2f, getWidth() / 2f - boaderwidth, mWavePaint);//画波浪
```

重新实现的代码可以参考

https://github.com/lynn8570/VoiceView/blob/master/animlib/src/main/java/anim/lynn/voice/VoiceWave.java

