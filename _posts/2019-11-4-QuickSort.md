---
title: 《漫谈数据结构与算法之：Quick Sort (1)》
description:
date:  Tue, 05 Apr 2022 20:21:13
tags:
  - Algorithm
categories:
- Algorithm
---

最近闲来无事，重新学习了一下算法，选用的教材则是大名鼎鼎的《算法导论》。不得不说算法导论真的是一本神书，只是当年自己过于年轻而不得其精髓。在看算法导论之前，自己对快排的理解只停留在实现和使用上，但是对里面的思想却一无所知。总的来说快排是分治思想的一种体现，而书中对快排优化时采用的随机算法和概率分析，让我不禁感慨理论才是指导实践的第一标准。

#### 1. 经典的快排算法

快速排序的基本思想是：先选一个“主元”，用它对整个待排序列进行筛选，以保证：其左边的元素都不大于它，其右边的元素都不小于它。这样，排序问题就被分割为两个子区间。再分别对子区间排序就可以了。

##### 简单的递归实现：

~~~C
void QuickSort(int* array,int left,int right){
    //表示已经完成一个组
	if(left >= right){ 
		return;
	}
    //获取枢轴的位置，并保证主元左边的元素都不大于它，其右边的元素都不小于它
	int index = PartSort(array,left,right);
	QuickSort(array,left,index - 1);
	QuickSort(array,index + 1,right);
}
~~~

其中代码的核心部分便是 **PartSort()**函数，而PartSort()函数又有三种实现方式，包括 **左右指针法，挖坑法和前后指针法**。下面分别介绍这三种算法的实现方式和原理：

##### 1. 左右指针法

**原理：**

>1. 选取一个关键字(key)作为枢轴，一般取整组记录的第一个数/最后一个，这里采用选取序列最后一个数为枢轴。
>2. 设置两个变量left = 0;right = N - 1;从left一直向后走，直到找到一个大于key的值，right从后至前，直至找到一个小于key的值，然后交换这两个数。
>3. 重复第三步，一直往后找，直到left和right相遇，这时将key放置left的位置即可。

**代码：**

~~~c
int PartSort(int* array,int left,int right){
	int& key = array[right];
	while(left < right){
		while(left < right && array[left] <= key){
			++left;
		}
		while(left < right && array[right] >= key){
			--right;
		}
		swap(array[left],array[right]);
	}
	swap(array[left],key);
	return left;
}
~~~

##### 2. 挖坑法

**原理：**

>1. 选取一个关键字(key)作为枢轴，一般取整组记录的第一个数/最后一个，这里采用选取序列最后一个数为枢轴，也是初始的坑位。
>2. 设置两个变量left = 0;right = N - 1;
>3. 从left一直向后走，直到找到一个大于key的值，然后将该数放入坑中，坑位变成了array[left]。
>4. right一直向前走，直到找到一个小于key的值，然后将该数放入坑中，坑位变成array[right]。
>5. 重复3和4的步骤，直到left和right相遇，然后将key放入最后一个坑位。

**代码：**

~~~c
int PartSort(int* array,int left,int right){
	int key = array[right];
	while(left < right){
		while(left < right && array[left] <= key){
			++left;
		}
		array[right] = array[left];
		while(left < right && array[right] >= key){
			--right;
		}
		array[left] = array[right];	 
	}
	array[right] = key;
	return right;
}
~~~

##### 3. 前后指针法

**原理:**

>1. 定义变量cur指向序列的开头，定义变量pre指向cur的前一个位置。
>2. 当array[cur] < key时，cur和pre同时往后走，如果array[cur]>key，cur往后走，pre留在大于key的数值前一个位置。
>3. 当array[cur]再次 < key时，交换array[cur]和array[pre]。

**代码**

~~~c
int PartSort(int* array,int left,int right){
	if(left < right){
		int key = array[right];
		int cur = left;
		int pre = cur - 1;
		while(cur < right){
            //如果找到小于key的值，并且cur和pre之间有距离时则进行交换。
			while(array[cur] < key && ++pre != cur)	{
				swap(array[cur],array[pre]);
			}
			++cur;
		}
		swap(array[++pre],array[right]);
		return pre;
	}
	return -1;
}
~~~

#### 2. 快排的时间复杂度分析

快速排序的时间复杂度和每次划分的比例相关其公式如下所示:

```
T（n）= T（n/a） + T(n/b) + n
T（1）= 0  
```

##### **1 . 最优情况**

在最优情况下Partition每次都划分的很均匀，即每次都对半分。则   **T(n) = 2 T(n/2) + n** 

解得 **Ｔ(n) = O(nlogn)** 

##### **2. 最坏情况：**

而最坏情况Partition每次都划分的极不均匀，即每次都分成1个和n-1个。则  **T(n) =  T(n-1) + T(1) + n** 

解得 **Ｔ(n) = O(n^2)** 

由于markdown打公式非常累具体过程可以参照：<https://www.cnblogs.com/LzyRapx/p/9565827.html>

#### 3. 随机快排

从上面的分析可知，当快排遇到已排好的数据时时间复杂度会降到 **O(n^2)** 的复杂度，那么怎么避免出现这种情况呢，使得无论输入数据是什么都有一个较为稳定的性能。其实实现的方法非常简单因为传统的快排在选取主元的时候，每次都选取最右/左边的元素。当序列为有序时，会发现划分出来的两个子序列一个里面没有元素，而另一个则只比原来少一个元素。为了避免这种情况，我们可以引入一个随机化量来破坏这种有序状态。即随机取一个主元而不是取最右/左边的元素。

**代码：**

~~~c
int PartSort(int* array,int left,int right){
	random = randomValue(left, right);
    int key = array[random];
	while(left < right){
		while(left < right && array[left] <= key){
			++left;
		}
		array[right] = array[left];
		while(left < right && array[right] >= key){
			--right;
		}
		array[left] = array[right];	 
	}
	array[right] = key;
	return right;
}
~~~

由于使用了随机选取主元的方式，所以随机快排的时间复杂度和输入数据的顺序便不相关了，使得他趋避免了最坏的情况而更加近于平均时间复杂度O(nlogn)使得快排的性能得到了保障，避免了出现性能极差的情况。