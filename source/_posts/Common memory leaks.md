---
title: "常见的内存泄露"
date: "2015-09-02"
categories: 
    - "android"
params:
  Description = "内存泄露修复常见方法"
  notoc = true
---
利用[LeakCanary](https://github.com/square/leakcanary)检查内存泄露，在出现内存泄露的时候，会弹出内存泄露提醒，点击可显示内存泄露路径。现在我来归纳一下自己遇到的几种常见的内存泄露类型吧~~~~~~~
# 常见的泄露类型 #

- 对context，activity的static的引用
	
	没必要的static，移出static
	
	![](http://7xl98n.com1.z0.glb.clouddn.com/QQ截图20150806154414.jpg)

	上图中`RawContactEditorView.mContext`有时候存在这种没有必要的static应用，看到static的时候，请提高一百八十分的警惕，寻根是否有必要，是否可以用appcontext来替代等，以便防止对activity context的泄露

- 必要的static，试着用appcontext来替代

	![](http://7xl98n.com1.z0.glb.clouddn.com/QQ截图20150806154825.jpg)

	上图中`ContactListFilterController.sFilterController`为静态，而这sFilterController对activity的context存在引用，从而造成了context的泄露。但是为了实现单例，代码又不得不保留static，对于这种情况查看对context引用的实际用处，有时候你就回发现，很多时候可以appcontext来替代。

- 必须对activity static应用，控制生命周期

	![](http://7xl98n.com1.z0.glb.clouddn.com/QQ截图20150901135429.jpg)

	上图中，mDialog为静态，在外部需要直接访问必须为静态，dialog又对activity的context进行了引用，无法用appcontext来替代，像这种情况，可以在相应activity的生命周期中对引用进行解除。就是我们熟悉的`onDestroy()`中设置null，remove之类的操作

- 非静态的内部类

	![](http://7xl98n.com1.z0.glb.clouddn.com/QQ截图20150806154825.jpg)
	
	在activity中直接声明的内部类，通常对activity保有一个this的引用，一次容易照成泄露，如果不想去控制这个mPhoneStateListener的生命周期，就采用将其变为静态内部类，通过WeakReference来保留对this的引用。例如：

```
public static class PhoneStateListenerImpl extends PhoneStateListener {
		protected final WeakReference<DialpadFragment> mDialpadFragment; 
		public PhoneStateListenerImpl(DialpadFragment fragment){
			 mDialpadFragment = new WeakReference<DialpadFragment>(fragment);
		}
        @Override
        public void onCallStateChanged(int state, String incomingNumber) {
		 	final DialpadFragment fragment = mDialpadFragment.get();
			if(fragment==null)return;
			if(fragment.getActivity()==null)return;
            	…….// fragment!=null.
			Balabala…….
		}
}

```

- HandlerThread 没有quit，cursor没有close，网络连接、IO流没有及时关闭

# 总结 #

这些知识一些常见的内存泄露例子，平时我们经常见的在onDestroy的时候，unregistered，close，removelistener之类的操作都是避免因一直引用context而引发内存泄露；另外就是static变量的慎用，还有就是非静态内部类慎用。
遇到内漏的时候，主要是还是根据路径查看代码，具体问题，具体分析~~~~~~~~~~