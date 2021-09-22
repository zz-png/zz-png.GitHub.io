---
title: 算法笔记-Chapter2
date:  2021-09-18 19:23:14
category: note
tags: algorithm
excerpt: 算法笔记第二章知识总结
---

### 数据类型

- 长整型long long，赋大于2的31次方-1的初值时要在初值后面加LL，否则会编译报错？<br>
不加也没事？可能与编译器版本有关。

- 浮点型，能用double就double，尽量不要使用float（精度不够）

- 字符型，字符常量（必须是单个字符）必须用**单引号**（双引号不行）标注
- 字符串常量是由**双引号**标记的字符集，如"HelloWorld"就是一个字符串常量
<br>不能把字符串常量赋值给字符变量，如`char c = "abcd"`是不允许的
<br>尽量不要用printf输出string类型，会出错（可以使用cout或者用char[]）

- 布尔型，C++中可以直接使用，C语音中需要添加stdbool.h头文件
<br>true和false存储时分别为1和0;整型常量赋值给布尔型变量时会自动转换为true(非零,如1和-1)和false（零）

### 输入输出

- double类型输入格式符为%lf,输出格式符为%f，有时把输出格式写成%lf倒也不会出错，但尽量按标准来

- scanf的%c是可以读入空格跟换行的，除了%c以外，对其它格式符(如%d，%s)的输入是以空白符（空格，换行等）为结束标志的

- VS2019,scanf -> scanf_s，gets -> gets_s(不然会报错)

- gets_s可以接收空格，以换行符\n作为输入结束标志，因此如果scanf了一个整数（或者其它的类型）后用gets_s，需要先用getchar接收整数后的换行符（否则会认为输入了换行符或空白？就结束了）

- 使用getchar函数输入字符串时，一定要在末尾加入'\0'，否则printf和puts会因为无法识别字符串末尾而输出一堆乱码
<br>使用gets_s或scanf_s时会在输入的字符串后面自动添加空字符\0，并占用一个字符位

- 输出格式， %md（高位补空格），%0md（高位补0），%.mf（保留m位小数）

### typedef

给复杂的数据类型起一个别名，在使用时就可以用别名来代替原来的写法

```C++
#include<cstdio>
typedef long long LL;
int main(){
	LL a = 1234567890123LL;
	printf("%lld\n",a);
	return 0;
}
```

### 冒泡排序

```C++
//从小到大，N为数组arr[]大小
for(int i = 0; i < N - 1; i++)
	for (int j = 0; j < N - 1 - i; j++) {
		if (arr[j] > arr[j + 1]) {
			int temp = arr[j];
			arr[j] = arr[j + 1];
			arr[j + 1] = temp;
		}
	}
```

### memset(数组名，值，sizeof(数组名))——对数组中每一个元素赋相同的值

需要添加头文件string.h或cstring，并且memset使用的是按字节赋值，建议使用memset赋0或者-1
<br>因为0的二进制补码为全0，-1的二进制补码为全1
<br> [memset赋值0和-1，还能赋其他值吗？](https://blog.csdn.net/a474617146/article/details/69788105)

### string.h头文件

- strlen()

- strcmp(字符数组1，字符数组2)，返回两个字符串大小的比较结果，比较原则按字典序
<br>字符数组1 < 字符数组2，返回一个负整数（不一定是-1）
<br>字符数组1 == 字符数组2，返回0
<br>字符数组1 > 字符数组2，返回一个正整数（不一定是+1）

- strcpy(字符数组1，字符数组2)，把**字符数组2**复制给**字符数组1**

- strcat(字符数组1，字符数组2)，把字符数组2拼接到字符数组1后面

### sscanf与sprintf

- sscanf可理解为string+scanf，sprintf可理解为string+printf

```C++
//把字符数组str中的内容以%d的格式写到n中
sscanf(str,"%d",&n);
//把n以%d的格式写到str字符数组中
sprintf(str,"%d",n);
```

### 结构体

定义一个结构体的例子

```C++
struct studentInfo{
	int id;
	char name[20];
	char gender;
}Alice,Bob,stu[1000];

//也可以按照基本数据类型定义
studentInfo Tom;
studentInfo student[100];

//结构体内能定义除了自己本身外的任何数据类型
//虽然不能定义本身，但是可以定义自身类型的指针变量
struct node{
	node n;  //不能定义node型变量
	node* next;  //可以定义node*型指针变量
};
```