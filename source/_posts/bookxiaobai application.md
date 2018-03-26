---
title: "书小白书单功能实现"
date: "2018-02-21"
description: "书小白书单功能实现"
categories: 
    - "ANDROID"
---

# Bookxiaobai 书小白书单

书小白书单是个保存书单的小应用。

主要功能是通过**搜索书名**或者通过**扫描书本的isbn**号来获取书籍信息，并将书籍收入清单中。

并可通过设置书本状态，设置已读未读。以timeline的形式形成自己的读书清单列表。

其中书本的信息主要访问的是豆瓣的API

代码地址：[https://github.com/lynn8570/BookXiaobai](https://github.com/lynn8570/BookXiaobai)



## 主要的功能实现介绍



###  采用mockplus初步的原型设计 ###

  [Mokcplus](https://www.mockplus.cn/)是一个非常简单好用的原型设计工具，官网上有一些简易的视频教程，看看几本就会一些几本的功能。不管会不会，撸起袖子加油干，试试，就会了。

  按照几本功能设计的原型大改如下图，细节与最后的实现略有差别

  ![](http://7xl98n.com1.z0.glb.clouddn.com/bookxiaobai_protype.gif)

主要包括主页面采用timeline的形式显示收藏的书单列表，顶部为搜索框，右边为条形码扫描按钮。

### 扫描图书二维码获取isbn号 ###

> ZXing ("zebra crossing") is an open-source, multi-format 1D/2D barcode image processing
> library implemented in Java, with ports to other languages.

直接使用zxing库来实现条码的扫描，源码地址：[zxing github地址](https://github.com/zxing/zxing)

1. 添加依赖

   ```
   compile 'com.google.zxing:core:3.3.2'
   compile 'com.journeyapps:zxing-android-embedded:3.5.0'
   ```

2. 启用内置的activity进行条码扫描

   ```
   imageViewScan.setOnClickListener(new View.OnClickListener() {
               @Override
               public void onClick(View view) {
                   new IntentIntegrator(MainActivity.this).initiateScan();
               }
   });
   ```

   在onClick的时候，启动并初始化扫描页面

3. 在onActivityResult中获取扫描结果

   ```
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
           IntentResult result = IntentIntegrator.parseActivityResult(requestCode, resultCode, data);
           if (result != null) {
               if (result.getContents() == null) {
                   Toast.makeText(this, "Cancelled", Toast.LENGTH_LONG).show();
               } else {
                   Toast.makeText(this, "Scanned: " + result.getContents(), Toast.LENGTH_LONG).show();
                   String strIsbn = result.getContents();//得到isbn号
                   if (!TextUtils.isEmpty(strIsbn)) {
                       mBookPresenter.getBookByIsbn(strIsbn);
                   }
               }
           } else {
               super.onActivityResult(requestCode, resultCode, data);
           }
       }
   ```



###  通过isbn号查询豆瓣API获取图书信息  ###

  豆瓣根据isbn号获取图书的API接口：[api地址](https://developers.douban.com/wiki/?title=book_v2#get_isbn_book)

  例如通过访问 https://api.douban.com/v2/book/isbn/9787115412744

  即可获取图书信息

  1. 采用MVP的结构，我们可以从接口获取到的json信息直接生成model中的BookBean类：具体方式可搜索插件**GsonFormat**自动生成javabean
  2. 使用[restrofit](http://square.github.io/retrofit/)封装几个需要用到的豆瓣图书API：通过isbn号获取图书信息，通过图书ID获取图书信息。

- 采用object box保持本地数据

  [Objectbox](http://objectbox.io/) is designed for mobile. It is an object-oriented embedded database and a full alternative for SQLite. ObjectBox is *incidentally* also well-suited for IoT.

  具体介绍可查看官网。

  1. 添加依赖

     ```
     apply plugin: 'io.objectbox'
     ....
     ....
     android{
         //add object box
         compile "io.objectbox:objectbox-android:1.4.0"
     }
     ```

  2. 初始化，在application 的onCreate中进行初始化

     ```
     BoxConfig.boxStore = MyObjectBox.builder().androidContext(app).build();
     Log.i(TAG,"init()............boxstore="+boxStore);
     if (BuildConfig.DEBUG) {
         new AndroidObjectBrowser(boxStore).start(app);
     }
     ```

  3. 对象注释

     ObjectBox跟其他的ORM框架一样，通过对象属性注解来决定是否要持久化某个对象，或者某个属性。

     > @Entity：这个对象需要持久化。
     >
     > @Id：这个对象的主键。
     >
     > @Index：这个对象中的索引。对经常大量进行查询的字段创建索引，会提高你的查询性能。
     >
     > @NameInDb：有的时候数据库中的字段跟你的对象字段不匹配的时候，可以使用此注解。
     >
     > @Transient:如果你有某个字段不想被持久化，可以使用此注解。
     >
     > @Relation:做一对多，多对一的注解

  4. 基础操作，增删改查
  

		 ```
		 package com.lynn.bookxiaobai.boxstore.base;
		 import com.lynn.bookxiaobai.boxstore.BoxConfig;
		 import java.util.List;
		 import io.objectbox.Box;
		 import io.objectbox.query.QueryBuilder;
		
		 /**
		  * Created by lynn on 2018/2/10.
		  * @param <T> database entity
		  *
		  */
		 public abstract class BaseBoxManager<T> {
		     protected Box<T> mBox;
		     Class<T> tClass;
		
		     public BaseBoxManager(Class<T> entityClazz){
		         this.tClass=entityClazz;
		         mBox= BoxConfig.getBoxStore().boxFor(tClass);
		     }
		
		     protected final void closeDatabase(){
		         mBox.closeThreadResources();
		     }
		     /**
		      * 插入一条记录
		      *
		      * @return The ID of the object within its box.
		      */
		     public long insert(T entity) {
		         if (entity == null) return -1;
		         return mBox.put(entity);
		     }
		
		
		     /**
		      * 插入多条记录
		      */
		     public void insert(List<T> entities) {
		         if (entities != null)
		             mBox.put(entities);
		     }
		
		
		     /**
		      * 删除所有数据
		      */
		     public void deleteAll() {
		         mBox.removeAll();
		     }
		
		
		     /**
		      * 根据条件删除数据库中的数据
		      */
		     public void delete(T object) {
		         mBox.remove(object);
		     }
		
		     /**
		      * 删除多条数据
		      */
		     public void deleteList(List<T> objects) {
		         mBox.remove(objects);
		     }
		
		
		     /**
		      * 更新一条记录
		      */
		     public long update(T object) {
		
		         return mBox.put(object);
		     }
		
		     /**
		      * 更新一条记录
		      */
		     public void update(List<T> objects) {
		         mBox.put(objects);
		     }
		
		     /**
		      * @return Returns a builder to create queries for Object matching supplied criteria.
		      */
		     public QueryBuilder<T> getQueryBuilder() {
		         return mBox.query();
		     }
		
		     /**
		      * 查询并返回所有对象的集合
		      */
		     public List<T> queryAll() {
		         return getQueryBuilder().build().find();
		     }
		
		     /**
		      * 查询并返回 第一个对象
		      */
		     public T QueryFirst() {
		         return getQueryBuilder().build().findFirst();
		     }
		
		
		     /**
		      * 获取对应的表名
		      */
		     public abstract String getTableName();
		 }
		
		 ```


### 采用glide获取网络图片 ###

  1. 添加依赖

     ```
     //glide
     implementation 'com.github.bumptech.glide:glide:4.6.1'
     annotationProcessor 'com.github.bumptech.glide:compiler:4.6.1'
     ```

  2. 只需添加一行就可以加载图片

     ```
      private void downloadImage(){
             String imgUrl=mBookbean.getImages().getSmall();
             if(!TextUtils.isEmpty(imgUrl)){
                 Glide.with(this).load(imgUrl).into(imgBook);//下载图片并显示到imageview上
             }
         }
     ```

### timeline组件 ###

  时间线显示列表的组件，找到了一个GitHub上star比较多的组件。[Timeline-View](https://github.com/vipulasri/Timeline-View) 上有详细介绍使用方法，只需要根据设计的样式稍作调整即可集成。

### ButterKnife使用  ###

  使用butterknife可以减少繁琐的findviewbyid这样的代码，可直接通过注释来绑定组件。

  1. 添加依赖

     ```
     //butterknife
     compile 'com.jakewharton:butterknife:8.8.1'
     annotationProcessor 'com.jakewharton:butterknife-compiler:8.8.1'
     ```

  2. 注释组件

     ```
      @BindView(R.id.txt_book_name)
      TextView txtBookName;
      @BindView(R.id.txt_author)
      TextView txtAuthor;
     ```

  3. 绑定

     ```
     ButterKnife.bind(BookDetailActivity.this);
     ```

### Todo: 添加已读未读存储，数据后台同步 ###





