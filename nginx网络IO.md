## nginx源码网络IO笔记
源码版本1.12.1

一些做笔记过程中有帮助的文章http://blog.csdn.net/lengzijian/article/details/7598996

### nginx模块(module)
模块(module)是nginx设计中的重要部分,在代码的很多地方可以看到如下或者类似的循环,通过调用每个模块里的回调函数来一步步处理数据.

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
---
    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->init_process) {
            if (cycle->modules[i]->init_process(cycle) == NGX_ERROR) {
                /* fatal */
                exit(2);
            }
        }
    }

#### 模块(module)类型
nginx有四种类型的模块:core,event,http,mail;  
模块的定义分散在各个文件里,并相应得注入了与该模块相关的回调的函数,如:  
+ nginx.c里的ngx_core_module;
+ ngx_event.c里的ngx_events_module,ngx_event_core_module等等;

#### 模块(module)管理
刚开始看nginx源码时,一直找不到上面循环调用的cycle->modules是在哪里初始化并把来自源代码各个地方的模块加到这个数组里的,后来才知道nginx源码在配置后(./confugure)会生成一个ngx_modules.c文件,里面就包含了如下的代码,然后直接引用这个ngx_modules变量并循环.突然一下也让我明白了为什么nginx新增模块要重启(于此相对的,apache很多模块新增不需要重启,以后再写个apache源码的笔记),这种静态调用模块的方式可能也是nginx性能更好(这个观点是"据很多人说",等我自己把apache和nginx两边源码都过一遍再做个判断)的原因之一把.

ngx_modules.c  

    ngx_module_t *ngx_modules[] = {
        &ngx_core_module,
        &ngx_errlog_module,
        &ngx_conf_module,
        &ngx_regex_module,
        &ngx_events_module,
        &ngx_event_core_module,
        &ngx_epoll_module,
        &ngx_http_module,
        &ngx_http_core_module,
        &ngx_http_log_module,
        &ngx_http_upstream_module,
        ...
        NULL
    };


### 程序执行过程中的模块(module)
nginx 的 main函数中 ngx_init_cycle函数会执行一些模块的初始化工作,期中一些模块前期配置相关的初始化在ngx_init_cycle中,而实际进程中的模块操作初始化会在ngx_xxx_process_cycle函数里进行;

nginx.c

    main(int argc, char *const *argv)
    {
        ...
        cycle = ngx_init_cycle(&init_cycle);
        ...
        if (ngx_process == NGX_PROCESS_SINGLE) {
            ngx_single_process_cycle(cycle);

        } else {
            ngx_master_process_cycle(cycle);
        }
    }

#### 模块(module)的create_conf
在ngx_init_cycle函数中,首先调用了每个core模块的create_conf回调函数

ngx_cycle.c

    ngx_init_cycle(ngx_cycle_t *old_cycle)
    {
        ...
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
        ...
    }