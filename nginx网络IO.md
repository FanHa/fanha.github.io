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
```c
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
```

### 程序执行过程中的模块(module)
nginx 的 main函数中 ngx_init_cycle函数会执行一些模块的初始化工作,期中一些模块前期配置相关的初始化在ngx_init_cycle中,而实际进程中的模块操作初始化会在ngx_xxx_process_cycle函数里进行;

nginx.c
```c
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
```

#### ngx_init_cycle
ngx_cycle.c
```c
    ngx_init_cycle(ngx_cycle_t *old_cycle)
    {
        ...
        // 第一部分
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
        //第二部分
        conf.ctx = cycle->conf_ctx;
        conf.cycle = cycle;
        conf.pool = pool;
        conf.log = log;
        conf.module_type = NGX_CORE_MODULE;
        conf.cmd_type = NGX_MAIN_CONF;
        
        if (ngx_conf_param(&conf) != NGX_CONF_OK) {
            environ = senv;
            ngx_destroy_cycle_pools(&conf);
            return NULL;
        }

        ...
        //第三部分
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
        ...
    }
```

+ 第一部分调用了每个CORE_MODULE的create_conf回调函数;
+ 第二部分ngx_conf_param函数其实隐藏大量的模块的初始化的过程,它由各个CORE_MODULE再调用想对应的子模块的配置解析,但大多与这次笔记的联系相关不大,少数联系紧密的会在后面再提[跳转2];
+ 第三部分则调用各个CORE_MODULE的init_conf回调函数;

#### ngx_master_process_cycle
ngx_process_cycle.c  

```c
ngx_master_process_cycle(ngx_cycle_t *cycle){
        ...
    ngx_start_worker_processes(cycle, ccf->worker_processes,
                               NGX_PROCESS_RESPAWN);
        ...
}

//
ngx_start_worker_processes(ngx_cycle_t *cycle, ngx_int_t n, ngx_int_t type){
        ...
        for (i = 0; i < n; i++) {
            ngx_spawn_process(cycle, ngx_worker_process_cycle,
                            (void *) (intptr_t) i, "worker process", type);

            ...
        }
}

//
ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)
    {
        ...
        ngx_worker_process_init(cycle, worker);
        ...
        for ( ;; ) {
            ...
            ngx_process_events_and_timers(cycle);
            ...
        }
    }
```

ngx_master_process_cycle 通过ngx_spawn_process复制出多个ngx_worker进程(这里只考虑进程情况,实际nginx也有线程选项),每个worker进程相对独立的初始化(ngx_worker_process_init),和执行无限循环处理事件;

#### worker进程初始化(ngx_worker_process_init)[跳转1]
ngx_process_cycle.c 
```c
    ngx_worker_process_init(ngx_cycle_t *cycle, ngx_int_t worker){
        ...
        for (i = 0; cycle->modules[i]; i++) {
            if (cycle->modules[i]->init_process) {
                if (cycle->modules[i]->init_process(cycle) == NGX_ERROR) {
                    /* fatal */
                    exit(2);
                }
            }
        }
        ...
    }
```

执行了每个模块的init_process回调函数

#### 事件处理无限循环(ngx_process_events_and_timers(cycle))
ngx_event.c
```c
    ngx_process_events_and_timers(ngx_cycle_t *cycle){
        ...
        (void) ngx_process_events(cycle, timer, flags);
        ...
        ngx_event_process_posted(cycle, &ngx_posted_accept_events);
        ...
        ngx_event_process_posted(cycle, &ngx_posted_events);
    }
```
这里处理了集中不同情况的event,这次暂时只分析最基本的ngx_process_events

#### ngx_process_events
ngx_process_events是个宏,实际调用的是ngx_event_actions.process_events.

    #define ngx_process_events   ngx_event_actions.process_events
这个ngx_event_actions是根据不同的异步IO方式封装的一层函数组,这里我们只考虑epoll的情况:
ngx_epoll_module.c
```c
    ngx_epoll_init(ngx_cycle_t *cycle, ngx_msec_t timer)
    {
        ...
        ngx_event_actions = ngx_epoll_module_ctx.actions;
        ...
    }
//
    static ngx_event_module_t  ngx_epoll_module_ctx = {
        &epoll_name,
        ngx_epoll_create_conf,               /* create configuration */
        ngx_epoll_init_conf,                 /* init configuration */

        //以下皆为action中的内容
        {
            ngx_epoll_add_event,             /* add an event */
            ngx_epoll_del_event,             /* delete an event */
            ngx_epoll_add_event,             /* enable an event */
            ngx_epoll_del_event,             /* disable an event */
            ngx_epoll_add_connection,        /* add an connection */
            ngx_epoll_del_connection,        /* delete an connection */
    #if (NGX_HAVE_EVENTFD)
            ngx_epoll_notify,                /* trigger a notify */
    #else
            NULL,                            /* trigger a notify */
    #endif
            ngx_epoll_process_events,        /* process the events */
            ngx_epoll_init,                  /* init the events */
            ngx_epoll_done,                  /* done the events */
        }
    };
```

因此ngx_process_event实际调用的是ngx_epoll_process_events函数

##### epoll事件处理(ngx_epoll_process_events)
ngx_epoll_module.c
```c
    ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags){
        ...
        events = epoll_wait(ep, event_list, (int) nevents, timer);
        ...
        for (i = 0; i < events; i++) {
            ...
            rev = c->read;
            ...
            if ((revents & EPOLLIN) && rev->active) {

                if (flags & NGX_POST_EVENTS) {
                    ...
                } else {
                    rev->handler(rev);
                }
            }

            wev = c->write;
            if ((revents & EPOLLOUT) && wev->active) {
                ...
                if (flags & NGX_POST_EVENTS) {
                    ...
                } else {
                    wev->handler(wev);
                }
            }
        }
    }
```

从epoll异步IO系统中取出事件,然后根据类别调用读(或写)事件的回调函数(rev->handler,wev->handler),那么这个回调函数是在哪里被注册的呢,这里可以回过头来[跳到跳转1],ngx_event_core_module模块的init_process回调函数ngx_event_process_init有初始建立连接的回调函数.

#### ngx_event_process_init
ngx_event.c
```c
    ngx_event_process_init(ngx_cycle_t *cycle){
        ...
        rev = cycle->read_events;
        ...
        ls = cycle->listening.elts;
        for (i = 0; i < cycle->listening.nelts; i++) {
            ...
            rev->handler = (c->type == SOCK_STREAM) ? ngx_event_accept
                                                : ngx_event_recvmsg;
            ...
        }
        ...
    }
```
这里看出给每个监听套接字的读事件注册了一个回调函数ngx_event_accept(或ngx_event_recvmsg),一般我们收到的nginx连接都是tcp,即type==SOCK_STREAM,所以这里我们认为回调函数注册的是ngx_event_accept,这样当nginx服务器收到来自客户端的第一个信息时,调用这个函数来建立tcp连接,并为接下来的客户端请求注册新的回调函数;

#### 接受TCP连接的回调函数ngx_event_accept
ngx_event_accept.c
```c
    ngx_event_accept(ngx_event_t *ev){
        ...
        lc = ev->data;
        ls = lc->listening;
        ...
        do {
            ...
            s = accept(lc->fd, &sa.sockaddr, &socklen);
            ...
            ls->handler(c);
            ...
        }
        ...
    }
```
这里accpet建立了连接后,调用了ls->handler,这个回调函数是在哪里注册的呢,[跳到跳转2]在ngx_conf_param一层层解析到ngx_conf_handler函数.

    ngx_conf_param  
        ngx_conf_parse  
            ngx_conf_handler  

ngx_conf_file.c
```c
    ngx_conf_handler(ngx_conf_t *cf, ngx_int_t last){
        for (i = 0; cf->cycle->modules[i]; i++) {

        cmd = cf->cycle->modules[i]->commands;
        if (cmd == NULL) {
            continue;
        }

        for ( /* void */ ; cmd->name.len; cmd++) {
            ...
            rv = cmd->set(cf, cmd, conf);
            ...
        }
        ...
    }
```
这里调用了每个模块的每个commands的set函数,期中ngx_http_module模块的"http"cmd中有个set函数为ngx_http_block.

##### ngx_http_block
ngx_http.c
```c
    ngx_http_block(ngx_conf_t *cf, ngx_command_t *cmd, void *conf){
        ...
        if (ngx_http_optimize_servers(cf, cmcf, cmcf->ports) != NGX_OK) {
            return NGX_CONF_ERROR;
        }
        ...
    }
//
    ngx_http_optimize_servers(ngx_conf_t *cf, ngx_http_core_main_conf_t *cmcf,
    ngx_array_t *ports)
    {
        ...
        for (p = 0; p < ports->nelts; p++) {
            ...
            if (ngx_http_init_listening(cf, &port[p]) != NGX_OK) {
                return NGX_ERROR;
            }
        }
        ...
    }
//
    ngx_http_init_listening(ngx_conf_t *cf, ngx_http_conf_port_t *port)
    {
        ...
        ls = ngx_http_add_listening(cf, &addr[i]);
        ...
    }
//
    ngx_http_add_listening(ngx_conf_t *cf, ngx_http_conf_addr_t *addr)
    {
        ...
        ls->handler = ngx_http_init_connection;
        ...
    }
```
经过一层一层的调用,终于找到了最后定义ls->handler回调函数的地方,在这里也可以看出,nginx在初始化时已经把监听,连接等方方面面与网络io方面的信息都安排妥当,只等客户端发送请求后,把客户的信息填到这些初始化的结构里,就可以直接执行了;而不用动态的配置,选择要执行的方法.

#### ngx_http_init_connection
由上可知nginx在与客户端建立tcp连接后调用了ngx_http_init_connection函数;

ngx_http_request.c
```c
    ngx_http_init_connection(ngx_connection_t *c)
    {
        ...
        rev = c->read;
        rev->handler = ngx_http_wait_request_handler;
        ...
        if (ngx_handle_read_event(rev, 0) != NGX_OK) {
            ngx_http_close_connection(c);
            return;
        }
        ...
    }
```
这里为了解析方便,省略掉了很多旁支的处理,如ssl,如tcp连接复用,等等.
通过ngx_handle_read_event异步IO循环里注册了一个新的读事件,这个读事件的回调函数是ngx_http_wait_request_handler,这样下次客户端发送信息到nginx,在主循环里,执行这个信息处理的回调函数就是ngx_http_wait_request_handler.

#### ngx_http_wait_request_handler
ngx_http_request.c
```c
    ngx_http_wait_request_handler(ngx_event_t *rev)
    {
        ...
        ngx_http_process_request_line(rev);
    }
```

层层解析调用,最终调用了ngx_http_process_request来处理http请求

ngx_http_request.c
```c
    ngx_http_process_request(ngx_http_request_t *r){
        ...
        c->read->handler = ngx_http_request_handler;
        c->write->handler = ngx_http_request_handler;
        r->read_event_handler = ngx_http_block_reading;

        ngx_http_handler(r);

        ngx_http_run_posted_requests(c);
    }
```

#### ngx_http_handler
ngx_http_core_module.c
```c
    ngx_http_handler(ngx_http_request_t *r)
    {
        ...
        r->write_event_handler = ngx_http_core_run_phases;
        ngx_http_core_run_phases(r);
        ...
    }
//
    ngx_http_core_run_phases(ngx_http_request_t *r)
    {
        ...
        ph = cmcf->phase_engine.handlers;

        while (ph[r->phase_handler].checker) {

            rc = ph[r->phase_handler].checker(r, &ph[r->phase_handler]);

            if (rc == NGX_OK) {
                return;
            }
        }
    }
```
这里http请求按照预设的步骤一步步解析处理请求的内容,把请求得到的内容返回给客户端的写事件应该也是在这些步骤里.
[Nginx 源码阅读笔记 HTTP模块](https://fanha.github.io/Nginx%20HTTP%E6%A8%A1%E5%9D%97)