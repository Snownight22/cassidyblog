---
layout: post
title: 排序算法之-计数排序 
date:   2022-04-24 17:31:00 +0800
categories: Algorithm
tags: C
topping: true
---

计数排序适用于取值在一定范围内的列表，例如取值范围在 0-k，可以建立一个数组，数组下标代表列表中值有大小，数组值代表该值在列表中出现的次数。先遍历原列表，列表中值出现一次，则对应数组下标的值加一，遍历完列表后对数组进行遍历，根据数组下标和值一一还原队列，还原完原列表达到有序。  

### 算法步骤

1. 创建计数数组。  
2. 遍历原列表，在计数数组中记录原列表值出现的次数。  
3. 从前到后/从后向前遍历计数数组，按顺序将计数数组下标放入原列表中，直到计数数组值为 0。  
4. 遍历计数数组结束，原队列有序。  

### 动图演示

![countingSort.webp]({{site.imgurl}}/styles/images/algorithm/countingSort.webp)  


### 代码示例

```
#define SORT_OK    (0)
#define SORT_ERR    (-1)

int countingSort(int* array, int arrLen)
{
    if (array == NULL || arrLen < 0)
        return SORT_ERR;
    int tmp[10] = { 0 };

    for (int i = 0; i < arrLen; i++)
    {
        if (array[i] >= arrLen)
            return SORT_ERR;
        tmp[array[i]]++;
    }

    int index = 0;
    for (int i = 0; i < 10; i++)
    {
        while (tmp[i]-- > 0)
            array[index++] = i;
    }

    return SORT_OK;
}

```
