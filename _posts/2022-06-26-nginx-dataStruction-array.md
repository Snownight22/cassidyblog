---
layout: post
title: Nginx源码分析-数据结构之数组array 
date:   2022-06-26 16:50:00 +0800
categories: Nginx源码分析
tags: C nginx
topping: true
---

Nginx 数组数据结构定义为 ngx_array_t，与链表的定义有些类似，数组内存也是基于 nginx 的内存池进行分配的，其操作也与内存池紧密关联。  

### 数组数据结构

#### ngx_array_t 定义

```
//array结构体定义
typedef struct {
    void        *elts;    //数组数据存储地址
    ngx_uint_t   nelts;    //数组存储个数
    size_t       size;    //数组元素大小
    ngx_uint_t   nalloc;    //array结构总共能容纳数据个数
    ngx_pool_t  *pool;    //内存池
} ngx_array_t;
```

ngx_array_t 数组头结构内定义了数组结构总共能够容纳数据的个数 nalloc，数组元素的大小 size，数组已存储个数 nelts，以及申请内存所用到的内存池指针。  

#### 数组结构图

![memoryPoolDataStruction_array.svg]({{site.baseurl}}/styles/images/nginx/memoryPoolDataStruction_array.svg)  

### 数组操作函数

#### 数组的创建与销毁 

数组的创建会从内存池中申请一个数组内存头加初始化时申请的数组数据内存，并对数组各数据进行初始化。  

```
//初始化array
static ngx_inline ngx_int_t
ngx_array_init(ngx_array_t *array, ngx_pool_t *pool, ngx_uint_t n, size_t size)
{
    /*
     * set "array->nelts" before "array->elts", otherwise MSVC thinks
     * that "array->nelts" may be used without having been initialized
     */

    array->nelts = 0;
    array->size = size;
    array->nalloc = n;
    array->pool = pool;

    array->elts = ngx_palloc(pool, n * size);
    if (array->elts == NULL) {
        return NGX_ERROR;
    }

    return NGX_OK;
}

ngx_array_t *
ngx_array_create(ngx_pool_t *p, ngx_uint_t n, size_t size)
{
    ngx_array_t *a;

    a = ngx_palloc(p, sizeof(ngx_array_t));
    if (a == NULL) {
        return NULL;
    }

    if (ngx_array_init(a, p, n, size) != NGX_OK) {
        return NULL;
    }

    return a;
}
```

销毁数组内存会判断数组内存部分，如果可能的话将内存归还内存池。  

```
//销毁ngx_array_t内存
void
ngx_array_destroy(ngx_array_t *a)
{
    ngx_pool_t  *p;

    p = a->pool;

    //判断a中数据部分是否为内存池最后申请部分，如果是，则归还内存池内存
    if ((u_char *) a->elts + a->size * a->nalloc == p->d.last) {
        p->d.last -= a->size * a->nalloc;
    }

    //判断a中array_t头部分是否为内存池最后申请部分，如果是，则归还内存池内存
    if ((u_char *) a + sizeof(ngx_array_t) == p->d.last) {
        p->d.last = (u_char *) a;
    }
}
```

#### 数组添加元素

添加元素有两个接口，向数组中添加一个数据 ngx_array_push 和向数组中添加 n 个数据 ngx_array_push_n。  
这两个接口在每次添加新元素时会对当前存储个数进行判断，如果剩余内存不足以存储要放入的个数，则需要申请新内存。因为数组的内存是从内存池中申请的，因此在重新申请时会对内存池中数据进行判断，查看内存池中最后的数据部分是否为数组内存，如果是，则直接将这部分内存扩容给数组，省去了重新申请并拷贝的消耗。  

```
void *
ngx_array_push(ngx_array_t *a)
{
    void        *elt, *new;
    size_t       size;
    ngx_pool_t  *p;

    //当array中数据个数达到最大值时，申请新的内存
    if (a->nelts == a->nalloc) {

        /* the array is full */

        size = a->size * a->nalloc;

        p = a->pool;
        //申请新内存时会在内存池中判断，如果内存池数据最后部分正好存储的是array_t的数据，且还有空间可添加新元素，则直接将这部分内存池扩容给a->elt, 省去了重新申请的消耗
        if ((u_char *) a->elts + size == p->d.last
            && p->d.last + a->size <= p->d.end)
        {
            /*
             * the array allocation is the last in the pool
             * and there is space for new allocation
             */

            p->d.last += a->size;
            a->nalloc++;

        } else {//当内存池不满足情况时，则再申请一块内存来存储数据，每次容量*2
            /* allocate a new array */

            new = ngx_palloc(p, 2 * size);
            if (new == NULL) {
                return NULL;
            }

            ngx_memcpy(new, a->elts, size);
            a->elts = new;
            a->nalloc *= 2;
        }
    }

    //将要写入元素的地址返回，并将数组元素个数++
    elt = (u_char *) a->elts + a->size * a->nelts;
    a->nelts++;

    return elt;
}


void *
ngx_array_push_n(ngx_array_t *a, ngx_uint_t n)
{
    void        *elt, *new;
    size_t       size;
    ngx_uint_t   nalloc;
    ngx_pool_t  *p;

    size = n * a->size;

    if (a->nelts + n > a->nalloc) {

        /* the array is full */

        p = a->pool;

        //这里与ngx_array_push相同，只是判断可容纳容量时变为了n*a->size，即n个元素
        if ((u_char *) a->elts + a->size * a->nalloc == p->d.last
            && p->d.last + size <= p->d.end)
        {
            /*
             * the array allocation is the last in the pool
             * and there is space for new allocation
             */

            p->d.last += size;
            a->nalloc += n;

        } else {//再申请内存时会判断n与a->nalloc，取较大值
            /* allocate a new array */

            nalloc = 2 * ((n >= a->nalloc) ? n : a->nalloc);

            new = ngx_palloc(p, nalloc * a->size);
            if (new == NULL) {
                return NULL;
            }

            ngx_memcpy(new, a->elts, a->nelts * a->size);
            a->elts = new;
            a->nalloc = nalloc;
        }
    }

    //将要写入元素的地址返回，并将数组元素个数+n
    elt = (u_char *) a->elts + a->size * a->nelts;
    a->nelts += n;

    return elt;
}
```

---
nginx源码基于nginx-1.22.0版本  
**参考**：  
[Nginx 数组结构 ngx_array_t](https://www.kancloud.cn/digest/understandingnginx/202590)   
