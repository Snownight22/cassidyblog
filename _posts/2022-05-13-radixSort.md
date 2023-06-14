---
layout: post
title: 排序算法之-基数排序 
date:   2022-05-13 14:20:00 +0800
categories: Algorithm
tags: C
topping: true
---

基数排序就是把原列表中的数按位数分割成不同的数字，然后从比较各位的大小来进行排序的方法。基数排序是桶排序的一种。  

基数排序有两种排序方式，最低位优先法 LSD (Least significant digital) 和最高位优先法 MSD (Most significant digital)，LSD 从最低位到最高位依次按位进行排序，MSD 从最高位向最低位依次按位进行排序。  

### 算法步骤

这里给出 LSD 方式的算法步骤：  

1. 设 n=1。  
2. 遍历原列表，把列表中元素按第 n (从右到左) 位的数字大小放到对应桶中。  
3. 遍历所有桶，按顺序将桶中元素放到已排序列表中。  
4. 设置 n++，重复步骤 2, 3，直到 n 到最大位。  

### 动图演示

![radixSort.webp]({{site.baseurl}}/styles/images/algorithm/radixSort.webp)  


### 代码示例

```
#define SORT_OK    (0)
#define SORT_ERR    (-1)

int radixSort(int* array, int arrLen)
{
    if (array == NULL || arrLen < 0)
        return SORT_ERR;

    int tmp[10][arrLen + 1];
    int power = 1;
    int max = array[0];

    int i = 0;
    for (i = 0; i < 10; i++)
        tmp[i][0] = 0;

    //计算最大位数
    i = 0;
    while (i < arrLen)
    {
        if (array[i] > max)
            max = array[i];
        i++;
    }

    while (power <= max)
    {
        //每一位按大小放入对应桶内
        for (i = 0; i < arrLen; i++)
        {
            int tmpValue = array[i] / power;
            int index = tmpValue % 10;
            tmp[index][++tmp[index][0]] = array[i];
        }

        //遍历所有桶，按顺序还原数组
        int aIndex = 0;
        for (i = 0; i < arrLen; i++)
        {
            for (int j = 1; j <= tmp[i][0]; j++)
                array[aIndex++] = tmp[i][j];
            tmp[i][0] = 0;
        }

        //对下一位进行排序
        power *= 10;
    }

    return SORT_OK;
}

```
