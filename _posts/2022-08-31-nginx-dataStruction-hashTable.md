---
layout: post
title: Nginx源码分析-数据结构之哈希表hashTable 
date:   2022-08-31 13:35:00 +0800
categories: Nginx源码分析
tags: C nginx
topping: true
---

Nginx 中定义了自己的哈希表 ngx_hash_t，定义在 ngx_hash.c/h 中，有关哈希表可参考[散列表（一）- 散列表及常用字符串哈希函数]({{site.baseurl}}/2022/07/15/hashTable) 及 [散列表（二）- 散列表解决冲突的方式]({{site.baseurl}}/2022/07/18/hashTable-collisionResolution)。  

### 数据结构定义

哈希表元素定义为 ngx_hash_elt_t，它的链值采用的是变长数组的方式，len 记录链值的长度。哈希表元素及哈希表结构定义：  

```
//哈希元素结构，键值对
typedef struct {
    void             *value;    //哈希元素值
    u_short           len;    //哈希元素键长度
    u_char            name[1];    //哈希元素键值，这里采用的是变长数组，name是键首地址
} ngx_hash_elt_t;

//哈希表结构
typedef struct {
    ngx_hash_elt_t  **buckets;    //哈希桶
    ngx_uint_t        size;    //哈希表桶个数
} ngx_hash_t;

```

nginx 的哈希表在创建之后就不会改变，因此不会有哈希表的插入删除操作。nginx 哈希表解决冲突的方式是链接法，但与一般链接法不同的是，它不是采用的链表结构，而是在一开始初始化时就把所有哈希表的元素放到一个数组中，然后每个哈希桶指向对应数组的索引地址上。如下图中的哈希元素数组。哈希表初始化时使用了一个结构 ngx_hash_init_t，用于记录哈希表的各个信息：  

```
typedef struct {
    ngx_hash_t       *hash;    //哈希表结构
    ngx_hash_key_pt   key;    //哈希函数

    ngx_uint_t        max_size;    //哈希桶个数最大值
    ngx_uint_t        bucket_size;    //桶大小最大值

    char             *name;    //哈希结构名称
    ngx_pool_t       *pool;    //分配内存的内存池
    ngx_pool_t       *temp_pool;
} ngx_hash_init_t;
```

哈希表结构图：  

![ngxDataStruct_hash.png]({{site.imgurl}}/styles/images/nginx/ngxDataStruct_hash.png)  

nginx 中还定义了用于通配符的哈希表：  

```
//通配符哈希表
typedef struct {
    ngx_hash_t        hash;
    void             *value;
} ngx_hash_wildcard_t;

typedef struct {
    ngx_hash_t            hash;    //普通哈希表
    ngx_hash_wildcard_t  *wc_head;    //通配符在前面的哈希表
    ngx_hash_wildcard_t  *wc_tail;    //通配符在后面的哈希表
} ngx_hash_combined_t;
```

另外还定义了两个处理哈希表 key 的结构 ngx_hash_key_t 和 ngx_hash_keys_arrays_t，ngx_hash_key_t 在哈希表初始化时以数组形式存储所有的哈希表 key，ngx_hash_keys_arrays_t 在初始化哈希表前对所有的 key 进行处理，包括普通 key，前缀通配符 key 和后缀通配符 key：  

```
//用来添加到哈希表的键值对结构
typedef struct {
    ngx_str_t         key;    //键
    ngx_uint_t        key_hash;    //键的哈希值
    void             *value;    //值
} ngx_hash_key_t;

typedef struct {
    ngx_uint_t        hsize;    //要构建的哈希表桶的个数，在ngx_hash_keys_array_init函数中对其进行初始化赋初值

    ngx_pool_t       *pool;    //使用的内存池
    ngx_pool_t       *temp_pool;    //临时内存池

    ngx_array_t       keys;    //存储所有非通配符key
    ngx_array_t      *keys_hash;    //存储所有非通配符key的哈希结构，用于查找是否有重复

    ngx_array_t       dns_wc_head;    //存储所有通配符在前面的key，这里会对其进行处理下，按点号分隔反转
    ngx_array_t      *dns_wc_head_hash;    //存储所有通配符在前的key的哈希结构，用于查找是否有重复

    ngx_array_t       dns_wc_tail;    //存储所有通配符在后面的key，这里会对其进行处理，将末尾的通配符去除
    ngx_array_t      *dns_wc_tail_hash;    //存储所有通配符在后的key的哈希结构，用于查找是否有重复
} ngx_hash_keys_arrays_t;
```

### 哈希表初始化

nginx 中采用的哈希函数：  

```
//哈希函数，用的是BKDRHash方法
ngx_uint_t
ngx_hash_key(u_char *data, size_t len)
{
    ngx_uint_t  i, key;

    key = 0;

    for (i = 0; i < len; i++) {
        key = ngx_hash(key, data[i]);
    }

    return key;
}

//转为小写的哈希函数
ngx_uint_t
ngx_hash_key_lc(u_char *data, size_t len)
{
    ngx_uint_t  i, key;

    key = 0;

    for (i = 0; i < len; i++) {
        key = ngx_hash(key, ngx_tolower(data[i]));
    }

    return key;
}

//将src转为小写放到dst中，并返回其hash值
ngx_uint_t
ngx_hash_strlow(u_char *dst, u_char *src, size_t n)
{
    ngx_uint_t  key;

    key = 0;

    while (n--) {
        *dst = ngx_tolower(*src);
        key = ngx_hash(key, *dst);
        dst++;
        src++;
    }

    return key;
}
```

ngx_hash_elt_t 元素为变长结构，且按地址进行了对齐，因此哈希元素占用空间计算如下：  

```
//ngx_hash_key 转为 ngx_hash_elt_t 结构后的大小，value + len + str;
#define NGX_HASH_ELT_SIZE(name)                                               \
    (sizeof(void *) + ngx_align((name)->key.len + 2, sizeof(void *)))
```

哈希表初始化时会申请一个大的数组，将所有 key 按哈希值顺序放到数组中，然后将哈希桶指向不同的索引。因此在创建哈希桶前需要判断每个哈希桶中包含几个数组元素，所以 nginx 在初始化时会遍历所有的 key，计算每个哈希值对应的元素个数、长度用所有元素占用空间大小，然后申请内存，将元素写入数组，最后将哈希桶指向对应索引的地址。  

```
ngx_int_t
ngx_hash_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names, ngx_uint_t nelts)
{
    u_char          *elts;
    size_t           len;
    u_short         *test;
    ngx_uint_t       i, n, key, size, start, bucket_size;
    ngx_hash_elt_t  *elt, **buckets;

    if (hinit->max_size == 0) {
        ngx_log_error(NGX_LOG_EMERG, hinit->pool->log, 0,
                      "could not build %s, you should "
                      "increase %s_max_size: %i",
                      hinit->name, hinit->name, hinit->max_size);
        return NGX_ERROR;
    }

    if (hinit->bucket_size > 65536 - ngx_cacheline_size) {
        ngx_log_error(NGX_LOG_EMERG, hinit->pool->log, 0,
                      "could not build %s, too large "
                      "%s_bucket_size: %i",
                      hinit->name, hinit->name, hinit->bucket_size);
        return NGX_ERROR;
    }

    //循环所有names，判断是否有较长有name，如果容量大于bucket_size，则需要增长bucket_size大小
    for (n = 0; n < nelts; n++) {
        if (names[n].key.data == NULL) {
            continue;
        }

        if (hinit->bucket_size < NGX_HASH_ELT_SIZE(&names[n]) + sizeof(void *))
        {
            ngx_log_error(NGX_LOG_EMERG, hinit->pool->log, 0,
                          "could not build %s, you should "
                          "increase %s_bucket_size: %i",
                          hinit->name, hinit->name, hinit->bucket_size);
            return NGX_ERROR;
        }
    }

    //初始化会先计算names所需哈希桶的最小个数，再计算每个桶容量大小，
    //然后分配一整块内存，存放names转化的halt_elt结构，将每个hash桶元素指针指向整块内存的对应结点，即按顺序及每个桶容量来分配每个哈希桶指向的指针。
    //详细可参见hash表结构图
    test = ngx_alloc(hinit->max_size * sizeof(u_short), hinit->pool->log);
    if (test == NULL) {
        return NGX_ERROR;
    }

    //末尾要加一个空指针做为结尾，所以减去这一个指针的空间
    bucket_size = hinit->bucket_size - sizeof(void *);

    //ngx_hash_elt_t结构最少使用两个指针的存储空间，按最少使用两个指针的存储空间来计算哈希桶的最少使用个数。
    start = nelts / (bucket_size / (2 * sizeof(void *)));
    start = start ? start : 1;    //最少为1

    if (hinit->max_size > 10000 && nelts && hinit->max_size / nelts < 100) {
        start = hinit->max_size - 1000;
    }

    //从最少使用的个数start开始，判断是否可以容纳下所有元素，不可以则将哈希桶个数递增
    for (size = start; size <= hinit->max_size; size++) {

        ngx_memzero(test, size * sizeof(u_short));

        for (n = 0; n < nelts; n++) {
            if (names[n].key.data == NULL) {
                continue;
            }

            key = names[n].key_hash % size;
            len = test[key] + NGX_HASH_ELT_SIZE(&names[n]);

#if 0
            ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                          "%ui: %ui %uz \"%V\"",
                          size, key, len, &names[n].key);
#endif

            if (len > bucket_size) {
                goto next;
            }

            test[key] = (u_short) len;
        }

        goto found;

    next:

        continue;
    }

    size = hinit->max_size;

    ngx_log_error(NGX_LOG_WARN, hinit->pool->log, 0,
                  "could not build optimal %s, you should increase "
                  "either %s_max_size: %i or %s_bucket_size: %i; "
                  "ignoring %s_bucket_size",
                  hinit->name, hinit->name, hinit->max_size,
                  hinit->name, hinit->bucket_size, hinit->name);

found:

    //开始计算每个桶需要占用的存储空间大小，存在test里，这里先将最后的空指针记录在内
    for (i = 0; i < size; i++) {
        test[i] = sizeof(void *);
    }

    //将所有元素映射到哈希桶中，计算占用空间
    for (n = 0; n < nelts; n++) {
        if (names[n].key.data == NULL) {
            continue;
        }

        key = names[n].key_hash % size;
        len = test[key] + NGX_HASH_ELT_SIZE(&names[n]);

        if (len > 65536 - ngx_cacheline_size) {
            ngx_log_error(NGX_LOG_EMERG, hinit->pool->log, 0,
                          "could not build %s, you should "
                          "increase %s_max_size: %i",
                          hinit->name, hinit->name, hinit->max_size);
            ngx_free(test);
            return NGX_ERROR;
        }

        test[key] = (u_short) len;
    }

    len = 0;

    //对每个hash桶的内存做cpuCache对齐，减少cache失效
    for (i = 0; i < size; i++) {
        if (test[i] == sizeof(void *)) {
            continue;
        }

        test[i] = (u_short) (ngx_align(test[i], ngx_cacheline_size));

        len += test[i];    //计算总长度
    }

    //申请hash表头内存，如果hinit->hash是空时，需要添加一个ngx_hash_wildcard_t头，然后是哈希表桶结构
    if (hinit->hash == NULL) {
        hinit->hash = ngx_pcalloc(hinit->pool, sizeof(ngx_hash_wildcard_t)
                                             + size * sizeof(ngx_hash_elt_t *));
        if (hinit->hash == NULL) {
            ngx_free(test);
            return NGX_ERROR;
        }

        buckets = (ngx_hash_elt_t **)
                      ((u_char *) hinit->hash + sizeof(ngx_hash_wildcard_t));

    } else {
        buckets = ngx_pcalloc(hinit->pool, size * sizeof(ngx_hash_elt_t *));
        if (buckets == NULL) {
            ngx_free(test);
            return NGX_ERROR;
        }
    }

    //申请一块块内存，组成一个哈希结点的链表，存储所有的哈希结点
    elts = ngx_palloc(hinit->pool, len + ngx_cacheline_size);
    if (elts == NULL) {
        ngx_free(test);
        return NGX_ERROR;
    }

    elts = ngx_align_ptr(elts, ngx_cacheline_size);

    //计算每个桶对应的存储哈希结点的内存的位置，test[i]为第i个哈希桶所占用的内存，
    for (i = 0; i < size; i++) {
        //只占用一个void*大小时，说明此哈希桶中没有结点，不占用空间。
        if (test[i] == sizeof(void *)) {
            continue;
        }

        buckets[i] = (ngx_hash_elt_t *) elts;
        elts += test[i];
    }

    for (i = 0; i < size; i++) {
        test[i] = 0;
    }

    //将结点写入哈希桶中
    for (n = 0; n < nelts; n++) {
        if (names[n].key.data == NULL) {
            continue;
        }

        key = names[n].key_hash % size;
        elt = (ngx_hash_elt_t *) ((u_char *) buckets[key] + test[key]);

        elt->value = names[n].value;
        elt->len = (u_short) names[n].key.len;

        ngx_strlow(elt->name, names[n].key.data, names[n].key.len);

        test[key] = (u_short) (test[key] + NGX_HASH_ELT_SIZE(&names[n]));
    }

    //最后一个结点置为空指针，这里是将最后一个结点的value值置为NULL。
    for (i = 0; i < size; i++) {
        if (buckets[i] == NULL) {
            continue;
        }

        elt = (ngx_hash_elt_t *) ((u_char *) buckets[i] + test[i]);

        elt->value = NULL;
    }

    ngx_free(test);

    hinit->hash->buckets = buckets;
    hinit->hash->size = size;

#if 0

    for (i = 0; i < size; i++) {
        ngx_str_t   val;
        ngx_uint_t  key;

        elt = buckets[i];

        if (elt == NULL) {
            ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                          "%ui: NULL", i);
            continue;
        }

        while (elt->value) {
            val.len = elt->len;
            val.data = &elt->name[0];

            key = hinit->key(val.data, val.len);

            ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                          "%ui: %p \"%V\" %ui", i, elt, &val, key);

            elt = (ngx_hash_elt_t *) ngx_align_ptr(&elt->name[0] + elt->len,
                                                   sizeof(void *));
        }
    }

#endif

    return NGX_OK;
}
```

nginx 在初始化前会对所用的 key 进行处理，这里用到两个处理函数，初始化 key 数组的函数 ngx_hash_keys_array_init 和添加 key 的函数 ngx_hash_add_key：  

```
//初始化ha，给结构体内数据赋初始值或申请内存
ngx_int_t
ngx_hash_keys_array_init(ngx_hash_keys_arrays_t *ha, ngx_uint_t type)
{
    ngx_uint_t  asize;

    if (type == NGX_HASH_SMALL) {
        asize = 4;
        ha->hsize = 107;

    } else {
        asize = NGX_HASH_LARGE_ASIZE;
        ha->hsize = NGX_HASH_LARGE_HSIZE;
    }

    if (ngx_array_init(&ha->keys, ha->temp_pool, asize, sizeof(ngx_hash_key_t))
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    if (ngx_array_init(&ha->dns_wc_head, ha->temp_pool, asize,
                       sizeof(ngx_hash_key_t))
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    if (ngx_array_init(&ha->dns_wc_tail, ha->temp_pool, asize,
                       sizeof(ngx_hash_key_t))
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    ha->keys_hash = ngx_pcalloc(ha->temp_pool, sizeof(ngx_array_t) * ha->hsize);
    if (ha->keys_hash == NULL) {
        return NGX_ERROR;
    }

    ha->dns_wc_head_hash = ngx_pcalloc(ha->temp_pool,
                                       sizeof(ngx_array_t) * ha->hsize);
    if (ha->dns_wc_head_hash == NULL) {
        return NGX_ERROR;
    }

    ha->dns_wc_tail_hash = ngx_pcalloc(ha->temp_pool,
                                       sizeof(ngx_array_t) * ha->hsize);
    if (ha->dns_wc_tail_hash == NULL) {
        return NGX_ERROR;
    }

    return NGX_OK;
}

//将key值进行处理并添加进ha中，方便后续创建哈希表
ngx_int_t
ngx_hash_add_key(ngx_hash_keys_arrays_t *ha, ngx_str_t *key, void *value,
    ngx_uint_t flags)
{
    size_t           len;
    u_char          *p;
    ngx_str_t       *name;
    ngx_uint_t       i, k, n, skip, last;
    ngx_array_t     *keys, *hwc;
    ngx_hash_key_t  *hk;

    last = key->len;

    //flags标识是否支持通配符还是只读
    if (flags & NGX_HASH_WILDCARD_KEY) {

        /*
         * supported wildcards:
         *     "*.example.com", ".example.com", and "www.example.*"
         */

        n = 0;

        //判断key值是否合法
        for (i = 0; i < key->len; i++) {

            if (key->data[i] == '*') {
                if (++n > 1) {
                    return NGX_DECLINED;
                }
            }

            if (key->data[i] == '.' && key->data[i + 1] == '.') {
                return NGX_DECLINED;
            }

            if (key->data[i] == '\0') {
                return NGX_DECLINED;
            }
        }

        //判断是否有通配符及通配符在前还是在后，记录其位置，如果有通配符则直接跳到wildcard标号进行处理
        if (key->len > 1 && key->data[0] == '.') {
            skip = 1;
            goto wildcard;
        }

        if (key->len > 2) {

            if (key->data[0] == '*' && key->data[1] == '.') {
                skip = 2;
                goto wildcard;
            }

            if (key->data[i - 2] == '.' && key->data[i - 1] == '*') {
                skip = 0;
                last -= 2;
                goto wildcard;
            }
        }

        if (n) {
            return NGX_DECLINED;
        }
    }

    /* exact hash */

    //不带通配符的处理
    k = 0;
    //计算其哈希值
    for (i = 0; i < last; i++) {
        if (!(flags & NGX_HASH_READONLY_KEY)) {
            key->data[i] = ngx_tolower(key->data[i]);
        }
        k = ngx_hash(k, key->data[i]);
    }

    k %= ha->hsize;    //k计算为哈希表索引

    /* check conflicts in exact hash */

    //判断此key值是否有重复
    name = ha->keys_hash[k].elts;
    //name为空说明还未存储值，则需要对其进行初始化，不为空则遍历查找是否重复，这里使用的都是链接法
    if (name) {
        for (i = 0; i < ha->keys_hash[k].nelts; i++) {
            if (last != name[i].len) {
                continue;
            }

            if (ngx_strncmp(key->data, name[i].data, last) == 0) {
                return NGX_BUSY;
            }
        }

    } else {
        if (ngx_array_init(&ha->keys_hash[k], ha->temp_pool, 4,
                           sizeof(ngx_str_t))
            != NGX_OK)
        {
            return NGX_ERROR;
        }
    }

    //未找到则添加进keys_hash[k]中
    name = ngx_array_push(&ha->keys_hash[k]);
    if (name == NULL) {
        return NGX_ERROR;
    }

    *name = *key;

    //把key添加进ha->keys数组中
    hk = ngx_array_push(&ha->keys);
    if (hk == NULL) {
        return NGX_ERROR;
    }

    hk->key = *key;
    hk->key_hash = ngx_hash_key(key->data, last);
    hk->value = value;

    return NGX_OK;


wildcard:

    /* wildcard hash */
    //转为小写并计算其哈希值
    k = ngx_hash_strlow(&key->data[skip], &key->data[skip], last - skip);

    k %= ha->hsize;    //计算索引

    //以点结尾的去除前面的点后的字符串存于ha->keys及ha->keys_hash中
    if (skip == 1) {
        //处理过程同前面普通字符串处理相同
        /* check conflicts in exact hash for ".example.com" */

        name = ha->keys_hash[k].elts;

        if (name) {
            len = last - skip;

            for (i = 0; i < ha->keys_hash[k].nelts; i++) {
                if (len != name[i].len) {
                    continue;
                }

                if (ngx_strncmp(&key->data[1], name[i].data, len) == 0) {
                    return NGX_BUSY;
                }
            }

        } else {
            if (ngx_array_init(&ha->keys_hash[k], ha->temp_pool, 4,
                               sizeof(ngx_str_t))
                != NGX_OK)
            {
                return NGX_ERROR;
            }
        }

        name = ngx_array_push(&ha->keys_hash[k]);
        if (name == NULL) {
            return NGX_ERROR;
        }

        name->len = last - 1;
        name->data = ngx_pnalloc(ha->temp_pool, name->len);
        if (name->data == NULL) {
            return NGX_ERROR;
        }

        ngx_memcpy(name->data, &key->data[1], name->len);
    }

    //前面带'.'及'*.'的，将其去除前面通配符及点，剩余部分按点分隔反转。如下面注释中:将"*.example.com" 转化为 "com.example.\0" ".example.com" 转化为"com.example\0"
    if (skip) {

        /*
         * convert "*.example.com" to "com.example.\0"
         *      and ".example.com" to "com.example\0"
         */

        p = ngx_pnalloc(ha->temp_pool, last);
        if (p == NULL) {
            return NGX_ERROR;
        }

        len = 0;
        n = 0;

        //这里是按点分隔反转操作
        for (i = last - 1; i; i--) {
            if (key->data[i] == '.') {
                ngx_memcpy(&p[n], &key->data[i + 1], len);
                n += len;
                p[n++] = '.';
                len = 0;
                continue;
            }

            len++;
        }

        if (len) {
            ngx_memcpy(&p[n], &key->data[1], len);
            n += len;
        }

        p[n] = '\0';

        hwc = &ha->dns_wc_head;
        keys = &ha->dns_wc_head_hash[k];

    } else {

        /* convert "www.example.*" to "www.example\0" */
        //带后缀的，将其后面的'.*'去除，如注释: 将"www.example.*" 转化为 "www.example\0" 
        last++;

        p = ngx_pnalloc(ha->temp_pool, last);
        if (p == NULL) {
            return NGX_ERROR;
        }

        ngx_cpystrn(p, key->data, last);

        hwc = &ha->dns_wc_tail;
        keys = &ha->dns_wc_tail_hash[k];
    }


    /* check conflicts in wildcard hash */

    //检查是否有重复
    name = keys->elts;

    if (name) {
        len = last - skip;

        for (i = 0; i < keys->nelts; i++) {
            if (len != name[i].len) {
                continue;
            }

            if (ngx_strncmp(key->data + skip, name[i].data, len) == 0) {
                return NGX_BUSY;
            }
        }

    } else {
        if (ngx_array_init(keys, ha->temp_pool, 4, sizeof(ngx_str_t)) != NGX_OK)
        {
            return NGX_ERROR;
        }
    }

    //分别将其加入到对应的数组中
    name = ngx_array_push(keys);
    if (name == NULL) {
        return NGX_ERROR;
    }

    name->len = last - skip;
    name->data = ngx_pnalloc(ha->temp_pool, name->len);
    if (name->data == NULL) {
        return NGX_ERROR;
    }

    ngx_memcpy(name->data, key->data + skip, name->len);


    /* add to wildcard hash */

    hk = ngx_array_push(hwc);
    if (hk == NULL) {
        return NGX_ERROR;
    }

    hk->key.len = last - 1;
    hk->key.data = p;
    hk->key_hash = 0;
    hk->value = value;

    return NGX_OK;
}
```

带通配符的哈希表的初始化过程：  

```
//对带通配符的哈希表进行初始化，如果前缀相同，则将其放于同一表项，并在该表项ngx_hash_elt_t的value部分存储一个由这些相同前缀字符串的后缀组成的哈希表
ngx_int_t
ngx_hash_wildcard_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names,
    ngx_uint_t nelts)
{
    size_t                len, dot_len;
    ngx_uint_t            i, n, dot;
    ngx_array_t           curr_names, next_names;
    ngx_hash_key_t       *name, *next_name;
    ngx_hash_init_t       h;
    ngx_hash_wildcard_t  *wdc;

    //curr_names存储当前哈希表的字符串数组
    if (ngx_array_init(&curr_names, hinit->temp_pool, nelts,
                       sizeof(ngx_hash_key_t))
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    //next_names存储具有相同前缀的字符串数组
    if (ngx_array_init(&next_names, hinit->temp_pool, nelts,
                       sizeof(ngx_hash_key_t))
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    //遍历所有字符串，对其进行分类
    for (n = 0; n < nelts; n = i) {

#if 0
        ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                      "wc0: \"%V\"", &names[n].key);
#endif

        dot = 0;

        //查找到第一个"."，将点前的部分计算哈希值并存储到当前哈希表的字符串数组中
        for (len = 0; len < names[n].key.len; len++) {
            if (names[n].key.data[len] == '.') {
                dot = 1;
                break;
            }
        }

        name = ngx_array_push(&curr_names);
        if (name == NULL) {
            return NGX_ERROR;
        }

        name->key.len = len;
        name->key.data = names[n].key.data;
        name->key_hash = hinit->key(name->key.data, name->key.len);
        name->value = names[n].value;

#if 0
        ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                      "wc1: \"%V\" %ui", &name->key, dot);
#endif

        dot_len = len + 1;

        if (dot) {
            len++;
        }

        next_names.nelts = 0;

        //"."后有数据则将其放到next_names数组中
        if (names[n].key.len != len) {
            next_name = ngx_array_push(&next_names);
            if (next_name == NULL) {
                return NGX_ERROR;
            }

            next_name->key.len = names[n].key.len - len;
            next_name->key.data = names[n].key.data + len;
            next_name->key_hash = 0;
            next_name->value = names[n].value;

#if 0
            ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                          "wc2: \"%V\"", &next_name->key);
#endif
        }
        //查找后面的字符是否带有相同的前缀，如果带有则放到next_names数组中
        for (i = n + 1; i < nelts; i++) {
            if (ngx_strncmp(names[n].key.data, names[i].key.data, len) != 0) {
                break;
            }

            if (!dot
                && names[i].key.len > len
                && names[i].key.data[len] != '.')
            {
                break;
            }

            next_name = ngx_array_push(&next_names);
            if (next_name == NULL) {
                return NGX_ERROR;
            }

            next_name->key.len = names[i].key.len - dot_len;
            next_name->key.data = names[i].key.data + dot_len;
            next_name->key_hash = 0;
            next_name->value = names[i].value;

#if 0
            ngx_log_error(NGX_LOG_ALERT, hinit->pool->log, 0,
                          "wc3: \"%V\"", &next_name->key);
#endif
        }

        //如果next_names.nelts不为0，说明有前缀相同的字符串，此时递归调用ngx_hash_wildcard_init创建新的哈希表，将哈希表地址放到name->value中，并对其地址进行特殊处理，后两位添加标志来标识不同的匹配字符串
        if (next_names.nelts) {

            h = *hinit;
            h.hash = NULL;

            if (ngx_hash_wildcard_init(&h, (ngx_hash_key_t *) next_names.elts,
                                       next_names.nelts)
                != NGX_OK)
            {
                return NGX_ERROR;
            }

            wdc = (ngx_hash_wildcard_t *) h.hash;

            if (names[n].key.len == len) {
                wdc->value = names[n].value;
            }

            name->value = (void *) ((uintptr_t) wdc | (dot ? 3 : 2));

        } else if (dot) {
            name->value = (void *) ((uintptr_t) name->value | 1);
        }
    }

    //对curr_names创建哈希表
    if (ngx_hash_init(hinit, (ngx_hash_key_t *) curr_names.elts,
                      curr_names.nelts)
        != NGX_OK)
    {
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

### 哈希表查找

nginx 中的哈希表创建后不会再添加删除元素，因此哈希表只有查找操作。  

查找普通哈希表：  

```
//在hash表中查找以name为键的value值， hash-nginx哈希表结构，key-name通过哈希算法得出的键值，len-name的长度
void *
ngx_hash_find(ngx_hash_t *hash, ngx_uint_t key, u_char *name, size_t len)
{
    ngx_uint_t       i;
    ngx_hash_elt_t  *elt;

#if 0
    ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0, "hf:\"%*s\"", len, name);
#endif

    //通过键值key，取出对应的哈希桶。
    elt = hash->buckets[key % hash->size];

    if (elt == NULL) {
        return NULL;
    }

    //按顺序比较name值，如果相同则找到了，将对应的value值返回，直到elt-value为空//哈希桶在初始化时会将桶中的最后一个元素值初始化为空，参见ngx_hash_init
    while (elt->value) {
        if (len != (size_t) elt->len) {
            goto next;
        }

        for (i = 0; i < len; i++) {
            if (name[i] != elt->name[i]) {
                goto next;
            }
        }

        return elt->value;

    next:

        //桶中的元素存储了name值且元素初始地址按sizeof(void*)对齐，因此以下列方式取下一个元素。
        elt = (ngx_hash_elt_t *) ngx_align_ptr(&elt->name[0] + elt->len,
                                               sizeof(void *));
        continue;
    }

    return NULL;
}
```

查找带通配符的哈希表：  

```
//查找通配符在前面的hash表
//通配符在前面的hash表在建立时会将字符串转化， "*.example.com" 转化为 "com.example.\0"　 ".example.com" 转化为 "com.example\0", 即按点分隔后做反转，转化过程可见ngx_hash_add_key函数
void *
ngx_hash_find_wc_head(ngx_hash_wildcard_t *hwc, u_char *name, size_t len)
{
    void        *value;
    ngx_uint_t   i, n, key;

#if 0
    ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0, "wch:\"%*s\"", len, name);
#endif

    n = len;

    //从后面按点分隔取每一部分
    while (n) {
        if (name[n - 1] == '.') {
            break;
        }

        n--;
    }

    key = 0;

    for (i = n; i < len; i++) {
        key = ngx_hash(key, name[i]);
    }

#if 0
    ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0, "key:\"%ui\"", key);
#endif

    value = ngx_hash_find(&hwc->hash, key, &name[n], len - n);

#if 0
    ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0, "value:\"%p\"", value);
#endif

    if (value) {

        /*
         * the 2 low bits of value have the special meaning:
         *     00 - value is data pointer for both "example.com"
         *          and "*.example.com";
         *     01 - value is data pointer for "*.example.com" only;
         *     10 - value is pointer to wildcard hash allowing
         *          both "example.com" and "*.example.com";
         *     11 - value is pointer to wildcard hash allowing
         *          "*.example.com" only.
         */

        //value哈希表地址按地址字节对齐的，后两位在建立哈希表时加入了其它意义，如上面注释
        if ((uintptr_t) value & 2) {

            if (n == 0) {

                /* "example.com" */

                if ((uintptr_t) value & 1) {
                    return NULL;
                }

                hwc = (ngx_hash_wildcard_t *)
                                          ((uintptr_t) value & (uintptr_t) ~3);    //这里取真实地址
                return hwc->value;
            }

            //递归查找
            hwc = (ngx_hash_wildcard_t *) ((uintptr_t) value & (uintptr_t) ~3);

            value = ngx_hash_find_wc_head(hwc, name, n - 1);

            if (value) {
                return value;
            }

            return hwc->value;
        }

        if ((uintptr_t) value & 1) {

            if (n == 0) {

                /* "example.com" */

                return NULL;
            }

            return (void *) ((uintptr_t) value & (uintptr_t) ~3);
        }

        return value;
    }

    return hwc->value;
}

//查找通配符在后面的哈希表，这里比较好理解，参见哈希表的创建过程ngx_hash_wildcard_init函数
void *
ngx_hash_find_wc_tail(ngx_hash_wildcard_t *hwc, u_char *name, size_t len)
{
    void        *value;
    ngx_uint_t   i, key;

#if 0
    ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0, "wct:\"%*s\"", len, name);
#endif

    key = 0;
    //从左到右按点分隔，查找每一部分的哈希表
    for (i = 0; i < len; i++) {
        if (name[i] == '.') {
            break;
        }

        key = ngx_hash(key, name[i]);
    }

    if (i == len) {
        return NULL;
    }

#if 0
    ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0, "key:\"%ui\"", key);
#endif

    value = ngx_hash_find(&hwc->hash, key, name, i);

#if 0
    ngx_log_error(NGX_LOG_ALERT, ngx_cycle->log, 0, "value:\"%p\"", value);
#endif

    if (value) {

        /*
         * the 2 low bits of value have the special meaning:
         *     00 - value is data pointer;
         *     11 - value is pointer to wildcard hash allowing "example.*".
         */

        //value结尾比特为00是数据指针，结尾是11是哈希表，需要递归查找
        if ((uintptr_t) value & 2) {

            i++;

            hwc = (ngx_hash_wildcard_t *) ((uintptr_t) value & (uintptr_t) ~3);

            value = ngx_hash_find_wc_tail(hwc, &name[i], len - i);

            if (value) {
                return value;
            }

            return hwc->value;
        }

        return value;
    }

    return hwc->value;
}
```

---
nginx源码基于nginx-1.22.0版本  
**参考**：  
[nginx平台初探](http://tengine.taobao.org/book/chapter_02.html#ngx-hash-t-100)   
[Nginx源码分析 - 基础数据结构篇 - hash表结构 ngx_hash.c（07）](https://blog.csdn.net/initphp/article/details/50675500)  

