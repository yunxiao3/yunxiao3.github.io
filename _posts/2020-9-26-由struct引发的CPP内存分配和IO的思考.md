---
title: 《一起由struct引发的C/C++内存分配和IO的思考》
description:  
date:  Sun, 27 Sep 2020 22:21:42 +0800
tags:
  - CPP
categories:
- CPP
---



代码：

~~~c++
#include <iostream>
using namespace std;
struct student{
    char num[3];
    char name[20];
}students;
int main(){
            cin >>  students.num;
			cin >> students.name;
			cout << students.num << endl;
    return 0;
}
~~~

输入：111 name

输出：111name



待续.....

