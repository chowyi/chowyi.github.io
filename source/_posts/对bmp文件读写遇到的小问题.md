---
title: 对bmp文件读写遇到的小问题
date: 2013-04-17 21:19:29
tags:
- 图像读写
- Qt
categories:
- C++
---


先把代码放上来，算是个记录吧。
<!--more-->

- main.cpp
```
#include <QtGui/QApplication>
#include "mainwindow.h"
#include "rgb.h"
#include <windows.h>
#include <QFile>
#include <QDebug>

int main(int argc, char *argv[])
{

    QApplication a(argc, argv);
    MainWindow w;
    w.show();

    BITMAPFILEHEADER fileHeader;
    BITMAPINFOHEADER infoHeader;

    QFile testBmp("test.bmp");
    QFile outBmp("out.bmp");
    testBmp.open(QIODevice::ReadOnly);
    testBmp.read((char*)&fileHeader,sizeof(BITMAPFILEHEADER));
    testBmp.read((char*)&infoHeader,sizeof(BITMAPINFOHEADER));
    if(infoHeader.biBitCount == 24)
    {
        int infoSize = infoHeader.biHeight*infoHeader.biWidth*sizeof(RGB);
        int fillSize = fileHeader.bfSize-infoSize-sizeof(BITMAPFILEHEADER)-sizeof(BITMAPINFOHEADER);
        RGB img[infoHeader.biHeight][infoHeader.biWidth];
        testBmp.read((char*)img,infoSize);
        char fillByte[fillSize];
        testBmp.read(fillByte,fillSize);

        for(int i=0;i<infoHeader.biWidth;i++)
        {
            img[50][i].r = 0;
            img[50][i].g = 0;
            img[50][i].b = 0;
        }

        outBmp.open(QIODevice::WriteOnly);
        outBmp.write((char*)&fileHeader,sizeof(BITMAPFILEHEADER));
        outBmp.write((char*)&infoHeader,sizeof(BITMAPINFOHEADER));
        outBmp.write((char*)img,infoSize);
        outBmp.write(fillByte,fillSize);
        outBmp.close();
    }
    testBmp.close();
    
    return a.exec();
}
```

- rgb.h
```
#ifndef RGB_H
#define RGB_H

class RGB
{
public:
    RGB();
    unsigned char r;
    unsigned char b;
    unsigned char g;
};

#endif // RGB_H
```

第一次对图像进行读写，这里的代码是想做个试验。在第50行画条线。

问题来了，生成的图像是下面这样的：
![origin](origin.jpg)

仔细看右边就会发现，黑线偏移了一个像素。

放大图：
![zoom](zoom.png)


这里看得很清楚，画出来的黑线有一个像素变成了棕色，接下来一个像素向下偏移了一像素，并且是蓝色。

//原因还不知道，先记录下问题。

已经找到问题的原因了。
算是我对bmp储存结构的理解有问题。
在前面的blog中已经对bmp格式的数据结构有了详细介绍。
位图按行存贮，位图的宽度不等于位图的行字节数。每个像素占用 biBitCount/8 个字节，且每行占用的字节数必须是4字节的整数倍！不是4字节的整数倍时要进行填充。
注意：要对每一行都进行填充，填充的字节跟在每一行的后面！ 一开始我就是这里理解错了。