---
layout: post
title: 排序算法之-冒泡排序 
date:   2022-03-31 18:49:00 +0800
categories: Algorithm
tags: C
topping: true
---



冒泡排序是在每次扫描列表时通过比较相邻两个元素的大小，如果顺序不对就交换过来。就像冒泡一样,每扫描完一次整个列表会将最大/最小的元素排到列表最后。  

### 算法步骤

1. 依次比较相邻两个元素的大小，如果第一个元素大于第二个，就交换两个元素。这样遍历一遍后会把最大的一个元素排列到列表最后。  
2. 重复步骤1，直至整个队列有序。  

### 动图演示

![bubblingSort.webp]({{site.imgurl}}/styles/images/algorithm/bubblingSort.webp)  

### 代码示例

```
#define SORT_OK    (0)
#define SORT_ERR    (-1)

static void swap(int* a, int* b)
{
    int tmp = *a;
    *a = *b;
    *b = tmp;
}

int bubblingSort(int* array, int arrLen)
{
    if (array == NULL || arrLen < 0)
        return SORT_ERR;

    for (int i = 0; i < arrLen; ++i)
    {
        for (int j = arrLen - 1; j > i; --j)
        {
            if (array[j] < array[j - 1])
            {
                swap(&array[j], &array[j - 1]);
            }
        }
    }

    return SORT_OK;
}

```

### 拓展

当列表末尾有一段已经排好序的序列，并且这段序列前面的元素都不大于序列中的元素中时，可以记录这段序列与前面序列的边界，减少比较次数。代码如下：  


```
int bubblingSort3(int* array, int arrLen)
{
    if (array == NULL || arrLen < 0)
        return SORT_ERR;
    
    //用来记录排序序列与未排序序列的边界, 初始化为队列尾.    
    int notOrderLen = arrLen;
    while (notOrderLen > 1)
    {
        int tmp = 0;
        for (int j = 1; j < notOrderLen; j++)
        {
            if (array[j] < array[j - 1])
            {
                swap(&array[j], &array[j - 1]);
                tmp = j;
            }
        }
        //记录排序边界
        notOrderLen = tmp;
    }
    return SORT_OK;
}
```

---
**参考**  
[面试官：手写一个冒泡排序，并对其改进](https://baijiahao.baidu.com/s?id=1643890238963997356&wfr=spider&for=pc)
  
