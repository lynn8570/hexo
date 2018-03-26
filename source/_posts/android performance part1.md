---
title: "Android performance par1: Render"
date: "2016-07-01"
categories: 
    - "ANDROID"
---
# 前言 #
这几篇文章是根据谷歌出的Android性能的专题视频的总结以及实验的笔记，这个视频主题从Render渲染、compute算法、memory算法、battery电池这几个方面来讲述如何对Android应用进行性能上的优化。主要是介绍了一些检测的工具，和优化的策略。

#Render#

CPU负责把UI组件计算成Polygons，Texture纹理，然后交给GPU进行栅格化渲染。
每次从CPU转移到GPU是一件很麻烦的事情，OpenGL ES可以把那些需要渲染的纹理Hold在GPU Memory里面，在下次需要渲染的时候直接进行操作
在Android里面那些由主题所提供的资源，例如Bitmaps，Drawables都是一起打包到统一的Texture纹理当中，然后再传递到 GPU里面
当然随着UI组件的越来越丰富，有了更多演变的形态。例如显示图 片的时候，需要先经过CPU的计算加载到内存中，然后传递给GPU进行渲染。文字的显示更加复杂，需要先经过CPU换算成纹理，然后再交给GPU进行渲 染，回到CPU绘制单个字符之后，再重新引用经过GPU渲染的内容。动画则是一个更加复杂的操作流程。
![](http://7xl98n.com1.z0.glb.clouddn.com/%E5%9B%BE%E7%89%871.png)

## 卡顿原因 ##


手机的刷新频率为60fps,如果当前需要处理的事情太多了，以至于无法再16ms内完成frame的刷新，就造成了丢帧，在视觉上，就给用户卡顿的感觉。如果此时用户正在进行一些交互，比如滑动屏幕、输入文字等，这种卡顿就会尤为明显。
![](http://7xl98n.com1.z0.glb.clouddn.com/%E5%9B%BE%E7%89%872.png)

另外，虚拟机在执行GC垃圾回收操作时所有线程（包括UI线程）都需要暂停，当GC垃圾回收完成之后所有线程才能够继续执行。所以如果此时有大量的GC操作，也会引起卡顿问题。


## UI性能优化策略 ##


1. CPU side: 不必要的layout和invalidations。过多的view hierarchy和不必要的重绘导致CPU过度的刷新displaylist和相关的GPU资源

2. GPU side: overdraw。

![](http://7xl98n.com1.z0.glb.clouddn.com/%E5%9B%BE%E7%89%873.png)
	
### Overdraw ###

Overdraw(过度绘制)描述的是屏幕上的某个像素在同一帧的时间内被绘制了多次
在多层次重叠的UI结构里面，如果不可见的UI也在做绘制的操作，会导致某些像素区域被绘制了多次。这样就会浪费大量的CPU以及GPU资源。

![](http://7xl98n.com1.z0.glb.clouddn.com/%E5%9B%BE%E7%89%874.png)

overdraw检测：在手机的开发者选项中，选择'调试GPU过度绘制'->'显示过度绘制区域'

我们的目标是，减少红色区域的面积大小

![](http://7xl98n.com1.z0.glb.clouddn.com/%E5%9B%BE%E7%89%875.png?imageView2/1/w/200/h/348/q/75)

![](http://7xl98n.com1.z0.glb.clouddn.com/%E5%9B%BE%E7%89%876.png)

示例一：

去除background

![](http://7xl98n.com1.z0.glb.clouddn.com/%E5%9B%BE%E7%89%877.png?imageView2/1/w/200/h/348/q/75)

对应代码，查看是否可以减少红色区域的面积

	<FrameLayout
            android:layout_width="fill_parent"
            android:layout_height="fill_parent"
            android:background="@color/white">

            <include layout="@layout/main_content" />

            <com.zowee.lib.widget.HeaderLayout
                xmlns:app="http://schemas.android.com/apk/res-auto"
                android:id="@+id/header_hl"
                android:layout_width="match_parent"
                android:layout_height="@dimen/title_height"
                android:background="@color/main_title_bg"
                app:hlNavigationIcon="@drawable/main_menu_btn"
                app:hlNavigationMinWidth="@dimen/title_height"
                app:hlNavigationScaleType="centerInside"
                app:hlSupportTranslucentStatus="true"
                app:hlTitleText="@string/main_title" />
        </FrameLayout>

发现代码中可能之前无意加的 'android:background="@color/white"',把背景去掉试试

![](http://7xl98n.com1.z0.glb.clouddn.com/%E5%9B%BE%E7%89%878.png?imageView2/1/w/200/h/348/q/75)

果然红色区域去掉了~~~~~~~~~~~~


官方示例:

    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        // Don't draw anything until all the Asynctasks are done and all the DroidCards are ready.
        if (mDroids.length > 0 && mDroidCards.size() == mDroids.length) {
            // Loop over all the droids, except the last one.
            int i;

            for (i = 0; i < mDroidCards.size()-1; i++) {
                // Each card is laid out a little to the right of the previous one.
                mCardLeft = i * mCardSpacing;
                // Save the canvas state
                canvas.save();

                // Restrict the drawing area to what is visible
                canvas.clipRect(mCardLeft, 0, mCardLeft+mCardSpacing, mDroidCards.get(i).getHeight());

                drawDroidCard(canvas, mDroidCards.get(i), mCardLeft, 0);

                // Restore canvas to non-clipping state
                canvas.restore();
            }
            // Draw the final card without clipping
            drawDroidCard(canvas, mDroidCards.get(i), mCardLeft + mCardSpacing, 0);
        }
        // Invalidate the whole view. Doing this calls onDraw() if the view is visible.
        invalidate();
    }

canvas.clipRect(mCardLeft, 0, mCardLeft+mCardSpacing, mDroidCards.get(i).getHeight()); 指明画布的可见区域。对比clipRect后，避免过度重绘的效果

![](http://7xl98n.com1.z0.glb.clouddn.com/QQ%E6%88%AA%E5%9B%BE20170710115604.png)


### Hierachy ###
Hierachy用于查看界面的层级结构。

![](http://7xl98n.com1.z0.glb.clouddn.com/hierarchy.gif)


尽量减少界面的层级结构，尽量使用include、merge、ViewStub标签，尽量不存在冗余嵌套及过于复杂布局，尽量使用GONE替换INVISIBLE，使用weight后尽量将width和heigh设置为0dp减少运算，Item存在非常复杂的嵌套时考虑使用自定义Item View来取代，减少measure与layout次数等。自定义View等绘图与布局优化；尽量避免在draw、measure、layout中做过于耗时及耗内存操作，尤其是draw方法中，尽量减少draw、measure、layout等执行次数。





### Lint ###


代码区点击右键->Analyze->Inspect Code–>界面选择你要检测的模块->点击确认开始检测

![](http://7xl98n.com1.z0.glb.clouddn.com/%E5%9B%BE%E7%89%8716.png)

检测结果：

![](http://7xl98n.com1.z0.glb.clouddn.com/lint-1.png)



## UI性能分析解决总结 ##



- 布局优化；尽量使用include、merge、ViewStub标签，尽量不存在冗余嵌套及过于复杂布局（譬如10层就会直接异常），尽量使用GONE替换INVISIBLE，使用weight后尽量将width和heigh设置为0dp减少运算，Item存在非常复杂的嵌套时考虑使用自定义Item View来取代，减少measure与layout次数等。
- 列表及Adapter优化；尽量复用getView方法中的相关View，不重复获取实例导致卡顿，列表尽量在滑动过程中不进行UI元素刷新等。
- 背景和图片等内存分配优化；尽量减少不必要的背景设置，图片尽量压缩处理显示，尽量避免频繁内存抖动等问题出现。
- 自定义View等绘图与布局优化；尽量避免在draw、measure、layout中做过于耗时及耗内存操作，尤其是draw方法中，尽量减少draw、measure、layout等执行次数。
- canvas.clipRect 限制draw的可见区域



