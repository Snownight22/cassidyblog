---
layout: post
title: 排序算法之-桶排序 
date:   2022-05-10 13:45:00 +0800
categories: Algorithm
tags: C
topping: true
---

桶排序就是将列表中的元素拆分成多个桶，然后将处于同一值域的元素放到同一个桶中。每个桶中的元素可通过插入排序、选择排序、冒泡排序或桶排序等各种排序方法再进行排序，使桶中的元素达到有序的效果。然后按顺序将桶中的元素还原，使整个列表达到有序。  

### 算法步骤

1. 根据待排序列表的最大值和最小值确定桶个数及每个桶的存储范围。  
2. 遍历原列表，将列表中的元素按桶的范围确定相应的桶，再使用插入排序(或其它排序)将元素放到相应桶的对应位置。  
3. 遍历所有桶，按顺序将桶中元素放到已排序列表中。  

### 动图演示

![bucketSort.webp]({{site.baseurl}}/styles/images/algorithm/bucketSort.webp)  


### 代码示例

```
#define SORT_OK    (0)
#define SORT_ERR    (-1)

int bucketSort(int* array, int arrLen)
{
    if (array == NULL || arrLen < 0)
        return SORT_ERR;

    int minValue = array[0];
    int maxValue = array[0];
    int bucket[5][arrLen + 1];

    for (int i = 0; i < 5; i++)
        bucket[i][0] = 0;

    //找出整个数组最大元素和最小元素
    for (int i = 1; i < arrLen; i++)
    {
        if (minValue > array[i])
            minValue = array[i];
        if (maxValue < array[i])
            maxValue = array[i];
    }

    //计算每个桶存储范围
    int interval = (maxValue - minValue) / 5 + 1;

    //用插入排序将原列表中元素放到桶中
    for (int i = 0; i < arrLen; i++)
    {
        int index = (array[i] - minValue) / interval;
        int j = bucket[index][0];
        for (; j > 0; j--)
        {
            if (array[i] < bucket[index][j])
                bucket[index][j+1] = bucket[index][j];
            else
                break;
        }
        bucket[index][j+1] = array[i];
        bucket[index][0]++;
    }

    //按顺序还原列表
    int aIndex = 0;
    for (int i = 0; i < 5; i++)
    {
        for (int j = 1; j <= bucket[i][0]; j++)
            array[aIndex++] = bucket[i][j];
    }

    return SORT_OK;
}

```
