---
layout: post
title: Nginx源码分析-模块及其初始化 
date:   2022-09-05 18:00:00 +0800
categories: Nginx源码分析
tags: C nginx
topping: true
---
 
### Nginx 中的模块

ngx_modules 变量是包含了 nginx 中所有模块的数组，它定义在 ngx_module.h 文件中，在编译前通过 configure 命令可以确定所包含的模块。在编译前执行 ./configure 命令，可生成一个 objs/ngx_modules.c 文件，这个文件内包含了 ngx_modules 数组，如下：  

```
ngx_module_t *ngx_modules[] = {
    &ngx_core_module,
    &ngx_errlog_module,
    &ngx_conf_module,
    &ngx_openssl_module,
    &ngx_regex_module,
    &ngx_events_module,
    &ngx_event_core_module,
    &ngx_epoll_module,
    &ngx_http_module,
    &ngx_http_core_module,
    &ngx_http_log_module,
    &ngx_http_upstream_module,
    &ngx_http_static_module,
    &ngx_http_autoindex_module,
    &ngx_http_index_module,
    &ngx_http_mirror_module,
    &ngx_http_try_files_module,
    &ngx_http_auth_basic_module,
    &ngx_http_access_module,
    &ngx_http_limit_conn_module,
    &ngx_http_limit_req_module,
    &ngx_http_geo_module,
    &ngx_http_map_module,
    &ngx_http_split_clients_module,
    &ngx_http_referer_module,
    &ngx_http_rewrite_module,
    &ngx_http_proxy_module,
    &ngx_http_fastcgi_module,
    &ngx_http_uwsgi_module,
    &ngx_http_scgi_module,
    &ngx_http_memcached_module,
    &ngx_http_empty_gif_module,
    &ngx_http_browser_module,
    &ngx_http_upstream_hash_module,
    &ngx_http_upstream_ip_hash_module,
    &ngx_http_upstream_least_conn_module,
    &ngx_http_upstream_random_module,
    &ngx_http_upstream_keepalive_module,
    &ngx_http_upstream_zone_module,
    &ngx_http_write_filter_module,
    &ngx_http_header_filter_module,
    &ngx_http_chunked_filter_module,
    &ngx_http_range_header_filter_module,
    &ngx_http_gzip_filter_module,
    &ngx_http_postpone_filter_module,
    &ngx_http_ssi_filter_module,
    &ngx_http_charset_filter_module,
    &ngx_http_userid_filter_module,
    &ngx_http_headers_filter_module,
    &ngx_http_copy_filter_module,
    &ngx_http_range_body_filter_module,
    &ngx_http_not_modified_filter_module,
    &ngx_mail_module,
    &ngx_mail_core_module,
    &ngx_mail_pop3_module,
    &ngx_mail_imap_module,
    &ngx_mail_smtp_module,
    &ngx_mail_auth_http_module,
    &ngx_mail_proxy_module,
    &ngx_mail_realip_module,
    &ngx_stream_module,
    &ngx_stream_core_module,
    &ngx_stream_log_module,
    &ngx_stream_proxy_module,
    &ngx_stream_upstream_module,
    &ngx_stream_write_filter_module,
    &ngx_stream_limit_conn_module,
    &ngx_stream_access_module,
    &ngx_stream_geo_module,
    &ngx_stream_map_module,
    &ngx_stream_split_clients_module,
    &ngx_stream_return_module,
    &ngx_stream_set_module,
    &ngx_stream_upstream_hash_module,
    &ngx_stream_upstream_least_conn_module,
    &ngx_stream_upstream_random_module,
    &ngx_stream_upstream_zone_module,
    NULL
};
```

Nginx 中的模块以下几种模块：NGX_CORE_MODULE, NGX_HTTP_MODULE, NGX_EVENT_MODULE, NGX_MAIL_MODULE, NGX_STREAM_MODULE，模块关系图如下：  

![ngxModules.png]({{site.imgurl}}/styles/images/nginx/ngxModules.png)  

### 数据结构定义

#### ngx_module_t

ngx_module_t 是模块的结构体，定义了模块的一些属性。  

```
typedef struct ngx_module_s          ngx_module_t;

//模块结构体定义
struct ngx_module_s {
    ngx_uint_t            ctx_index;    //不同类型模块的索引
    ngx_uint_t            index;    //所有模块中的模块索引

    char                 *name;    //模块名称

    ngx_uint_t            spare0;
    ngx_uint_t            spare1;

    ngx_uint_t            version;    //模块版本
    const char           *signature;    //签名

    void                 *ctx;    //模块上下文属性
    ngx_command_t        *commands;    //模块指令数组
    ngx_uint_t            type;    //模块类型

    //回调函数
    ngx_int_t           (*init_master)(ngx_log_t *log);    //初始化master

    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);    //初始化模块

    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);    //初始化进程
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);    //初始化线程
    void                (*exit_thread)(ngx_cycle_t *cycle);    //退出线程
    void                (*exit_process)(ngx_cycle_t *cycle);    //退出进程

    void                (*exit_master)(ngx_cycle_t *cycle);    //退出master

    uintptr_t             spare_hook0;
    uintptr_t             spare_hook1;
    uintptr_t             spare_hook2;
    uintptr_t             spare_hook3;
    uintptr_t             spare_hook4;
    uintptr_t             spare_hook5;
    uintptr_t             spare_hook6;
    uintptr_t             spare_hook7;
};
```

#### ngx_command_t

ngx_command_t 是模块的配置指令，也是出现在配置文件中的指令，name 为命令的名称，set 是命令的设置函数，如下：  

```
typedef struct ngx_command_s         ngx_command_t;

struct ngx_command_s {
    ngx_str_t             name;    //命令名称
    ngx_uint_t            type;    //命令类型
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);    //设置配置函数句柄
    ngx_uint_t            conf;
    ngx_uint_t            offset;    //设置参数在配置结构中的偏移量
    void                 *post;    //post函数句柄
};
```

### 模块初始化

#### 模块预初始化　ngx_preinit_modules

ngx_preinit_modules 给所有模块初始化其索引 index 和模块名称 name，并记录模块总数。  

```
//预初始化模块，遍历所有模块，给其赋值index和name, 模块数为ngx_modules_n, 最大模块数为模块数与动态模块数最大值的和
ngx_int_t
ngx_preinit_modules(void)
{
    ngx_uint_t  i;

    for (i = 0; ngx_modules[i]; i++) {
        ngx_modules[i]->index = i;
        ngx_modules[i]->name = ngx_module_names[i];
    }

    ngx_modules_n = i;
    ngx_max_module = ngx_modules_n + NGX_MAX_DYNAMIC_MODULES;

    return NGX_OK;
}
```

#### 初始化 cycle 的模块信息　ngx_cycle_modules

ngx_cycle_modules 用于初始化 cycle 中的模块信息，给模块申请内存并将 ngx_modules 中的信息拷贝到 cycle->modules中。  

```
//初始化 cycle 中的模块信息
ngx_int_t
ngx_cycle_modules(ngx_cycle_t *cycle)
{
    /*
     * create a list of modules to be used for this cycle,
     * copy static modules to it
     */

    cycle->modules = ngx_pcalloc(cycle->pool, (ngx_max_module + 1)
                                              * sizeof(ngx_module_t *));
    if (cycle->modules == NULL) {
        return NGX_ERROR;
    }

    ngx_memcpy(cycle->modules, ngx_modules,
               ngx_modules_n * sizeof(ngx_module_t *));

    cycle->modules_n = ngx_modules_n;

    return NGX_OK;
}
```

#### 每个模块进行初始化　ngx_init_modules

ngx_init_modules 循环所有模块，调用初始化函数 init_module 来进行模块的初始化。  

```
//初始化模块信息，调用模块的初始化函数句柄 init_module 来初始化。
ngx_int_t
ngx_init_modules(ngx_cycle_t *cycle)
{
    ngx_uint_t  i;

    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->init_module) {
            if (cycle->modules[i]->init_module(cycle) != NGX_OK) {
                return NGX_ERROR;
            }
        }
    }

    return NGX_OK;
}
```

#### 统计模块数 ngx_count_modules

ngx_count_modules 用于统计每个类型 type 下有多少模块。  

```
//从 index 开始找到一个类型为 type 的，没有使用过的 index
static ngx_uint_t
ngx_module_ctx_index(ngx_cycle_t *cycle, ngx_uint_t type, ngx_uint_t index)
{
    ngx_uint_t     i;
    ngx_module_t  *module;

again:

    /* find an unused ctx_index */

    for (i = 0; cycle->modules[i]; i++) {
        module = cycle->modules[i];

        if (module->type != type) {
            continue;
        }

        if (module->ctx_index == index) {
            index++;
            goto again;
        }
    }

    /* check previous cycle */

    if (cycle->old_cycle && cycle->old_cycle->modules) {

        for (i = 0; cycle->old_cycle->modules[i]; i++) {
            module = cycle->old_cycle->modules[i];

            if (module->type != type) {
                continue;
            }

            if (module->ctx_index == index) {
                index++;
                goto again;
            }
        }
    }

    return index;
}

ngx_int_t
ngx_count_modules(ngx_cycle_t *cycle, ngx_uint_t type)
{
    ngx_uint_t     i, next, max;
    ngx_module_t  *module;

    next = 0;
    max = 0;

    /* count appropriate modules, set up their indices */

    for (i = 0; cycle->modules[i]; i++) {
        module = cycle->modules[i];

        if (module->type != type) {
            continue;
        }

        if (module->ctx_index != NGX_MODULE_UNSET_INDEX) {

            /* if ctx_index was assigned, preserve it */

            if (module->ctx_index > max) {
                max = module->ctx_index;
            }

            if (module->ctx_index == next) {
                next++;
            }

            continue;
        }

        /* search for some free index */

        module->ctx_index = ngx_module_ctx_index(cycle, type, next);

        if (module->ctx_index > max) {
            max = module->ctx_index;
        }

        next = module->ctx_index + 1;
    }

    /*
     * make sure the number returned is big enough for previous
     * cycle as well, else there will be problems if the number
     * will be stored in a global variable (as it's used to be)
     * and we'll have to roll back to the previous cycle
     */

    if (cycle->old_cycle && cycle->old_cycle->modules) {

        for (i = 0; cycle->old_cycle->modules[i]; i++) {
            module = cycle->old_cycle->modules[i];

            if (module->type != type) {
                continue;
            }

            if (module->ctx_index > max) {
                max = module->ctx_index;
            }
        }
    }

    /* prevent loading of additional modules */

    cycle->modules_used = 1;

    return max + 1;
}
```

#### 添加一个模块 ngx_add_module

ngx_add_module 用于添加一个模块。  

```
//从0开始，获取一个未使用的module->index
static ngx_uint_t
ngx_module_index(ngx_cycle_t *cycle)
{
    ngx_uint_t     i, index;
    ngx_module_t  *module;

    index = 0;

again:

    /* find an unused index */

    for (i = 0; cycle->modules[i]; i++) {
        module = cycle->modules[i];

        if (module->index == index) {
            index++;
            goto again;
        }
    }

    /* check previous cycle */

    if (cycle->old_cycle && cycle->old_cycle->modules) {

        for (i = 0; cycle->old_cycle->modules[i]; i++) {
            module = cycle->old_cycle->modules[i];

            if (module->index == index) {
                index++;
                goto again;
            }
        }
    }

    return index;
}

//添加一个模块
ngx_int_t
ngx_add_module(ngx_conf_t *cf, ngx_str_t *file, ngx_module_t *module,
    char **order)
{
    void               *rv;
    ngx_uint_t          i, m, before;
    ngx_core_module_t  *core_module;

    if (cf->cycle->modules_n >= ngx_max_module) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "too many modules loaded");
        return NGX_ERROR;
    }

    if (module->version != nginx_version) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "module \"%V\" version %ui instead of %ui",
                           file, module->version, (ngx_uint_t) nginx_version);
        return NGX_ERROR;
    }

    if (ngx_strcmp(module->signature, NGX_MODULE_SIGNATURE) != 0) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "module \"%V\" is not binary compatible",
                           file);
        return NGX_ERROR;
    }

    for (m = 0; cf->cycle->modules[m]; m++) {
        if (ngx_strcmp(cf->cycle->modules[m]->name, module->name) == 0) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "module \"%s\" is already loaded",
                               module->name);
            return NGX_ERROR;
        }
    }

    /*
     * if the module wasn't previously loaded, assign an index
     */

    if (module->index == NGX_MODULE_UNSET_INDEX) {
        module->index = ngx_module_index(cf->cycle);

        if (module->index >= ngx_max_module) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "too many modules loaded");
            return NGX_ERROR;
        }
    }

    /*
     * put the module into the cycle->modules array
     */

    before = cf->cycle->modules_n;

    if (order) {
        for (i = 0; order[i]; i++) {
            if (ngx_strcmp(order[i], module->name) == 0) {
                i++;
                break;
            }
        }

        for ( /* void */ ; order[i]; i++) {

#if 0
            ngx_log_debug2(NGX_LOG_DEBUG_CORE, cf->log, 0,
                           "module: %s before %s",
                           module->name, order[i]);
#endif

            for (m = 0; m < before; m++) {
                if (ngx_strcmp(cf->cycle->modules[m]->name, order[i]) == 0) {

                    ngx_log_debug3(NGX_LOG_DEBUG_CORE, cf->log, 0,
                                   "module: %s before %s:%i",
                                   module->name, order[i], m);

                    before = m;
                    break;
                }
            }
        }
    }

    /* put the module before modules[before] */

    if (before != cf->cycle->modules_n) {
        ngx_memmove(&cf->cycle->modules[before + 1],
                    &cf->cycle->modules[before],
                    (cf->cycle->modules_n - before) * sizeof(ngx_module_t *));
    }

    cf->cycle->modules[before] = module;
    cf->cycle->modules_n++;

    if (module->type == NGX_CORE_MODULE) {

        /*
         * we are smart enough to initialize core modules;
         * other modules are expected to be loaded before
         * initialization - e.g., http modules must be loaded
         * before http{} block
         */

        core_module = module->ctx;

        if (core_module->create_conf) {
            rv = core_module->create_conf(cf->cycle);
            if (rv == NULL) {
                return NGX_ERROR;
            }

            cf->cycle->conf_ctx[module->index] = rv;
        }
    }

    return NGX_OK;
}
```

---
nginx源码基于nginx-1.22.0版本  
**参考**：  
[nginx源码分析—模块及其初始化](https://blog.csdn.net/livelylittlefish/article/details/6571497)   
[Nginx源码分析 - 主流程篇 - 模块的初始化（12）](https://blog.csdn.net/initphp/article/details/51898955)  
[菜鸟nginx源码剖析 框架篇（一） 从main函数看nginx启动流程](https://blog.csdn.net/chen19870707/article/details/41050379)  
