---
title: "svg路径动画"
date: "2018-04-13"
categories: 
    - "animation"
---
# Svg 路径动画效果

项目地址：[Github](https://github.com/geftimov/android-pathview)

动画效果：

![svg path动画](https://github.com/geftimov/android-pathview/blob/master/art/fill-after-resize-new.gif)

## 什么是svg

Scalable Vector Graphics (SVG)可缩放的矢量图形，应用几个定义[wiki](https://en.wikipedia.org/wiki/Scalable_Vector_Graphics)

> - SVG 指可伸缩矢量图形 (Scalable Vector Graphics)
> - SVG 用来定义用于网络的基于[矢量](https://baike.baidu.com/item/%E7%9F%A2%E9%87%8F/1400417)的图形
> - SVG 使用 XML 格式定义图形
> - SVG 图像在放大或改变尺寸的情况下其图形质量不会有所损失
> - SVG 是万维网[联盟](https://baike.baidu.com/item/%E8%81%94%E7%9B%9F/4799014)的标准
> - SVG 与诸如 [DOM](https://baike.baidu.com/item/DOM/50288)和 [XSL](https://baike.baidu.com/item/XSL) 之类的[W3C](https://baike.baidu.com/item/W3C)标准是一个整体



## PathView做了什么

通过XML方式获取svg资源id 

```
<com.eftimoff.androipathview.PathView
        android:id="@+id/pathView"
        android:layout_width="350dp"
        android:layout_height="350dp"
        app:svg="@raw/monitor"  //在res/raw文件夹中，
        app:pathColor="@android:color/white"
        app:pathWidth="2dp"/>
```

在onSizeChanged的时候加载svg文件

```
  @Override
    protected void onSizeChanged(final int w, final int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);

        if (mLoader != null) {
            try {
                mLoader.join();//必须在loader运行完毕后，再运行
            } catch (InterruptedException e) {
                Log.e(LOG_TAG, "Unexpected error", e);
            }
        }
        if (svgResourceId != 0) {
            mLoader = new Thread(new Runnable() {
                @Override
                public void run() {

                    svgUtils.load(getContext(), svgResourceId);通过svg包加载资源文件到 mSvg

                    synchronized (mSvgLock) {
                        width = w - getPaddingLeft() - getPaddingRight();
                        height = h - getPaddingTop() - getPaddingBottom();
                        paths = svgUtils.getPathsForViewport(width, height);
                        updatePathsPhaseLocked();
                    }
                }
            }, "SVG Loader");
            mLoader.start();
        }
    }
```

其中load主要是加载资源文件

```
    /**
     * Loading the svg from the resources.
     *
     * @param context     Context object to get the resources.
     * @param svgResource int resource id of the svg.
     */
    public void load(Context context, int svgResource) {
        if (mSvg != null) 
            return;
        try {
            mSvg = SVG.getFromResource(context, svgResource);//通过 com.caverock.androidsvg api加载
            mSvg.setDocumentPreserveAspectRatio(PreserveAspectRatio.UNSCALED);
        } catch (SVGParseException e) {
            Log.e(LOG_TAG, "Could not load specified SVG resource", e);
        }
    }
```

getPathsForViewport，将

```
    /**
     * Render the svg to canvas and catch all the paths while rendering.
     *
     * @param width  - the width to scale down the view to,
     * @param height - the height to scale down the view to,
     * @return All the paths from the svg.
     */
    public List<SvgPath> getPathsForViewport(final int width, final int height) {
        final float strokeWidth = mSourcePaint.getStrokeWidth();
        Canvas canvas = new Canvas() {
            private final Matrix mMatrix = new Matrix();

            @Override
            public int getWidth() {
                return width;
            }

            @Override
            public int getHeight() {
                return height;
            }

            @Override
            public void drawPath(Path path, Paint paint) {
                Path dst = new Path();

                //noinspection deprecation
                getMatrix(mMatrix);
                path.transform(mMatrix, dst);//将path中的点进行mMatrix变化后，并将最终的path写到dst path中
                paint.setAntiAlias(true);//抗锯齿
                paint.setStyle(Paint.Style.STROKE);
                paint.setStrokeWidth(strokeWidth);
                mPaths.add(new SvgPath(dst, paint));
            }
        };

        rescaleCanvas(width, height, strokeWidth, canvas);

        return mPaths;
    }
```

rescaleCanvas：

```
 /**
     * Rescale the canvas with specific width and height.
     *
     * @param width       The width of the canvas.
     * @param height      The height of the canvas.
     * @param strokeWidth Width of the path to add to scaling.
     * @param canvas      The canvas to be drawn.
     */
    private void rescaleCanvas(int width, int height, float strokeWidth, Canvas canvas) {
        if (mSvg == null) 
            return;
        final RectF viewBox = mSvg.getDocumentViewBox();

        final float scale = Math.min(width
                        / (viewBox.width() + strokeWidth),
                height / (viewBox.height() + strokeWidth));

        canvas.translate((width - viewBox.width() * scale) / 2.0f,
                (height - viewBox.height() * scale) / 2.0f);
        canvas.scale(scale, scale);

        mSvg.renderToCanvas(canvas);
    }
```

对于Matrix的学习扩展，请参阅[Matrix源码详解](https://blog.csdn.net/abcdef314159/article/details/52813313)

[matrix原理](https://github.com/GcsSloop/AndroidNote/blob/master/CustomView/Advance/[09]Matrix_Basic.md)



动画影响的属性是percentage

```
       /**
         * Default constructor.
         *
         * @param pathView The view that must be animated.
         */
        public AnimatorBuilder(final PathView pathView) {
            anim = ObjectAnimator.ofFloat(pathView, "percentage", 0.0f, 1.0f);
        }
```

通过影响percentage的值，从而影响

```
    /**
     * Animate this property. It is the percentage of the path that is drawn.
     * It must be [0,1].
     *
     * @param percentage float the percentage of the path.
     */
    public void setPercentage(float percentage) {
        if (percentage < 0.0f || percentage > 1.0f) {
            throw new IllegalArgumentException("setPercentage not between 0.0f and 1.0f");
        }
        progress = percentage;
        synchronized (mSvgLock) {
            updatePathsPhaseLocked();//更新path
        }
        invalidate();再重新绘制
    }
```

根据progress，更新svgPath

```
    /**
     * This refreshes the paths before draw and resize.
     */
    private void updatePathsPhaseLocked() {
        final int count = paths.size();
        for (int i = 0; i < count; i++) {
            SvgUtils.SvgPath svgPath = paths.get(i);
            svgPath.path.reset();
            svgPath.measure.getSegment(0.0f, svgPath.length * progress, svgPath.path, true);
            //Given a start and stop distance, return in dst the intervening segment(s). If the segment is zero-length, return false, else return true. startD and stopD are pinned to legal values (0..getLength()). If startD <= stopD then return false (and leave dst untouched). Begin the segment with a moveTo if startWithMoveTo is true
            // Required only for Android 4.4 and earlier
            svgPath.path.rLineTo(0.0f, 0.0f);
        }
    }
```

从而实现，path动画

