---
title: bmp格式
date: 2013-04-17 21:19:29
tags:
- bmp格式
categories:
- C++
---


摘自：http://blog.csdn.net/you_lan_hai/article/details/6770471

24位以上的位图文件包含3个部分：位图文件头（BITMAPFILEHEADER）、位图信息头(BITMAPINFOHEADER)、位图数据。
<!--more-->

下面是MSDN中两个信息头的具体定义：
```
typedef struct tagBITMAPFILEHEADER { 
  WORD    bfType; //图像类型：必须是‘BM’(BM的16进制编码为：0x4d42)
  DWORD   bfSize; //文件大小。包含着两个结构，以及数据的总大小。

  WORD    bfReserved1; //保留值。必须为0
  WORD    bfReserved2; //保留值。必须为0
  DWORD   bfOffBits;   //从文件开头到数据的偏移量。也就是结构BITMAPFILEHEADER和结构BITMAPINFOHEADER的大小

} BITMAPFILEHEADER, *PBITMAPFILEHEADER; 
typedef struct tagBITMAPINFOHEADER{
  DWORD  biSize; //当前结构体的大小。
  LONG   biWidth; //位图的宽度。单位是像素
  LONG   biHeight; //位图高度。单位是像素
  WORD   biPlanes; //位图平面个数。必须是1
  WORD   biBitCount //位图的位数，也就是位图深度。可以是1、4、8、16、24、32。16位以下的位图文件含有调色板信息。
  DWORD  biCompression; //压缩方式。位图没有压缩，赋予BI_RGB参数（也就是0）。
  DWORD  biSizeImage; //位图数据占用的字节数。可以设置为默认值0。
  LONG   biXPelsPerMeter; //指定目标设备水平分辨率，单位是每米的像素个数。可设置为0
  LONG   biYPelsPerMeter; //指定目标设备垂直分辨率，同上。
  DWORD  biClrUsed; //指定本图像实际用到的像素。可设置为0
  DWORD  biClrImportant; //指定本图像重要的颜色个数。可设置为0，表示所有颜色都重要。
} BITMAPINFOHEADER, *PBITMAPINFOHEADER; 
```

接下来就是位图数据了：

位图按行存贮，位图的宽度不等于位图的行字节数。每个像素占用 biBitCount/8 个字节，且每行占用的字节数必须是4字节的整数倍！例如：101\*101\*24的位图，行占用字节数为：101\*3=303，而303%4！=0，所以行 占用字节数需修改为304，此时位图数据的实际大小为304\*101，上面的biSizeImage就可设置为304\*101。

下面是保存位图文件的结构体填充例：
```
BITMAPFILEHEADER bmheader;
    memset(&bmheader,0,sizeof(bmheader));
    bmheader.bfType=0x4d42;     //图像格式。必须为'BM'格式。
    bmheader.bfOffBits = sizeof(BITMAPFILEHEADER)+sizeof(BITMAPINFOHEADER); //从文件开头到数据的偏移量
    bmheader.bfSize = m_nWidthBytes*m_nHeight + bmheader.bfOffBits;//文件大小

    BITMAPINFOHEADER bmInfo;
    memset(&bmInfo,0,sizeof(bmInfo));
    bmInfo.biSize = sizeof(bmInfo);
    bmInfo.biWidth = m_nWidth; 
    bmInfo.biHeight = m_nHeight; 
    bmInfo.biPlanes = 1; 
    bmInfo.biBitCount =  m_nDeep;
    bmInfo.biCompression = BI_RGB;
```

注：m_nWidthBytes 可这样计算：int m_nWidthBytes = ((m_nWidth\*m_nDeep/8 + 3)/4) \* 4;