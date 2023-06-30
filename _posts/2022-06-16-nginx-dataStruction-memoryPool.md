---
layout: post
title: Nginx源码分析-数据结构之内存池 
date:   2022-06-16 16:20:00 +0800
categories: Nginx源码分析
tags: C nginx
topping: true
---

Nginx内存管理相关文件主要有：  
1. ngx_alloc.h, ngx_alloc.c  
这两个文件封装了内存相关的基本操作，包括 ngx_alloc, ngx_calloc, ngx_memalign 等。  
2. ngx_palloc.h, ngx_palloc.c
这两个文件主要是nginx内存池的操作，包括内存池的创建，销毁以及申请内存的操作。  

### 内存池数据结构

Nginx内存池主结构为 ngx_pool_t，定义如下：  

```
typedef struct ngx_pool_s            ngx_pool_t;

struct ngx_pool_s {
    ngx_pool_data_t       d;    //内存池数据块信息
    size_t                max;    //数据块大小，也是小块内存最大值
    ngx_pool_t           *current;    //当前可用于申请内存的内存池结点
    ngx_chain_t          *chain;    //ngx_chain_t链表
    ngx_pool_large_t     *large;    //大内存分配链表
    ngx_pool_cleanup_t   *cleanup;    //需要特殊清理工作的内存链表
    ngx_log_t            *log;    //日志指针
};
```

ngx_pool_t 结构相当于一个内存头，nginx 会申请一定大小的内存池，并在这个内存池前部存入这个内存头信息，其中 ngx_pool_data_t 结构里存储了这个内存池的数据块信息：  

```
typedef struct {
    u_char               *last;    //当前已分配内存的最后位置，即可分配内存的开始位置
    u_char               *end;    //当前内存池的结束位置
    ngx_pool_t           *next;    //指向下一个内存池节点
    ngx_uint_t            failed;    //当前内存池节点分配内存失败次数，失败达到一定次数则不再使用该节点
} ngx_pool_data_t;
```

内存池结构图如下：  

![memoryPoolDataStruction.svg]({{site.baseurl}}/styles/images/nginx/memoryPoolDataStruction.svg)  

ngx_pool_t 内存储了两个单向链表：large 和 cleanup，large 是一个 ngx_pool_large_t 的结构，当 nginx 需要大于内存池数据大小的内存时，会另外申请内存并将它挂载到 large 上，cleanup 是一个 ngx_pool_cleanup_t 结构，它里面的内存也是通过内存池函数 ngx_palloc 申请到的，只是在销毁内存池时需要先调用此结构里的清理句柄 handler 来进行特殊清理工作。这两个结构体定义：  

```
typedef void (*ngx_pool_cleanup_pt)(void *data);    //清理函数句柄定义

typedef struct ngx_pool_cleanup_s  ngx_pool_cleanup_t;

struct ngx_pool_cleanup_s {
    ngx_pool_cleanup_pt   handler;    //清理数据句柄
    void                 *data;    //内存指针
    ngx_pool_cleanup_t   *next;    //链表指向下一节点
};


typedef struct ngx_pool_large_s  ngx_pool_large_t;

struct ngx_pool_large_s {
    ngx_pool_large_t     *next;    //链表指向下一节点
    void                 *alloc;    //内存指针
};
```

结构图：  

![memoryPoolDataStruction_list.svg]({{site.baseurl}}/styles/images/nginx/memoryPoolDataStruction_list.svg)  

### 内存池操作函数

#### 创建内存池 ngx_create_pool  

这里申请了一块 size 大小的内存(除去内存头 ngx_pool_t 外其它部分用于小内存的申请)，并初始化了内存头的信息。  

```
ngx_pool_t *
ngx_create_pool(size_t size, ngx_log_t *log)
{
    ngx_pool_t  *p;

    p = ngx_memalign(NGX_POOL_ALIGNMENT, size, log);    //申请size大小内存
    if (p == NULL) {
        return NULL;
    }

    p->d.last = (u_char *) p + sizeof(ngx_pool_t);    //初始化数据指针，跳过内存头
    p->d.end = (u_char *) p + size;    //记录内存结束位置
    p->d.next = NULL;    //初始化内存池链表信息，新创建的内存池节点放到链表最后
    p->d.failed = 0;    //初始化申请内存失败次数

    size = size - sizeof(ngx_pool_t);    //计算数据块大小
    p->max = (size < NGX_MAX_ALLOC_FROM_POOL) ? size : NGX_MAX_ALLOC_FROM_POOL;    //ngx_pagesize默认初始值为０，此处max取size，后面ngx_pagesize初始化为getpagesize()

    p->current = p;    //初始化置为此节点
    p->chain = NULL;
    p->large = NULL;
    p->cleanup = NULL;
    p->log = log;

    return p;
}
```

#### 销毁内存池 ngx_destroy_pool  

销毁内存池操作会先循环cleanup链表，做内存相应的清理工作，然后释放大内存，最后释放内存池

```
void
ngx_destroy_pool(ngx_pool_t *pool)
{
    ngx_pool_t          *p, *n;
    ngx_pool_large_t    *l;
    ngx_pool_cleanup_t  *c;

    //这里先循环cleanup链表，调用handler，做清理工作
    for (c = pool->cleanup; c; c = c->next) {
        if (c->handler) {
            ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                           "run cleanup: %p", c);
            c->handler(c->data);
        }
    }

    //释放大内存
    //Debug模式下会记录日志
#if (NGX_DEBUG)

    /*
     * we could allocate the pool->log from this pool
     * so we cannot use this log while free()ing the pool
     */

    for (l = pool->large; l; l = l->next) {
        ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0, "free: %p", l->alloc);
    }

    for (p = pool, n = pool->d.next; /* void */; p = n, n = n->d.next) {
        ngx_log_debug2(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                       "free: %p, unused: %uz", p, p->d.end - p->d.last);

        if (n == NULL) {
            break;
        }
    }

#endif
    //释放内存，大内存的内存头也申请于内存池，所以这里不用单独释放
    for (l = pool->large; l; l = l->next) {
        if (l->alloc) {
            ngx_free(l->alloc);
        }
    }

    //释放内存池
    for (p = pool, n = pool->d.next; /* void */; p = n, n = n->d.next) {
        ngx_free(p);

        if (n == NULL) {
            break;
        }
    }
}
```

#### 重置内存池　ngx_reset_pool  

重置内存池时只释放大内存，不释放内存池，只将内存池置为未使用时的状态。  

```
void
ngx_reset_pool(ngx_pool_t *pool)
{
    ngx_pool_t        *p;
    ngx_pool_large_t  *l;

    //大内存直接释放，然后将pool->large置空
    for (l = pool->large; l; l = l->next) {
        if (l->alloc) {
            ngx_free(l->alloc);
        }
    }

    //内存池不需要释放，只要将里面的数据部分置为未使用时即可，即将内存最后使用位置置到内存头(ngx_pool_t)后
    for (p = pool; p; p = p->d.next) {
        p->d.last = (u_char *) p + sizeof(ngx_pool_t);
        p->d.failed = 0;
    }

    pool->current = pool;    //可使用内存池节点置为链表的第一个节点
    pool->chain = NULL;
    pool->large = NULL;
}
```

#### 内存的申请与释放

ngx_alloc.h, ngx_alloc.c 两个文件中封装了内存相关的基本操作：  

```
#define ngx_free          free  
void * ngx_alloc(size_t size, ngx_log_t *log)  
void * ngx_calloc(size_t size, ngx_log_t *log)  
void * ngx_memalign(size_t alignment, size_t size, ngx_log_t *log)  
```

其中，ngx_free 就是库函数 void free(void *ptr) 的别名。ngx_alloc 内部调用 malloc 来申请内存，ngx_calloc 除了调用 ngx_alloc 来申请内存外还会对申请的内存进行赋初始值 0 的操作，ngx_memalign 调用 posix_memalign 或 memalign 来申请内存以使申请的内存按 alignment 对齐。  

内存池申请释放内存有以下函数：  

```
void * ngx_palloc(ngx_pool_t *pool, size_t size)
void * ngx_pnalloc(ngx_pool_t *pool, size_t size)
void * ngx_pmemalign(ngx_pool_t *pool, size_t size, size_t alignment)
void * ngx_pcalloc(ngx_pool_t *pool, size_t size)
ngx_int_t ngx_pfree(ngx_pool_t *pool, void *p)
```

ngx_palloc 与 ngx_pnalloc 都会在申请内存前比较要申请内存大小与内存池的大小，如果不大于内存池大小则在内存池中申请内存，如果大于内存池大小则当作大内存处理，单独申请内存，两个函数的区别在于在内存池中申请小内存时是否需要作对齐操作。  

```
void *
ngx_palloc(ngx_pool_t *pool, size_t size)
{
    //查看要申请的内存是否大于内存池最大值，如果不大于则在内存池中申请，如果大于则当作大内存处理
#if !(NGX_DEBUG_PALLOC)
    if (size <= pool->max) {
        return ngx_palloc_small(pool, size, 1);    //最后一个操作为１表示申请内存时按 NGX_ALIGNMENT 对齐
    }
#endif

    return ngx_palloc_large(pool, size);
}


void *
ngx_pnalloc(ngx_pool_t *pool, size_t size)
{
#if !(NGX_DEBUG_PALLOC)
    if (size <= pool->max) {
        return ngx_palloc_small(pool, size, 0);    //不需要对齐操作
    }
#endif

    return ngx_palloc_large(pool, size);
}
```

小内存申请使用 ngx_palloc_small 函数，该函数从可申请内存池节点开始遍历内存池链表，直到找到可用大小的内存则返回。  

```
static ngx_inline void *
ngx_palloc_small(ngx_pool_t *pool, size_t size, ngx_uint_t align)
{
    u_char      *m;
    ngx_pool_t  *p;

    p = pool->current;

    do {
        //p->d.last为可使用内存开始位置
        m = p->d.last;

        //如果需要对齐，则将m调整为按NGX_ALIGNMENT 对齐的最近的地址
        if (align) {
            m = ngx_align_ptr(m, NGX_ALIGNMENT);
        }

        //判断内存池中剩余内存是否满足 size 大小，如果满足则将此地址返回，并将此段地址置为已使用
        if ((size_t) (p->d.end - m) >= size) {
            p->d.last = m + size;

            return m;
        }

        //如果不满足大小则判断下一个内存池节点
        p = p->d.next;

    } while (p);

    //遍历完内存池链表还没有找到可用内存则需要另外申请一个内存池节点
    return ngx_palloc_block(pool, size);
}
```

如果未找到可用的大小，则调用 ngx_palloc_block 来申请一个新的内存池节点，在新节点中返回一块可用内存。申请完后会对整个内存池链表进行整理，如果失败次数较多，则下次申请直接跳过，直接在可用内存池节点上申请。  

```
static void *
ngx_palloc_block(ngx_pool_t *pool, size_t size)
{
    u_char      *m;
    size_t       psize;
    ngx_pool_t  *p, *new;

    psize = (size_t) (pool->d.end - (u_char *) pool);    //每个内存池节点大小

    m = ngx_memalign(NGX_POOL_ALIGNMENT, psize, pool->log);    //以对齐方式申请一块内存块当作内存池节点
    if (m == NULL) {
        return NULL;
    }

    //初始化内存池头
    new = (ngx_pool_t *) m;

    new->d.end = m + psize;
    new->d.next = NULL;
    new->d.failed = 0;

    //在新申请的内存中开辟一块 size 大小的空间给用户返回使用，并将其置为已使用
    m += sizeof(ngx_pool_data_t);
    m = ngx_align_ptr(m, NGX_ALIGNMENT);
    new->d.last = m + size;

    //这里从当前可申请内存池节点开始遍历了一下内存池链表，将其申请失败次数++，
    //如果其失败次数大于4次，则将当前可用节点置为下一个，下次申请直接从current开始，
    //避免申请时做无用的遍历
    for (p = pool->current; p->d.next; p = p->d.next) {
        if (p->d.failed++ > 4) {
            pool->current = p->d.next;
        }
    }

    //将新申请的内存池放到内存池链表最后
    p->d.next = new;

    return m;
}
```

当要申请内存大于内存池大小时调用　ngx_palloc_large 来申请大内存。大内存的内存节点信息 ngx_pool_large_t 是从内存池中申请并挂载到内存池中large节点上的。申请到内存后遍历 pool->large 链表寻找空节点是与 ngx_free 对应的，因为释放大内存时只释放了申请的大内存，内存池中申请的 ngx_pool_large_t 未释放。  

```
static void *
ngx_palloc_large(ngx_pool_t *pool, size_t size)
{
    void              *p;
    ngx_uint_t         n;
    ngx_pool_large_t  *large;

    //直接调用ngx_alloc申请size 大小的内存
    p = ngx_alloc(size, pool->log);
    if (p == NULL) {
        return NULL;
    }

    //从large链表中找一个未使用的链表节点，将申请的内存挂载到这里
    //计数n 为防止遍历较多时影响效率
    n = 0;
    for (large = pool->large; large; large = large->next) {
        if (large->alloc == NULL) {
            large->alloc = p;
            return p;
        }

        if (n++ > 3) {
            break;
        }
    }

    //从内存池中申请一块ngx_pool_large_t结构，将申请到的p挂载到这里放到large链表头
    large = ngx_palloc_small(pool, sizeof(ngx_pool_large_t), 1);
    if (large == NULL) {
        ngx_free(p);
        return NULL;
    }

    large->alloc = p;
    large->next = pool->large;
    pool->large = large;    //这里放到链表头，效率高

    return p;
}
```

ngx_free 只用于释放大内存。  

```
ngx_int_t
ngx_pfree(ngx_pool_t *pool, void *p)
{
    ngx_pool_large_t  *l;

    for (l = pool->large; l; l = l->next) {
        if (p == l->alloc) {
            ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                           "free: %p", l->alloc);
            //只将申请的大内存释放并将l-alloc置空，未释放从内存池申请的ngx_pool_large_t链表节点
            ngx_free(l->alloc);
            l->alloc = NULL;

            return NGX_OK;
        }
    }

    return NGX_DECLINED;
}
```

ngx_pcalloc 函数中调用 ngx_palloc 来申请内存，只是在申请完内存后对其进行初始化为 0 操作。  

ngx_pmemalign 函数申请一块以 alignment 对齐的大内存，并将其挂载到 large 链表中。  

#### cleanup 内存操作

ngx_pool_cleanup_add 用于注册一个cleanup节点。  

```
ngx_pool_cleanup_t *
ngx_pool_cleanup_add(ngx_pool_t *p, size_t size)
{
    ngx_pool_cleanup_t  *c;

    //申请一个ngx_pool_cleanup_t结构用于挂载到cleanup链表
    c = ngx_palloc(p, sizeof(ngx_pool_cleanup_t));
    if (c == NULL) {
        return NULL;
    }

    //调用ngx_palloc申请一块size大小的数据
    if (size) {
        c->data = ngx_palloc(p, size);
        if (c->data == NULL) {
            return NULL;
        }

    } else {
        c->data = NULL;
    }

    //挂到内存池cleanup链表上
    c->handler = NULL;
    c->next = p->cleanup;

    p->cleanup = c;

    ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, p->log, 0, "add cleanup: %p", c);

    return c;
}
```

cleanup机制可以通过回调函数清理数据，典型应用是对文件的关闭和删除。nginx 也提供了两个函数来实现文件的回调：  

```
void ngx_pool_cleanup_file(void *data);    //关闭文件
void ngx_pool_delete_file(void *data)    //删除文件
```

ngx_pool_run_cleanup_file 用于遍历 cleanup 链表关闭文件。  


---
nginx源码基于nginx-1.22.0版本  
**参考**：  
[Nginx 内存池管理](https://www.kancloud.cn/digest/understandingnginx/202588)   
[Nginx源码分析 - 基础数据结构篇 - 内存池 ngx_palloc.c（02）](https://blog.csdn.net/initphp/article/details/50588790)  
