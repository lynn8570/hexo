---
title: "TDD测试驱动开发学习（1）"
date: "2015-09-16"
categories: 
    - "TDD"
---
# 前言 #
说到测试驱动，就好像各种模式一样，你知道她好，但是总是没有好好的去坚持，去实施，这事，就和女人的减肥事业一样，磨练人的意志。
让我重新开始决定好好学习TDD原因，最基础的时间条件来说，我最近是比较空闲，处于项目空档期；另外，这几天在看美剧[《硅谷》(SILICON VALLEY)](http://v.baidu.com/tv/22488.htm?video_uri=tv.basic.002211.1427779458.0 "")。其中有一集就是，有个小毛孩把男主的程序数据库全毁了，男主花了好大心事修复，而最后一个镜头就是，他的所有测试用例，一排绿灯通过，这下，他才开始放心揍那个高科技骗子。虽然这个镜头只有一秒不到的时间，但这个镜头在我还算幼小的心灵，烙下了一个我要好好学TDD的念头，哈哈哈O(∩_∩)O哈哈~

# 从0开始 #
我follow的这本书是[测试驱动开发的艺术](http://book.douban.com/subject/5326182/)，
以下是我根据书中内容，一步一步敲打代码的步骤总结，只为了，让自己梳理一下。

# Android studio 添加 junit 测试 #
当然，书里面没有讲怎么在 Android studio 集成IDE中添加junit测试，下面是我摸索的结果。

1. 创建一个Android项目，这里，也可以不是，随便java项目也可以，我们只是做一个基本的java Junit测试的demo程序而已。我创建的是一个Android的项目按AS的模板默认设置就可以；默认的项目目录结构如下图（采用project视图）：

	![](http://7xl98n.com1.z0.glb.clouddn.com/android_project.jpg)

2. 打开`build.gradle`文件，将junit:junit:4.12加入编译

	```
	dependencies {
	    compile fileTree(dir: 'libs', include: ['*.jar'])
	    compile 'com.android.support:appcompat-v7:23.0.0'
	    compile  'junit:junit:4.12'
	    testCompile 'junit:junit:4.12'
	}
	```
	然后点击`Sync Project with Grandle files`的那个图标按钮，这个时候，我们就可以在代码中访问`@Test`了
    
    	package com.example.lynn.myapplication;
    	import org.junit.Test;
	
		/**
		 * Created by lynn8570 on 2015-9-16.
		 * mail: lynn8570@gmail.com
		 */
		public class TestDemo {
		    @Test
		    public void testOne() throws  Exception{
		
		    }
		}
	
3. 告诉编译器，你的测试代码在哪里，一般android项目的默认测试代码放在`/src/androidTest/java`中，因此我们需要在gradle文件中添加如下内容：
	
		android {
	 	sourceSets{
        	test {
            	java.srcDir file('src/androidTest/java')
        	}
    	}
		}

4. 由于我们只是运行junit测试，而android默认的测试是`Android Instrumentation Tests`，所以需要在运行测试的时候，点击`Build Variants`（一般在左下角），将`Test Artifact`设置为`Unit Tests`。
5. 随便写一个测试看下我们的测试可否运行，代码如下：

		@Test
    	public void testOne() throws  Exception{
        	assertNotNull(new Object());
    	}
	
6. 配置测试运行参数，在`Run/Debug Configurations`对话框中，点击“+”号，添加Junit运行配置。你可以只运行一个class或者运行package下所有的包。
7. 运行测试，测试通过配置完成，event log输出如下内容的话，表示前期准备OK

		13:46:00 Compilation completed successfully in 4s 393ms
		13:46:00 Tests Passed: 1 passed in 0.032 s


# 可能会遇到一些错误 #

- 默认的ApplicationTestCase未删掉，因为AS默认创建的Android项目是有一个默认的ApplicationTestCase测试，由于我们将build variantants中的Test Artifact改为Unit Test，会造成报错如下：

		Exception in thread "main" java.lang.NoClassDefFoundError: android/test/ApplicationTestCase
		 at java.lang.ClassLoader.defineClass1(Native Method)
		 at java.lang.ClassLoader.defineClass(ClassLoader.java:800)
		 at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:142)
		 at java.net.URLClassLoader.defineClass(URLClassLoader.java:449)
		 at java.net.URLClassLoader.access$100(URLClassLoader.java:71)
		 at java.net.URLClassLoader$1.run(URLClassLoader.java:361)
		 at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
		 at java.security.AccessController.doPrivileged(Native Method)
		 at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
		 at java.lang.ClassLoader.loadClass(ClassLoader.java:425)
		 at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:308)
		 at java.lang.ClassLoader.loadClass(ClassLoader.java:358)
		 at java.lang.Class.forName0(Native Method)
		 at java.lang.Class.forName(Class.java:270)
		 at com.intellij.junit4.JUnit4TestRunnerUtil.loadTestClass(JUnit4TestRunnerUtil.java:302)
		 at com.intellij.junit4.JUnit4TestRunnerUtil.appendTestClass(JUnit4TestRunnerUtil.java:286)
		 at com.intellij.junit4.JUnit4TestRunnerUtil.buildRequest(JUnit4TestRunnerUtil.java:80)
		 at com.intellij.junit4.JUnit4IdeaTestRunner.startRunnerWithArgs(JUnit4IdeaTestRunner.java:39)
		 at com.intellij.rt.execution.junit.JUnitStarter.prepareStreamsAndStart(JUnitStarter.java:212)
		 at com.intellij.rt.execution.junit.JUnitStarter.main(JUnitStarter.java:68)
		 at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
		 at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
		 at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
		 at java.lang.reflect.Method.invoke(Method.java:606)
		 at com.intellij.rt.execution.application.AppMain.main(AppMain.java:140)
		Caused by: java.lang.ClassNotFoundException: android.test.ApplicationTestCase
		 at java.net.URLClassLoader$1.run(URLClassLoader.java:366)
		 at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
		 at java.security.AccessController.doPrivileged(Native Method)
		 at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
		 at java.lang.ClassLoader.loadClass(ClassLoader.java:425)
		 at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:308)
		 at java.lang.ClassLoader.loadClass(ClassLoader.java:358)
		 ... 25 more


如果这样，请删除默认的ApplicationTestCase

- 如果运行时，提示错误信息：

		Error running TestDemo: Class 'com.example.lynn.myapplication.TestDemo' not found in module 'app'
	检查是否添加了

		sourceSets{
	        	test {
	            	java.srcDir file('src/androidTest/java')
	        	}
	    	}

# 总结 #

通过上面的环境准备，基本就可以按照书本里面的步骤开始一步步敲代码啦~~

至此书本上的第一个demo演练已经过了一遍，主要是模板解析的一个演示程序，在不断写测试代码，完成功能，重构优化的一轮轮中，通过测试代码作为基本保障，让程序代码不断的完善。