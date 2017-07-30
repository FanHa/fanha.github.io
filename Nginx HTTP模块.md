### HTTP模块初始化
HTTP模块占据了nginx模块的很大一部分,HTTP模块的初始化入口是在ngx_http_module模块里,ngx_http_module的模块类型是NGX_CORE_MODULE,在ngx_init_cycle里
ngx_conf_param
    ngx_conf_parse
        ngx_conf_handler
最终调用了ngx_http_module->ngx_http_commands里的ngx_http_block,这个ngx_http_block就是整个HTTP模块初始化的入口.

### ngx_http_block
ngx_http_block函数首先调用每个HTTP模块的一些回调函数.

ngx_http.c

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

然后切换cmd_type调用下一层(NGX_HTTP_MAIN_CONF)的cmd回调函数.


    /* parse inside the http{} block */

    cf->module_type = NGX_HTTP_MODULE;
    cf->cmd_type = NGX_HTTP_MAIN_CONF;
    rv = ngx_conf_parse(cf, NULL);

接着调用每个HTTP模块的init_main_conf函数,并ngx_http_merge_servers.(这里我认为应该是nginx配置文件里可能里,外层定义了同样的变量,会合并去掉这个冗余的变量,但这里暂时不深究.)

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


接下来到了http模块初始化的的核心循环.
nginx把处理http请求放在了一个一个的按顺序排列的phase阶段里,每个http模块可以把处理http请求的方法注册到某个phase里.当nginx把请求转发给HTTP模块处理流程后,依次调用每个phase里的处理方法,完成对http请求的处理.

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

+ postconfiguration回调:  
    每个http模块在这个回调函数里把自己想要在哪个phase阶段,用哪个函数处理注册到 main_conf里.
+ ngx_http_init_phase_handlers:
    把放在main_conf里的回调函数取出来,按照注册的信息依次放到每个phase,并附上相应的check函数.

ngx_http.c

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

#### 怎样进入http模块的处理流程
nginx建立好tcp连接后对http请求的处理最终会进入到ngx_http_core_run_phases函数(怎么到这一步参考[引用nginx网络IO]).这里就是调用每个phase的回掉函数,并通过前面注册的check函数判断是否进入下一个phase阶段的处理或者返回.  
ngx_http_core_module.c

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