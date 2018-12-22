### 阶段
+ ngx_event_accept
+ ngx_http_ssl_handshake
+ ngx_http_wait_request_handler

#### ngx_event_accept
第一个读事件的触发是在,客户端与服务端建立TCP链接后,服务端初始化链接数据结构等
```c
// src/event/ngx_event_accept.c
ngx_event_accept(ngx_event_t *ev)
{
    socklen_t          socklen;
    ngx_err_t          err;
    ngx_log_t         *log;
    ngx_uint_t         level;
    ngx_socket_t       s;
    ngx_event_t       *rev, *wev;
    ngx_sockaddr_t     sa;
    ngx_listening_t   *ls;
    ngx_connection_t  *c, *lc;
    ngx_event_conf_t  *ecf;

    lc = ev->data;
    ls = lc->listening;

    do {
        // 这一步代表tcp三次握手成功,服务器可以初始化连接结构,并设置接下来的读事件的回调
        s = accept(lc->fd, &sa.sockaddr, &socklen);
        // 初始化链接结构
        c = ngx_get_connection(s, ev->log);
        // 这里是在程序初始化阶段设置的handler为 ngx_http_init_connection
        ls->handler(c);
    } while (ev->available);
}

```
```c
// src/http/ngx_http_request.c
ngx_http_init_connection(ngx_connection_t *c)
{
    ngx_uint_t              i;
    ngx_event_t            *rev;
    struct sockaddr_in     *sin;
    ngx_http_port_t        *port;
    ngx_http_in_addr_t     *addr;
    ngx_http_log_ctx_t     *ctx;
    ngx_http_connection_t  *hc;

    c->data = hc;

    rev = c->read;

#if (NGX_HTTP_SSL)
    {
    ngx_http_ssl_srv_conf_t  *sscf;

    // ssl配置
    sscf = ngx_http_get_module_srv_conf(hc->conf_ctx, ngx_http_ssl_module);

    if (sscf->enable || hc->addr_conf->ssl) {
        hc->ssl = 1;
        c->log->action = "SSL handshaking";
        // 设置下一次该链接产生的读事件的回调 ngx_http_ssl_handshake,等待客户端发起ssl握手
        rev->handler = ngx_http_ssl_handshake;
    }
    }
}
```

#### ngx_http_ssl_handshake
```c
static void
ngx_http_ssl_handshake(ngx_event_t *rev)
{
    u_char                    *p, buf[NGX_PROXY_PROTOCOL_MAX_HEADER + 1];
    size_t                     size;
    ssize_t                    n;
    ngx_err_t                  err;
    ngx_int_t                  rc;
    ngx_connection_t          *c;
    ngx_http_connection_t     *hc;
    ngx_http_ssl_srv_conf_t   *sscf;
    ngx_http_core_loc_conf_t  *clcf;

    c = rev->data;
    hc = c->data;

    n = recv(c->fd, (char *) buf, size, MSG_PEEK);

    if (n == 1) {
        if (buf[0] & 0x80 /* SSLv2 */ || buf[0] == 0x16 /* SSLv3/TLSv1 */) {
            // 从配置中获取ssl相关配置
            clcf = ngx_http_get_module_loc_conf(hc->conf_ctx,
                                                ngx_http_core_module);

            sscf = ngx_http_get_module_srv_conf(hc->conf_ctx,
                                                ngx_http_ssl_module);

            // 初始化ssl连接结构
            if (ngx_ssl_create_connection(&sscf->ssl, c, NGX_SSL_BUFFER)
                != NGX_OK)
            {
                ngx_http_close_connection(c);
                return;
            }

            // 这里调用的openssl提供的接口函数,与客户端进行ssl握手,这里就把客户端和服务端的证书,加密方式等的交互交给了openssl这个底层库,这个交互需要来回传递多次信息,也会触发服务器的多次读事件回调,直到ngx_ssl_handshake所代表的底层openssl库完成ssl连接,则接着执行下一步ngx_http_ssl_handshake_handler,另外这个函数返回成功(即建立了ssl连接)后,会改写连接的读,写回调,后续经过此连接的信息流都会经过先经过加密解密,再交给后面http处理或发给客户端
            rc = ngx_ssl_handshake(c);
            if (rc == NGX_AGAIN) {
                if (!rev->timer_set) {
                    ngx_add_timer(rev, c->listening->post_accept_timeout);
                }

                ngx_reusable_connection(c, 0);

                c->ssl->handler = ngx_http_ssl_handshake_handler;
                return;
            }
            ngx_http_ssl_handshake_handler(c);

            return;
        }
    }
}

ngx_int_t
ngx_ssl_handshake(ngx_connection_t *c)
{
    int        n, sslerr;
    ngx_err_t  err;
    ngx_ssl_clear_error(c->log);

    n = SSL_do_handshake(c->ssl->connection);
    if (n == 1) {

        if (ngx_handle_read_event(c->read, 0) != NGX_OK) {
            return NGX_ERROR;
        }

        if (ngx_handle_write_event(c->write, 0) != NGX_OK) {
            return NGX_ERROR;
        }

        c->ssl->handshaked = 1;

        // 将该连接的recv,send回调替换为ssl版本的
        c->recv = ngx_ssl_recv;
        c->send = ngx_ssl_write;
        c->recv_chain = ngx_ssl_recv_chain;
        c->send_chain = ngx_ssl_send_chain;
    }
}

ngx_http_ssl_handshake_handler(ngx_connection_t *c)
{
    if (c->ssl->handshaked) {
        // ssl 握手成功后,接下来该连接的读事件回调可以切换成正常的http执行逻辑了,但因为前面把链接的recv 置换成了ngx_ssl_recv,所以在http处理时会先对读到得内容作ssl解密处理
        c->read->handler = ngx_http_wait_request_handler;
        /* STUB: epoll edge */ c->write->handler = ngx_http_empty_handler;

        ngx_reusable_connection(c, 1);

        ngx_http_wait_request_handler(c->read);
    }
}
```

#### ngx_http_wait_request_handler
```c
// src/http/ngx_http_request.c
static void
ngx_http_wait_request_handler(ngx_event_t *rev)
{
    u_char                    *p;
    size_t                     size;
    ssize_t                    n;
    ngx_buf_t                 *b;
    ngx_connection_t          *c;
    ngx_http_connection_t     *hc;
    ngx_http_core_srv_conf_t  *cscf;

    c = rev->data;

    hc = c->data;

    b = c->buffer;

    // 这里的recv调用的是前面已经替换成ssl解析的recv,经过这里后,加密后的数据已经变成了正常的http数据,可以走http处理流程了
    n = c->recv(c, b->last, size);

    c->data = ngx_http_create_request(c);
    if (c->data == NULL) {
        ngx_http_close_connection(c);
        return;
    }

    rev->handler = ngx_http_process_request_line;
    ngx_http_process_request_line(rev);
}
```

#### ssl_recv 和 ssl_send
```c
// src/event/ngx_event_openssl.c
ssize_t
ngx_ssl_recv(ngx_connection_t *c, u_char *buf, size_t size)
{
    for ( ;; ) {
        //调用ssl提供的接口函数SSL_read实现解密
        n = SSL_read(c->ssl->connection, buf, size);

    }
}

ssize_t
ngx_ssl_write(ngx_connection_t *c, u_char *data, size_t size)
{
    // 调用ssl提供的接口函数SSL_write实现数据的加密再传输
    n = SSL_write(c->ssl->connection, data, size);
}
```
