---
title: 《一起由struct引发的C/C++内存分配和IO的思考》
description:  
date:  Sun, 27 Sep 2020 22:21:42 +0800
tags:
  - CPP
categories:
- CPP
---

##### 1. 奇奇怪怪的BUG

代码：

~~~c++
#include <iostream>
using namespace std;
struct Student{
    char num[3];
    char name[20];
}student;
int main(){
    cin >>  student.num;
	cin >> student.name;
	cout << student.num << endl;
    return 0;
}
~~~

输入：111 name

输出：111name

##### 2. 上述BUG产生的原因

​	之前看到一个数据说：C/C++项目的bug，80%都是源于内存错误。之前以为是但是很多时候真的是因为有些错误实在是不明显。

例如在上述bug中，num定义为char 3， name 定义为char 10。并且在输入的时候也没有什么问题

但是因为输入的num是三位数，并且cin在直接对char*类型输入时会在末尾设置一位'\0'。



但是在struct中，内存是分配在一起的，因此当输入的字符，超过了3位时，‘/0’。刚好赋值在了name的起始位置。但是因为name是之后输出的，所以原本属于num的终止符‘/0’。被name覆盖了。所以在输出num的时候，内存中的实际分布是”111name\0“；而C/C++的输出机制是遇到 '\0'才终止，因此



