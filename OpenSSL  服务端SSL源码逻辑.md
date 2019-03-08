### nginx使用SSL的方式
`ssl 握手在 tcp握手后进行`

以nginx使用ssl为例
```c
    // 服务器端在tcp握手结束后,设置下一次该链接产生的读事件的回调 ngx_http_ssl_handshake,等待客户端发起ssl握手
    rev->handler = ngx_http_ssl_handshake;
```
```c
    // ngx_http_ssl_handshake 重复调用ngx_ssl_handshake
    // 直到ssl握手完成
    rc = ngx_ssl_handshake(c);

    if (rc == NGX_AGAIN) {

        c->ssl->handler = ngx_http_ssl_handshake_handler;
        return;
    }
```
```c
    //ngx_ssl_handshake 是对ssl_lib里 的 SSL_do_handshake函数的一层封装,实际nginx服务端在收到读事件时,就是反复调用SSL_do_handshake这个API,握手的具体 客户端《-》服务端的信息交互都隐藏在这层API里,
    //SSL_do_handshake 返回1时 代表ssl握手完成, nginx可以正式接受http应用层的信息了
ngx_ssl_handshake(ngx_connection_t *c)
{
    n = SSL_do_handshake(c->ssl->connection);

    // n==1 时代表ssl握手成功了,可以把当前连接的读写回调改成更靠近http应用层的回调, 即读前解密,写前加密等等环节
    if (n == 1) {

        if (ngx_handle_read_event(c->read, 0) != NGX_OK) {
            return NGX_ERROR;
        }

        if (ngx_handle_write_event(c->write, 0) != NGX_OK) {
            return NGX_ERROR;
        }

#if (NGX_DEBUG)
        ngx_ssl_handshake_log(c);
#endif

        c->ssl->handshaked = 1;

        c->recv = ngx_ssl_recv;
        c->send = ngx_ssl_write;
        c->recv_chain = ngx_ssl_recv_chain;
        c->send_chain = ngx_ssl_send_chain;

        return NGX_OK;
    }
    
    if (sslerr == SSL_ERROR_WANT_READ) {
        // ...
        return NGX_ERROR;
        // ...
        // 当握手还在正常的进行中时会返回NGX_AGAIN,便于nginx下次再重复此流程,直到SSL API返回连接成功(或失败)的结果
        return NGX_AGAIN;
    }

    if (sslerr == SSL_ERROR_WANT_WRITE) {
        // ... 
        return NGX_ERROR;
        // ...
        return NGX_AGAIN;
    }
}
```

### SSL_do_handshake
 TODO 