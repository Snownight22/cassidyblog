---
layout: post
title: 排序算法之-插入排序 
date:   2022-04-01 13:52:00 +0800
categories: Algorithm
tags: C
topping: true
---

插入排序中将原列表分为两部分:有序区和无序区，插入排序就是从无序区取出一个元素，按照它的大小插入到有序区内相应位置。  

### 算法步骤

1. 开始时列表中所有元素都在无序区，取第一个元素，它可以被认为是已经有序。  
2. 再从无序区取一个元素A，从后向前循环比较有序区的元素，如果有序区的元素较大，则将其向后移动一个位置。  
3. 直到有序区的元素不大于A，则将A放到这个元素的后面。  
4. 重复步骤2、3，直到无序区的元素都放到有序区。  

### 动图演示

![insertSort.webp]({{site.baseurl}}/styles/images/algorithm/insertSort.webp)  

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

int insertSort(int* array, int arrLen)
{
    if (array == NULL || arrLen < 0)
        return SORT_ERR;

    for (int i = 0; i < arrLen; ++i)
    {
        int tmp = array[i];
        int j = i - 1;
        while (j >= 0 && array[j] > tmp)
        {
            array[j + 1] = array[j];
            j--;
        }
        array[j + 1] = tmp;
    }
    return SORT_OK;
}
```
