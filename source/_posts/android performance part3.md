---
title: "Android performance par3: Memory"
date: "2016-08-18"
categories: 
    - "ANDROID"
---
# 关于内存的一些小tips #

1. Android 的每个应用程序都会使用一个专有的Dalvik虚拟机实例来运行，每个应用程序都是在属于自己的进程中运行的。
2. 如果程序在运行过程中出现了内存泄漏的问题，仅仅会使得自己的进程被杀掉，而不会影响其他进程（如果是system_process 等系统进程出问题的话，则会引起系统重启）。
3. Android为不同类型的进程分配了不同的内存使用上限，如果应用进程使用的内存超过了这个上限， 则会被系统视为内存泄漏，从而被杀掉Android会根据进程中运行的组件类别以及组件的状态来判断该进程的重要性，从高到低一共有五个级别：前台进程、可见进程、服务进程、后台进程、空进程。
4. Android设备出厂以后，java虚拟机对单个应用的最大内存分配就确定下来了，超出这个值就会OOM。这个属性值是定义在/system/build.prop文件中的 dalvik.vm.heapsize=36m
5. 当GC发生时，虚拟机会从GC Roots 开始去扫描当前的对象树，发现通过任何reference chain(引用链)无法访问某个对象的时候，该对象即被回收
6. Java中包含4种对象引用：强引用、软引用、弱引用、虚引用
7. Shallow size就是对象本身占用内存的大小，不包含对其他对象的引用，也就是对象头加成员变量（不是成员变量的值）的总和
8. Retained size是该对象自己的shallow size，加上只能从该对象能直接或间接访问到对象的shallow size之和


# Memory monitor  #

1. 发现内存抖动的场景 
2. 发现大内存对象分配的场景 
3. 发现内存不断增长的场景 
4. 确定卡顿问题是否因为执行了GC操作


# OOM  #
之前我们知道Android的应用程序所能申请的最大内存都是有限的，OOM是指APP向系统申请内存的请求超过了应用所能有的最大阀值的内存，系统无法再分配多余的空间，就会造成OOM error。

在Android平台下，除了之前所说的持续发生了内存泄漏(Memory Leak)

一次性申请很多内存，比如说一次创建大的数组或者是载入大的文件如图片的时候

# Leakcanary  #

关于Leakcanary 工具可参考之前的文章。

# 内存优化总结  #

-  static：static声明变量的生命周期其实是和APP的生命周期一 样的，请合理使用。无关引用：比如比较有代表性的Context泄漏，很多情况下当Activity 结束掉后，由于仍被其他的对象指向导致一直迟迟不能回收，这就造成了内存泄漏

-  SoftReference/WeakReference/LruCache：如果对内存的开销比较关注的APP，可以考虑使用WeakReference，当GC回收扫过这块内存区域时 就会回收；如果不是那么关注的话，可以使用SoftReference，它会在内存申请不足的情况下自动释放，同样也能解决OOM问题
  
-  谨慎handler：handler运行于UI 线程，不断处理来自MessageQueue的消息，如果handler还有消息需要处理但是Activity页面已经结束的情况下，Activity的 引用其实并不会被回收，这就造成了内存泄漏。解决方案，一是在Activity的onDestroy方法中调用handler.removeCallbacksAndMessages(null);取消所有的消息的处理，包括待处理的消息；二是声明handler的内部类为static。

-  Bitmap：BitmapFactory.Options的inSampleSize属性进行控制，加载网络图片的时候，使用软引用或者弱引用并进 行本地缓存，图片加载的项目 picasso等

-  Cursor和I/O流及时关闭

-  ListView和GridView的item缓存

-  页面背景和图片加载：在布局和代码中设置背景和图片的时候，如果是纯色，尽量使用color；如果是规则图形，尽量使用shape画图；如果稍微复杂点，可以使用9patch图；如果不能使用9patch的情况下，针对几种主流分辨率的机型进行切图。

-  线程：开启线程数量不易过多，一般和自己机器内核数一样最好。4？
 
-  String/StringBuffer