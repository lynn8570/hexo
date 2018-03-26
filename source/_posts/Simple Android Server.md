---
title: "简易 Android 服务器"
date: "2016-04-23"
categories: 
    - "android-server"
---
# 在Android上部署一个简单的服务器 #
主要的应用场景：在android端建一个简单的服务器，这样比如在同一个局域网的其他设备，可以通过浏览器访问“ip地址+端口”，访问android服务器端的一些资源。就好像我们经常见，通过浏览器设置wifi配置那样。

因为业务需求相对简单，而且给我们的研发投入就是，一个人，一周左右的时间。所以，我看到jetty什么的有点头大，就没有用一些商业的现有的android web 服务器，倒是apache的httpcore.jar包解决了我大部分的问题，但是这个在4.0之后都被弃用了。所以需要的，请下载对应的jar 包，下载地址[httpcomponents/httpcore](http://www.apache.org/dist/httpcomponents/httpcore/source/)

# 搭建web server #

首先，我们主要参考一篇博文[Android中如何搭建一个WebServer](http://blog.csdn.net/jiangwei0910410003/article/details/27861135)。

大部分的服务器代码就是参考他的几个基本模块，可以让你快速的搭建一个Android 端的web服务器。主要的核心代码就是，在Android建一个`ServerSocket`，然后用httpcore中的一些类来负责处理`serversocket.accept()`每次接收的socket；分析请求，返回响应。


# 动态网页 #

大部分的服务器代码如所链接的博文那样，基本上，另外一个连接的设备，通过浏览器访问对应地址，就可以收到回复。但是，在我的简单需求里面，还有一点点小复杂的东西：获取android服务器端的某些信息，需要写到发送给客户端的html代码里面。

基本思路是：将写号的html 模板（大部分是html代码，具体数值方面，用`$valuename`来替代)，这样，在response中写入反馈的时候，可以先将template模板中的变量进行过滤，替换为对应的数值和html标签，最终生成一份html代码，在反馈给客户端。

这就是一个伪动态网页了

# 获取post请求中的参数 #

另外博文中没有提到的一个很重要的功能，就是获取网页代码中，提交的form表单数据。原先我以为会很简单，比如在request中用个类似于`getParameter（XXXX）`不就搞定了么。可事实上，这样，你得到的永远都是`null`。

后来找了很久，发现了这个[echo-received-post-data](http://stackoverflow.com/questions/7199969/apache-httpcore-simple-server-to-echo-received-post-data)。

所以获取html 表单数据类似于需要这样：

## html表单代码 ##

例如，我们有一个如下的表单需要提交，而且，一般有密码的表单都是用POST请求，因为我们可不想浏览器的地址上赫赫然地写着passwork=abc;

    <form method="POST" action="/submit" name="forms" enctype="text/plain" onsubmit="return toVaild()">
	Network SSID: 
	<input type="text" name="SSID" id="SSID" value="AndroidAp" />
	<br />
	Security: 
	<select name="Security"> 
	<option value="0">None</option> 
	<option value="1">WPA PSK</option> 
	<option value="2"  selected="selected">WPA2 PSK</option> 
	</select> 
	<br />
	Password: 
	<span id=box><input type="password" name="password"  id="password" value="" /></span> <span id=click><a href="javascript:showps()">Show password</a></span>
	<br />
	The password must have at least 8 characters
	
	<br />
	<input type="submit" id="submit" value="Submit" />
	
	</form>

## 服务器端handler代码 ##

根据验证发现：当表单采用post方式提交数据后，`HttpRequestHandler`的handler方法参数中的request，实际上会变成一个`BasicHttpEntityEnclosingRequest`实例，而普通的get请求request则是`BasicHttpRequest`实例。

服务器获取post请求中的数据相关代码：

	static class WebServiceHandler implements HttpRequestHandler {
			private PageEntity mPageEntity;
	
			public WebServiceHandler(Context context) {
				mPageEntity = new PageEntity(context);
			}
	
			@Override
			public void handle(HttpRequest request, HttpResponse response,
					HttpContext context) throws HttpException, IOException {
	
				Log.i(TAG, "WebServiceHandler handle request=" + request);
	
				String method = request.getRequestLine().getMethod()
						.toUpperCase(Locale.ENGLISH);
				String target = request.getRequestLine().getUri();
	
				Log.i(TAG, "WebServiceHandler handle method=" + method);
	
				Log.i(TAG, "WebServiceHandler handle target=" + target);
	
				Log.i(TAG, "WebServiceHandler handle request.getParams()="
						+ request.getParams()); //放心，我们要的parameter不在这个里面，never
				
				if(request instanceof BasicHttpEntityEnclosingRequest){
					HttpEntity entity = ((BasicHttpEntityEnclosingRequest) request).getEntity();
					
					byte[] data;
			        if (entity == null) {
			            data = new byte [0];
			        } else {
			            data = EntityUtils.toByteArray(entity);
			        }
	
			        Log.i(TAG,new String(data));//把这段话答应出来，里面就是post表单中的键值对。
				}
				response.setStatusCode(HttpStatus.SC_OK);
				response.setEntity(mPageEntity.getPage("index.html"));
			}
	}

我们运行之后，果然在log中打印出来post表单中的键值对`SSID=AndroidAp&Security=2&password=ttyyyyyyyyy`;

这样完成服务器和网页客户端的数据互通。其他的问题，就是一个搬运和时间的问题了。


# 获取get请求中的参数 #

另外提一下，get请求，因为参数直接写在uri中，我们可以通过

			String method = request.getRequestLine().getMethod()
					.toUpperCase(Locale.ENGLISH);
			String target = request.getRequestLine().getUri();

直接在target中得到`target=/submit?SSID=AndroidAp&Security=2&password=ffffffgggfffff`
所需要的参数。
# 效果展示 #
基本这个服务器可以完成图片、css、js、文件等传送，看看首页效果：

![](http://7xl98n.com1.z0.glb.clouddn.com/QQ%E6%88%AA%E5%9B%BE20160625154803.png)


# 总结 #

因为整体的业务逻辑和需求比较简单，我们估计只需要一个信息页面，然后这个页面可以提交数据，最多再加一两个正确和错误的提示页面。所以这个简单web server就是这样。如果要是复杂的，当然在看资料的是，也发现有人用KSWEB，或者android i-jetty把android 设备作为服务器，运行wordpress等。感觉也可以很强大，不过我下了都是要收费的。我这个小project就算了。还是回到原始，解决问题，最关键。O(∩_∩)O