---
layout: post
title: 埃拉托斯特尼筛法
date:   2023-04-27 17:00:00 +0800　
categories: Algorithm
tags: C
---

### 介绍

埃拉托斯特尼筛法是一种对素数进行检定的方法，它是希腊数学家埃拉托斯特尼提出的一种算法，因此命名，又称埃氏筛或爱氏筛。  

### 算法介绍

要获取 n 以内的素数，就要不大于 $ \sqrt[2]{n} \quad $的所有素数的倍数全部剔除，剩下的就是素数。  

比如给定一个自然数 n，要找到不大于 n 的所有素数。首先用最小的素数 2 来筛选，把 2 留下，把所有 2 的倍数从中筛除。再用 3 去筛，然后用 5, 7, 11, ${\cdots}$，直到 $ \sqrt[2]{n}$，剩下的数就是素数。  

埃拉托斯特尼筛法的算法复杂度为 $ n\log{n} $。  

### 算法实现

算法的 C 语言实现代码如下：  

```
void sieve1(char* prime, int n)
{
    int max = (int)sqrt(n);
    for (int i = 2; i <= max; i++) {
        for (int j = i + i; j < n; j += i) {
            prime[j] = 1;
        }
    }

    printf("[sieve1]Prime numbers less than %d:\n", n - 1);
    int primeCount = 0;
    for (int i = 2; i < n; i++) {
        if (prime[i] == 0) {
            printf("%d", i);
            if (primeCount++ % 10 == 9)
                printf("\n");
            else
                printf("\t");
        }
    }
    printf("\n");
}
```

根据原理：除 2 以外的所有偶数都不是素数，和本算法原理：所有不大于 $ \sqrt[2]{n} \quad $的倍数都剔除，剩下的是素数。可以简化算法，减少循环：  

```
void sieve2(char* prime, int n)
{
    int max = (int)sqrt(n);
    for (int i = 3; i <= max; i += 2) {
        for (int j = i * i; j < n; j += i) {
            prime[j] = 1;
        }
    }

    printf("[sieve2]Prime numbers less than %d:\n", n - 1);
    int primeCount = 1;
    printf("2\t");
    for (int i = 3; i < n; i += 2) {
        if (prime[i] == 0) {
            printf("%d", i);
            if (primeCount++ % 10 == 9)
                printf("\n");
            else
                printf("\t");
        }
    }
    printf("\n");
}
```



--- 
**参考**：  
[百度百科-埃拉托斯特尼筛法](https://baike.baidu.com/item/%E5%9F%83%E6%8B%89%E6%89%98%E6%96%AF%E7%89%B9%E5%B0%BC%E7%AD%9B%E6%B3%95/374984?fr=aladdin)   
