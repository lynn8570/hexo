---
title: "MTK feature phone上生成二维码"
date: "2016-03-10"
categories: 
    - "QRCODE"
---
# 添加LibQREncode #

首先呢，我们在github上可以搜到很多LibQREncode相关的项目，LibQREncode使用C语言编写的二维码编码库 github项目地址：[libqrencode](https://github.com/fukuchi/libqrencode)

然后我还不小心找到一个QRCODE 生成器的c++代码项目
[QRGenerator](https://github.com/thomasdic2000/QRGenerator/)

那么我接下来的工作就比较简单啦，就将这个C++的二维码生成器移植到feature phone平台上，稍作改良。


# 生成二维码bmp图片 #

根据QRGenerator可以很快改造一下，调用libqrencode库生成二维码bmp图片。代码如下：

bitmap图片编码的一些头文件定义：

	#define OUT_FILE_PIXEL_PRESCALER		4
	#define PIXEL_COLOR_R				0
	#define PIXEL_COLOR_G				0
	#define PIXEL_COLOR_B				0
	//rgb 0 0 0,代表 黑色
	#define BI_RGB						0L
	#pragma pack(push, 1)
	
	typedef struct	
	{
		U16	bfType;
		U32	bfSize;
		U16	bfReserved1;
		U16	bfReserved2;
		U32	bfOffBits;
	} BITMAPFILEHEADER;
	
	typedef struct 
	{
		U32	   biSize;
		S32	   biWidth;
		S32	   biHeight;
		U16	   biPlanes;
		U16	   biBitCount;
		U32	   biCompression;
		U32	   biSizeImage;
		S32	   biXPelsPerMeter;
		S32	   biYPelsPerMeter;
		U32	   biClrUsed;
		U32	   biClrImportant;
	} BITMAPINFOHEADER;
	
	typedef struct
	{
		BITMAPFILEHEADER kFileHeader;
		BITMAPINFOHEADER kInfoHeader;
		unsigned char g_rgb_data[100000];
	} BITMAPDATA;
	#pragma pack(pop)

实现二维码编码具体代码：
	BITMAPDATA bitmapData;//定义一个全局变量，用户存放编码图片信息

	// ox要显示的图片的x坐标，Y坐标
    // resize_w 调整图片大小
    // szSourceString 要转换为二维码的字符串，比如网址啊之类的
	int mmi_getQRcode(S32 ox,S32 oy,S32 resize_w, char* szSourceString){
		//char* szSourceString = QRCODE_TEXT;
		unsigned int unWidth,x,y,l,n,unWidthAdjusted,unDataBytes;
		unsigned char* pRGBData,*pSourceData,*pDestData;
		QRcode*  pQRC;
		S32 fd;
		S32 fs_ret;
		U32 nLen;
		S32 w,h,rec_x,rec_y,rec_w;
		
		S32 resize_h=resize_w;
	
		pQRC = QRcode_encodeString(szSourceString,0,QR_ECLEVEL_H,QR_MODE_8,1);
		//调用lib库中的函数，获取二维码编码信息。然后接下来的工作都是将获取的这个二维码编码信息转换为图片
		unWidth = pQRC->width;
		kal_prompt_trace(MOD_MMI,"%d unWidth --linlian .\n",unWidth);
		unWidthAdjusted = unWidth * OUT_FILE_PIXEL_PRESCALER * 3;
		kal_prompt_trace(MOD_MMI,"%d unWidthAdjusted --linlian .\n",unWidthAdjusted);
		if (unWidthAdjusted % 4)
			unWidthAdjusted = (unWidthAdjusted / 4 + 1) * 4;
	
		unDataBytes = unWidthAdjusted * unWidth * OUT_FILE_PIXEL_PRESCALER;
	
		kal_prompt_trace(MOD_MMI,"%d unDataBytes --linlian .\n",unDataBytes);
	
		if (unDataBytes > sizeof(bitmapData.g_rgb_data))
		{
			kal_prompt_trace(MOD_MMI,"%d unDataBytes out of memory --linlian .\n",unDataBytes);
			return 0; // out of memory
		}
		pRGBData = bitmapData.g_rgb_data;
		
		memset(pRGBData, 0xff, unDataBytes);//将所有的点设置为0XFF，则为白色
		bitmapData.kFileHeader.bfType = 0x4d42;  // "BM"
		bitmapData.kFileHeader.bfSize =	sizeof(BITMAPFILEHEADER) +
								sizeof(BITMAPINFOHEADER) +
								unDataBytes;
		bitmapData.kFileHeader.bfReserved1 = 0;
		bitmapData.kFileHeader.bfReserved2 = 0;
		bitmapData.kFileHeader.bfOffBits =	sizeof(BITMAPFILEHEADER) +
								sizeof(BITMAPINFOHEADER);
		bitmapData.kInfoHeader.biSize = sizeof(BITMAPINFOHEADER);
		bitmapData.kInfoHeader.biWidth = unWidth * OUT_FILE_PIXEL_PRESCALER;
		bitmapData.kInfoHeader.biHeight = ((int)unWidth * OUT_FILE_PIXEL_PRESCALER);
		bitmapData.kInfoHeader.biPlanes = 1;
		bitmapData.kInfoHeader.biBitCount = 24;
		bitmapData.kInfoHeader.biCompression = BI_RGB;
		bitmapData.kInfoHeader.biSizeImage = 0;
		bitmapData.kInfoHeader.biXPelsPerMeter = 0;
		bitmapData.kInfoHeader.biYPelsPerMeter = 0;
		bitmapData.kInfoHeader.biClrUsed = 0;
		bitmapData.kInfoHeader.biClrImportant = 0;
		pSourceData = pQRC->data;
		qijiReverseData(pSourceData,unWidth);//linlian@2016.03.04 reverse qrcode image vertically
		for(y = 0; y < unWidth; y++)
		{
			pDestData = pRGBData + unWidthAdjusted * y * OUT_FILE_PIXEL_PRESCALER;
			for(x = 0; x < unWidth; x++)
				{
				if (*pSourceData & 1)//在pSourceData数组中值为1的点，将其颜色设置为BGR 000，则对应点变为黑色
					{
					for(l = 0; l < OUT_FILE_PIXEL_PRESCALER; l++)
						{
						for(n = 0; n < OUT_FILE_PIXEL_PRESCALER; n++)
							{
							*(pDestData +		n * 3 + unWidthAdjusted * l) =	PIXEL_COLOR_B;
							*(pDestData + 1 +	n * 3 + unWidthAdjusted * l) =	PIXEL_COLOR_G;
							*(pDestData + 2 +	n * 3 + unWidthAdjusted * l) =	PIXEL_COLOR_R;
							}
						}
					}
				pDestData += 3 * OUT_FILE_PIXEL_PRESCALER;
				pSourceData++;
			}
		}
		//前面这段代码的意思是，初始化好bmp图片文件的头部。
		//并且在后面追加每个点的颜色定义。最后我们获取了一系列黑白点阵


		//OslMfree(pRGBData);
		nLen = sizeof(BITMAPFILEHEADER)+sizeof(BITMAPINFOHEADER)+sizeof(unsigned char)*unDataBytes;
		gdi_image_get_dimension_mem(GDI_IMAGE_TYPE_BMP,(U8*)&bitmapData,nLen,&w,&h);
		kal_prompt_trace(MOD_MMI," %d w --linlian .\n",w);
		kal_prompt_trace(MOD_MMI," %d h --linlian .\n",h);
	
		img_width =w;
		img_height=h;
		
		if(w < resize_w || resize_w==0){ //can't be larger than original width
			resize_w = w;
			resize_h = h;
		}
		if(resize_w>100){
			resize_w =100;
			resize_h =100;
		}
		if(ox<=0||oy<=0){
			ox=(UI_device_width-resize_w)/2;
			oy=(UI_device_height-resize_h)/2+10;
		}
		kal_prompt_trace(MOD_MMI," %d resize_w --linlian .\n",resize_w);
		kal_prompt_trace(MOD_MMI," %d resize_h--linlian .\n",resize_h);
	    //gdi_image_draw_mem(ox,oy,(U8*)&bitmapData,GDI_IMAGE_TYPE_BMP,nLen);
	
		rec_x = ox-2;
		rec_y = oy-2;
		rec_w = resize_w +4;
		gdi_draw_solid_rect(rec_x,rec_y,rec_x+rec_w,rec_y+rec_w,GDI_COLOR_WHITE);
		gdi_image_draw_resized_mem(ox,oy,resize_w,resize_h,(U8*)&bitmapData,GDI_IMAGE_TYPE_BMP,nLen);
	
		preWidth = resize_w;
		QRcode_free(pQRC);
		return 1;
	}


这边要说明一下，在pc平台上，将`bitmapData.kInfoHeader.biHeight = -((int)unWidth * OUT_FILE_PIXEL_PRESCALER);` 也就是biHeight设置为负值的话，可以实现图片的垂直翻转，但是在feature phone上，如果biHeight直接设置为负值，那么图片将不被识别，所以，我们只能将biHeight设置为`+((int)unWidth * OUT_FILE_PIXEL_PRESCALER)`。因此，在原始的0101数据开始被rgb化的时候，我们先将这串数据组成一个二维正方形数组，并进行以中间高度为中心的上下翻转。即`qijiReverseData(pSourceData,unWidth);`。对应的代码为

	// vertical reverse image 
	void qijiReverseData(unsigned char* data, unsigned int unWidth){
		unsigned int i,j;
		unsigned char temp;
		for( i=0 ;i<(unWidth/2);i++){
			for(j=0;j<unWidth;j++){
				temp = data[j+i*unWidth];
				data[j+i*unWidth] = data[(unWidth-1-i)*unWidth+j];
				data[(unWidth-1-i)*unWidth+j]=temp;
			}
		}
	}

# 平台适配调整 #
基本上qrcode lib库的代码在feature phone的模拟器上运行时没有问题的，但是在实际的机子上，需要调整一些内存分配等方法，进行平台适配。例如：



- data = (unsigned char *)**malloc**(bstream->length + arg->length);
需要改为
data = (unsigned char *)**OslMalloc**(bstream->length + arg->length);
- **free**(bstream->data); --> **OslMfree**(bstream->data);
- raw->rsblock = (RSblock *)**calloc**(raw->blocks, sizeof(RSblock));改为	raw->rsblock = (RSblock *)**OslMalloc**((raw->blocks)*(sizeof(RSblock)));memset(raw->rsblock, 0x00, (raw->blocks)*(sizeof(RSblock)));

# 总结 #

有时候我们都不知道自己是怎么完成这一步步的瞎子摸象。例如我突然想不起来，曾经写过一个给校准树写DLL的经历，我只记得接到任务的时候，只有“懵逼”来形容，什么事校准树，什么事DLL，特么我是写android的，C是什么鬼，我只在大学学过C++。然后，我在万用的路上越走越远。也许有圣人相助，反正每个感觉完不成的任务，最后也都完成了，再见了feature phone~~~~~~~