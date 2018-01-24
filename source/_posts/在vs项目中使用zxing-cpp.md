---
title: 在vs项目中使用zxing-cpp
date: 2018-01-24 12:28:17
categories:
 编码人生
tags:
 [Zxing]
---
<Excerpt in index | 首页摘要>
<!-- more -->
<The rest of contents | 余下全文>

#在vs项目中使用zxing-cpp
### 前言
    最近接手一个项目需要识别通过监控摄像机识别二维码，本客户端已经做好RtspClient收流和H.264和H.265解码。这就只需要通过在播放器解码后获取到YUV数据解析得到二维码数据。
### 环境与工具
>* window10
>* visual studio 2017
>* OpenCv 3.4.0
>* cmake gui

## zxing-cpp系列
>* [Zxing-cpp的编译](https://alvincyy.github.io/2018/01/23/win32下zxing的使用/)<br>
>* [在vs项目中使用zxing-cpp](https://alvincyy.github.io/2018/01/23/在vs项目中使用zxing-cpp/)<br>

## 正文
  通过前面，我们已经获得了lib文件和实例Demo，但是实例Demo是解析图片文件，而项目需求是解析YUV数据，这就需要修改成解析Yuv数据。虽然也可以通过将YUV数据保存成文件后在识别，但是因为在项目的实时传输流情况下不能影响视频的播放和显示，显然不现实，而且YUV数据直接解析二维码效率更高。
  因为已经生成了静态库lib库，和头文件(code/src文件夹中)。可以直接导入已经存在的工程，直接开始使用。如果不知道怎么导入lib库请找度娘。
  导入后因为原先的工程是VS2005的导致这个2017编译的静态库总是出现接口未定义的错误，最后无奈又重新用vs2005将zxing-cpp编译了一次。再次导入出现了未找到stdint.h头文件的错误，因为stdint.h是c99标准的头文件，vs2005不支持从VS2017的\VC\include文件夹找到复制到vs2005这个目录下面。
  再次运行成功。
  开始编写支持YUV数据解析的接口。具体的过程就不一一道明了，直接放代码。
  参考Demo实例的ImageReaderSource类编写出来CBufferReaderSource类
 >* BufferReaderSource.h文件
 
```c++
#pragma once
#include <zxing/LuminanceSource.h> 
#include <stdio.h>  
#include <stdlib.h>  
using namespace zxing; 

class CBufferReaderSource :
	public zxing::LuminanceSource
{
private:  
	typedef LuminanceSource Super;  
	int width, height;   
	ArrayRef<char> buffer;   
public:
	CBufferReaderSource(int inWidth, int inHeight, ArrayRef<char> inBuffer);
	~CBufferReaderSource(void);
	int getWidth() const;   
	int getHeight() const;   
	ArrayRef<char> getRow(int y, ArrayRef<char> row) const;   
	ArrayRef<char> getMatrix() const;
};
```
 >* BufferReaderSource.cpp文件
 
```c++
#include "BufferReaderSource.h"

CBufferReaderSource::CBufferReaderSource(int inWidth, int inHeight, ArrayRef<char> inBuffer)
 :Super(inWidth,inHeight),buffer(inBuffer)
{
	width = inWidth;   
	height = inHeight;   
	buffer = inBuffer;
}

CBufferReaderSource::~CBufferReaderSource(void)
{
}

int CBufferReaderSource::getWidth() const  
{  
	return width;   
}  

int CBufferReaderSource::getHeight() const  
{  
	return height;   
}  

ArrayRef<char> CBufferReaderSource::getRow(int y, ArrayRef<char> row) const  
{  
	if (y < 0 || y >= height)   
	{  
		fprintf(stderr, "ERROR, attempted to read row %d of a %d height image.\n", y, height);   
		return NULL;   
	}  
	// WARNING: NO ERROR CHECKING! You will want to add some in your code.   
	if (row == NULL) row = ArrayRef<char>(getWidth());  
	for (int x = 0; x < width; x ++)  
	{  
		row[x] = buffer[y*width+x];   
	}  
	return row;   
}  

ArrayRef<char> CBufferReaderSource::getMatrix() const  
{  
	return buffer;   
}  
```
>* ParseQRInfo.h 文件

```c++
#pragma once
//#include <iostream>
//#include <fstream>
//#include <string>
#include "BufferReaderSource.h"
#include "ImageReaderSource.h"
#include <zxing/common/Counted.h>
#include <zxing/Binarizer.h>
#include <zxing/MultiFormatReader.h>
#include <zxing/Result.h>
#include <zxing/ReaderException.h>
#include <zxing/common/GlobalHistogramBinarizer.h>
#include <zxing/common/HybridBinarizer.h>
#include <exception>
#include <zxing/Exception.h>
#include <zxing/common/IllegalArgumentException.h>
#include <zxing/BinaryBitmap.h>
#include <zxing/DecodeHints.h>
#include <zxing/qrcode/QRCodeReader.h>
#include <zxing/multi/qrcode/QRCodeMultiReader.h>
#include <zxing/multi/ByQuadrantReader.h>
#include <zxing/multi/MultipleBarcodeReader.h>
#include <zxing/multi/GenericMultipleBarcodeReader.h>

using namespace std;
using namespace zxing;
using namespace zxing::multi;
using namespace zxing::qrcode;
namespace {  

	bool more = false;  
	bool test_mode = false;  
	bool try_harder = false;  
	bool search_multi = false;  
	bool use_hybrid = false;  
	bool use_global = false;  
	bool verbose = false;  

} 

class CParseQRInfo
{
public:
	CParseQRInfo(void);
	~CParseQRInfo(void);
	BOOL parseQRInfo(int width,int height,char* buffer,string &QRResult);
	BOOL parseQRInfo(string filename,string& QRResult);
};
```
>* ParseQRInfo.cpp

```c++
#include "StdAfx.h"
#include "ParseQRInfo.h"

CParseQRInfo::CParseQRInfo(void)
{
}

CParseQRInfo::~CParseQRInfo(void)
{
}

BOOL CParseQRInfo::parseQRInfo(int width,int height,char* buffer,string &QRResult){  
	try{  
		// Convert the buffer to something that the library understands.   
		ArrayRef<char> data((char*)buffer, width*height);  
		Ref<LuminanceSource> source (new CBufferReaderSource(width, height, data));   

		// Turn it into a binary image.   
		Ref<Binarizer> binarizer (new GlobalHistogramBinarizer(source));   
		Ref<BinaryBitmap> image(new BinaryBitmap(binarizer));  

		// Tell the decoder to try as hard as possible.   
		DecodeHints hints(DecodeHints::DEFAULT_HINT);   
		hints.setTryHarder(true);   
		hints.addFormat(BarcodeFormat::QR_CODE);
		// Perform the decoding.   
		QRCodeReader reader;  
		Ref<Result> result(reader.decode(image, hints));  

		// Output the result.   
		cout << result->getText()->getText() << endl;  
		QRResult = result->getText()->getText();  
		

	}  
	catch (zxing::Exception& e)   
	{  
		cerr << "Error: " << e.what() << endl; 
		//DebugLog("QR show Failed = %s",e.what());
		return false;  
	}  
	return true;  

}  
```
>* 下面一小段代码是我在项目中调用这个接口。

```c++
CParseQRInfo test;
	string str;
	if (test.parseQRInfo(w,h,pBuffer,str))
	{
		if (str.length() > 0)
		{
			USES_CONVERSION;
			if (m_pParent && m_pParent->m_strQRcode.Compare(A2W(str.c_str())) != 0)
			{
				m_pParent->m_strQRcode = A2W(str.c_str());
				DebugLog("QRcode = %S",m_pParent->m_strQRcode);
				if (m_pParent->m_fQRCallback)
				{
					m_pParent->m_fQRCallback(m_pParent->m_nPlayStreamId, m_pParent->m_strQRcode, m_pParent->m_pCallBackUserDataQR);
				}
			}
		}

	}
```

>* 以上是关于解析YUV数据中二维码的代码，至于解析图片中的二维码在源代码的实例代码中就有，就不在这献丑了。

> 参考文献
>* [如何在visual studio下编译zxing cpp，以及zxing c++的使用](http://blog.csdn.net/sinat_29957455/article/details/60467090)<br>
>* [C++用zxing识别二维码](http://blog.csdn.net/coolingcoding/article/details/25804129)<br>

> 最后的话<br>
>* 如果遇到问题可以给我发邮件[chenyiyu@gmail.com](mailto:chenyiyu@gmail.com)<br>
>* 转载请注明原地址，[Alvin的博客](http://alvinCyy.github.io) 谢谢！

