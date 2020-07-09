### HTTP模块初始化
HTTP模块占据了nginx模块的很大一部分,HTTP模块的初始化入口是在ngx_http_module模块里,ngx_http_module的模块类型是NGX_CORE_MODULE,在ngx_init_cycle里
ngx_conf_param
    ngx_conf_parse
        ngx_conf_handler
最终调用了ngx_http_module->ngx_http_commands里的ngx_http_block,这个ngx_http_block就是整个HTTP模块初始化的入口.

### ngx_http_block
ngx_http_block函数首先调用每个HTTP模块的一些回调函数.

ngx_http.c
```c
    ngx_http_block(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
    {
        ...
        for (m = 0; cf->cycle->modules[m]; m++) {
            if (cf->cycle->modules[m]->type != NGX_HTTP_MODULE) {
                continue;
            }

            module = cf->cycle->modules[m]->ctx;
            mi = cf->cycle->modules[m]->ctx_index;

            if (module->create_main_conf) {
                ctx->main_conf[mi] = module->create_main_conf(cf);
                if (ctx->main_conf[mi] == NULL) {
                    return NGX_CONF_ERROR;
                }
            }

            if (module->create_srv_conf) {
                ctx->srv_conf[mi] = module->create_srv_conf(cf);
                if (ctx->srv_conf[mi] == NULL) {
                    return NGX_CONF_ERROR;
                }
            }

            if (module->create_loc_conf) {
                ctx->loc_conf[mi] = module->create_loc_conf(cf);
                if (ctx->loc_conf[mi] == NULL) {
                    return NGX_CONF_ERROR;
                }
            }
        }
        
        pcf = *cf;
        cf->ctx = ctx;

        for (m = 0; cf->cycle->modules[m]; m++) {
            if (cf->cycle->modules[m]->type != NGX_HTTP_MODULE) {
                continue;
            }

            module = cf->cycle->modules[m]->ctx;

            if (module->preconfiguration) {
                if (module->preconfiguration(cf) != NGX_OK) {
                    return NGX_CONF_ERROR;
                }
            }
        }
        ...
    }
```
然后切换cmd_type调用下一层(NGX_HTTP_MAIN_CONF)的cmd回调函数.

```c
    /* parse inside the http{} block */

    cf->module_type = NGX_HTTP_MODULE;
    cf->cmd_type = NGX_HTTP_MAIN_CONF;
    rv = ngx_conf_parse(cf, NULL);
```

接着调用每个HTTP模块的init_main_conf函数,并ngx_http_merge_servers.(这里我认为应该是nginx配置文件里可能里,外层定义了同样的变量,会合并去掉这个冗余的变量,但这里暂时不深究.)

```c
    for (m = 0; cf->cycle->modules[m]; m++) {
        if (cf->cycle->modules[m]->type != NGX_HTTP_MODULE) {
            continue;
        }

        module = cf->cycle->modules[m]->ctx;
        mi = cf->cycle->modules[m]->ctx_index;

        /* init http{} main_conf's */

        if (module->init_main_conf) {
            rv = module->init_main_conf(cf, ctx->main_conf[mi]);
            if (rv != NGX_CONF_OK) {
                goto failed;
            }
        }

        rv = ngx_http_merge_servers(cf, cmcf, module, mi);
        if (rv != NGX_CONF_OK) {
            goto failed;
        }
    }
```

接下来到了http模块初始化的的核心循环.
nginx把处理http请求放在了一个一个的按顺序排列的phase阶段里,每个http模块可以把处理http请求的方法注册到某个phase里.当nginx把请求转发给HTTP模块处理流程后,依次调用每个phase里的处理方法,完成对http请求的处理.

```c
    if (ngx_http_init_phases(cf, cmcf) != NGX_OK) {
        return NGX_CONF_ERROR;
    }

    ...
    for (m = 0; cf->cycle->modules[m]; m++) {
        if (cf->cycle->modules[m]->type != NGX_HTTP_MODULE) {
            continue;
        }

        module = cf->cycle->modules[m]->ctx;

        if (module->postconfiguration) {
            if (module->postconfiguration(cf) != NGX_OK) {
                return NGX_CONF_ERROR;
            }
        }
    }
    ...
    if (ngx_http_init_phase_handlers(cf, cmcf) != NGX_OK) {
        return NGX_CONF_ERROR;
    }
```

+ postconfiguration回调:  
    每个http模块在这个回调函数里把自己想要在哪个phase阶段,用哪个函数处理注册到 main_conf里.
+ ngx_http_init_phase_handlers:
    把放在main_conf里的回调函数取出来,按照注册的信息依次放到每个phase,并附上相应的check函数.

ngx_http.c
```c
    ngx_http_init_phase_handlers(ngx_conf_t *cf, ngx_http_core_main_conf_t *cmcf)
    {
        ...
        cmcf->phase_engine.handlers = ph;
        n = 0;

        for (i = 0; i < NGX_HTTP_LOG_PHASE; i++) {
            h = cmcf->phases[i].handlers.elts;

            switch (i) {

            case NGX_HTTP_SERVER_REWRITE_PHASE:
                if (cmcf->phase_engine.server_rewrite_index == (ngx_uint_t) -1) {
                    cmcf->phase_engine.server_rewrite_index = n;
                }
                checker = ngx_http_core_rewrite_phase;

                break;

            case NGX_HTTP_FIND_CONFIG_PHASE:
                find_config_index = n;

                ph->checker = ngx_http_core_find_config_phase;
                n++;
                ph++;

                continue;

            case NGX_HTTP_REWRITE_PHASE:
                if (cmcf->phase_engine.location_rewrite_index == (ngx_uint_t) -1) {
                    cmcf->phase_engine.location_rewrite_index = n;
                }
                checker = ngx_http_core_rewrite_phase;

                break;

            case NGX_HTTP_POST_REWRITE_PHASE:
                if (use_rewrite) {
                    ph->checker = ngx_http_core_post_rewrite_phase;
                    ph->next = find_config_index;
                    n++;
                    ph++;
                }

                continue;

            case NGX_HTTP_ACCESS_PHASE:
                checker = ngx_http_core_access_phase;
                n++;
                break;

            case NGX_HTTP_POST_ACCESS_PHASE:
                if (use_access) {
                    ph->checker = ngx_http_core_post_access_phase;
                    ph->next = n;
                    ph++;
                }

                continue;

            case NGX_HTTP_TRY_FILES_PHASE:
                if (cmcf->try_files) {
                    ph->checker = ngx_http_core_try_files_phase;
                    n++;
                    ph++;
                }

                continue;

            case NGX_HTTP_CONTENT_PHASE:
                checker = ngx_http_core_content_phase;
                break;

            default:
                checker = ngx_http_core_generic_phase;
            }

            n += cmcf->phases[i].handlers.nelts;

            for (j = cmcf->phases[i].handlers.nelts - 1; j >=0; j--) {
                ph->checker = checker;
                ph->handler = h[j];
                ph->next = n;
                ph++;
            }
        }

        return NGX_OK;
    }
```

#### 怎样进入http模块的处理流程
nginx建立好tcp连接后对http请求的处理最终会进入到ngx_http_core_run_phases函数(怎么到这一步参考[Nginx 源码阅读笔记 网络IO](https://fanha.github.io/nginx%E7%BD%91%E7%BB%9CIO)).这里就是调用每个phase的回掉函数,并通过前面注册的check函数判断是否进入下一个phase阶段的处理或者返回.  
ngx_http_core_module.c
```c
    ngx_http_core_run_phases(ngx_http_request_t *r)
    {
        ngx_int_t                   rc;
        ngx_http_phase_handler_t   *ph;
        ngx_http_core_main_conf_t  *cmcf;

        cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);

        ph = cmcf->phase_engine.handlers;

        while (ph[r->phase_handler].checker) {

            rc = ph[r->phase_handler].checker(r, &ph[r->phase_handler]);

            if (rc == NGX_OK) {
                return;
            }
        }
    }
```

### nginx转发php请求和静态文件请求
上面处理http模块的phase的NGX_HTTP_CONTENT_PHASE阶段时,调用check函数是 ngx_http_core_content_phase.

ngx_http_core_module.c
```c
    ngx_http_core_content_phase(ngx_http_request_t *r,
        ngx_http_phase_handler_t *ph)
    {
        ...
        if (r->content_handler) {
            r->write_event_handler = ngx_http_request_empty_handler;
            ngx_http_finalize_request(r, r->content_handler(r));
            return NGX_OK;
        }

        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                    "content phase: %ui", r->phase_handler);

        rc = ph->handler(r);
        ...
    }
```
这里可以看到这个阶段先判断r->content_handler是否存在,只有在不存在content_handler时才会调用ph->handler.  
那么这个content_handler什么时候会存在呢?转发fastCGI或者获取静态文件时,会存在.   
在往前看phase处理代码,找到NGX_HTTP_FIND_CONFIG_PHASE阶段的check函数ngx_http_core_find_config_phase,这个函数调用了ngx_http_update_location_config.

ngx_http_core_module.c  
```c
    ngx_http_update_location_config(ngx_http_request_t *r){
        ngx_http_core_loc_conf_t  *clcf;

        clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
        ...
        if (clcf->handler) {
            r->content_handler = clcf->handler;
        }
    }
```
这里可以看到如果loc_conf(即clcf)里本身已注册了handler,则把这个函数赋予r->content_handler.


#### 通过FastCGI转发php请求
我们来看fastCGI模块的内容,
ngx_http_fastcgi_module.c
```c
    static ngx_http_module_t  ngx_http_fastcgi_module_ctx = {
        ngx_http_fastcgi_add_variables,        /* preconfiguration */
        NULL,                                  /* postconfiguration */

        ngx_http_fastcgi_create_main_conf,     /* create main configuration */
        NULL,                                  /* init main configuration */

        NULL,                                  /* create server configuration */
        NULL,                                  /* merge server configuration */

        ngx_http_fastcgi_create_loc_conf,      /* create location configuration */
        ngx_http_fastcgi_merge_loc_conf        /* merge location configuration */
    };
```

这里最后一个ngx_http_fastcgi_merge_loc_conf就是我们要找的注册了loc_conf的handler,而nginx在初始化配置阶段会调用这个merge location 函数[Nginx 源码阅读笔记 网络IO](https://fanha.github.io/nginx%E7%BD%91%E7%BB%9CIO).

```c
    ngx_http_fastcgi_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
    {
        ...
        clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
        ...
        if (clcf->lmt_excpt && clcf->handler == NULL
            && (conf->upstream.upstream || conf->fastcgi_lengths))
        {
            clcf->handler = ngx_http_fastcgi_handler;
        }
        ...
    }
```
这里可以看到clcf的handler是ngx_http_fastcgi_handler.

#### ngx_http_fastcgi_handler
ngx_http_fastcgi_module.c
```c
    ngx_http_fastcgi_handler(ngx_http_request_t *r)
    {
        ...
        ngx_http_upstream_t           *u;
        ...
        if (ngx_http_upstream_create(r) != NGX_OK) {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }
        ...
        u = r->upstream;
        ...
        u->create_request = ngx_http_fastcgi_create_request;
        u->reinit_request = ngx_http_fastcgi_reinit_request;
        u->process_header = ngx_http_fastcgi_process_header;
        u->abort_request = ngx_http_fastcgi_abort_request;
        u->finalize_request = ngx_http_fastcgi_finalize_request;
        ...
        rc = ngx_http_read_client_request_body(r, ngx_http_upstream_init);
        ...
    }
```
首先ngx_http_upstream_create初始创建upsteam(主要是分配内存空间和几个数值参数),然后注册了upstream的一些回调函数,最后调用ngx_http_read_client_request_body处理请求.  
注意这里ngx_http_read_client_request_body的第二个参数 ngx_http_upstream_init是一个函数.

ngx_http_request_body.c
```c
    ngx_http_read_client_request_body(ngx_http_request_t *r,
        ngx_http_client_body_handler_pt post_handler)
    {
        ...
        if (r != r->main || r->request_body || r->discard_body) {
            r->request_body_no_buffering = 0;
            post_handler(r);
            return NGX_OK;
        }
        ...
    }
```
这里直接调用了作为参数传进来的post_handler,也就是前面的ngx_http_upstream_init.

#### ngx_http_upstream_init
ngx_http_upstream.c
```c
    ngx_http_upstream_init(ngx_http_request_t *r)
    {
        ngx_connection_t     *c;
        c = r->connection;
        ...
        if (ngx_event_flags & NGX_USE_CLEAR_EVENT) {

            if (!c->write->active) {
                if (ngx_add_event(c->write, NGX_WRITE_EVENT, NGX_CLEAR_EVENT)
                    == NGX_ERROR)
                {
                    ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
                    return;
                }
            }
        }

        ngx_http_upstream_init_request(r);
    }
 ```   
上面代码可以看到先注册了一个写事件到多路复用IO(epoll)里,然后接着处理请求.
```c
    ngx_http_upstream_init_request(ngx_http_request_t *r)
    {
        ...
        ngx_http_upstream_t            *u;
        ...
        u = r->upstream;
        ...
        if (u->create_request(r) != NGX_OK) {
            ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }
        ...
        ngx_http_upstream_connect(r, u);
        ...
    }
```
---
```c
    ngx_http_upstream_connect(ngx_http_request_t *r, ngx_http_upstream_t *u)
    {
        ngx_connection_t  *c;
        ...
        rc = ngx_event_connect_peer(&u->peer);
        ...
        c = u->peer.connection;

        c->data = r;
        ...
        c->write->handler = ngx_http_upstream_handler;
        c->read->handler = ngx_http_upstream_handler;

        u->write_event_handler = ngx_http_upstream_send_request_handler;
        u->read_event_handler = ngx_http_upstream_process_header;

        c->sendfile &= r->connection->sendfile;
        u->output.sendfile = c->sendfile;
        ...
        ngx_http_upstream_send_request(r, u, 1);
    }
```
上面代码看到注册了一些事件回调函数后,调用ngx_http_upstream_send_request.
```c
    ngx_http_upstream_send_request(ngx_http_request_t *r, ngx_http_upstream_t *u,
        ngx_uint_t do_write)
    {
        ...
        rc = ngx_http_upstream_send_request_body(r, u, do_write);
        ...
        ngx_add_timer(c->read, u->conf->read_timeout);

        if (c->read->ready) {
            ngx_http_upstream_process_header(r, u);
            return;
        }
        ...
    }
```
上面代码将请求转发给上游(如php-fpm),然后加了个时间事件处理上游返回的内容;(注:这里有点疑问,以后细看)
但如果这个时候,读事件(c->read->ready)已经ready里,直接调用处理返回内容的函数ngx_http_upstream_process_header.

#### ngx_http_upstream_process_header
ngx_http_upstream.c
```c
    ngx_http_upstream_process_header(ngx_http_request_t *r, ngx_http_upstream_t *u)
    {
        ...
        for ( ;; ) {

            n = c->recv(c, u->buffer.last, u->buffer.end - u->buffer.last);
            ...
            rc = u->process_header(r);
            ...
        }
        ...
        /* rc == NGX_OK */
        ...
        if (!r->subrequest_in_memory) {
            ngx_http_upstream_send_response(r, u);
            return;
        }
        ...
    }
```
上面代码从上游(php-fpm)获取数据,然后把处理结果通过ngx_http_upstream_send_response返回个客户端.
```c
    ngx_http_upstream_send_response(ngx_http_request_t *r, ngx_http_upstream_t *u)
    {
        ...
        ngx_http_upstream_process_non_buffered_downstream(r);
        ...
    }
```
---
```c
    ngx_http_upstream_process_non_buffered_downstream(ngx_http_request_t *r)
    {
        ngx_http_upstream_process_non_buffered_request(r, 1);
    }
```
---
```c
    ngx_http_upstream_process_non_buffered_request(ngx_http_request_t *r,
        ngx_uint_t do_write)
    {
        ...
        rc = ngx_http_output_filter(r, u->out_bufs);
        ...
        if (ngx_handle_write_event(downstream->write, clcf->send_lowat)
                != NGX_OK){
                    ...
                }
        ...
    }
```
由上面代码可以看到,上游返回的内容加到了写回客户端的buff中,然后添加了一个写事件,写回客户端.至此http模块的处理内容结束.

### 总结
这篇笔记摈弃了很多处理的分支(如缓存,如一个请求需要发多个包才能把内容传输完,各种错误处理等等),只分析了最简单的,从客户端发了一个请求到nginx,nginx把请求发送给上游php-fpm,php-fpm返回给nginx,nginx再把结果返回给客户端.

