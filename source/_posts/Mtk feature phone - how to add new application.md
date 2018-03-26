---
title: "MTK Feature Phone-Hello World"
date: "2016-03-05"
categories: 
    - "feature phone"
---
# What, Feature phone #
一开始我是拒绝的，然并卵，要你干，你就得干，返璞归真，走一走先人的路。一搜几乎都是零几年的文章，那个时候，我还在备战高考，那个时候，还不知道编程为何物。从年前一段时间到现在，这个项目有了个初始版本，终于闲下来可以列点总结，以备记忆。么么哒~~~~~


# 如何建一个应用 #
MTK的文档是有，但是因为年代久远，平台更新，总是有一些变化，让你没办法依葫芦画瓢，最后郁郁寡欢；但一般这种状态持续一阵子后，总有茅塞顿开的一天。几经摸索，我终于可以建立一个，随后我才发现，认识旧世界，这仅仅只是个开始。如下总结下下，也许没准多少年后，我又回来了呢。

## 文件目录 ##
大部分的应用都是放在`plutommi\mmi\`下面的，每个应用一个文件夹，文件夹夹下面的目录结构大体是这样的，例如`QRcode`目录下应该是


	QRcodeInc   //存放头文件
	--QRcodeAppGprot.h  //对外的接口，供外部调用
	--QRcodeAppProt.h   //内部接口，内部原型声明
	--QRcodeAppResDef.h //资源定义
	QRcodeSrc   //存放源文件C
	--QRcodeMain.c

## 将项目加入编译 ##
将我们的代码加入编译，在`make/plutommi/mmi_app/mmi_app.mak`
文件中追加：

	//add 源文件
	SRC_LIST += XXXXXX \
			plutommi\mmi\QRcode\QRcodeSrc\QRcodeMain.c \
			plutommi\mmi\HelloWorld\HelloWorldSrc\QRcodeMain2.c \

	......
	//add 头文件目录
	INC_DIR = XXXXXX＼
		plutommi\mmi\QRcode\QRcodeInc			
## 程序开关 ##
程序开关，可以不要，全都省了，要加的话，差不多就是

1. 在plutommi\mmi\Inc\MMI_features_switch.h添加`#define CFG_MMI_HELLOWORLD	(__ON__)`

2. 在plutommi\Customer\CustResource\PLUTO_MMI\ MMI_features_switchPLUTO.h末尾添加	`#define CFG_MMI_HELLOWORLD (__ON__)`


3. 在\plutommi\Customer\CustResource\项目名_MMI 的feature.h文件中追加`#define CFG_MMI_HELLOWORLD	(__ON__)`


4. 在plutommi\mmi\Inc\MMI_features.h添加

		#if defined(CFG_MMI_HELLOWORLD) && ((CFG_MMI_HELLOWORLD == __ON__)||(CFG_MMI_HELLOWORLD == __AUTO__)) 
			#ifndef __MMI_HELLOWORLD__
			#define __MMI_HELLOWORLD__
			#endif
		#endif

完成之后，我们就可以在程序中添加如：
		
		#ifdef __MMI_HELLOWORLD__
        #endif
这样的代码开关了，但是我觉得菊紧~，不想弄；这步骤不要了；

## 界面显示 ##

如果你的程序不需要文字资源，不需要图片资源，那么上面两步，已经完成了一个应用的添加，接下来要做的事情不过是在c文件里面给一个程序入口，然后再这个程序路口里面，负责显示你的应用界面；最后再把这个程序路口暴露给随便谁谁谁，可以在launcher里面，可以在一个按键的响应里面，反正一按，就跳到你的界面就对了。

例如：

	void mmi_qrcode_exit(void){
		mmi_frm_scrn_close_active_id(); //关闭页面
	}
	
	void mmi_qrcode_entry(char* szSourceString){
		S32 x,y,w,h;
		EntryNewScreen(SCR_QRCODE_MAIN,NULL,mmi_qrcode_entry,NULL); //SCR_QRCODE_MAIN暂时没有，可以随便拿一个src用一下，后面我们会说到怎么定义这些东西
		entry_full_screen();//hide status bar
		clear_screen();
		gui_measure_string((UI_string_type)GetString(STR_ID_QRCODE_NOTE),&w,&h);
		x=(UI_device_width-w)/2;
		y=(UI_device_width-h)/2;
		gui_move_text_cursor(x,y);
		gui_print_text((UI_string_type)GetString(STR_ID_QRCODE_NOTE));
		
		gui_BLT_double_buffer(0,0,UI_device_width-1,UI_device_height-1);
		SetKeyHandler(mmi_qrcode_exit,KEY_LSK,KEY_EVENT_DOWN);
	}

在C的代码中，我们做的事情就是，进入我们的页面，获取一段字符串，显示，然后设置点击软键盘左键的时候，退出我们的界面；

相应的，在头文件中我们需要加入原型声明

	#ifndef _QRCODE_GPROT_H
	#define _QRCODE_GPROT_H
	
	#include "MMI_features.h"
	#include "MMIDataType.h"
	#include "kal_general_types.h"
	
	//interface for outside
	
	extern void mmi_qrcode_exit(void);
	extern void mmi_qrcode_entry(char* szSourceString);
	
	#endif//_QRCODE_GPROT_H

## 添加资源 ##

那上面的这些 STR_ID_QRCODE_NOTE、SCR_QRCODE_MAIN怎么加入的呢？
先说下字符串资源

1. 首先，平台的资源放在plutommi/Customer/CustResource/PLUTO_MMI/ref_list.txt 文件中，请使用 excell 来开打文件，会看到一列一列的资源,在底部添加一句

```STR_ID_QRCODE_NOTE	Qrcode	note	tell the use to scan qrcode	Scan to banding	請掃描二維碼綁定	请扫描二维码绑定 ```

底部的数字加上你追加的条目+n； 

2. 在plutommi/Customer/CustResource/PLUTO_MMI/Res_MMI/下添加一个文件Res_QRcode.c。内容如下


		#ifndef __QRCODE_RES_DEF_H__
		#define __QRCODE_RES_DEF_H__
		#include "MMI_features.h"
		#include "MMIDataType.h"
		#include "CustDataProts.h"
		#include "GlobalMenuItems.h"
		#include "StdC.h"
		#ifdef DEVELOPER_BUILD_FIRST_PASS
		#include "PopulateRes.h"
		#include "QRcodeAppResDef.h"
		void PopulateQRcodeRes(void)
		{
		    ADD_APPLICATION_STRING2(STR_ID_QRCODE_NOTE,"Scan to bind","QRcode");
		
			
		
			   ADD_APPLICATION_IMAGE2(
		        IMG_ID_QRCODE,
		        CUST_IMG_PATH"\\\\MainLCD\\\\IdleScreen\\\\Wallpaper\\\\analog.bmp",
		        "analog image test");
		}
		
		#endif//DEVELOPER_BUILD_SECOND_PASS
		#endif

该文件主要讲我们在资源list中添加的字符串资源装载进来，同样的，将对应目录下的图片也可以通过第二个方法装载进来

3. 那么这个文件中的方法由谁来调用呢？我们需要修改 plutommi/mmi/Resource/PopulateRes.c

		//声明外部函数
		extern void PopulateQRcodeRes(void);
		.....
		
		#ifdef __MMI_VUI_ENGINE__
		    PopulateVCPRes();
		#endif
		RESPOP_LOG_V(("Populating QRCODE APP Resource"));
		PopulateQRcodeRes(); //追加我们的资源装载
	 

4. 在应用目录下的资源声明头文件中添加 QRcodeAppResDef.h

		#include "MMI_features.h"
		#include "MMIDataType.h"
		#include "mmi_res_range_def.h"
		/* Screen IDs */
		typedef enum
		{
		    SCR_QRCODE_MAIN = QRCODE_APP_BASE+1,
		}SCREEN_LIST_QRCODE;
		
		
		/* String IDs */
		
		typedef enum
		{
			STR_ID_QRCODE_NOTE = QRCODE_APP_BASE+1,
		}STRING_LIST_QRCODE;
		
		/* Image IDs */
		typedef enum
		{
			IMG_ID_QRCODE = QRCODE_APP_BASE+1,
		}IMAGE_LIST_QRCODE;

5.QRCODE_APP_BASE哪里来的？在plutommi/mmi/Inc/mmi_res_range_def.h，看文件名的大概意思就是，资源范围定义。追加：
	
		RESOURCE_BASE_RANGE(APP_QRCODE,              100),
	
		.....
		#define QRCODE_APP_BASE                        ((U16) GET_RESOURCE_BASE(APP_QRCODE))
		#define QRCODE_APP_BASE_MAX                   ((U16) GET_RESOURCE_MAX(APP_QRCODE))
		RESOURCE_BASE_TABLE_ITEM(APP_QRCODE)
不要问我为什么，反正其他应用也这么写，参照着写了；

6. 最后我们要把声明的这些id告诉给编译器，在plutommi/Custmer/ResGenerator/resgen_default_inc.txt中追加一行`../../MMI/QRcode/QRcodeInc`告诉编译器`QRcodeAppResDef.h`的位置，不然会找不到里面定义的这些id。


7. 图片的位置是放在plutommi/Customer/Images/屏幕对应尺寸/iamge.zip中



# 新版本差别 #
这篇总要是列出来新建一个应用的大体框架和资源添加的方式，新的版本可以通过在应用目录下添加res文件来声明资源ID。例如添加QRcode/QRcodeRes/QRcode.res文件，写法上也简便很多

	#include "MMI_features.h"
	#include "CustResDef.h"
	
	
	<?xml version="1.0" encoding="UTF-8"?>
	<APP id="APP_QIJI" type="venus" name ="STR_ID_QIJI_EXER">
	    <INCLUDE file="GlobalResDef.h"/>
	    <!----- String Resource Area ---------------------------------------------->
	    <STRING id="STR_ID_QRCODE_NOTE"/>
	   
	    <!----- Image Resource Area ----------------------------------------------->
		<IMAGE id="IMG_ID_QRCODE">CUST_IMG_PATH"\\\\MainLCD\\\\QiJi\\\\QJ_FONT24_0.png"</IMAGE>
	
	    <!----- Screen Resource Area ---------------------------------------------->
	    <SCREEN id="SCR_QRCODE_MAIN"/>
	   
	    <!----- Menu Resource Area ------------------------------------------------>
	
	
	    <!----- Timer Resource Area ---- ------------------------------------------>
	</APP>

最后需要将新加入的res文件加入编译，修改plutommi/mmi/Inc/mmi_pluto_res_range_def.h文件

	MMI_RES_DECLARE(APP_QRCODE, 100, ".\\mmi\\QRcode\\QRcodeRes\\")
	#define APP_QRCODE_BASE							((U16) GET_RESOURCE_BASE(APP_QRCODE))
	#define APP_QIJI_BASE_MAX						((U16) GET_RESOURCE_MAX(APP_QRCODE))

# 总结 #

我们大概了解了一个应用的搭建框架。看名字也知道，哈哈我现在要弄一个二维码的转换显示。虽然这个只是整个流程环节中微不足道的一环，但是面对全新的平台，还是捏了一把汗。但是，本宝宝最后还是完成了，点赞点赞。