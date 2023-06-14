---
layout: post
title: 散列表（二）- 散列表解决冲突的方式
date:   2022-07-18 15:43:00 +0800
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

### 散列表冲突  {#hashTableCollision}  

当哈希函数把不同的关键字映射到同一个散列表地址（即哈希数组索引）上时，就产生了冲突(collision)，当关键字集比散列表地址（哈希数组个数）大时，关键字映射必然会出现冲突。我们用链接法（chaining）和开放寻址法（open addressing）来处理冲突。  

### 使用链接法处理冲突    {#hashTableChaining}

链接法，又叫开链法，顾名思义，把映射到同一散列表地址中的所有元素都放在一个链表中，如下图所示：  

![hashTableChaining]({{site.baseurl}}/styles/images/algorithm/hashTable/hashTableChaining-introductionToAlgorithm(P11-2).png)  
<center>(图片取自算法导论图11-2)</center>  

链接法的优点：链接法处理冲突比较简单，处理数据存储比较灵活，数据处理的平均性能也较高效。   

#### 代码实现

这里为了方便，采用关键字为整数，值也为整数的 key-value 结构，哈希函数采用代码中 integerhash 函数内整数取模的方式。  

```
#define HASH_OK    (0)
#define HASH_ERR    (-1)

typedef struct hash_node
{
    struct hash_node* prev;    //哈希结点前向指针
    struct hash_node* next;    //哈希结点后向指针
    int key;    //关键字
    int value;    //值
}stHashNode;


typedef int (*hashFunc)(int key, unsigned int tableSize);

typedef struct hash_table
{
    stHashNode** table;    //哈希表
    hashFunc keyHash;    //哈希函数
    unsigned int count;    //哈希数组个数
}stHashTable;

//整数哈希函数
int integerHash(int intKey, unsigned int tableSize)
{
    return intKey % tableSize;
}

static stHashNode* mallocNode(int key, int value)
{
    stHashNode* node = (stHashNode*) malloc(sizeof(stHashNode));
    if (node == NULL)
        return NULL;

    node->key = key;
    node->value = value;
    node->prev = NULL;
    node->next = NULL;

    return node;
}

static void freeNode(stHashNode* node)
{
    free(node);
}

//初始化哈希表
int hashTableInit(stHashTable* hashHandler, unsigned int count, hashFunc func)
{
    if (func == NULL)
        return HASH_ERR;

    hashHandler->table = (stHashNode**) malloc(count * sizeof(stHashNode*));
    if (hashHandler->table == NULL)
        return HASH_ERR;

    hashHandler->count = count;
    hashHandler->keyHash = func;

    for (unsigned int i = 0; i < count; i++) 
        hashHandler->table[i] = NULL;

    return HASH_OK;
}

//查找哈希表结点
stHashNode* hashTableFindNode(stHashTable* table, int key, int value)
{
    int index = table->keyHash(key, table->count);
    stHashNode* tableListHead = table->table[index];
    stHashNode* node = tableListHead;
    while (node != NULL)
    {
        if (node->value == value)
            return node;
        node = node->next;
    }

    return NULL;
}

//插入结点
int hashTableInsertNode(stHashTable* table, int key, int value)
{
    stHashNode* node = mallocNode(key, value);
    int index = table->keyHash(node->key, table->count);
    stHashNode* tableListHead = table->table[index];
    if (tableListHead != NULL)
        tableListHead->prev = node;
    node->next = tableListHead;
    table->table[index] = node;
    return HASH_OK;
}

//删除结点
int hashTableDelNode(stHashTable* table, int key, int value)
{
    int index = table->keyHash(key, table->count);
    stHashNode* tableListHead = table->table[index];
    stHashNode* node = tableListHead;

    while(node != NULL)
    {
        if (node->value == value)
        {
            if (node->prev == NULL)
                table->table[index] = node->next;
            else
                node->prev->next = node->next;

            if (node->next != NULL)
                node->next->prev = node->prev;

            freeNode(node);

            return HASH_OK;
        }
        node = node->next;
    }

    return HASH_ERR;
}
```

### 使用开放寻址法处理冲突    {#hashTableOpenAddressing}

在开放寻址法中，所有元素都存放在散列表中。也就是说，每个表项或包含动态集合的一个元素，可包含NULL。当查找某个元素时，要系统地检查所有的表项，直到找到所需的元素，或最终查明该元素不在表中。在开放寻址法中，散列表可能会被填满，这时就要扩充散列表，扩充时需要将散列表的容量（即数组的大小）扩大，并将原散列表中的数据重新散列到新散列表中，这里我们代码未实现。  

开放寻址法中，当出现冲突时，需要探查元素下一个可使用的地址，探查地址的技术有三种，因此我们也有三种方式来实现开放寻址法：  

#### 线性探查（linear probing）    {#linearProbing}

设 m 为散列表大小，i 为探查次数，则对关键字 k 进行第 i 次探查得到的散列值为：  

h(k, i) = (h'(k) + i) mod m,  i = 0, 1, ..., m-1  

为了方便查找和删除，我们给散列表结点置上状态，STATUS_NULL, STATUS_DEL, STATUS_EXIST, 初始状态为 STATUS_NULL，添加结点时状态置为 STATUS_EXIST，删除结点时不需要物理删除，只需要将结点置为 STATUS_DEL 状态即可。  

代码实现：  

```
typedef enum node_status
{
    STATUS_NULL = 0,    //空结点
    STATUS_DEL = 1,    //结点删除
    STATUS_EXIST = 2    //结点在使用
}eNodeStatus;

typedef struct hash_node
{
    int key;
    int value;
    eNodeStatus status;
}stHashNode;

typedef int (*hashFunc)(int key, unsigned int tableSize);

typedef struct hash_table
{
    stHashNode* table;
    hashFunc keyHash;
    unsigned int count;
}stHashTable;

int hashTableInit(stHashTable* hashHandler, unsigned int count, hashFunc func)
{
    if (func == NULL)
        return HASH_ERR;

    hashHandler->table = (stHashNode*) malloc(count * sizeof(stHashNode));
    if (hashHandler->table == NULL)
        return HASH_ERR;

    hashHandler->count = count;
    hashHandler->keyHash = func;
    memset(hashHandler->table, 0, count * sizeof(stHashNode));

    return HASH_OK;
}

int hashTableDelNode(stHashTable* table, int key, int value)
{
    stHashNode* node = hashTableFindNode(table, key, value);
    if (node == NULL)
        return HASH_ERR;
    node->status = STATUS_DEL;
}

stHashNode* hashTableFindNode(stHashTable* table, int key, int value)
{
    int index = table->keyHash(key, table->count);
    int linearIndex = index;
    while (table->table[linearIndex].status != STATUS_NULL && table->table[linearIndex].key != key)
    {
        linearIndex = (linearIndex + 1) % table->count;    //h(k, i) = (h'(k) + i) mod m
        if (linearIndex == index)
            return NULL;
    }

    if (table->table[linearIndex].status == STATUS_EXIST)
        return &table->table[linearIndex];
    return NULL;
}

int hashTableInsertNode(stHashTable* table, int key, int value)
{
    int index = table->keyHash(key, table->count);
    int linearIndex = index;
    while (table->table[linearIndex].status == STATUS_EXIST)
    {
        //已存在
        if (table->table[linearIndex].key == key && table->table[linearIndex].value == value)
            return HASH_OK;
        linearIndex = (linearIndex + 1) % table->count;    //h(k, i) = (h'(k) + i) mod m
        //未找到空位，返回错误
        if (linearIndex == index)
            return HASH_ERR;
    }

    //找到空位，插入
    table->table[linearIndex].key = key;
    table->table[linearIndex].value = value;
    table->table[linearIndex].status = STATUS_EXIST;
    return HASH_OK;
}
```

#### 二次探查（quadratic probing）    {#quadraticProbing}

二次探查的散列函数：  

h(k, i) = (h'(k) + c1\*i + c2\*i<sup>2</sup>) mod m  

此方式数据结构与初始化代码与线性探查相同，只在查找与添加结点进行探查时代码不同。  

代码实现：  

```
stHashNode* hashTableFindNode(stHashTable* table, int key, int value)
{
    int index = table->keyHash(key, table->count);
    int odd = 0;
    int i = 1;
    int quadIndex = index;
    while (table->table[quadIndex].status != STATUS_NULL && table->table[quadIndex].key != key)
    {
        if (odd == 0)
        {
            odd = 1;
            quadIndex = index + i * i;
            quadIndex %= table->count;
        }
        else
        {
            odd = 0;
            quadIndex = index - i * i;
            i++;
            while (quadIndex < 0)
                quadIndex += table->count;
        }
        if (i > table->count / 2)
            return NULL;
    }

    if (table->table[quadIndex].status == STATUS_EXIST)
        return &table->table[quadIndex];
    return NULL;
}

int hashTableInsertNode(stHashTable* table, int key, int value)
{
    int index = table->keyHash(key, table->count);
    int quadIndex = index;
    int odd = 0;
    int i = 1;
    while (table->table[quadIndex].status == STATUS_EXIST)
    {
        //已存在
        if (table->table[quadIndex].key == key && table->table[quadIndex].value == value)
            return HASH_OK;

        if (odd == 0)
        {
            odd = 1;
            quadIndex = index + i * i;
            quadIndex %= table->count;
        }
        else
        {
            odd = 0;
            quadIndex = index - i * i;
            i++;
            while (quadIndex < 0)
                quadIndex += table->count;
        }

        //未找到
        if (i > table->count / 2)
            return HASH_ERR;
    }

    //插入
    table->table[quadIndex].key = key;
    table->table[quadIndex].value = value;
    table->table[quadIndex].status = STATUS_EXIST;
    return HASH_OK;
}
```

#### 双重散列（double hashing）    {#doubleHashing}

双重散列的散列函数：  

h(k, i) = (h1(k) + i\*h2(k)) mod m  

为了能够查找整个散列表，h2(k) 的值必须要与表大小 m 互素，我们这里取小于 m 的最大素数。代码实现如下：  

```
//是否为素数
int isPrimer(int num)
{
    for (int i = 2; i * i <= num; i++)
    {
        if (num % i == 0)
            return 0;
    }
    return 1;
}

//获取小于表大小的最大素数
static int getPrimer(unsigned int tableSize)
{
    int num = tableSize - 1;
    while (isPrimer(num) == 0)
        num--;

    return num;
}

//h2(k), primer为小于表大小的最大素数
int integerHash2(int intKey, int primer)
{
    return primer - (intKey % primer);
}

stHashNode* hashTableFindNode(stHashTable* table, int key, int value)
{
    int index = table->keyHash(key, table->count);
    int index2 = integerHash2(key, getPrimer(table->count));
    int i = 1;
    int quadIndex = index;
    while (table->table[quadIndex].status != STATUS_NULL && table->table[quadIndex].key != key)
    {
        quadIndex = (index + i * index2) % table->count;    //h(k, i) = (h1(k) + i*h2(k)) mod m  
        if (quadIndex == index)
            return NULL;
        i++;
    }

    if (table->table[quadIndex].status == STATUS_EXIST)
        return &table->table[quadIndex];
    return NULL;
}

int hashTableInsertNode(stHashTable* table, int key, int value)
{
    int index = table->keyHash(key, table->count);
    int index2 = integerHash2(key, getPrimer(table->count));
    int i = 1;
    int quadIndex = index;
    while (table->table[quadIndex].status == STATUS_EXIST)
    {
        if (table->table[quadIndex].key == key && table->table[quadIndex].value == value)
            return HASH_OK;

        quadIndex = (index + i * index2) % table->count;    //h(k, i) = (h1(k) + i*h2(k)) mod m  
        if (quadIndex == index)
            return HASH_ERR;
        i++;
    }

    table->table[quadIndex].key = key;
    table->table[quadIndex].value = value;
    table->table[quadIndex].status = STATUS_EXIST;
    return HASH_OK;
}

```

--- 
**参考**：  
[散列表 4篇](https://cloud.tencent.com/developer/inventory/1271)  
《算法导论》第三版中文版  P11   
 
 
