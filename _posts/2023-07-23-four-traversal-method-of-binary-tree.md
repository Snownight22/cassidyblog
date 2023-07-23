---
layout: post
title: 二叉树的四种遍历方法 
date:   2023-07-23 22:30:00 +0800
categories: [Algorithm, 数据结构]
tags: [C]
topping: true
---

二叉树有四种遍历方式，先序遍历，中序遍历，后序遍历以及层次遍历。  

### 先序遍历

二叉树的先序，中序，后序遍历表示访问二叉树中结点的先后顺序，先序表示先访问结点，中序表示中间访问结点，后序表示最后访问结点。  

先序遍历二叉树的顺序：  
1. 访问根结点。  
2. 先序遍历左子树。  
3. 先序遍历右子树。  

采用递归的方式来编写遍历二叉树的代码是最直观的。代码实现：  

```
//遍历时结点操作函数指针
typedef void (*operateFunc)(tagTreeNode* node);

void traversalInPreOrder(tagBinTree* root, operateFunc func)
{
    if (root ==  NULL)
        return ;
    if (func)
        func(root);
    traversalInPreOrder(root->left, func);
    traversalInPreOrder(root->right, func);
}
```

也可以采用非递归的方式来实现遍历，此时需要借助于栈的数据结构：  
1. 将根结点入栈。  
2. 从栈中弹出一个结点并访问。  
3. 结点右儿子非空则入栈。  
4. 结点左儿子非空则入栈。  
5. 重复步骤2,3,4，直到栈为空，遍历结束。  

代码实现：  

```
void traversalInPreOrder(tagBinTree* root, operateFunc func)
{
    if (root == NULL)
        return ;
    ttStack* stack = stackInit();
    if (stack == NULL)
        return ;

    push(stack, root);
    while (!stackEmpty(stack)) {
        tagBinTree* node = pop(stack);
        if (func)
            func(node);
        if (node->right != NULL)
            push(stack, node->right);
        if (node->left != NULL)
            push(stack, node->left);
    }

    stackDestory(stack);
}
```

### 中序遍历

中序遍历二叉树的顺序：  
1. 中序遍历左子树。  
2. 访问根结点。  
3. 中序遍历右子树。  

代码实现：  

```
void traversalInPostOrder(tagBinTree* root, operateFunc func)
{
    if (root == NULL)
        return ;
    traversalInPostOrder(root->left, func);
    traversalInPostOrder(root->right, func);
    if (func)
        func(root);
}
```

非递归实现同先序遍历，略。  

### 后序遍历

后序遍历二叉树的顺序：  
1. 后序遍历左子树。  
2. 后序遍历右子树。  
3. 访问根结点。  

代码实现：  

```
void traversalInOrder(tagBinTree* root, operateFunc func)
{
    if (root == NULL)
        return ;
    traversalInOrder(root->left, func);
    if (func)
        func(root);
    traversalInOrder(root->right, func);
}
```

非递归实现同先序遍历，略。  

### 层次遍历

层次遍历就是按照树的结构从上到下，从左到右的顺序依次访问每个结点。  

层次遍历需要借助于一个抽象数据类型-队列：  
1. 将树的根结点入队列。  
2. 从队列中取出一个结点并访问。  
3. 判断结点的左右儿子是否为空，不为空则入队列。  
4. 重复步骤2, 3，直到队列为空。遍历完成。  

代码实现：  

```
void traversalInLevelOrder(tagBinTree* root, operateFunc func) 
{
    ttQueue* queue = createQueue(0);
    if (queue == NULL)
        return ;
    enqueue(queue, root);
    while (!queueEmpty(queue)) {
        tagBinTree* node = (tagBinTree*)dequeue(queue);
        if (node != NULL) {
            if (node->left != NULL)
                enqueue(queue, node->left);
            if (node->right != NULL)
                enqueue(queue, node->right);
            if (func != NULL)
                func(node);
        }
    }
    disposeQueue(queue);
}
```
---
**参考**：  
<数据结构与算法分析-C语言描述> P4