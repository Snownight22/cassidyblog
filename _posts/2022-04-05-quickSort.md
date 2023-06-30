---
layout: post
title: 排序算法之-快速排序 
date:   2022-04-10 16:25:00 +0800
categories: Algorithm
tags: C
topping: true
---

快速排序的基本思想是：通过每一趟排序，将列表分为两部分，其中一部分元素值比另一部分元素值都要小，再按这种方法对这两部分分别进行快速排序。最终使整个列表达到有序。  

### 算法步骤

1. 从列表中选一个元素的值作为基准值。  
2. 对列表进行排序，比基准小的排在基准前面，比基准大的排在后面。  
3. 递归重复步骤 1、2， 对小于基准的列表和大于基准的列表进行排序。  

### 动图演示

步骤2排序有两种方式。  
方式一：  

![quickSort1.webp]({{site.baseurl}}/styles/images/algorithm/quickSort1.webp)  

方式二：  

![quickSort2.webp]({{site.baseurl}}/styles/images/algorithm/quickSort2.webp)  

### 代码示例

```
#define SORT_OK    (0)
#define SORT_ERR    (-1)

//快排第一种方式

int quickSort1(int* array, int begin, int end)
{
    if (begin >= end)
        return SORT_OK;

    int baseValue = array[begin];
    int i = begin;
    int j = end;
    while (j > i)
    {
        while (j > i && array[j] > baseValue)
            j--;
        array[i] = array[j];
        if (j <= i)
            break;
        while(j > i && array[i] < baseValue)
            i++;
        array[j] = array[i];
    }

    array[i] = baseValue;

    quickSort1(array, begin, i - 1);
    quickSort1(array, i + 1, end);

    return SORT_OK;
}

//快排第二种方式

static void swap(int* a, int* b)
{
    int tmp = *a;
    *a = *b;
    *b = tmp;
}

int quickSort2(int* array, int begin, int end)
{
    if (begin >= end)
        return SORT_OK;

    int baseValue = array[begin];
    int index = begin + 1;
    for(int i = begin + 1; i <= end; i++)
    {
        if (array[i] < baseValue)
        {
            swap(&array[i], &array[index]);
            index++;
        }

    }
    swap(&array[begin], &array[index - 1]);

    quickSort2(array, begin, index - 2);
    quickSort2(array, index, end);

    return SORT_OK;
}

```
