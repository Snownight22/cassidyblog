---
layout: post
title: 排序算法之-归并排序 
date:   2022-04-05 21:37:00 +0800
categories: Algorithm
tags: C
topping: true
---

归并排序是采用分治法的一个典型应用。将列表分为两个子序列，对每个子序列再进行分治，直到每个序列元素个数为1，则这个序列必定是有序的，一步步合并每个子序列，最终整个列表达到有序的状态。  

### 算法步骤

1. 把长度为 n 的列表分成两个长度为 n/2 的子序列。  
2. 对这两个子序列分别采用归并排序。  
3. 将两个排序好的子序列合并成一个最终的排序序列。  

### 动图演示

![mergeSort.webp]({{site.baseurl}}/styles/images/algorithm/mergeSort.webp)  

### 代码示例

```
#define SORT_OK    (0)
#define SORT_ERR    (-1)

void merge(int* array, int* resultArray, int begin, int mid, int end)
{
    int index = begin;
    int j = mid + 1;
    int i = begin;

    while (i <= mid && j <= end)
        resultArray[index++] = (array[i] > array[j] ? array[j++] : array[i++]);

    while (i <= mid)
        resultArray[index++] = array[i++];

    while (j <= end)
        resultArray[index++] = array[j++];

    memcpy(&array[begin], &resultArray[begin], sizeof(int) * (end - begin + 1));
}

void mergeSortIterator(int* array, int* resultArray, int begin, int end)
{
    if (begin >= end)
        return ;

    int mid = (begin + end) / 2;
    mergeSortIterator(array, resultArray, begin, mid);
    mergeSortIterator(array, resultArray, mid + 1, end);
    merge(array, resultArray, begin, mid, end);
}

int mergeSort(int* array, int arrLen)
{
    if (array == NULL || arrLen < 0)
        return SORT_ERR;

    int* tmpArray = (int*) malloc(sizeof(int) * arrLen);
    if (tmpArray == NULL)
        return SORT_ERR;

    mergeSortIterator(array, tmpArray, 0, arrLen - 1);

    free(tmpArray);
    return SORT_OK;
}

```
