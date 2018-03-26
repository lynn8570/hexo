---
title: "Leakcanary部分泄露警报无需修复"
date: "2015-09-14"
categories: 
    - "android"
---
# 前言 #
使用leakcanary检查内存泄露之后，由于他的工作原理，造成所有的在上下文关闭之后，还未被释放的资源就会引爆内存泄露通知。但是不是所有的泄露都需要修复的。下面总结几个我的血泪史，希望以后不要重蹈覆辙。


#InputMethodManager.sInstance泄露

输入法泄露，具体的泄露路径类似于

![](http://7xl98n.com1.z0.glb.clouddn.com/M_1.jpg)

提示InputMethodManager.sInstance这个静态实例可能通过各种路径对context进行了泄露。具体的路径可能会不一样，但是归根到最后都是提示InputMethodManager.sInstance静态引用泄露。
通过网上搜索，可能有类似的修复，如下
    public static final class TypedObject
	{
	    private final Object object;
	    private final Class type;

	    public TypedObject(final Object object, final Class type)
	    {
	    this.object = object;
	    this.type = type;
	    }

	    Object getObject()
	    {
	        return object;
	    }

	    Class getType()
	    { 
	        return type;
	    }
	}

	public static void invokeMethodExceptionSafe(final Object methodOwner, final String method, final TypedObject... arguments)
	{
	    if (null == methodOwner)
	    {
	        return;
	    }

	    try
	    {
	        final Class<?>[] types = null == arguments ? new Class[0] : new Class[arguments.length];
	        final Object[] objects = null == arguments ? new Object[0] : new Object[arguments.length];

	        if (null != arguments)
	        {
	            for (int i = 0, limit = types.length; i < limit; i++)
	            {
	                types[i] = arguments[i].getType();
	                objects[i] = arguments[i].getObject();
	            }
	        }

	        final Method declaredMethod = methodOwner.getClass().getDeclaredMethod(method, types);

	        declaredMethod.setAccessible(true);
	        declaredMethod.invoke(methodOwner, objects);
	    }
	    catch (final Throwable ignored)
	    {
	    }
	}
	
	public static void fixInputMethodManager(Activity activity)
	{
	    final Object imm = activity.getSystemService(Context.INPUT_METHOD_SERVICE);

	    final Reflector.TypedObject windowToken
	        = new Reflector.TypedObject(activity.getWindow().getDecorView().getWindowToken(), IBinder.class);

	    Reflector.invokeMethodExceptionSafe(imm, "windowDismissed", windowToken);

	    final Reflector.TypedObject view
	        = new Reflector.TypedObject(null, View.class);

	    Reflector.invokeMethodExceptionSafe(imm, "startGettingWindowFocus", view);
	}

主要通过反射，修复内存泄露。

但是，下面的话，非常重要，非常重要，非常重要，重要的事情说三遍：这个属于系统级别的泄露，也就是说，你不泄露，别人也会泄露，而且整个android系统，只保留一个static instance的引用，所以这个修复，对整个系统的内存没有太大的改善。而且这个修复的隐患是，你有可能会在页面跳转的时候，遇到各种不可预测的编辑框无法获取焦点的问题。所以，我建议，这个泄露，可以忽略。

# AsyncQueryHandler 没有quit #
有时候我们会遇到HandlerThread没有quit而爆出的泄露，泄露路径如下：

![](http://7xl98n.com1.z0.glb.clouddn.com/M_2.jpg)

而android系统中有个AsyncQueryWorker从源代码看

	public AsyncQueryHandler(ContentResolver cr) {
		super();
		mResolver = new WeakReference<ContentResolver>(cr);
		synchronized (AsyncQueryHandler.class) {
	    	if (sLooper == null) {
	        	HandlerThread thread = new HandlerThread	("AsyncQueryWorker");
	        	thread.start();
	        	sLooper = thread.getLooper();
	    	}
		}
		mWorkerThreadHandler = createHandler(sLooper);
	}

这个名为AsyncQueryWorker的HandlerThread自从start之后就没人管了。这个时候，leakcanary也会提示泄露。
于是我曾自作聪明，通过反射将这个sLooper强制退出，代码如下：

	//linlian@2015.06.01 release static sLooper in AsyncQueryHandler
	public static void fixAsyncQueryWorker(){
		Field sLooperCached = null;
        try {
            sLooperCached = Class.forName("android.content.AsyncQueryHandler").getDeclaredField("sLooper");
            sLooperCached.setAccessible(true);
        } catch (Exception ex) {
            ex.printStackTrace();
        }
		if (sLooperCached == null) return;
		Looper looper = null;
		 try {
            // Get reference to the sLooperCached 
            looper = (Looper)sLooperCached.get(null);
			if(looper!=null){
				looper.quit();
				sLooperCached.set(null,null);
			}
        } catch (Exception ex) {
            ex.printStackTrace();
        }
	
这个修复的惨痛后果是，有些后台线程直接结束了，所以，对于AsyncQueryHandler的泄露，我也是建议不修复。

# TextLine.sCached 泄露 #

![](http://7xl98n.com1.z0.glb.clouddn.com/M_3.jpg)

和inputManager一样，这个泄露，属于系统的静态引用而造成的泄露，但是这个泄露的修复，目前没有发现什么不良的影响，但是泄露修复的价值意义不知道大不大，也是你不泄露，别人也会泄露，反正总有一个这样的引用存在的。修复的效果也不是很明显
修复代码如下

	public static void clearTextLineCache(){
		Field textLineCached = null;
        try {
            textLineCached = Class.forName("android.text.TextLine").getDeclaredField("sCached");
            textLineCached.setAccessible(true);
        } catch (Exception ex) {
            ex.printStackTrace();
        }
		if (textLineCached == null) return;
        Object cached = null;
        try {
            // Get reference to the TextLine sCached array.
            cached = textLineCached.get(null);
        } catch (Exception ex) {
            ex.printStackTrace();
        }
        if (cached != null) {
            // Clear the array.
            for (int i = 0, size = Array.getLength(cached); i < size; i ++) {
                Array.set(cached, i, null);
            }
        }
	}

# 总结 #

LeakCanary是一个很好检查内存泄露的工具，但不是所有的泄露都需要修复，有些事android系统的泄露bug，通过各种方式曲线救国之后，不一定会达到一个很好的内存改善结果，所以干脆不要去动他，以免引起不可预测的运行异常。只有自己非常肯定的，由于使用不规范等引起的内存泄露才是我们重点关注的部分