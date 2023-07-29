---
layout: post
title: 抽象数据类型(ADT)-二叉搜索树 
date:   2023-07-29 22:30:00 +0800
categories: [数据结构]
tags: [C]
topping: false
---


### 二叉搜索树

二叉搜索树是一种特殊的二叉树，对于树中的每个结点 X，它左子树上的每个结点的值都小于 X 的值，它右子树上的每个结点的值都大于 X 的值。  

### 二叉搜索树的实现

二叉搜索树上的结点都是有序的，因此不同于普通二叉树，二叉搜索树可以按照元素的大小，向特定的分支去查找，因此二叉搜索树有自己的插入、删除、查找的方法。  

结构体定义与普通二叉树相同。  

```
typedef struct binTree {
    struct binTree* parent;
    struct binTree* left;
    struct binTree* right;
    ELEMENT_TYPE data;
}tagBinTree;

tagBinTree* binSearchTreeFind(tagBinTree* root, ELEMENT_TYPE value);
tagBinTree* binSearchTreeFindMin(tagBinTree* root);
tagBinTree* binSearchTreeFindMax(tagBinTree* root);
tagBinTree* binSearchTreeInsert(tagBinTree* root, ELEMENT_TYPE data);
tagBinTree* binSearchTreeDelete(tagBinTree* root, ELEMENT_TYPE data);
```

接口可以采用递归与非递归两种方式来实现，递归方式：  

```
tagBinTree* binSearchTreeFind(tagBinTree* root, ELEMENT_TYPE value)
{
    if (root == NULL)
        return NULL;
    if (root->data == value)
        return root;
    else if (value < root->data)
        return binSearchTreeFind(root->left, value);
    else
        return binSearchTreeFind(root->right, value);
}

tagBinTree* binSearchTreeFindMin(tagBinTree* root)
{
    if (root == NULL)
        return NULL;
    if (root->left != NULL)
        return binSearchTreeFindMin(root->left);
    else
        return root;
}

tagBinTree* binSearchTreeFindMax(tagBinTree* root)
{
    if (root == NULL)
        return NULL;
    if (root->right != NULL)
        return binSearchTreeFindMax(root->right);
    else
        return root;
}

tagBinTree* binSearchTreeInsert(tagBinTree* root, ELEMENT_TYPE data)
{
    if (root == NULL) {
        return binTreeMallocNode(data);
    }
    if (data < root->data) {
        root->left = binSearchTreeInsert(root->left, data);
        if (root->left != NULL)
            root->left->parent = root;
    }
    else {
        root->right = binSearchTreeInsert(root->right, data);
        if (root->right != NULL)
            root->right->parent = root;
    }
    return root;
}

tagBinTree* binSearchTreeDelete(tagBinTree* root, ELEMENT_TYPE data)
{
    if (root == NULL)
        return NULL;
    if (data < root->data) {
        root->left = binSearchTreeDelete(root->left, data);
        if (root->left != NULL)
            root->left->parent = root;
        return root;
    } else if (data > root->data) {
        root->right = binSearchTreeDelete(root->right, data);
        if (root->right != NULL)
            root->right->parent = root;
        return root;
    } else {
        if (root->left != NULL && root->right != NULL) {
            tagBinTree* successor = binSearchTreeFindMin(root->right);
            root->data = successor->data;
            root->right = binSearchTreeDelete(root->right, successor->data);
            return root;
        } else {
            tagBinTree* node = NULL;
            if (root->left != NULL)
                node = root->left;
            else if (root->right != NULL)
                node = root->right;
            if (node != NULL)
                node->parent = root->parent;
            free(root);
            return node;
        }
    }
}
```

非递归方式：  

```
tagBinTree* binSearchTreeFind(tagBinTree* root, ELEMENT_TYPE value)
{
    if (root == NULL)
        return NULL;
    tagBinTree* node = root;
    while (node != NULL) {
        if (node->data == value)
            return node;
        else if (value < root->data)
            node = node->left;
        else
            node = node->right;
    }
    return NULL;
}

tagBinTree* binSearchTreeFindMin(tagBinTree* root)
{
    if (root == NULL)
        return NULL;
    tagBinTree* node = root;
    while (node != NULL) {
        if (node->left == NULL)
            return node;
        node = node->left;
    }
    return node;
}

tagBinTree* binSearchTreeFindMax(tagBinTree* root)
{
    if (root == NULL)
        return NULL;
    tagBinTree* node = root;
    while (node != NULL) {
        if (node->right == NULL)
            return node;
        node = node->right;
    }
    return node;
}

tagBinTree* binSearchTreeInsert(tagBinTree* root, ELEMENT_TYPE data)
{
    if (root == NULL) {
        return binTreeMallocNode(data);
    }
    tagBinTree* father = NULL;
    tagBinTree* node = root;
    while (node != NULL) {
        if (data == node->data) {
            printf("isExist!\n");
            return node;
        } else if (data < node->data) {
            father = node;
            node = node->left;
        } else {
            father = node;
            node = node->right;
        }
    }
    tagBinTree* new = binTreeMallocNode(data);
    if (new == NULL) {
        printf("malloc error!\n");
        return NULL;
    }
    new->parent = father;
    if (data < father->data)
        father->left = new;
    else
        father->right = new;

    return new;
}

tagBinTree* binSearchTreeDelete(tagBinTree* root, ELEMENT_TYPE data)
{
    if (root == NULL)
        return NULL;
    tagBinTree* node = root;
    while (node != NULL) {
        if (data < node->data)
            node = node->left;
        else if (data > node->data)
            node = node->right;
        else {
            if (node->left != NULL && node->right != NULL) {
                tagBinTree* successor = binSearchTreeFindMin(node->right);
                node->data = successor->data;
                if (successor->parent->left == successor)
                    successor->parent->left = successor->right;
                else //只有一种情况
                    successor->parent->right = successor->right;
                if (successor->right != NULL)
                    successor->right->parent = successor->parent;
                free(successor);
                return NULL;
            } else {
                tagBinTree* next = NULL;
                if (node->left != NULL)
                    next = node->left;
                else if (node->right != NULL)
                    next = node->right;
                if (node->parent != NULL) {
                    if (node->parent->left == node)
                        node->parent->left = next;
                    if (node->parent->right == node)
                        node->parent->right = next;
                }
                if (next != NULL)
                    next->parent = node->parent;
                free(node);
                return NULL;
            }
        }
    }
    return NULL;
}
```
---
**参考**：  
<数据结构与算法分析-C语言描述> P4