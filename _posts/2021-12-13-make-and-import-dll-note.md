---
title: DLL动态链接库创建与调用
date:  2021-12-13 11:31:39
category: work
tags: DLL
excerpt: 创建与调用DLL动态链接库总结
---

### 一、创建DLL动态链接库

#### （1）打开VS2019，创建新项目
项目类型选择**动态链接库(DLL)**，自定义项目名称以及工程位置，点击创建。工程初始界面包括4个文件（两个头文件和两个源文件）:
+ 头文件
    1. framework.h
    2. pch.h
+ 源文件
    1. dllmain.cpp
    2. pch.cpp
   
上述4个文件都不需要做任何修改，我们只需要实现自己的动态链接库的头文件以及源文件

#### （2）demo示例
头文件：myDLL.h，用来声明需要导出的函数接口

```C++
#pragma once
#include<stdio.h>
#if 1
#define DLL_API extern "C" _declspec(dllexport)
#else
#define DLL_API extern "C" _declspec(dllimport)
#endif

struct Point
{
	float x, y;
};

DLL_API int add(int a, int b);
DLL_API float addf(float a, float b);
DLL_API void Print_point(Point* p);
```

`extern "C"`用于告诉编译器将被它修饰的代码按C语言的方式进行编译，更详细的用法可以参考[extern "C"的用法解析](https://www.cnblogs.com/rollenholt/archive/2012/03/20/2409046.html)<br>

`_declspec(dllexport)`用于告诉编译器和链接器被它修饰的函数或变量需要从DLL中导出，供其它应用程序使用；相应的，`_declspec(dllimport)`说明被它修饰的函数或变量需要从DLL中导入<br>

源文件：myDLL.cpp，实现函数接口

```C++
#include "pch.h"
#include "myDLL.h"

int add(int a, int b) {
	return a + b;
}

float addf(float a, float b) {
	return a + b;
}

void Print_point(Point* p) {
	if (p) {
		printf("x: %f, y: %f", p->x, p->y);
	}
}
```

#### （3）生成解决方案，编译DLL
编译成功后出现无法启动程序的错误是正常现象，点击确定即可。此时在Debug文件夹下已经生成了\$(项目名称).lib以及\$(项目名称).dll

### 二、调用DLL动态链接库

#### （1）创建新的工程项目
重新创建一个需要调用DLL的工程，选择**空项目**即可，自定义项目名称以及工程位置

#### （2）配置环境
将上一节中编写的头文件**myDLL.h**以及生成的动态链接库**testDLL.dll**和**testDLL.lib**复制到当前项目文件夹下，然后在头文件下添加现有项，选择myDLL.h<br>

在VS侧边栏**解决方案资源管理器**中选中项目，右键 -> 属性 -> 链接器 -> 常规(/输入) -> 附加库目录(/附加依赖项)
1. 在附加库目录中添加**testDLL.lib**文件路径
2. 在附加依赖中添加**testDLL.lib**项，以分号间隔
3. (可选，没做好像也没问题)在C/C++ -> 常规 -> 附加包含目录中添加调用的DLL动态链接库(不确定...)所在路径

#### （3）修改myDLL.h头文件，编写测试程序
将上述复制过来的myDLL.h文件中的`DLL_API`宏定义修改为`extern "C" _declspec(dllimport)`，即被它修饰的函数或变量需要从DLL中导入，修改后的myDLL.h头文件为

```C++
#pragma once
#include<stdio.h>
#define DLL_API extern "C" _declspec(dllimport)

struct Point
{
	float x, y;
};

DLL_API int add(int a, int b);
DLL_API float addf(float a, float b);
DLL_API void Print_point(Point* p);
```

创建测试用例test.cpp源文件，生成解决方案，成功运行

```C++
#include "myDLL.h"

int main() {

	int x = add(12, 25);
	printf("Yes, x: %d\n", x);
	float y = addf(10.0, 13.4);
	printf("y: %f", y);
    Point p;
	p.x = 1.3;
	p.y = 2.4;
	Print_point(&p);
	return 0;
}
```

### Reference
1. [Dll的分析与编写](https://www.cnblogs.com/hicjiajia/archive/2010/08/27/1809997.html)
2. [VS2019环境下C++动态链接库（DLL）的创建与调用](https://blog.csdn.net/qq_30139555/article/details/103621955?utm_term=vs2019%E5%8A%A0%E8%BD%BDdll%E6%96%87%E4%BB%B6&utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~all~sobaiduweb~default-0-103621955&spm=3001.4430)

**PS: 注意应用程序以及所创建的DLL动态链接库是32位还是64位的**