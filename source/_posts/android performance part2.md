---
title: "Android performance par2: Compute"
date: "2016-07-08"
categories: 
    - "ANDROID"
---
# 前言 #
前文主要讲在渲染阶段，我们可以做的优化。本文主要是分析算法对程序运行速度的影响。举个栗子，int的运算速度最快，short次之，byte再次之，long再次之。float和double运算速度最慢；除法比乘法慢的太多，基本上除法是乘法的9倍时间......等等，真的要这么较真吗？？？我只是举个栗子，这说明，我们的代码，实实在在的在影响程序的运行速度。

当然如果我们遇到的只是一个方法耗时太久，那么问题似乎更简单些，我们只需要优化一个方法函数，而整个的运行速度就会有很大的提高。而更麻烦的就是，我们遇到的是，每个方法都慢了一点点，所以我们花了很大的时间，优化了每个方法，但是整体的速度只提高了一点点。

![](http://7xl98n.com1.z0.glb.clouddn.com/%E5%9B%BE%E7%89%8721.png)

![](http://7xl98n.com1.z0.glb.clouddn.com/%E5%9B%BE%E7%89%8722.png)


# batching and cashing #

性能优化中最常用的一项技术就是，批处理和缓存。这可以减少我们进行耗时运算的次数，从而从整体上降低方法的运行时间。

视频举了个斐波那契的数列1,1,2，3，5，8，13，21，34，......数列从第3项起，每一项是前两项的和，这就是有名的斐波拉契数列。

所以计算斐波那契的数列可以用递归的方式来写：

    /**
     *  Why store things when you can recurse instead?  Don't let evidence, personal experience,
     *  or rational arguments from your peers fool you.  The elegant solution is the best solution.
     *
     * @param positionInFibSequence  The position in the fibonacci sequence to return.
     * @return the nth number of the fibonacci sequence.  Seriously, try to keep up.
     */
    public int computeFibonacciRecursive(int positionInFibSequence) {
        if (positionInFibSequence <= 2) {
            return 1;
        } else {
            return computeFibonacciRecursive(positionInFibSequence - 1)
                    + computeFibonacciRecursive(positionInFibSequence - 2);
        }
    }


似乎看起来没有问题，但实际运行过程中，我们发现随着数列越来越大，递归算法造成的运行时间成指数级的增长。

我抓了两个，分别是数列35 和 数列37 的计算trace文件

![](http://7xl98n.com1.z0.glb.clouddn.com/QQ%E6%88%AA%E5%9B%BE20170725093716.png?imageView2/2/w/650/h/600/q/75)

![](http://7xl98n.com1.z0.glb.clouddn.com/QQ%E6%88%AA%E5%9B%BE20170725093830.png?imageView2/2/w/650/h/600/q/75)

35的时候耗时已经31秒多了，而37时，增加到81秒多。更别提如果是1000的大小的数列。程序的运行时间将达到一个什么级别。

然而在充分理解了函数意义之后，采用算法优化之后，可大大降低运行速度。优化如下：


  	/**
     * It is important to understand what your code is doing, no matter how simple the task. For
     * example, most people know better than to compute Fibonacci numbers recursively, but it is
     * not unusual to unintentionally redo work in your application. Check your app for places
     * where you can cache current results for future re-use.
     *
     * In this case, recursive Fibonacci calls fib8 which calls fib7 and fib6, but that fib7 call
     * calls fib6 again and fib5, So now you've got two fib6's and one fib5 call, but each of those
     * fib6 calls will have a fib5 and fib4, so now you have three calls to calculate fib5, blah,
     * blah, blah.  Recursive fibonacci is terrible.  Iterating lets you calculate fibX once,
     * use that result twice, and move on.
     *
     * @param positionInFibSequence  The position in the fibonacci sequence to return.
     * @return the nth number of the fibonacci sequence.  Seriously, try to keep up.
     */
    public int computeFibonacci(int positionInFibSequence) {

        int prev = 0;
        int current = 1;
        int newValue;
        for (int i=1; i<positionInFibSequence; i++) {
            newValue = current + prev;
            prev = current;
            current = newValue;
        }

        return current;
    }

将之前的值保存起来，这样，不用每一次都递归去重新计算之前的数列值。

再测试下trace运行速度。

![](http://7xl98n.com1.z0.glb.clouddn.com/QQ%E6%88%AA%E5%9B%BE20170725094936.png?imageView2/2/w/650/h/600/q/75)


计算数列大小为1000的序列值。总共大概三秒。。

这个列子大概演示了，cashing，在程序算法优化上的作用。我们将之前的值cashing起来，而不是再重新计算。大大减少了运算次数。从而将运算性能大大的提高了。


## trace view 的使用 ##

找到最耗时的方法，定位性能瓶颈。traceview可以将每个方法的运行时间调用的层级关系清晰的展现出来，方便我们分析。

Android device monitor->devices->package name->		start method profiling-> interact with app ->


使用monitor来获取trace文件，录制了一个动画：

![](http://7xl98n.com1.z0.glb.clouddn.com/profiling_trace.gif)

用代码的方式来获取trace文件

1. Debug的以下静态方法方法来启动： startMethodTracing(String traceName)， stopMethodTracing ()
2. adb pull /sdcard/XXXX.trace d:XXX.trace
3. 再用DDMS打开trace文件


# UI thread #

应用启动，系统会创建一个主线程（main thread）。这个主线程负责向UI组件分发事件（包括绘制事件）

1. 不要阻塞UI线程。
2. 不要在UI线程之外访问Android UI toolkit

如果线程被阻塞了，那么我们可能会得到一个ANR：

应用开发中常见的ANR主要有如下几类：


- 按键触摸事件派发超时ANR，一般阈值为5s（设置中开启ANR弹窗，默认有事件派发才会触发弹框ANR）；
- 广播阻塞ANR，一般阈值为10s（设置中开启ANR弹窗，默认不弹框，只有log提示）；
- 服务超时ANR，一般阈值为20s（设置中开启ANR弹窗，默认不弹框，只有log提示）；

发现ANR的时候可以通过如下命令，查看一些anr的信息，ANR文件的分析，我过个阶段会整理下遇到的问题和解决方式。

adb pull /data/anr/ arn/

为了不要阻塞UI线程，我们需要将耗时的工作放在其他线程中。

这个就不具体举例了。


# 数据结构对性能的影响 #

我们大概也是举个例子：

ArrayList和LinkedList的大致区别： 
- ArrayList是实现了基于动态数组的数据结构，LinkedList基于链表的数据结构。 -对于随机访问get和set，ArrayList觉得优于LinkedList，因为LinkedList要移动指针。 
- 对于新增和删除操作add和remove，LinedList比较占优势，因为ArrayList要移动数据。 


这里引入另外一个查看方法执行时间的工作，system trace.

![](http://7xl98n.com1.z0.glb.clouddn.com/systemtrace.gif)


对于想检测的函数方法，我们可以在前后加上代码，然后在抓trace的时候，需要中选应用对应的包名，才可以看到对应的标签。


![](http://7xl98n.com1.z0.glb.clouddn.com/%E5%9B%BE%E7%89%8723.png)


即上图中的 Trace.begingSection()和 Trace.endSection()

![](http://7xl98n.com1.z0.glb.clouddn.com/QQ%E6%88%AA%E5%9B%BE20170725112812.png)



# 算法优化 #

本章主要介绍在算法优化上两个我们可以使用作为分析的辅助工具 TraceView和systemtrace。
具体的算法优化方法有批处理，缓存、多线程、数据结构优化，这边只是举了视频中的例子。
