---
layout: post
title: 秦九韶算法(霍纳法则)
date:   2023-02-16 10:30:00 +0800　
categories: Algorithm
tags: C
---

### 介绍

秦九韶算法是中国南宋时期的数学家秦九韶提出的一种多项式简化算法。在西方被称作霍纳算法。秦九韶（约公元1202年－1261年），字道古，南宋末年人，出生于鲁郡（今山东曲阜一带人）。  

19世纪初，英国数学家威廉·乔治·霍纳重新发现并证明，后世称作霍纳算法（Horner's method、Horner scheme）。  

### 算法介绍

这个算法用于多项式 $\sum_{i=1}^n A_iX^i$ <!--$\sum\limits_{i=1}^n A_iX^i$--> 求值，即求 $f(x)=a_nx^n+a_{n-1}x^{n-1}+{\cdots}+a_1x+a_0$ 的值。该多项式可以改写成以下形式：  

$f(x)$  
$=a_nx^n+a_{n-1}x^{n-1}+{\cdots}+a_1x+a_0$  
$=(a_nx^{n-1}+a_{n-1}x^{n-2}+{\cdots}+a_2x+a_1)x+a_0$  
$=((a_nx^{n-2}+a_{n-1}x^{n-3}+{\cdots}+a_3x+a_2)x+a_1)x+a_0$  
$=({\ldots}((a_nx+a_{n-1})x+a_{n-2})x+{\cdots}+a_1)x+a_0$    

这样，可以先计算最内层括号内一次多项式的值，即 $v_1=a_nx+a_{n-1}$，然后由内向外逐层计算一次多项式的值，即：  

$v_2=v_1x+a_{n-2}$  
$v_3=v_2x+a_{n-3}$  
${\cdots}$  
$v_n=v_{n-1}x+a_0$  

这样就会将一个 n 次多项式 $f(x)$ 的值转化为求 n 个一次多项式的值。对一个 n 次多项式，至多做 n 次乘法和 n 次加法。  

### 算法实现

```
int horner_scheme(int *an, int n, int x)
{
    int anx = 0;
    for (int i = n; i >= 0; i--) {
        anx = anx * x + an[i];
    }
    return anx;
}
```




--- 
**参考**：  
[百度百科-秦九韶算法](https://baike.baidu.com/item/%E7%A7%A6%E4%B9%9D%E9%9F%B6%E7%AE%97%E6%B3%95/449196?fromModule=lemma_inlink)   
