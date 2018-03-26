---
title: "如何使用leakcanary进行内存检查"
date: "2015-09-01"
categories: 
    - "android"
---
# LeakCanary介绍 #

如官网所说[LeakCanary](https://github.com/square/leakcanary)is a memory leak detection library for Android and Java.

有了他，我们用于做内存泄露的检查就方便了许多，经过部署和添加之后，运行我们的应用，经过常规的使用操作，如果应用存在内存泄露，那么leakcanary将会发出notification提醒，点击提醒，可以进入leakcanary的泄露路径界面，将内存泄露路径打出来。


# 如何添加leakcanary #
## 适用于eclipse的使用方式##
github上的代码，采用的是gradle来build的，之前我还在用eclipse的时候，还弄了一份适用于eclipse目录结构的代码，然后作为jar包提供给应用使用，检查应用的内存泄露情况。
[下载转换后的代码](/media/DisplayLeak.rar)

1. 将下载好的代码，导入到eclipse中；
2. 将DisplayLeak作为lib使用，打开DisplayLeak的properties对话框，选择android勾选 Is Library
3. 在需要使用内存泄露的应用中，同样打开properties对话框，选择android在library中选择add,添加刚才的DisplayLeak作为lib使用

## Android studio使用##
以上只是因为当时工作的特殊集成环境，于是走了一条曲线救国的道路，现在用AS，只需在dependencies添加写代码就可以直接使用了，好用到哭~~~~(>_<)~~~~
 
你只需要在app module的build.gradle文件中添加如下依赖，就可以使用leakcanary了。

```java
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile 'com.android.support:appcompat-v7:23.0.0'
    compile 'com.squareup.leakcanary:leakcanary-android-no-op:1.3.1'
    releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.3.1'
    releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.3.1'
}
```
# 如何使用 #

- 添加组件到AndroidManifest.xml

	```
	<!-- leak canary-->
		<service
            android:name="com.squareup.leakcanary.internal.HeapAnalyzerService"
            android:enabled="false"
            android:process=":leakcanary" />
        <service
            android:name="com.squareup.leakcanary.internal.DisplayLeakService"
            android:enabled="false" />

        <activity
            android:name="com.squareup.leakcanary.internal.DisplayLeakActivity"
            android:enabled="true"
            android:icon="@drawable/__leak_canary_icon"
            android:label="@string/__leak_canary_display_activity_label"
            android:taskAffinity="com.squareup.leakcanary"
            android:theme="@style/__LeakCanary.Base" >
        </activity>
	<!-- leak canary-->

	```
- 项目中加入LeakCanary代码
	- 检查Activity泄露
	检查activity泄露只需要在application中添加如下代码

    ```java
		import android.app.Application;
		import android.content.Context;
		import com.squareup.leakcanary.LeakCanary;
		import com.squareup.leakcanary.RefWatcher;
		/**
		 * Created by Administrator on 2015-8-31.
		 */
		public class ExampleApplication extends Application {
		    public static RefWatcher getRefWatcher(Context context) {
		        ExampleApplication application = (ExampleApplication) context.getApplicationContext();
		        return application.refWatcher;
		    }
		    private RefWatcher refWatcher;
		    @Override
		    public void onCreate() {
		        super.onCreate();
		        refWatcher = LeakCanary.install(this);
		    }
		}
	```
	- 检查fragment泄露
	在fragment的onDestroy方法中添加

	```
	public abstract class BaseFragment extends Fragment {
 	@Override
 	public void onDestroy() {
    	super.onDestroy();
    	RefWatcher refWatcher = ExampleApplication.getRefWatcher(getActivity());
    	refWatcher.watch(this);
  		}
	}
	```

# LeakCanary原理 #
所有的这些准备结束之后，我们只需要对编译的debug版本的app进行常规的一些操作，当所检测的应用出现内存泄露的时候，LeakCanary将自动的发出内存泄露的通知。

Leakcanary的工作原理：

1. 在onDestroy中创建了KeyedWeakReference的对象来监视需要监视的fragment，activity或其他一些对象等；进行弱引用
2. 在一个后台线程，会去查看是否有这个KeyedWeakReference对象，是否被cleared了；如果没有，则强制执行GC
3. 如果GC之后，还是没有clear这个reference，那么久保存一个heap堆栈的hrof文件；
4. 然后一个独立的HeapAnalyzerService服务进程将启动，并通过HeapAnalyzer集成HAHA另外一个开源项目的代码来对hrof进行自动的分析。
5. HeapAnalyzer将去定位KeyedWeakReference
6. HeapAnalyzer计算出这个引用到GC root一条最短的强引用链路，来确定是否存在leak
7. 最终再讲这个结果传递给DisplayLeakService并发出通知，显示泄露路径 

# 总结 #
Leakcanary方便我们查找内存泄露路径，当发现内存泄露之后，我们需要根据具体的泄露路径来解决泄露问题，下一篇，我会总结下在检查出发现的常见的泄露类型 