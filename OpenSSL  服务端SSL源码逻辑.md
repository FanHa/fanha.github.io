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

#### 读,写操作机制
openssl分为读操作(解析从客户端发来的包)和写操作(发送消息到客户端);
这两个操作通过挂钩函数一步步向前推进,完成一次交互;

```c
// ssl/statem/statem.c
// 写操作钩子函数和状态变化
static SUB_STATE_RETURN write_state_machine(SSL *s)
{
    OSSL_STATEM *st = &s->statem;

    if (s->server) {
        transition = ossl_statem_server_write_transition;
        pre_work = ossl_statem_server_pre_work;
        post_work = ossl_statem_server_post_work;
        get_construct_message_f = ossl_statem_server_construct_message;
    } 
    while (1) {
        switch (st->write_state) {
        case WRITE_STATE_TRANSITION:
            switch (transition(s)) {
            //...
            
            }

        case WRITE_STATE_PRE_WORK:
            switch (st->write_state_work = pre_work(s, st->write_state_work)) {
                //...
            }

        case WRITE_STATE_SEND:
            // 这里是写操作里的主逻辑
            ret = statem_do_write(s);

            //...

        case WRITE_STATE_POST_WORK:
            switch (st->write_state_work = post_work(s, st->write_state_work)) {
            //...
            }
            break;

        default:
            SSLfatal(s, SSL_AD_INTERNAL_ERROR, SSL_F_WRITE_STATE_MACHINE,
                     ERR_R_INTERNAL_ERROR);
            return SUB_STATE_ERROR;
        }
    }
}

// 读操作钩子函数及状态变化
static SUB_STATE_RETURN read_state_machine(SSL *s)
{
    OSSL_STATEM *st = &s->statem;

    if (s->server) {
        transition = ossl_statem_server_read_transition;
        process_message = ossl_statem_server_process_message;
        max_message_size = ossl_statem_server_max_message_size;
        post_process_message = ossl_statem_server_post_process_message;
    } 
    while (1) {
        switch (st->read_state) {
        case READ_STATE_HEADER:
            if (SSL_IS_DTLS(s)) {

            } else {
                ret = tls_get_message_header(s, &mt);
            }
            if (!transition(s, mt))
                return SUB_STATE_ERROR;

            st->read_state = READ_STATE_BODY;
            /* Fall through */
            // ...
        case READ_STATE_BODY:
            // 这里是主逻辑里的读包函数
            ret = process_message(s, &pkt);
            // ...
 
        case READ_STATE_POST_PROCESS:
            st->read_state_work = post_process_message(s, st->read_state_work);
            // ...
            break;

        default:
            /* Shouldn't happen */
            SSLfatal(s, SSL_AD_INTERNAL_ERROR, SSL_F_READ_STATE_MACHINE,
                     ERR_R_INTERNAL_ERROR);
            return SUB_STATE_ERROR;
        }
    }
```

#### 握手交互
服务端在实际的握手中需要进行多次读和多次写,且读写所调用的函数不尽相同,需要通过另一个状态变量来决定
当前连接的握手到了哪一步,并调用哪个函数处理当前的请求
比如下面同样是读从客户端发来的包,但因为当前所处的握手状态不同而转向了不同的处理逻辑
```c
// ssl/statem/statem_srvr.c
/*
 * Process a message that the server has received from the client.
 */
MSG_PROCESS_RETURN ossl_statem_server_process_message(SSL *s, PACKET *pkt)
{
    OSSL_STATEM *st = &s->statem;

    switch (st->hand_state) {
    default:
        /* Shouldn't happen */
        SSLfatal(s, SSL_AD_INTERNAL_ERROR,
                 SSL_F_OSSL_STATEM_SERVER_PROCESS_MESSAGE,
                 ERR_R_INTERNAL_ERROR);
        return MSG_PROCESS_ERROR;

    case TLS_ST_SR_CLNT_HELLO:
        return tls_process_client_hello(s, pkt);

    case TLS_ST_SR_END_OF_EARLY_DATA:
        return tls_process_end_of_early_data(s, pkt);

    case TLS_ST_SR_CERT:
        return tls_process_client_certificate(s, pkt);

    case TLS_ST_SR_KEY_EXCH:
        return tls_process_client_key_exchange(s, pkt);

    case TLS_ST_SR_CERT_VRFY:
        return tls_process_cert_verify(s, pkt);

#ifndef OPENSSL_NO_NEXTPROTONEG
    case TLS_ST_SR_NEXT_PROTO:
        return tls_process_next_proto(s, pkt);
#endif

    case TLS_ST_SR_CHANGE:
        return tls_process_change_cipher_spec(s, pkt);

    case TLS_ST_SR_FINISHED:
        return tls_process_finished(s, pkt);

    case TLS_ST_SR_KEY_UPDATE:
        return tls_process_key_update(s, pkt);

    }
}
```

##### 解析clientHello
服务端最开始的状态是‘等待clientHello’,这个时候调用的函数是解析从客户端发来的clientHello握手请求
```c
// ssl/statem/statem_srvr.c
MSG_PROCESS_RETURN tls_process_client_hello(SSL *s, PACKET *pkt)
{
    /* Parse the message and load client random. */
    if (clienthello->isv2) {
        //...
    } else {
        // 保存好从客户端传来的’随机数‘ 和 ’sessionId‘
        if (!PACKET_copy_bytes(pkt, clienthello->random, SSL3_RANDOM_SIZE)
            || !PACKET_get_length_prefixed_1(pkt, &session_id)
            || !PACKET_copy_all(&session_id, clienthello->session_id,
                    SSL_MAX_SSL_SESSION_ID_LENGTH,
                    &clienthello->session_id_len)) {
            // ...
            goto err;
        }

        // 保存从客户端传来的 所支持的加密算法
        if (!PACKET_get_length_prefixed_2(pkt, &clienthello->ciphersuites)) {
            SSLfatal(s, SSL_AD_DECODE_ERROR, SSL_F_TLS_PROCESS_CLIENT_HELLO,
                     SSL_R_LENGTH_MISMATCH);
            goto err;
        }

        // 保存客户端支持的压缩方式
        if (!PACKET_get_length_prefixed_1(pkt, &compression)) {
            SSLfatal(s, SSL_AD_DECODE_ERROR, SSL_F_TLS_PROCESS_CLIENT_HELLO,
                     SSL_R_LENGTH_MISMATCH);
            goto err;
        }

        // 保存客户端的其他扩展信息
        if (PACKET_remaining(pkt) == 0) {
            PACKET_null_init(&clienthello->extensions);
        } else {
            if (!PACKET_get_length_prefixed_2(pkt, &clienthello->extensions)
                    || PACKET_remaining(pkt) != 0) {
                SSLfatal(s, SSL_AD_DECODE_ERROR, SSL_F_TLS_PROCESS_CLIENT_HELLO,
                         SSL_R_LENGTH_MISMATCH);
                goto err;
            }
        }
    }

    s->clienthello = clienthello;

    return MSG_PROCESS_CONTINUE_PROCESSING;

 err:
    if (clienthello != NULL)
        OPENSSL_free(clienthello->pre_proc_exts);
    OPENSSL_free(clienthello);

    return MSG_PROCESS_ERROR;
}
```

##### 服务端ServerHello
 TODO 
