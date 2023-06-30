---
layout: post
title: Nginx源码分析-全局变量cycle初始化
date:   2022-10-31 15:30:00 +0800
categories: Nginx源码分析
tags: C nginx
topping: true
---
 
Nginx 启动过程中会调用 ngx_init_cycle 初始化一个全局变量 ngx_cycle, 这是 nginx 的一个核心结构体，类型为 ngx_cycle_t 。  

### 结构体定义

ngx_cycle_t 结构体定义：  

```
typedef struct ngx_cycle_s           ngx_cycle_t;

struct ngx_cycle_s {
    void                  ****conf_ctx;    //各模块配置文件数组
    ngx_pool_t               *pool;    //内存池

    ngx_log_t                *log;    //日志指针
    ngx_log_t                 new_log;

    ngx_uint_t                log_use_stderr;  /* unsigned  log_use_stderr:1; */

    ngx_connection_t        **files;    //连接文件
    ngx_connection_t         *free_connections;    //空闲连接
    ngx_uint_t                free_connection_n;    //空闲连接个数

    ngx_module_t            **modules;    //模块数组
    ngx_uint_t                modules_n;    //模块个数
    ngx_uint_t                modules_used;    /* unsigned  modules_used:1; */

    ngx_queue_t               reusable_connections_queue;
    ngx_uint_t                reusable_connections_n;
    time_t                    connections_reuse_time;

    ngx_array_t               listening;    //监听数组
    ngx_array_t               paths;    //路径数组

    ngx_array_t               config_dump;    //配置拷贝
    ngx_rbtree_t              config_dump_rbtree;    //配置拷贝红黑树
    ngx_rbtree_node_t         config_dump_sentinel;    //红黑树哨兵结点

    ngx_list_t                open_files;    //打工文件列表
    ngx_list_t                shared_memory;    //共享内存列表

    ngx_uint_t                connection_n;    //连接数数
    ngx_uint_t                files_n;    //打开文件数

    ngx_connection_t         *connections;    //连接事件
    ngx_event_t              *read_events;    //读事件
    ngx_event_t              *write_events;    //写事件

    ngx_cycle_t              *old_cycle;    //旧的cycle

    ngx_str_t                 conf_file;    //配置文件
    ngx_str_t                 conf_param;    //配置参数
    ngx_str_t                 conf_prefix;    //配置文件前缀
    ngx_str_t                 prefix;    //前缀
    ngx_str_t                 error_log;    //错误日志
    ngx_str_t                 lock_file;    //锁文件
    ngx_str_t                 hostname;    //主机名
};
```
### 流程图

ngx_init_cycle 函数主流程图：  

![ngxCycleInit.png]({{site.baseurl}}/styles/images/nginx/ngxCycleInit.png)  

### 分步介绍

#### 更新时区和时间

```
//更新时区
ngx_timezone_update();

/* force localtime update with a new timezone */

tp = ngx_timeofday();
tp->sec = 0;

//更新时间
ngx_time_update();
```

#### 申请内存池

```
log = old_cycle->log;
//创建一个 16k的内存池
pool = ngx_create_pool(NGX_CYCLE_POOL_SIZE, log);
if (pool == NULL) {
    return NULL;
}
pool->log = log;
```

#### 申请一个 Cycle 并初始化

```
cycle = ngx_pcalloc(pool, sizeof(ngx_cycle_t));
if (cycle == NULL) {
    ngx_destroy_pool(pool);
    return NULL;
}

cycle->pool = pool;
cycle->log = log;
cycle->old_cycle = old_cycle;
```

#### cycle 内各元素初始化

包括配置前缀，错误日志路径，配置文件路径， 配置参数，path 数组，config_dump数组，config_dump_rbtree 红黑树，打开文件，共享内存，reusable_connections_queue 列表，监听文件，配置上下文及主机名。  

```
//赋值配置前缀
cycle->conf_prefix.len = old_cycle->conf_prefix.len;
cycle->conf_prefix.data = ngx_pstrdup(pool, &old_cycle->conf_prefix);
if (cycle->conf_prefix.data == NULL) {
    ngx_destroy_pool(pool);
    return NULL;
}

cycle->prefix.len = old_cycle->prefix.len;
cycle->prefix.data = ngx_pstrdup(pool, &old_cycle->prefix);
if (cycle->prefix.data == NULL) {
    ngx_destroy_pool(pool);
    return NULL;
}

//赋值错误日志路径
cycle->error_log.len = old_cycle->error_log.len;
cycle->error_log.data = ngx_pnalloc(pool, old_cycle->error_log.len + 1);
if (cycle->error_log.data == NULL) {
    ngx_destroy_pool(pool);
    return NULL;
}
ngx_cpystrn(cycle->error_log.data, old_cycle->error_log.data,
            old_cycle->error_log.len + 1);

//赋值配置文件路径
cycle->conf_file.len = old_cycle->conf_file.len;
cycle->conf_file.data = ngx_pnalloc(pool, old_cycle->conf_file.len + 1);
if (cycle->conf_file.data == NULL) {
    ngx_destroy_pool(pool);
    return NULL;
}
ngx_cpystrn(cycle->conf_file.data, old_cycle->conf_file.data,
            old_cycle->conf_file.len + 1);

//赋值配置参数
cycle->conf_param.len = old_cycle->conf_param.len;
cycle->conf_param.data = ngx_pstrdup(pool, &old_cycle->conf_param);
if (cycle->conf_param.data == NULL) {
    ngx_destroy_pool(pool);
    return NULL;
}


//初始化paths数组
n = old_cycle->paths.nelts ? old_cycle->paths.nelts : 10;

if (ngx_array_init(&cycle->paths, pool, n, sizeof(ngx_path_t *))
    != NGX_OK)
{
    ngx_destroy_pool(pool);
    return NULL;
}

ngx_memzero(cycle->paths.elts, n * sizeof(ngx_path_t *));


//初始化config_dump数组
if (ngx_array_init(&cycle->config_dump, pool, 1, sizeof(ngx_conf_dump_t))
    != NGX_OK)
{
    ngx_destroy_pool(pool);
    return NULL;
}

//初始化 config_dump_rbtree 红黑树
ngx_rbtree_init(&cycle->config_dump_rbtree, &cycle->config_dump_sentinel,
                ngx_str_rbtree_insert_value);

//计算打开文件数目
if (old_cycle->open_files.part.nelts) {
    n = old_cycle->open_files.part.nelts;
    for (part = old_cycle->open_files.part.next; part; part = part->next) {
        n += part->nelts;
    }

} else {
    n = 20;
}

//打开文件链表初始化，??什么时候赋值
if (ngx_list_init(&cycle->open_files, pool, n, sizeof(ngx_open_file_t))
    != NGX_OK)
{
    ngx_destroy_pool(pool);
    return NULL;
}


//计算共享内存数目
if (old_cycle->shared_memory.part.nelts) {
    n = old_cycle->shared_memory.part.nelts;
    for (part = old_cycle->shared_memory.part.next; part; part = part->next)
    {
        n += part->nelts;
    }

} else {
    n = 1;
}

//初始化共享内存列表
if (ngx_list_init(&cycle->shared_memory, pool, n, sizeof(ngx_shm_zone_t))
    != NGX_OK)
{
    ngx_destroy_pool(pool);
    return NULL;
}

//初始化listening数组
n = old_cycle->listening.nelts ? old_cycle->listening.nelts : 10;

if (ngx_array_init(&cycle->listening, pool, n, sizeof(ngx_listening_t))
    != NGX_OK)
{
    ngx_destroy_pool(pool);
    return NULL;
}

ngx_memzero(cycle->listening.elts, n * sizeof(ngx_listening_t));

//初始化 reusable_connections_queue 列表
ngx_queue_init(&cycle->reusable_connections_queue);


//申请配置上下文内存，每个模块一个配置信息，以void*指针保存配置结构地址
cycle->conf_ctx = ngx_pcalloc(pool, ngx_max_module * sizeof(void *));
if (cycle->conf_ctx == NULL) {
    ngx_destroy_pool(pool);
    return NULL;
}


//获取主机名
if (gethostname(hostname, NGX_MAXHOSTNAMELEN) == -1) {
    ngx_log_error(NGX_LOG_EMERG, log, ngx_errno, "gethostname() failed");
    ngx_destroy_pool(pool);
    return NULL;
}

/* on Linux gethostname() silently truncates name that does not fit */

hostname[NGX_MAXHOSTNAMELEN - 1] = '\0';
cycle->hostname.len = ngx_strlen(hostname);

cycle->hostname.data = ngx_pnalloc(pool, cycle->hostname.len);
if (cycle->hostname.data == NULL) {
    ngx_destroy_pool(pool);
    return NULL;
}

ngx_strlow(cycle->hostname.data, (u_char *) hostname, cycle->hostname.len);
```

#### 初始化模块信息

```
//初始化 cycle 内的模块信息
if (ngx_cycle_modules(cycle) != NGX_OK) {
    ngx_destroy_pool(pool);
    return NULL;
}


//调用 create_conf 函数句柄来创建 NGX_CORE_MODULE 类型模块的配置
for (i = 0; cycle->modules[i]; i++) {
    if (cycle->modules[i]->type != NGX_CORE_MODULE) {
        continue;
    }

    module = cycle->modules[i]->ctx;

    if (module->create_conf) {
        rv = module->create_conf(cycle);
        if (rv == NULL) {
            ngx_destroy_pool(pool);
            return NULL;
        }
        cycle->conf_ctx[cycle->modules[i]->index] = rv;
    }
}
```

#### 初始化配置解析结构

解析配置时会用到一个 ngx_conf_t 结构体，定义为：  

```
typedef struct ngx_conf_s            ngx_conf_t;

struct ngx_conf_s {
    char                 *name;
    ngx_array_t          *args;    //解析出的参数

    ngx_cycle_t          *cycle;    //cycle
    ngx_pool_t           *pool;    //内存池
    ngx_pool_t           *temp_pool;    //临时内存池
    ngx_conf_file_t      *conf_file;    //配置文件信息
    ngx_log_t            *log;    //日志指针

    void                 *ctx;    //所有模块配置数组
    ngx_uint_t            module_type;    //配置模块类型
    ngx_uint_t            cmd_type;    //配置命令类型

    ngx_conf_handler_pt   handler;
    void                 *handler_conf;
};
```

初始化过程：  

```
//初始化配置解析结构
ngx_memzero(&conf, sizeof(ngx_conf_t));
/* STUB: init array ? */
//申请可带 10 个参数的数组，用于存储配置解析参数
conf.args = ngx_array_create(pool, 10, sizeof(ngx_str_t));
if (conf.args == NULL) {
    ngx_destroy_pool(pool);
    return NULL;
}

//申请一个临时内存池
conf.temp_pool = ngx_create_pool(NGX_CYCLE_POOL_SIZE, log);
if (conf.temp_pool == NULL) {
    ngx_destroy_pool(pool);
    return NULL;
}


//配置解析结构初始化，要解析的模块类型为 NGX_CORE_MODULE，命令类型为 NGX_MAIN_CONF
conf.ctx = cycle->conf_ctx;
conf.cycle = cycle;
conf.pool = pool;
conf.log = log;
conf.module_type = NGX_CORE_MODULE;
conf.cmd_type = NGX_MAIN_CONF;
```

#### 解析配置

```
//解析命令行中的配置，里面通过调用 ngx_conf_parse 来实现配置的解析
if (ngx_conf_param(&conf) != NGX_CONF_OK) {
    environ = senv;
    ngx_destroy_cycle_pools(&conf);
    return NULL;
}

//解析配置文件中的配置
if (ngx_conf_parse(&conf, &cycle->conf_file) != NGX_CONF_OK) {
    environ = senv;
    ngx_destroy_cycle_pools(&conf);
    return NULL;
}
```

#### 初始化各模块配置

```
//通过调用 init_conf 初始化函数句柄来初始化 NGX_CORE_MODULE 类型模块配置
for (i = 0; cycle->modules[i]; i++) {
    if (cycle->modules[i]->type != NGX_CORE_MODULE) {
        continue;
    }

    module = cycle->modules[i]->ctx;

    if (module->init_conf) {
        if (module->init_conf(cycle,
                                cycle->conf_ctx[cycle->modules[i]->index])
            == NGX_CONF_ERROR)
        {
            environ = senv;
            ngx_destroy_cycle_pools(&conf);
            return NULL;
        }
    }
}
```

#### 创建所需目录

```
//创建目录，一些配置中会存在目录的配置，在解析配置时添加进paths中，这里创建该目录
if (ngx_create_paths(cycle, ccf->user) != NGX_OK) {
    goto failed;
}
```

#### 创建并初始化共享内存

```
/* create shared memory */
//创建并初始化配置中需要的共享内存

part = &cycle->shared_memory.part;
shm_zone = part->elts;

for (i = 0; /* void */ ; i++) {

    if (i >= part->nelts) {
        if (part->next == NULL) {
            break;
        }
        part = part->next;
        shm_zone = part->elts;
        i = 0;
    }

    if (shm_zone[i].shm.size == 0) {
        ngx_log_error(NGX_LOG_EMERG, log, 0,
                        "zero size shared memory zone \"%V\"",
                        &shm_zone[i].shm.name);
        goto failed;
    }

    shm_zone[i].shm.log = cycle->log;

    opart = &old_cycle->shared_memory.part;
    oshm_zone = opart->elts;

    //查询原 old_cycle 中是否包含该共享内存，如果存在则直接将其复制过来
    for (n = 0; /* void */ ; n++) {

        if (n >= opart->nelts) {
            if (opart->next == NULL) {
                break;
            }
            opart = opart->next;
            oshm_zone = opart->elts;
            n = 0;
        }

        if (shm_zone[i].shm.name.len != oshm_zone[n].shm.name.len) {
            continue;
        }

        if (ngx_strncmp(shm_zone[i].shm.name.data,
                        oshm_zone[n].shm.name.data,
                        shm_zone[i].shm.name.len)
            != 0)
        {
            continue;
        }

        if (shm_zone[i].tag == oshm_zone[n].tag
            && shm_zone[i].shm.size == oshm_zone[n].shm.size
            && !shm_zone[i].noreuse)
        {
            shm_zone[i].shm.addr = oshm_zone[n].shm.addr;
#if (NGX_WIN32)
            shm_zone[i].shm.handle = oshm_zone[n].shm.handle;
#endif

            if (shm_zone[i].init(&shm_zone[i], oshm_zone[n].data)
                != NGX_OK)
            {
                goto failed;
            }

            goto shm_zone_found;
        }

        break;
    }

    //申请并初始化该共享内存
    if (ngx_shm_alloc(&shm_zone[i].shm) != NGX_OK) {
        goto failed;
    }

    if (ngx_init_zone_pool(cycle, &shm_zone[i]) != NGX_OK) {
        goto failed;
    }

    if (shm_zone[i].init(&shm_zone[i], NULL) != NGX_OK) {
        goto failed;
    }

shm_zone_found:

    continue;
}
```

#### 处理监听的 socket

```
/* handle the listening sockets */
//处理监听的sockets
if (old_cycle->listening.nelts) {
    ls = old_cycle->listening.elts;
    for (i = 0; i < old_cycle->listening.nelts; i++) {
        ls[i].remain = 0;
    }

    nls = cycle->listening.elts;
    for (n = 0; n < cycle->listening.nelts; n++) {

        for (i = 0; i < old_cycle->listening.nelts; i++) {
            if (ls[i].ignore) {
                continue;
            }

            if (ls[i].remain) {
                continue;
            }

            if (ls[i].type != nls[n].type) {
                continue;
            }

            if (ngx_cmp_sockaddr(nls[n].sockaddr, nls[n].socklen,
                                    ls[i].sockaddr, ls[i].socklen, 1)
                == NGX_OK)
            {
                nls[n].fd = ls[i].fd;
                nls[n].inherited = ls[i].inherited;
                nls[n].previous = &ls[i];
                ls[i].remain = 1;

                if (ls[i].backlog != nls[n].backlog) {
                    nls[n].listen = 1;
                }

#if (NGX_HAVE_DEFERRED_ACCEPT && defined SO_ACCEPTFILTER)

                /*
                    * FreeBSD, except the most recent versions,
                    * could not remove accept filter
                    */
                nls[n].deferred_accept = ls[i].deferred_accept;

                if (ls[i].accept_filter && nls[n].accept_filter) {
                    if (ngx_strcmp(ls[i].accept_filter,
                                    nls[n].accept_filter)
                        != 0)
                    {
                        nls[n].delete_deferred = 1;
                        nls[n].add_deferred = 1;
                    }

                } else if (ls[i].accept_filter) {
                    nls[n].delete_deferred = 1;

                } else if (nls[n].accept_filter) {
                    nls[n].add_deferred = 1;
                }
#endif

#if (NGX_HAVE_DEFERRED_ACCEPT && defined TCP_DEFER_ACCEPT)

                if (ls[i].deferred_accept && !nls[n].deferred_accept) {
                    nls[n].delete_deferred = 1;

                } else if (ls[i].deferred_accept != nls[n].deferred_accept)
                {
                    nls[n].add_deferred = 1;
                }
#endif

#if (NGX_HAVE_REUSEPORT)
                if (nls[n].reuseport && !ls[i].reuseport) {
                    nls[n].add_reuseport = 1;
                }
#endif

                break;
            }
        }

        if (nls[n].fd == (ngx_socket_t) -1) {
            nls[n].open = 1;
#if (NGX_HAVE_DEFERRED_ACCEPT && defined SO_ACCEPTFILTER)
            if (nls[n].accept_filter) {
                nls[n].add_deferred = 1;
            }
#endif
#if (NGX_HAVE_DEFERRED_ACCEPT && defined TCP_DEFER_ACCEPT)
            if (nls[n].deferred_accept) {
                nls[n].add_deferred = 1;
            }
#endif
        }
    }

} else {
    ls = cycle->listening.elts;
    for (i = 0; i < cycle->listening.nelts; i++) {
        ls[i].open = 1;
#if (NGX_HAVE_DEFERRED_ACCEPT && defined SO_ACCEPTFILTER)
        if (ls[i].accept_filter) {
            ls[i].add_deferred = 1;
        }
#endif
#if (NGX_HAVE_DEFERRED_ACCEPT && defined TCP_DEFER_ACCEPT)
        if (ls[i].deferred_accept) {
            ls[i].add_deferred = 1;
        }
#endif
    }
}

if (ngx_open_listening_sockets(cycle) != NGX_OK) {
    goto failed;
}

if (!ngx_test_config) {
    ngx_configure_listening_sockets(cycle);
}


/* commit the new cycle configuration */

if (!ngx_use_stderr) {
    (void) ngx_log_redirect_stderr(cycle);
}

pool->log = cycle->log;

//初始化所有模块
if (ngx_init_modules(cycle) != NGX_OK) {
    /* fatal */
    exit(1);
}
```

### 清理工作

清理工作包括释放不需要的共享内存，关闭不需要的 socket，关闭不需要的打开文件，销毁临时内存池。  

```
//释放不需要的共享内存，遍历old_cycle->shared_memory，查找每个共享内存在 cycle->shared_memory中是否存在，不存在则直接释放
opart = &old_cycle->shared_memory.part;
oshm_zone = opart->elts;

for (i = 0; /* void */ ; i++) {

    if (i >= opart->nelts) {
        if (opart->next == NULL) {
            goto old_shm_zone_done;
        }
        opart = opart->next;
        oshm_zone = opart->elts;
        i = 0;
    }

    part = &cycle->shared_memory.part;
    shm_zone = part->elts;

    for (n = 0; /* void */ ; n++) {

        if (n >= part->nelts) {
            if (part->next == NULL) {
                break;
            }
            part = part->next;
            shm_zone = part->elts;
            n = 0;
        }

        if (oshm_zone[i].shm.name.len != shm_zone[n].shm.name.len) {
            continue;
        }

        if (ngx_strncmp(oshm_zone[i].shm.name.data,
                        shm_zone[n].shm.name.data,
                        oshm_zone[i].shm.name.len)
            != 0)
        {
            continue;
        }

        if (oshm_zone[i].tag == shm_zone[n].tag
            && oshm_zone[i].shm.size == shm_zone[n].shm.size
            && !oshm_zone[i].noreuse)
        {
            goto live_shm_zone;
        }

        break;
    }

    ngx_shm_free(&oshm_zone[i].shm);

live_shm_zone:

    continue;
}

old_shm_zone_done:


/* close the unnecessary listening sockets */
//关闭不需要监听的sockets
ls = old_cycle->listening.elts;
for (i = 0; i < old_cycle->listening.nelts; i++) {

    if (ls[i].remain || ls[i].fd == (ngx_socket_t) -1) {
        continue;
    }

    if (ngx_close_socket(ls[i].fd) == -1) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_socket_errno,
                        ngx_close_socket_n " listening socket on %V failed",
                        &ls[i].addr_text);
    }

#if (NGX_HAVE_UNIX_DOMAIN)

    if (ls[i].sockaddr->sa_family == AF_UNIX) {
        u_char  *name;

        name = ls[i].addr_text.data + sizeof("unix:") - 1;

        ngx_log_error(NGX_LOG_WARN, cycle->log, 0,
                        "deleting socket %s", name);

        if (ngx_delete_file(name) == NGX_FILE_ERROR) {
            ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_socket_errno,
                            ngx_delete_file_n " %s failed", name);
        }
    }

#endif
}


/* close the unnecessary open files */
//关闭不需要的打开文件
part = &old_cycle->open_files.part;
file = part->elts;

for (i = 0; /* void */ ; i++) {

    if (i >= part->nelts) {
        if (part->next == NULL) {
            break;
        }
        part = part->next;
        file = part->elts;
        i = 0;
    }

    if (file[i].fd == NGX_INVALID_FILE || file[i].fd == ngx_stderr) {
        continue;
    }

    if (ngx_close_file(file[i].fd) == NGX_FILE_ERROR) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
                        ngx_close_file_n " \"%s\" failed",
                        file[i].name.data);
    }
}

//销毁临时内存池
ngx_destroy_pool(conf.temp_pool);

```

---
nginx源码基于nginx-1.22.0版本  
**参考**：  
[Nginx 启动初始化过程](https://www.kancloud.cn/digest/understandingnginx/202596)   
[Nginx源码分析 - 主流程篇 - 全局变量cycle初始化（11）](https://blog.csdn.net/initphp/article/details/51882804)  
