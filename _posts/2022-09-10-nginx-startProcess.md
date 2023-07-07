---
layout: post
title: Nginx源码分析-Nginx启动流程
date:   2022-09-10 11:00:00 +0800
categories: Nginx源码分析
tags: C nginx
topping: true
---
 
Nginx 启动流程是由 nginx.c 中的 main 函数完成的，这里是 nginx 的入口。它通过调用一系列函数来完成 nginx 的初始化。  

### 初始化流程

#### 初始化流程图

![ngxStartProcess.png]({{site.imgurl}}/styles/images/nginx/ngxStartProcess.png)  

#### 分步介绍

1. 调用 ngx_strerror_init 初始化错误列表，拷贝系统中的 errno 及其错误信息到全局变量。  

2. 调用 ngx_get_options 获取命令行参数，保存在全局变量中。  

    >命令参数如下表：  
    >
    >|命令行参数 |功能     |
    >|:----------|:------  |
    >|-?,-h      |帮助信息 |
    >|-v         |显示版本号并退出 |
    >|-V         |显示版本号和配置选项并退出 |
    >|-t         |测试配置文件是否正确并退出 |
    >|-T         |测试配置文件是否正确，复制该文件并退出 |
    >|-q         |在测试配置文件的时候，屏蔽无错误信息，即静默模式 |
    >|-s         |发送信号给 master 进程，信号包括：stop, quit, reopen, reload |
    >|-p         |设置前缀路径（默认：/opt/nginx/） |
    >|-c         |设置配置文件（默认：conf/nginx.conf） |
    >|-g         |设置配置文件外的全局指令 |

3. 调用 ngx_time_init, ngx_regex_init, ngx_log_init, ngx_ssl_init 进行时间，正则表达式，日志，openssl 的初始化。  

4. 初始化 init_cycle，申请一个内存池（ngx_create_pool），然后调用 ngx_save_argv 将命令行参数保存到全局变量，再调用 ngx_process_option 来处理命令行参数，将配置文件，错误日志文件，配置参数等信息写入 init_cycle 中。  

5. 调用 ngx_os_init 进行操作系统信息的初始化，包括系统内存页大小，cpu 个数，cpu 缓存，cpu 缓存的 cacheline，最大 sockets 数等信息。  

6. 调用 ngx_crc32_table_init 进行循环冗余校验表的初始化，循环冗余校验采用查表法来进行。  

7. 调用 ngx_preinit_modules 进行模块的预处理，给所有模块添加索引值和模块名称。  

8. 使用 init_cycle 来调用 ngx_init_cycle 初始化全局变量 ngx_cycle，ngx_cycle 是 nginx 运行的一个核心结构体。用于存储各种配置信息。  

9. 当命令行使用 `-s` 选项时，ngx_signal 会在解析命令行参数 ngx_get_options 时被赋值，这时要通过 ngx_signal_process 函数给进程发送信号。  

10. 调用 ngx_init_signals 来注册信号量。注册 signal 数组中的所有信号量。  
    
    signal 为 ngx_signal_t 结构体，结构体定义及初始化在 ngx_process.c 文件中，定义如下：  
    
    >```
    >typedef struct {
    >    int     signo;    //信号量值
    >    char   *signame;    //信号名
    >    char   *name;
    >    void  (*handler)(int signo, siginfo_t *siginfo, void *ucontext);    //信号量处理流程
    >} ngx_signal_t;
    >```
    
    结构体中的 handler 是信号处理函数，nginx 使用 ngx_signal_handler 函数来处理各个信号，它根据不同的信号设置不同的全局变量：ngx_quit, ngx_terminate, ngx_reopen 等，在主循环进程中根据这些全局变量来进行不同的处理。  
   
11. 调用 ngx_daemon 来创建守护进程。  

12. 调用 ngx_create_pidfile 来创建进程文件并写入进程 pid。  

13. 根据 ngx_process 确定启动单进程工作模式还是多进程工作模式。然后进入主进程循环。  

### 源码分析

```
int ngx_cdecl
main(int argc, char *const *argv)
{
    ngx_buf_t        *b;
    ngx_log_t        *log;
    ngx_uint_t        i;
    ngx_cycle_t      *cycle, init_cycle;
    ngx_conf_dump_t  *cd;
    ngx_core_conf_t  *ccf;

    ngx_debug_init();

    //拷贝错误码
    if (ngx_strerror_init() != NGX_OK) {
        return 1;
    }

    //获取命令参数
    if (ngx_get_options(argc, argv) != NGX_OK) {
        return 1;
    }

    if (ngx_show_version) {
        ngx_show_version_info();

        if (!ngx_test_config) {
            return 0;
        }
    }

    /* TODO */ ngx_max_sockets = -1;

    //初始化时间信息
    ngx_time_init();

#if (NGX_PCRE)
    ngx_regex_init();
#endif

    //设置pid及父pid
    ngx_pid = ngx_getpid();
    ngx_parent = ngx_getppid();

    //日志初始化
    log = ngx_log_init(ngx_prefix, ngx_error_log);
    if (log == NULL) {
        return 1;
    }

    /* STUB */
#if (NGX_OPENSSL)
    //openssl初始化
    ngx_ssl_init(log);
#endif

    /*
     * init_cycle->log is required for signal handlers and
     * ngx_process_options()
     */

    ngx_memzero(&init_cycle, sizeof(ngx_cycle_t));
    init_cycle.log = log;
    ngx_cycle = &init_cycle;

    //申请一个内存池
    init_cycle.pool = ngx_create_pool(1024, log);
    if (init_cycle.pool == NULL) {
        return 1;
    }

    //保存命令行参数
    if (ngx_save_argv(&init_cycle, argc, argv) != NGX_OK) {
        return 1;
    }

    //处理命令行选项，配置文件，错误日志文件，配置参数等信息，写入 init_cycle 中
    if (ngx_process_options(&init_cycle) != NGX_OK) {
        return 1;
    }

    //系统信息初始化
    if (ngx_os_init(log) != NGX_OK) {
        return 1;
    }

    /*
     * ngx_crc32_table_init() requires ngx_cacheline_size set in ngx_os_init()
     */
    //crc32表初始化，详参见 crc 校验
    if (ngx_crc32_table_init() != NGX_OK) {
        return 1;
    }

    /*
     * ngx_slab_sizes_init() requires ngx_pagesize set in ngx_os_init()
     */

    ngx_slab_sizes_init();

    if (ngx_add_inherited_sockets(&init_cycle) != NGX_OK) {
        return 1;
    }

    //预初始化模块
    if (ngx_preinit_modules() != NGX_OK) {
        return 1;
    }

    //初始化cycle
    cycle = ngx_init_cycle(&init_cycle);
    if (cycle == NULL) {
        if (ngx_test_config) {
            ngx_log_stderr(0, "configuration file %s test failed",
                           init_cycle.conf_file.data);
        }

        return 1;
    }

    if (ngx_test_config) {
        if (!ngx_quiet_mode) {
            ngx_log_stderr(0, "configuration file %s test is successful",
                           cycle->conf_file.data);
        }

        if (ngx_dump_config) {
            cd = cycle->config_dump.elts;

            for (i = 0; i < cycle->config_dump.nelts; i++) {

                ngx_write_stdout("# configuration file ");
                (void) ngx_write_fd(ngx_stdout, cd[i].name.data,
                                    cd[i].name.len);
                ngx_write_stdout(":" NGX_LINEFEED);

                b = cd[i].buffer;

                (void) ngx_write_fd(ngx_stdout, b->pos, b->last - b->pos);
                ngx_write_stdout(NGX_LINEFEED);
            }
        }

        return 0;
    }

    //发送信息时
    if (ngx_signal) {
        return ngx_signal_process(cycle, ngx_signal);
    }

    ngx_os_status(cycle->log);

    ngx_cycle = cycle;

    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);

    if (ccf->master && ngx_process == NGX_PROCESS_SINGLE) {
        ngx_process = NGX_PROCESS_MASTER;
    }

#if !(NGX_WIN32)

    //注册信号量
    if (ngx_init_signals(cycle->log) != NGX_OK) {
        return 1;
    }

    //创建守护进程
    if (!ngx_inherited && ccf->daemon) {
        if (ngx_daemon(cycle->log) != NGX_OK) {
            return 1;
        }

        ngx_daemonized = 1;
    }

    if (ngx_inherited) {
        ngx_daemonized = 1;
    }

#endif

    //创建 pid 文件写入pid
    if (ngx_create_pidfile(&ccf->pid, cycle->log) != NGX_OK) {
        return 1;
    }

    if (ngx_log_redirect_stderr(cycle) != NGX_OK) {
        return 1;
    }

    if (log->file->fd != ngx_stderr) {
        if (ngx_close_file(log->file->fd) == NGX_FILE_ERROR) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          ngx_close_file_n " built-in log failed");
        }
    }

    ngx_use_stderr = 0;

    //单进程工作模式和多进程工作模式
    if (ngx_process == NGX_PROCESS_SINGLE) {
        ngx_single_process_cycle(cycle);

    } else {
        ngx_master_process_cycle(cycle);
    }

    return 0;
}
```

---
nginx源码基于nginx-1.22.0版本  
**参考**：  
[Nginx 启动初始化过程](https://www.kancloud.cn/digest/understandingnginx/202596)   
[菜鸟nginx源码剖析 框架篇（一） 从main函数看nginx启动流程](https://blog.csdn.net/chen19870707/article/details/41050379)  
