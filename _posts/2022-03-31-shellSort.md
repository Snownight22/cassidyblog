---
layout: post
title: 排序算法之-希尔排序 
date:   2022-04-01 17:43:00 +0800
categories: Algorithm
tags: C
topping: true
---

希尔排序又称缩小增量排序，是由Donald.L.Shell于1959年提出而得名。先将原列表按一定增量gap分组，然后对每一组进行插入排序。之后缩小增量gap，再对每组进行排序，直到增量gap缩小为1，再进行排序后即得到最后的有序列表。  

比如，原列表为 R(R0, R1, R2, R3, R4, R5, R6, R7, R8, R9)，首先设置 gap = R.length/2 = 5，将原列表按增量分为5组，(R0, R5), (R1, R6), (R2, R7), (R3, R8) 和 (R4, R9)，对几组分别进行排序。再设置gap = gap/2 = 2，将上面排序后的列表按增量分为2组，(R0, R2, R4, R6, R8) 和 (R1, R3, R5, R7, R9)，对分组再进行排序。最后设置gap = gap/2 = 1，结整个列表进行排序。  

### 算法步骤

1. 设置gap=list.length/2。  
2. 按照增量gap对列表进行分组。  
3. 对每组进行插入排序。  
4. 将增量gap减半。  
5. 重复步骤2、3、4，直到gap = 0，结束排序。  

### 动图演示

![shellSort.webp]({{site.imgurl}}/styles/images/algorithm/shellSort.webp)  

### 代码示例

```
#define SORT_OK    (0)
#define SORT_ERR    (-1)

int shellSort(int* array, int arrLen)
{
    if (array == NULL || arrLen < 0)
        return SORT_ERR;

    int gap = arrLen / 2;
    while (gap >= 1)
    {
        for (int i = 0; i < gap; i++)
        {
            //做插入排序
            for (int j = i + gap; j < arrLen; j += gap)
            {
                int k = j - gap;
                int tmp = array[j];
                while (k >= 0 && array[k] > tmp)
                {
                    array[k + gap] = array[k];
                    k -= gap;
                }
                if (k + gap != j)
                    array[k + gap] = tmp;
            }
        }
        gap /= 2;
    }

    return SORT_OK;
}
```
