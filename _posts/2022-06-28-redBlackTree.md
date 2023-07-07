---
layout: post
title: 红黑树及其C语言实现
date:   2022-06-28 18:13:00 +0800
categories: Algorithm
tags: C
topping: true
---

### 红黑树及其性质    {#redBlackTreeFeature}

红黑树是一棵二叉搜索树，它在每个节点上存储了一位来表示结点的颜色，然后通过每个结点到叶子结点的颜色数目来约束以达到自平衡。它必须满足以下性质：  

1. 每个结点要不是红色，要不是黑色。  
2. 根结点是黑色。  
3. 叶子结点是黑色。(这里的叶子结点指的是T.Nil)  
4. 如果一个结点是红色的，那么它的子结点必须是黑色的。  
5. 从树中的任一结点到所有叶子结点的简单路径上，均包含同样数目的黑色结点。(即其黑高相同)  

### 红黑树的操作

红黑树数据结构定义：  

```
//红黑树结点颜色定义
typedef enum rb_tree_color
{
    RB_TREE_COLOR_BLACK = 0,    //黑色
    RB_TREE_COLOR_RED = 1,    //红色
}eRbTreeColor;

//红黑结结构定义
typedef struct rb_tree_node
{
    struct rb_tree_node* parent;    //父结点
    struct rb_tree_node* left;    //左结点
    struct rb_tree_node* right;    //右结点
    eRbTreeColor nodeColor;    //结点颜色
    int data;    //数据
}stRbTreeNode;
```

红黑树的操作有查询，添加和删除。当从红黑树添加或删除结点时，会破坏[红黑树的性质](#redBlackTreeFeature)。这时需要用到红黑树的旋转操作，以保持红黑树的平衡。  

#### 红黑树的旋转

红黑树的左旋就是把红黑树的结点变为其右结点的左子树，右旋就是将结点变为其左结点的右子树。  

![rbTree_rotate.svg]({{site.imgurl}}/styles/images/algorithm/rbTree/rbTree_rotate.svg)

代码实现：  

```
//左旋
static void rbTreeLeftRotate(stRbTreeNode** root, stRbTreeNode* node)
{
    stRbTreeNode* rNode = node->right;
    node->right = rNode->left;

    if (rNode->left != NULL) 
        rNode->left->parent = node;
    rNode->parent = node->parent;

    //父结点为空代表 node 结点为根结点
    if (node->parent == NULL) 
        *root = rNode;
    else if (node->parent->left == node)
        node->parent->left = rNode;
    else
        node->parent->right = rNode;

    rNode->left = node;
    node->parent = rNode;
}

//右旋
static void rbTreeRightRotate(stRbTreeNode** root, stRbTreeNode* node)
{
    stRbTreeNode* rNode = node->left;
    node->left = rNode->right;

    if (rNode->right != NULL)
        rNode->right->parent = node;
    rNode->parent = node->parent;
    
    //父结点为空代表 node 结点为根结点
    if (node->parent == NULL)
        *root = rNode;
    else if (node->parent->left == node)
        node->parent->left = rNode;
    else
        node->parent->right = rNode;

    rNode->right = node;
    node->parent = rNode;
}
```

#### 红黑树查找

因为红黑树是一棵二叉搜索树，因此可以利用二叉搜索树的性质，如果要查找的值小于当前结点的值，则查找当前结点的左子树，如果大于，则查找右子树，直到与结点值相等或找到叶子结点。  

![rbTreeSearch.webp]({{site.imgurl}}/styles/images/algorithm/rbTree/rbTreeSearch.webp)  

代码实现：  

```
//查找
stRbTreeNode* rbTreeSearch(stRbTreeNode* root, int data)
{
    stRbTreeNode* node = root;

    while (node != NULL)
    {
        if (data < node->data)
            node = node->left;
        else if (data > node->data)
            node = node->right;
        else
            break;
    }

    return node;
}
```

#### 红黑树添加结点 

红黑树添加结点可以分为三步：  

1. 查找结点的插入位置。  
    根据二叉搜索树的性质，可以很容易的找到要插入结点的位置，即找到其父结点。  
2. 将结点着为红色并插入相应位置。  
    这里把结点着为红色，不会影响其黑高，即红黑树性质 5，这样当其父节点为黑色时，不需要进行调整即可完成插入。  
3. 通过旋转和着色等操作，使其满足红黑树的性质。  
    当插入结点的父结点为黑色时，红黑树的性质并没有被破坏，因此不需要进行调整。当插入结点的父结点是红色时，红黑树性质 4 被破坏了，这时就需要调整红黑树。  
    
插入结点代码实现：  

```
//插入                                                                                                                                   
int rbTreeInsert(stRbTreeNode** root, stRbTreeNode* node)                                                                                
{                                                                                                                                        
    stRbTreeNode* parent = NULL; 
    
    //查找结点插入位置                                                                                                        
    stRbTreeNode* next = *root;                                                                                                          
    while (next != NULL)                                                                                                                 
    {                                                                                                                                    
        parent = next;                                                                                                                   
        if (node->data < next->data)                                                                                                     
            next = next->left;                                                                                                           
        else if (node->data > next->data)                                                                                                
            next = next->right;                                                                                                          
        else                                                                                                                             
            return 1;                                                                                                         
    }                                                                                                                                    
                                                                                                                                         
    if (parent == NULL)                                                                                                                  
    {       
        //parent为NULL代表红黑树为空，要插入的结点即为根结点                                                                                                                             
        node->nodeColor = RB_TREE_COLOR_BLACK;                                                                                           
        node->parent = NULL;                                                                                                             
        node->left = NULL;                                                                                                               
        node->right = NULL;                                                                                                              
        *root = node;                                                                                                                    
        return 0;                                                                                                                
    }                                                                                                                                    
         
    //将结点着为红色并插入到红黑树中                                                                                                                                     
    node->left = NULL;                                                                                                                   
    node->right = NULL;                                                                                                                  
    node->parent = parent;                                                                                                               
    if (parent->data > node->data)                                                                                                       
        parent->left = node;                                                                                                             
    else                                                                                                                                 
        parent->right = node;                                                                                                            
    node->nodeColor = RB_TREE_COLOR_RED;                                                                                                 
    
    //调整红黑树，使其满足红黑树性质                                                                                                                                     
    if (parent->nodeColor == RB_TREE_COLOR_RED)                                                                                          
        rbTreeInsertFixUp(root, node);                                                                                                   
    return 0;                                                                                                                    
}
```

当插入结点父结点为红色时，这时违反了红黑树的性质 4，出现了两个连续的红色结点，此时需要对红黑树进行调整，调整方法分为三种情况：  

1). 当插入结点的叔叔结点为红色时，因为根据红黑树的性质可知，插入结点的爷爷结点，即父结点的父结点肯定是黑色，这时需要将爷爷结点的颜色下移，即将其父结点和叔叔结点着为黑色，将爷爷结点着为红色，这样整棵树的黑高未改变，将爷爷结点当做新插入结点，再判断是否违反了红黑树的性质 4，从爷爷结点开始调整。如下图：　　

![rbTree_insert1.svg]({{site.imgurl}}/styles/images/algorithm/rbTree/rbTree_insert1.svg)  

调整后：  

![rbTree_insert11.svg]({{site.imgurl}}/styles/images/algorithm/rbTree/rbTree_insert11.svg)  

2). 当插入结点的叔叔结点为黑色时，这时不可以同步骤 1) 一样将颜色下移，因为下移后会改变红黑树的黑高。此时可以判断插入结点是父结点的左结点还是右结点。如果同父结点的左右位置不相同（即父结点是爷爷结点的左结点，插入结点是父结点的右结点，或父结点是爷爷结点的右结点，插入结点是父结点的左结点），此时可以通过对父结点进行左旋/右旋，使两个结点左右位置相同。然后将插入结点指向旋转后的子结点，将其当做插入结点进行操作。如下图：

![rbTree_insert2.svg]({{site.imgurl}}/styles/images/algorithm/rbTree/rbTree_insert2.svg)  

图中 x 是其父结点的右结点，x.p 是其父结点的左结点，则可以将 x.p 进行左旋，然后将左旋前的父结点当做插入结点。调整后：  

![rbTree_insert21.svg]({{site.imgurl}}/styles/images/algorithm/rbTree/rbTree_insert21.svg)    

3). 当插入结点的叔叔结点为黑色且插入结点同父结点的位置相同时（即父结点是爷爷结点的左结点，插入结点是父结点的左结点，或父结点是爷爷结点的右结点，插入结点是父结点的右结点），根据红黑树的性质，其爷爷结点必定是黑色，可以将其爷爷结点的颜色传递给其父结点，然后对其爷爷结点进行右旋/左旋，这样必定可以使原红黑树达到平衡。如下图：  

![rbTree_insert3.svg]({{site.imgurl}}/styles/images/algorithm/rbTree/rbTree_insert3.svg)  

图中 x 是其父结点的左结点，x.p 也是其父结点的左结点，则可以将其父结点着为黑色，将其爷爷结点着为红色，然后将 x.p.p 进行右旋，使树达到平衡：  

![rbTree_insert31.svg]({{site.imgurl}}/styles/images/algorithm/rbTree/rbTree_insert31.svg)    

插入结点后调整方法代码实现：

```
//插入结点后调整树
static void rbTreeInsertFixUp(stRbTreeNode** root, stRbTreeNode* node)
{
    while (node->parent != NULL && node->parent->nodeColor == RB_TREE_COLOR_RED)
    {
        stRbTreeNode* parent = node->parent;
        if (parent == parent->parent->left)
        {
            stRbTreeNode* uncle = parent->parent->right;
            //情况 1
            if (uncle != NULL && uncle->nodeColor == RB_TREE_COLOR_RED)
            {
                parent->nodeColor = RB_TREE_COLOR_BLACK;
                uncle->nodeColor = RB_TREE_COLOR_BLACK;
                parent->parent->nodeColor = RB_TREE_COLOR_RED;
                node = parent->parent;
            }
            //情况 2
            else if (node == parent->right)
            {
                node = parent;
                rbTreeLeftRotate(root, node);
            }
            //情况 3
            else
            {
                parent->parent->nodeColor = RB_TREE_COLOR_RED;
                parent->nodeColor = RB_TREE_COLOR_BLACK;
                rbTreeRightRotate(root, parent->parent);
            }
        }
        else
        {
            stRbTreeNode* uncle = parent->parent->left;
            //情况 1
            if (uncle->nodeColor == RB_TREE_COLOR_RED)                                                                                   
            {                                                                                                                            
                parent->nodeColor = RB_TREE_COLOR_BLACK;                                                                                 
                uncle->nodeColor = RB_TREE_COLOR_BLACK;                                                                                  
                parent->parent->nodeColor = RB_TREE_COLOR_RED;                                                                           
                node = parent->parent;                                                                                                   
            }                                                                                                                            
            //情况 2
            else if (node == parent->left)                                                                                               
            {                                                                                                                            
                node = parent;                                                                                                           
                rbTreeRightRotate(root, node);                                                                                           
            }                                                                                                                            
            //情况 3
            else                                                                                                                         
            {                                                                                                                            
                parent->parent->nodeColor = RB_TREE_COLOR_RED;                                                                           
                parent->nodeColor = RB_TREE_COLOR_BLACK;                                                                                 
                rbTreeLeftRotate(root, parent->parent);                                                                                  
            }                                                                                                                            
                                                                                                                                         
        }                                                                                                                                
    }                                                                                                                                    
                                                                                                                                         
    (*root)->nodeColor = RB_TREE_COLOR_BLACK;                                                                                            
}
```

#### 红黑树删除结点

红黑树删除结点也需要三个步骤：  

1. 查找到需要删除的结点。  
2. 删除该结点。  
    删除结点需要注意，为了不使树断掉，需要找到能代替删除结点的结点。如果要删除的结点只有一个子结点，则直接用该结点代替要删除的结点。如果要删除的结点有两个结点，则需要找到该结点的后继，将其代替该结点（继承该结点的颜色和位置）。（当然使用该结点的前驱也可以，这里使用后继）  
3. 对删除结点后的树进行调整，使其保持红黑树性质。  
    如果删除的结点是红色的，则红黑树仍然保持其性质，此时不需要调整。但是如果删除的结点是黑色的，那么红黑树的性质 5 肯定是不满足了，此时需要对红黑树进行调整。这里需要注意的是，如果删除结点的替代结点是其后继/前驱，则相当于删除了其后继/前驱结点，此时删除结点的颜色应该是其后继/前驱结点的颜色。  

删除结点代码实现：  

```
//红黑树结点后继
static stRbTreeNode* rbTreeSuccessor(stRbTreeNode* node)
{
    stRbTreeNode* suc = node->right;
    while (suc->left != NULL)
        suc = suc->left;
    return suc;
}


//以now结点代替ori
static void rbTreeTransplant(stRbTreeNode** root, stRbTreeNode* ori, stRbTreeNode* now)
{
    if (ori->parent == NULL)
        *root = now;
    else if (ori->parent->left == ori)
        ori->parent->left = now;
    else
        ori->parent->right = now;

    if (now != NULL)
        now->parent = ori->parent;
}

//删除
stRbTreeNode* rbTreeDelete(stRbTreeNode** root, int data)
{
    //查找要删除的结点
    stRbTreeNode* node = *root;
    while (node != NULL)
    {
        if (node->data == data)
            break;
        else if (node->data > data)
            node = node->left;
        else
            node = node->right;
    }

    if (node == NULL)
        return NULL;

    //删除结点，如果结点只有一个子结点，则直接替代该结点，
    eRbTreeColor oriColor = node->nodeColor;
    stRbTreeNode* adjNode = node;
    stRbTreeNode* parentNode = node->parent;
    if (node->left == NULL)
    {
        adjNode = node->right;
        rbTreeTransplant(root, node, node->right);
    }
    else if (node->right == NULL)
    {
        adjNode = node->left;
        rbTreeTransplant(root, node, node->left);
    }
    else
    {
        stRbTreeNode* suc = rbTreeSuccessor(node);
        oriColor = suc->nodeColor;
        parentNode = suc->parent;
        if (suc->parent == node)
            parentNode = node->parent;
        stRbTreeNode* sucRight = suc->right;
        rbTreeTransplant(root, suc, sucRight);
        rbTreeTransplant(root, node, suc);
        suc->left = node->left;
        suc->right = node->right;
        suc->nodeColor = node->nodeColor;
        if (node->left != NULL)
            node->left->parent = suc;
        if (node->right != NULL)
            node->right->parent = suc;
        adjNode = sucRight;
    }

    if (oriColor == RB_TREE_COLOR_BLACK)
        rbTreeDeleteFixUp(root, adjNode, parentNode);

    return node;
}
```

如果删除的结点是黑色，则肯定会破坏红黑树的性质 5，其删除结点路径的黑高会产生变化，此时需要对红黑树进行调整，使其满足红黑树性质。以删除结点为其父结点的左结点为例，将调整分为以下几种情况（删除结点为其父结点的右结点时与此情况操作正好相反，左右结点相反，左旋改为右旋等）：  

1). 当其兄弟结点颜色为红色时，其父结点肯定为黑色，此时对其父结点进行左旋，旋转后的父结点的右结点变为其兄弟结点，再进行之后的操作。如图：  

![rbTree_delete1.svg]({{site.imgurl}}/styles/images/algorithm/rbTree/rbTree_delete1.svg)    

2). 当其兄弟结点颜色为黑色，且其兄弟结点的左右两个结点也都为黑色时，将其兄弟结点着色为红色，将当前结点指向其父结点，再进行之后的操作。如图：  

![rbTree_delete2.svg]({{site.imgurl}}/styles/images/algorithm/rbTree/rbTree_delete2.svg)    

3). 当其兄弟结点颜色为黑色，兄弟结点的右结点为黑色且兄弟结点的左结点为红色时，可以将其兄弟结点的左结点着色为黑色，将其兄弟结点着色为红色，然后把兄弟结点进行右旋。右旋后就会转换成情况　4。如图：  

![rbTree_delete3.svg]({{site.imgurl}}/styles/images/algorithm/rbTree/rbTree_delete3.svg)    

4). 当其兄弟结点颜色为黑色，且兄弟结点的右结点为红色时。可以将兄弟结点着色为父结点的颜色，父结点着色为黑色，兄弟结点的右结点着色为黑色，然后对父结点进行左旋。左旋后的红黑树即满足性质 5。如图：

![rbTree_delete4.svg]({{site.imgurl}}/styles/images/algorithm/rbTree/rbTree_delete4.svg)    

其实，删除结点后调整红黑树的基本思想就是向其兄弟树借结点，为满足红黑树的性质，兄弟树上的黑色结点肯定是不能借的，只能借红色结点。所以会判断其兄弟结点或其兄弟结点的子结点是否有红色结点。当出现情况 4 时，只需要调整一次即可使红黑树达到平衡的效果。情况 3 可以通过一次旋转转换为情况 4。当都没有红色结点时，为保持其黑高，将其兄弟结点着色为红色，此时相当于删除了其父结点，将当前结点置为其父结点，再向上继续调整（情况 2 ）。  

删除结点调整代码实现：  

```
//红黑树删除结点后调整
static void rbTreeDeleteFixUp(stRbTreeNode** root, stRbTreeNode* node, stRbTreeNode* parent)
{
    while (node != *root && (node == NULL || node->nodeColor == RB_TREE_COLOR_BLACK))
    {
        if (parent->left == node)
        {
            stRbTreeNode* brother = parent->right;
            //情况 1
            if (brother->nodeColor == RB_TREE_COLOR_RED)
            {
                parent->nodeColor = RB_TREE_COLOR_RED;
                brother->nodeColor = RB_TREE_COLOR_BLACK;
                rbTreeLeftRotate(root, parent);
                brother = parent->right;
            }
            //情况 2
            if ((brother->left == NULL || 
                 brother->left->nodeColor == RB_TREE_COLOR_BLACK) &&
                (brother->right == NULL || 
                 brother->right->nodeColor == RB_TREE_COLOR_BLACK))
            {
                brother->nodeColor = RB_TREE_COLOR_RED;
                node = parent;
                parent = node->parent;
            }
            else
            {
                //情况 3
                if (brother->right != NULL && 
                    brother->right->nodeColor == RB_TREE_COLOR_RED)
                {
                    brother->nodeColor = parent->nodeColor;
                    brother->right->nodeColor = RB_TREE_COLOR_BLACK;
                    parent->nodeColor = RB_TREE_COLOR_BLACK;
                    rbTreeLeftRotate(root, parent);
                    node = *root;
                }
                //情况 4
                else
                {
                    brother->nodeColor = RB_TREE_COLOR_RED;
                    brother->left->nodeColor = RB_TREE_COLOR_BLACK;
                    rbTreeRightRotate(root, brother);
                    brother = parent->right;
                }
            }
        }
        else
        {
            stRbTreeNode* brother = parent->left;
            //情况 1
            if (brother->nodeColor == RB_TREE_COLOR_RED)
            {
                parent->nodeColor = RB_TREE_COLOR_RED;
                brother->nodeColor = RB_TREE_COLOR_BLACK;
                rbTreeRightRotate(root, parent);
                brother = parent->left;
            }
            //情况 2
            if ((brother->left == NULL || 
                 brother->left->nodeColor == RB_TREE_COLOR_BLACK) &&
                (brother->right == NULL || 
                 brother->right->nodeColor == RB_TREE_COLOR_BLACK))
            {
                brother->nodeColor = RB_TREE_COLOR_RED;
                node = parent;
                parent = node->parent;
            }
            else
            {
                //情况 3
                if (brother->left != NULL && 
                    brother->left->nodeColor == RB_TREE_COLOR_RED)
                {
                    brother->nodeColor = parent->nodeColor;
                    brother->left->nodeColor = RB_TREE_COLOR_BLACK;
                    parent->nodeColor = RB_TREE_COLOR_BLACK;
                    rbTreeRightRotate(root, parent);
                    node = *root;
                }
                //情况 4
                else
                {
                    brother->nodeColor = RB_TREE_COLOR_RED;
                    brother->right->nodeColor = RB_TREE_COLOR_BLACK;
                    rbTreeLeftRotate(root, brother);
                    brother = parent->left;
                }
            }
        }
    }

    if (node != NULL)
        node->nodeColor = RB_TREE_COLOR_BLACK;
}        
```

--- 
**参考**：  
<算法导论> 第三版中文版  P13
 
