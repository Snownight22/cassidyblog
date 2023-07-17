---
layout: post
title: 抽象数据类型(ADT)-二叉树 
date:   2023-07-15 22:30:00 +0800
categories: [数据结构]
tags: [C]
topping: true
---

### 二叉树定义

二叉树是一种特殊的树，对于树中的每一个结点，最多只有两个儿子的树称为二叉树。如图：  

![binaryTree.png]({{ site.imgurl }}/styles/images/algorithm/tree/binaryTree.png)  

### 二叉树的实现

因为二叉树的每个结点只有两个儿子，因此可以把在每个结点中记录下儿子的指针，为了方便查找，还可以记录下每个结点的父结点指针。  

数据结构定义：  

```
#define ELEMENT_TYPE int

#define BINTREE_OK    (0)
#define BINTREE_FAIL    (-1)

typedef struct binTree {
    struct binTree* parent;
    struct binTree* left;
    struct binTree* right;
    ELEMENT_TYPE data;
}tagBinTree;
```

可以定义一些二叉树的操作：  

```
// 创建二叉树根结点
tagBinTree* createTree(ELEMENT_TYPE rootValue);
// 销毁二叉树
void disposeTree(tagBinTree* root);
// 添加结点为左儿子
tagBinTree* addLeft(tagBinTree* parent, ELEMENT_TYPE value);
// 添加结点为右儿子
tagBinTree* addRight(tagBinTree* parent, ELEMENT_TYPE value);
```

```
tagBinTree* binTreeMallocNode(ELEMENT_TYPE value)
{
    tagBinTree* node = (tagBinTree*)malloc(sizeof(tagBinTree));
    if (node == NULL)
        return NULL;
    node->data = value;
    node->parent = NULL;
    node->left = NULL;
    node->right = NULL;
    return node;
}

void binTreeFreeNode(tagBinTree* node)
{
    free(node);
}

tagBinTree* createTree(ELEMENT_TYPE rootValue)
{
    printf("[createTree]element size:%d\n", sizeof(ELEMENT_TYPE));
    return binTreeMallocNode(rootValue);
}

void disposeTree(tagBinTree* root)
{
    if (root == NULL)
        return ;
    disposeTree(root->left);
    disposeTree(root->right);
    free(root);
}

tagBinTree* addLeft(tagBinTree* parent, ELEMENT_TYPE value)
{
    if (parent == NULL)
        return NULL;
    tagBinTree* node = binTreeMallocNode(value);
    if (node == NULL)
        return NULL;
    if (parent->left != NULL) {
        node->left = parent->left;
        parent->left->parent = node;
    }
    parent->left = node;
    node->parent = parent;
    return node;
}

tagBinTree* addRight(tagBinTree* parent, ELEMENT_TYPE value)
{
    if (parent == NULL)
        return NULL;
    tagBinTree* node = binTreeMallocNode(value);
    if (node == NULL)
        return NULL;
    if (parent->right != NULL) {
        node->right = parent->right;
        parent->right->parent = node;
    }
    parent->right = node;
    node->parent = parent;
    return node;
}
```

### 表达式树

表达式树就是一种二叉树，它的所有叶子结点都是操作数，其它结点都是操作符。下面是一棵表达式树：  

![expressionTree.png]({{ site.imgurl }}/styles/images/algorithm/tree/expressionTree.png)  

上述表达式树是由表达式 `1+2*3+(4*5-6)*7` 转化来的。  
我们可以通过后缀表达式来构建一棵表达式树。这里也需要用到栈的数据结构，注意此时栈中要存储的数据为指针，需要定义 `#define ELEMENT_TYPE void*`。上述表达式的后缀表达式为 `123*+45*6-7*+`。构建时从左到右扫描后缀表达式，当遇到操作数时，则创建一个树结点放到到栈中，当遇到一个操作符时，创建一个树结点，并从栈中弹出两个树结点，当做此操作符结点的左右儿子，然后将该操作符结点放入栈中，直到扫描完整个表达式，栈中应该只有一个操作符结点，即为该表达式树的根结点。  

代码实现：

```
tagBinTree* createTreeFromReversePolish(char* notation, int length)
{
    ttStack* stack = stackInit();
    tagBinTree* root = NULL;

    for (int i = 0; i < length; i++) {
        if (notation[i] == '+' || notation[i] == '-' || notation[i] == '*' || notation[i] == '/') {
            tagBinTree* node = binTreeMallocNode(notation[i]);
            if (node == NULL) {
                printf("[createTreeFromReversePolish]malloc error\n");
                continue;
            }
            tagBinTree* left = NULL, *right = NULL;
            if (!stackEmpty(stack))
                right = pop(stack);
            if (!stackEmpty(stack))
                left = pop(stack);
            if (left != NULL)
                left->parent = node;
            if (right != NULL)
                right->parent = node;
            node->left = left;
            node->right = right;
            push(stack, node);
        } else {
            tagBinTree* node = binTreeMallocNode(notation[i]);
            push(stack, node);
        }
    }

    if (!stackEmpty(stack))
        root = pop(stack);
    stackDestory(stack);

    return root;
}
```
---
**参考**：  
<数据结构与算法分析-C语言描述> P4