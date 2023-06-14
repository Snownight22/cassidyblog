---
layout: post
title: 散列表（一）- 散列表及常用字符串哈希函数
date:   2022-07-15 21:19:00 +0800
categories: Algorithm
tags: C
topping: true
---
<!--
本文档表格居中显示
-->
<style>
table {
width: 80%;
margin: auto;
}
</style>

### 散列表  {#hashTable}

散列表又称哈希表（hash table），是普通数组概念的推广。它将数据的关键字信息通过哈希函数映射到数组的索引上，然后将数据存储在数组对应的索引上，这样访问数据时就可以按照数组的方式来访问，可以加快访问速度。  

将关键字映射到数组索引的函数就叫做哈希函数。  

### 常用字符串哈希函数    {#StringHashFunction}

下面我们对一些常用的字符串哈希函数进行介绍，它们都是计算机科学家研究出来的，映射后的地址比较平均，冲突较少。函数都以科学家名字或项目名称命名：  

#### 1. SKDR Hash  

BKDRHash 是 Kernighan 和 Dennis 在《The C programming language》中提出的。  
    
```
#define UNSIGNED_INT_MAX    (0x7fffffff)

//BKDR Hash Function
unsigned int BKDRHash(char *str)
{
    unsigned int seed = 31;    //31,131, 1313, 13131,..
    unsigned hashVal = 0;

    while (*str)
        hashVal = hashVal * seed + (*str++);

    return (hashVal & UNSIGNED_INT_MAX);
}
```

#### 2. AP Hash  

Arash Partow 提出了这个算法。  

```
//Arash Partow, AP Hash function
unsigned int APHash(char *str)
{
    unsigned int hashVal = 0;

    for (int i = 0; *str != '\0'; i++)
    {
        if ((i & 1) == 0)    //偶数
            hashVal ^= ((hashVal << 7) ^ (*str++) ^ (hashVal >> 3));
        else    //奇数
            hashVal ^= (~((hashVal << 11) ^ (*str++) ^ (hashVal >> 5)));
    }

    return (hashVal & UNSIGNED_INT_MAX);
}
```

#### 3. DJB Hash  

Daniel J. Bernstein 在 comp.lang.c 邮件列表中发表的。  

```
//Daniel J. Bernstein, DJB Hash Function
unsigned int DJBHash(char *str)
{
    unsigned int hashVal = 5381;

    while (*str)
        hashVal += (hashVal << 5) + (*str++);

    return (hashVal & UNSIGNED_INT_MAX);
}
```

#### 4. JS Hash  

Justin Sobel 提出的基于位的函数函数。  

```
//Justin Sobel, JS Hash function
unsigned int JSHash(char *str)
{
    unsigned int hashVal = 1315423911;

    while (*str)
        hashVal ^= ((hashVal << 5) + (*str++) + (hashVal >> 2));

    return (hashVal & UNSIGNED_INT_MAX);
}
```

#### 5. RS Hash

Robert Sedgwicks 提出的这个哈希函数。  

```
//Robert Sedgwicks, RS Hash function
unsigned int RSHash(char *str)
{
    unsigned int b = 378551;
    unsigned int a = 63689;
    unsigned int hashVal = 0;

    while (*str)
    {
        hashVal = hashVal * a + (*str++);
        a *= b;
    }

    return (hashVal & UNSIGNED_INT_MAX);
}
```

#### 6. SDBM Hash

SDBM项目使用的哈希函数。  

```
//SDBM Hash function
unsigned int SDBMHash(char *str)
{
    unsigned int hashVal = 0;

    while (*str)
        hashVal = (*str++) + (hashVal << 6) + (hashVal << 16) - hashVal;

    return (hashVal & UNSIGNED_INT_MAX);
}
```

#### 7. PJW Hash

Peter J. Weinberger在其编译器著作中提出的。  

```
//Peter J.Weinberger PJWHash Hash function
unsigned int PJWHash(char *str)
{
    unsigned charBit = 8;
    unsigned int bitsInUnsignedInt = (unsigned int)(sizeof(unsigned int) * charBit);
    unsigned int threeQuarter = (unsigned int)((bitsInUnsignedInt * 3) / 4);
    unsigned int oneEighth = (unsigned int)(bitsInUnsignedInt / 8);
    unsigned int highBits = (unsigned int)(0xFFFFFFFF) << (bitsInUnsignedInt - oneEighth);
    unsigned int hashVal = 0;
    unsigned int test = 0;

    while (*str)
    {
        hashVal = (hashVal << oneEighth) + (*str++);
        if ((test = hashVal & highBits) != 0)
            hashVal = ((hashVal ^ (test >> threeQuarter)) & (~highBits));
    }

    return (hashVal & UNSIGNED_INT_MAX);
}
```  

#### 8. ELF Hash

Unix系统上面广泛使用的哈希函数。  

```
//UnixSystem, ELF Hash function
unsigned int ELFHash(char *str)
{
    unsigned int hashVal = 0;
    unsigned int x = 0;

    while (*str)
    {
        hashVal = (hashVal << 4) + (*str++);
        if ((x = hashVal & 0xF0000000L) != 0)
        {
            hashVal ^= (x >> 24);
            hashVal &= ~x;
        }
    }

    return (hashVal & UNSIGNED_INT_MAX);
}
```

#### 9. DEK Hash

高德纳(Donald E. Knuth)在《计算机程序设计的艺术》中提出的哈希函数。  

```
unsigned int DEKHash(char *str)
{
    unsigned int hashVal = strlen(str);

    while (*str)
        hashVal = (hashVal << 5) ^ (hashVal >> 27) ^ (*str++);

    return (hashVal & UNSIGNED_INT_MAX);
}
```

#### 10. BP Hash

```
//BP Hash function
unsigned int BPHash(char *str)
{
    unsigned int hashVal = strlen(str);

    while (*str)
        hashVal = (hashVal << 7) ^ (*str++);

    return (hashVal & UNSIGNED_INT_MAX);
}
```

#### 11. FNV Hash

FNVHash 算法全名为 Fowler-Noll-Vo 算法，是以三位发明人 Glenn Fowler，Landon Curt Noll，Phong Vo 的名字来命名的。  

```
//FNV Hash function, FNV-1
unsigned int FNVHash(char *str)
{
    int fnvprime = 0x811C9DC5;
    unsigned int hashVal = 0;

    while (*str)
    {
        hashVal *= fnvprime;
        hashVal ^= *str++;
        /*FNV-1a
        hashVal ^= *str++;
        hashVal *= fnvprime;    
        */
    }

    return (hashVal & UNSIGNED_INT_MAX);
}
```

#### 各哈希函数比较

哈希函数可能有其应用范围，这里我们只比较常用英文字符串的哈希效果。可以通过哈希函数计算的效率及计算后的最大冲突数来比较各哈希函数的优劣。样本有两个，一个是 linux 系统中的 words 文件，里面包含 45403 个单词，ubuntu 或 fedora 系统可以从 /usr/share/dict 中获取，其它可以从　<https://users.cs.duke.edu/~ola/ap/linuxwords> 中获取，另一个取自 <https://gist.github.com/WChargin/8927565>　里面包含 99171 个英文单词。  

对两个样本采用各个哈希函数分别进行了 3000 次哈希后总耗时除以 3000 得到每次哈希两个样本共 (99171 + 45403 =) 144574 个单词的平均耗时，如下表：  

|哈希函数|BKDRHash |APHash |DJBHash |JSHash |RSHash |SDBMHash |PJWHash |ELFHash |DEKHash |BPHash |FNVHash |
|:------:|:---:|:-----:|:------:|:-----:|:-----:|:-------:|:------:|:------:|:------:|:-----:|:------:|
|耗时<br/>（单位：毫秒）|5.91 |6.65 |5.39 |5.70 |6.27 |5.91 |7.69 |7.25 |6.25 |6.26 |6.26 |   

把哈希表大小分别设为100, 1000, 10000，对两个样本进行哈希后取其最大冲突数。    

样本1 （45403 个单词）：  

|哈希表大小 |100 |1000 |10000 |
|:---------:|:---:|:---:|:---:|
|BKDRHash |503  |67   |13   |
|APHash   |525  |69   |15   |
|DJBHash  |500  |66   |15   |
|JSHash   |493  |67   |17   |
|RSHash   |503  |70   |15   |
|SDBMHash |506  |68   |16   |
|PJWHash  |1919 |419  |102  |
|ELFHash  |1919 |419  |102  |
|DEKHash  |697  |109  |24   |
|BPHash   |3079 |1621 |1505 |
|FNVHash  |490  |69   |15   |

样本2 （99171 个单词）：  

|哈希表大小 |100 |1000 |10000 |
|:---------:|:---:|:---:|:---:|
|BKDRHash |1076 |136  |24   |
|APHash   |1054 |129  |25   |
|DJBHash  |1084 |134  |26   |
|JSHash   |1073 |135  |24   |
|RSHash   |1066 |135  |24   |
|SDBMHash |1077 |134  |23   |
|PJWHash  |4135 |890  |200  |
|ELFHash  |4135 |890  |200  |
|DEKHash  |1401 |204  |37   |
|BPHash   |7417 |3233 |2668 |
|FNVHash  |1068 |131  |24   |

从表一可以看到哈希效率最快的有 DJBHash, BKDRHash, SDBMHash，最慢的有 PJWHash 和 ELFHash，从表二和表三可以看到，哈希效果最差的是 BPHash，其次是 PJWHash 和 ELFHash，其它效果差别不大。  

--- 
**参考**：  
[字符串哈希函数](https://zhuanlan.zhihu.com/p/507990991)  
[FNV哈希算法](https://www.cnblogs.com/zrx1/p/16452263.html)  
《算法导论》 第三版中文版  P11   
《C程序设计语言》 第二版 P6.6  
 
