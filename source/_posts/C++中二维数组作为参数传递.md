---
title: C++中二维数组作为参数传递
date: 2013-09-15 21:19
tags:
- C++
categories:
- C++
---


c++中参数不能是二维数组，可以将二维数组转为一维数组传递。
<!--more-->

```
//可以强制转为一维指针
#include <stdio.h>

void disp(int *a, int m, int n)
{
    int i, j;
    for (i=0; i < m; i++)
    {
        for (j=0; j < n; j++)
            printf("%2d", a[n*i+j]);
        putchar('\n');
    }
}

int main()
{
    int a[2][2] = {1, 2, 3, 4};
    disp((int *)a, 2, 2);
    return 0;
}
```