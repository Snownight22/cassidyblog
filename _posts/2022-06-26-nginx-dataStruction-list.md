---
layout: post
title: Nginx源码分析-数据结构之链表list 
date:   2022-06-26 15:32:00 +0800
categories: Nginx源码分析
tags: C nginx
topping: true
---

Nginx 链表数据结构定义为 ngx_list_t，链表内存是基于 nginx 的内存池进行分配的，其操作也与内存池紧密关联。  

### 链表数据结构

#### ngx_list_t 定义

```
typedef struct ngx_list_part_s  ngx_list_part_t;

//链表元素部分结构体
struct ngx_list_part_s {
    void             *elts;    //数据部分使用内存
    ngx_uint_t        nelts;    //数据使用个数
    ngx_list_part_t  *next;    //指向链表下一结点
};


//链表头结构体
typedef struct {
    ngx_list_part_t  *last;    //指向链表最后一个元素
    ngx_list_part_t   part;    //链表第一个元素
    size_t            size;    //链表所存储数据的大小
    ngx_uint_t        nalloc;    //链表能够存储数据个数
    ngx_pool_t       *pool;    //内存池
} ngx_list_t;
```

ngx_list_t 链表头结构内包含了链表的第一个元素，last 指向链表最后一个元素。链表元素部分 ngx_list_part_t 内 *elts 是元素数据部分的指针，它是一个数组结构，按顺序存储了链表中的数据，数据部分大小为 size， 可存储个数为 nalloc，已使用个数为 nelts。  

#### 链表数据结构图

![memoryPoolDataStruction_listt.svg]({{site.baseurl}}/styles/images/nginx/memoryPoolDataStruction_listt.svg)  

### 链表操作函数

#### 链表创建 

链表创建时会从内存池中申请一个链表头，然后初始化链表头的各个部分，其中链表第一个元素中的 *elts 需要在此时给它申请内存，也从内存池中申请，申请大小为 nalloc * size。  

```
static ngx_inline ngx_int_t
ngx_list_init(ngx_list_t *list, ngx_pool_t *pool, ngx_uint_t n, size_t size)
{
    //为链表首元素申请数据内存空间
    list->part.elts = ngx_palloc(pool, n * size);
    if (list->part.elts == NULL) {
        return NGX_ERROR;
    }

    //初始化其它数据
    list->part.nelts = 0;
    list->part.next = NULL;
    list->last = &list->part;
    list->size = size;
    list->nalloc = n;
    list->pool = pool;

    return NGX_OK;
}

ngx_list_t *
ngx_list_create(ngx_pool_t *pool, ngx_uint_t n, size_t size)
{
    ngx_list_t  *list;

    //从内存池申请一个链表头
    list = ngx_palloc(pool, sizeof(ngx_list_t));
    if (list == NULL) {
        return NULL;
    }

	//初始化链表头
    if (ngx_list_init(list, pool, n, size) != NGX_OK) {
        return NULL;
    }

    return list;
}
```

#### 链表添加元素

向链表中添加元素都是添加到链表的最后位置，所以添加接口名为 ngx_list_push。添加元素时，会判断链表中是否有足够内存存储新节点，如果内存用完了再申请一块 nalloc * size 大小的内存挂载到链表最后。最后会把需要存储的下一个链表元素的内存返回。  

```
void *
ngx_list_push(ngx_list_t *l)
{
    void             *elt;
    ngx_list_part_t  *last;

    last = l->last;

    //当链表最后一个节点已使用完时，再申请一个part，挂载到链表最后。
    if (last->nelts == l->nalloc) {

        /* the last part is full, allocate a new list part */

        last = ngx_palloc(l->pool, sizeof(ngx_list_part_t));
        if (last == NULL) {
            return NULL;
        }

        last->elts = ngx_palloc(l->pool, l->nalloc * l->size);
        if (last->elts == NULL) {
            return NULL;
        }

        last->nelts = 0;
        last->next = NULL;

        l->last->next = last;
        l->last = last;
    }

    //获取下一个数据写入地址将其返回，并将链表使用数目++
    elt = (char *) last->elts + l->size * last->nelts;
    last->nelts++;

    return elt;
}
```

---
nginx源码基于nginx-1.22.0版本  
**参考**：  
[Nginx 链表结构 ngx_list_t](https://www.kancloud.cn/digest/understandingnginx/202591)   
[Nginx源码分析 - 基础数据结构篇 - 单向链表结构 ngx_list.c（06）](https://blog.csdn.net/initphp/article/details/50637104)  
