---
layout: post
title: 排序算法之-选择排序 
date:   2022-03-31 20:22:00 +0800
categories: Algorithm
tags: C
topping: true
---

选择排序每次选择一个未排序列表中最大/最小的元素，放到有序位置。  

### 算法步骤

1. 从列表中找到最小的元素，将它放在第一个位置。  
2. 从剩下的元素中找到最小的元素，将它放在第二个位置。  
3. 重复步骤1、2，直到所有数据都有序。  

### 动图演示

![chooseSort.webp]({{site.imgurl}}/styles/images/algorithm/chooseSort.webp)  

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

int chooseSort(int* array, int arrLen)
{
    if (array == NULL || arrLen < 0)
        return SORT_ERR;

    for (int i = 0; i < arrLen; ++i)
    {
        int minIndex = i;
        for (int j = i + 1; j < arrLen; ++j)
        {
            if (array[minIndex] > array[j])
                minIndex = j;
        }

        swap(&array[i], &array[minIndex]);
    }

    return SORT_OK;
}
```
