---
layout: post
title: Nginx源码分析-数据结构之双向链表queue 
date:   2022-07-27 15:44:00 +0800
categories: Nginx源码分析
tags: C nginx
topping: false
---

Nginx 中定义了一个同一般双向链表基本相同的双向链表结构 ngx_queue_t，定义在 ngx_queue.c/h 中。  

### 数据结构定义

```
typedef struct ngx_queue_s  ngx_queue_t;

struct ngx_queue_s {
    ngx_queue_t  *prev;    //链表前结点
    ngx_queue_t  *next;    //链表后结点
};
```

此结构一般会嵌入到其它结构中，作为其它数据结构的前后向指针使用，此时可通过链表结点指针找到原数据结构：  

```
//通过链表地址找到数据结构地址，q为链表地址，type为数据结构类型，link为数据结构type中的双向链表元素
#define ngx_queue_data(q, type, link)                                         \
    (type *) ((u_char *) q - offsetof(type, link))
```

### 链表操作

nginx 的双向链表需要定义一个 ngx_queue_t 结点作头结点，记录链表信息，相当于哨兵结点。  

链表头操作：  

```
//初始化链表头
#define ngx_queue_init(q)                                                     \
    (q)->prev = q;                                                            \
    (q)->next = q

//判断链表是否为空
#define ngx_queue_empty(h)                                                    \
    (h == (h)->prev)
    
//取链表第一个节点
#define ngx_queue_head(h)                                                     \
    (h)->next

//取链表最后一个节点
#define ngx_queue_last(h)                                                     \
    (h)->prev

//取链表头
#define ngx_queue_sentinel(h)                                                 \
    (h)    
```

链表的一些基本操作：  

```
//将节点插入到链表前面
#define ngx_queue_insert_head(h, x)                                           \
    (x)->next = (h)->next;                                                    \
    (x)->next->prev = x;                                                      \
    (x)->prev = h;                                                            \
    (h)->next = x


#define ngx_queue_insert_after   ngx_queue_insert_head

//将节点插入到链表末尾
#define ngx_queue_insert_tail(h, x)                                           \
    (x)->prev = (h)->prev;                                                    \
    (x)->prev->next = x;                                                      \
    (x)->next = h;                                                            \
    (h)->prev = x

//下一结点
#define ngx_queue_next(q)                                                     \
    (q)->next

//前一结点
#define ngx_queue_prev(q)                                                     \
    (q)->prev


#if (NGX_DEBUG)
//从链表中删除节点x
#define ngx_queue_remove(x)                                                   \
    (x)->next->prev = (x)->prev;                                              \
    (x)->prev->next = (x)->next;                                              \
    (x)->prev = NULL;                                                         \
    (x)->next = NULL

#else

#define ngx_queue_remove(x)                                                   \
    (x)->next->prev = (x)->prev;                                              \
    (x)->prev->next = (x)->next

#endif
```

链表的一些较复杂的操作有分隔链表，合并链表，查找链表的中间位置结点和链表排序。  

```
//将链表h从q节点分开为两个链表，第一个链表仍然以h为链表头节点，后一个链表以n为链表头节点
#define ngx_queue_split(h, q, n)                                              \
    (n)->prev = (h)->prev;                                                    \
    (n)->prev->next = n;                                                      \
    (n)->next = q;                                                            \
    (h)->prev = (q)->prev;                                                    \
    (h)->prev->next = h;                                                      \
    (q)->prev = n;

//合并两个链，h,n分别为两个链表头
#define ngx_queue_add(h, n)                                                   \
    (h)->prev->next = (n)->next;                                              \
    (n)->next->prev = (h)->prev;                                              \
    (h)->prev = (n)->prev;                                                    \
    (h)->prev->next = h;

//找到双向链表的中间节点，如果链表有奇数个元素，则返回中间节点，如果有偶数个节点，返回第二部分的首节点
ngx_queue_t *
ngx_queue_middle(ngx_queue_t *queue)
{
    ngx_queue_t  *middle, *next;

    middle = ngx_queue_head(queue);

    //当链表中没有节点或只有一个节点时，返回middle节点
    if (middle == ngx_queue_last(queue)) {
        return middle;
    }

    //定义两个指针，next循环链表所有结点，middle指向头结点到next结点的中间位置，每次循环next移动两次，middle移动一次，保持middle指向中间位置，直到next循环完整个链表，找到middle位置
    next = ngx_queue_head(queue);

    for ( ;; ) {
        middle = ngx_queue_next(middle);

        next = ngx_queue_next(next);

        if (next == ngx_queue_last(queue)) {
            return middle;
        }

        next = ngx_queue_next(next);

        if (next == ngx_queue_last(queue)) {
            return middle;
        }
    }
}


/* the stable insertion sort */
//利用插入排序对链表进行排序，cmp为链表元素比较函数
void
ngx_queue_sort(ngx_queue_t *queue,
    ngx_int_t (*cmp)(const ngx_queue_t *, const ngx_queue_t *))
{
    ngx_queue_t  *q, *prev, *next;

    q = ngx_queue_head(queue);

    if (q == ngx_queue_last(queue)) {
        return;
    }

    for (q = ngx_queue_next(q); q != ngx_queue_sentinel(queue); q = next) {

        prev = ngx_queue_prev(q);
        next = ngx_queue_next(q);

        ngx_queue_remove(q);

        do {
            if (cmp(prev, q) <= 0) {
                break;
            }

            prev = ngx_queue_prev(prev);

        } while (prev != ngx_queue_sentinel(queue));

        ngx_queue_insert_after(prev, q);
    }
}
```

---
nginx源码基于nginx-1.22.0版本  
