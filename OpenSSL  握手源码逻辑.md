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
openssl握手分为读操作(解析从客户端发来的包)和写操作(发送消息到客户端);
这两个操作通过挂钩函数一步步向前推进,完成一次交互;

```c
// ssl/statem/statem.c
// 写操作钩子函数和状态变化
static SUB_STATE_RETURN write_state_machine(SSL *s)
{
    OSSL_STATEM *st = &s->statem;

    if (s->server) {
        // 作为ssl服务端的情况
        transition = ossl_statem_server_write_transition;
        pre_work = ossl_statem_server_pre_work;
        post_work = ossl_statem_server_post_work;
        get_construct_message_f = ossl_statem_server_construct_message;
    } else {
        // 作为ssl客户端(发起者)的情况
        transition = ossl_statem_client_write_transition;
        pre_work = ossl_statem_client_pre_work;
        post_work = ossl_statem_client_post_work;
        get_construct_message_f = ossl_statem_client_construct_message;
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
            // 根据当前的hand_state选择构造消息的函数,并调用该函数将要发回客户端的信息准备好
            if (!get_construct_message_f(s, &pkt, &confunc, &mt)) {
                /* SSLfatal() already called */
                return SUB_STATE_ERROR;
            }
            // ...
            if (confunc != NULL && !confunc(s, &pkt)) {
                WPACKET_cleanup(&pkt);
                check_fatal(s, SSL_F_WRITE_STATE_MACHINE);
                return SUB_STATE_ERROR;
            }

        case WRITE_STATE_SEND:
            // 这里将构造号的内容发回客户端
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
        // 作为ssl服务端的情况
        transition = ossl_statem_server_read_transition;
        process_message = ossl_statem_server_process_message;
        max_message_size = ossl_statem_server_max_message_size;
        post_process_message = ossl_statem_server_post_process_message;
    } else {
        // 作为ssl客户端(发起者)的情况
        transition = ossl_statem_client_read_transition;
        process_message = ossl_statem_client_process_message;
        max_message_size = ossl_statem_client_max_message_size;
        post_process_message = ossl_statem_client_post_process_message;
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

#### 服务端握手状态机
服务端在实际的握手中需要进行多次读和多次写,且读写所调用的函数不尽相同,需要通过另一个状态变量(hand_state)来决定
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
服务端最开始的状态是‘未定义’,收到客户端发来的clientHello后,  
服务端在读状态机的‘READ_STATE_HEADER’阶段将hand_state设置为TLS_ST_SR_CLNT_HELLO,  
并在‘READ_STATE_BODY’阶段调用函数tls_process_client_hello解析从客户端发来的clientHello握手请求  
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
 + 服务端解析完clientHello后,读写状态及切换为写状态‘MSG_FLOW_WRITING‘,  
 + 然后在写状态的’WRITE_STATE_TRANSITION‘阶段将hand_state转换为'TLS_ST_SW_SRVR_HELLO',  
 + 写状态机在‘WRITE_STATE_PRE_WORK’阶段,根据hand_state状态调用了tls_construct_server_hello来构造要写的消息内容;
 + 构造要发的消息后,执行send阶段的函数,将消息”发“出去(这里并不一定真发出去了,优化可能会合并多个send操作)
 ```c
 // ssl/statem/statem_srvr.c
int tls_construct_server_hello(SSL *s, WPACKET *pkt)
{
    // ...
    // 服务端的版本和随机数
    version = usetls13 ? TLS1_2_VERSION : s->version;
    if (!WPACKET_put_bytes_u16(pkt, version)
               /*
                * Random stuff. Filling of the server_random takes place in
                * tls_process_client_hello()
                */
            || !WPACKET_memcpy(pkt,
                               s->hello_retry_request == SSL_HRR_PENDING
                                   ? hrrrandom : s->s3->server_random,
                               SSL3_RANDOM_SIZE)) {
        //...
    }
    // sessionId 及 服务端选择的 加密方式
    if (!WPACKET_sub_memcpy_u8(pkt, session_id, sl)
            || !s->method->put_cipher_by_char(s->s3->tmp.new_cipher, pkt, &len)
            || !WPACKET_put_bytes_u8(pkt, compm)) {
        // ...
    }
}
 ```

 ##### 服务端ServerCertificate
 + 服务端写完clientHello后,读写状态机依然处于写状态,这时会重复一次写状态的流程;
 + 但这次写状态及在’WRITE_STATE_TRANSITION‘阶段根据当前的hand_state转变为‘TLS_ST_SW_CERT’;
 + 然后在写状态机‘WRITE_STATE_PRE_WORK’阶段时,根据新的hand_state状态,调用了构造消息的函数tls_construct_server_certificate;
 + send;
```c
 // ssl/statem/statem_srvr.c
int tls_construct_server_certificate(SSL *s, WPACKET *pkt)
{
    CERT_PKEY *cpk = s->s3->tmp.cert;

    // 准备好服务端要发的证书
    if (!ssl3_output_cert_chain(s, pkt, cpk)) {
        /* SSLfatal() already called */
        return 0;
    }

    return 1;
}
```

##### 服务端HelloDone
+ 服务端发送完ServerCertificate后还有一些“客户端服务端”双向认证的选项;
+ 但一般常见的ssl握手服务端的Hello到发送完ServerCertificate就结束并将读写状态机置为读状态,hand_state变为'TLS_ST_SW_SRVR_DONE',等待客户端验证完成再次发来的消息
```c

    case TLS_ST_SW_CERT:
        // ...
        /* Fall through */

    case TLS_ST_SW_CERT_STATUS:
        // ...
        /* Fall through */

    case TLS_ST_SW_KEY_EXCH:
        // ...
        /* Fall through */

    case TLS_ST_SW_CERT_REQ:
        st->hand_state = TLS_ST_SW_SRVR_DONE;
        return WRITE_TRAN_CONTINUE;
```


##### 服务端收到KeyExchangeMessage
服务端再次收到客户端的消息是客户端认证了服务端的证书后,用服务端的公钥加密的KeyExchangeMessage信息,里面附带了客户端新生成的随机数(学名pre-master secret);
>> 注1: 前面服务端和客户端各生成了一个随机数,但这两个随机数的传输是明文的,而这里客户端新生成的随机数的传输是通过了公钥加密的
>> 注2: 既然只有最后一个随机数是加密的,那前面两个随机数的意义是什么? 使随机数更随机?

+ 服务端将hand_state状态变为'TLS_ST_SR_KEY_EXCH';
+ processes_message阶段,根据当前的hand_state状态,调用tls_process_client_key_exchange
```c
// ssl/statem/statem_srvr.c
    case TLS_ST_SW_SRVR_DONE:
        // ...
        st->hand_state = TLS_ST_SR_KEY_EXCH;
    break;
```
```c
switch (st->hand_state) {
    // ...

    case TLS_ST_SR_KEY_EXCH:
        return tls_process_client_key_exchange(s, pkt);
    // ...
```
```c
// ssl/statem/statem_srvr.c
// 以rsa算法为例
MSG_PROCESS_RETURN tls_process_client_key_exchange(SSL *s, PACKET *pkt)
{
    // ...
    if (alg_k & SSL_kPSK) {
        
    } else if (alg_k & (SSL_kRSA | SSL_kRSAPSK)) {
        if (!tls_process_cke_rsa(s, pkt)) {
            /* SSLfatal() already called */
            goto err;
        }
    }
    // ...
```
```c
static int tls_process_cke_rsa(SSL *s, PACKET *pkt)
{
    // 得到服务器的私钥
    rsa = EVP_PKEY_get0_RSA(s->cert->pkeys[SSL_PKEY_RSA].privatekey);
    // ...
    
    // 用私钥解密客户端发来的消息
    decrypt_len = (int)RSA_private_decrypt((int)PACKET_remaining(&enc_premaster),
                                           PACKET_data(&enc_premaster),
                                           rsa_decrypt, rsa, RSA_NO_PADDING);

    // 用解密后的pre_master_secret生成一个新的密码,这个密码就是接下来用来和客户端交互时加密的
    // 握手完成
    // 1. 因为客户端也是用同样的信息的同样的方法来生成这个密码,所以服务端和客户端的密码是一致的;
    // 2. 接下来对传输内容的加密是对称的,服务端客户端用同一个密码,同一种加解密方式对交互信息进行处理;
    
    if (!ssl_generate_master_secret(s, rsa_decrypt + padding_len,sizeof(rand_premaster_secret), 0)) {

    }
}
```

#### 客户端握手状态机
```c
// ssl/statem/statem.c
static int state_machine(SSL *s, int server)
{
    /* Initialise state machine */
    if (st->state == MSG_FLOW_UNINITED
            || st->state == MSG_FLOW_FINISHED) {
        if (st->state == MSG_FLOW_UNINITED) {

            // 状态机的初始状态从TLS_ST_BEFORE开始
            st->hand_state = TLS_ST_BEFORE;
            st->request_state = TLS_ST_BEFORE;
        }
    }
}
```
```c
// ssl/statem/statem_clnt.c
WRITE_TRAN ossl_statem_client_write_transition(SSL *s)
{
    OSSL_STATEM *st = &s->statem;

    switch (st->hand_state) {

    case TLS_ST_BEFORE:
        // transition阶段将初始的握手状态置为 TLS_ST_CW_CLNT_HELLO
        st->hand_state = TLS_ST_CW_CLNT_HELLO;
        return WRITE_TRAN_CONTINUE;
    }
}
```
```c
// ssl/statem/statem_clnt.c
int ossl_statem_client_construct_message(SSL *s, WPACKET *pkt,
                                         confunc_f *confunc, int *mt)
{
    OSSL_STATEM *st = &s->statem;

    switch (st->hand_state) {
    case TLS_ST_CW_CLNT_HELLO:
        // 调用tls_construct_client_hello构造握手的hello包
        *confunc = tls_construct_client_hello;
        *mt = SSL3_MT_CLIENT_HELLO;
        break;
    }
}

int tls_construct_client_hello(SSL *s, WPACKET *pkt)
{
    unsigned char *session_id;

    // 生成客户端随机数
    if (i && ssl_fill_hello_random(s, 0, p, sizeof(s->s3->client_random),
                                   DOWNGRADE_NONE) <= 0) {
    }

    // sessionId(从其他资料得知得知sessionId在客户端一直为0)
    session_id = s->session->session_id;

    // 客户端支持的加密算法
    if (!ssl_cipher_list_to_bytes(s, SSL_get_ciphers(s), pkt)) {
        /* SSLfatal() already called */
        return 0;
    }

    // 压缩方式
    if (ssl_allow_compression(s)
            && s->ctx->comp_methods
            && (SSL_IS_DTLS(s) || s->s3->tmp.max_ver < TLS1_3_VERSION)) {
        int compnum = sk_SSL_COMP_num(s->ctx->comp_methods);

    }

    // tls扩展
    if (!tls_construct_extensions(s, pkt, SSL_EXT_CLIENT_HELLO, NULL, 0)) {
        /* SSLfatal() already called */
        return 0;
    }

}

```
将一些基本信息附在clientHello包发送完后,读写状态的状态置为读,收到回信后转入read_state_machine处理
```c
// ssl/statem/statem_clnt.c
int ossl_statem_client_read_transition(SSL *s, int mt)
{

    case TLS_ST_CW_CLNT_HELLO:
        if (mt == SSL3_MT_SERVER_HELLO) {
            // transition阶段把握手状态置为 TLS_ST_CR_SRVR_HELLO
            st->hand_state = TLS_ST_CR_SRVR_HELLO;
            return 1;
        }
        break;
    }
}
```
```c
// ssl/statem/statem_clnt.c
MSG_PROCESS_RETURN ossl_statem_client_process_message(SSL *s, PACKET *pkt)
{

    case TLS_ST_CR_SRVR_HELLO:
        // process_message阶段,调用tls_process_server_hello
        return tls_process_server_hello(s, pkt);
    }
}

MSG_PROCESS_RETURN tls_process_server_hello(SSL *s, PACKET *pkt)
{
    // 读取服务端生成的随机数
    if (s->version == TLS1_3_VERSION
            && sversion == TLS1_2_VERSION
            && PACKET_remaining(pkt) >= SSL3_RANDOM_SIZE
            && memcmp(hrrrandom, PACKET_data(pkt), SSL3_RANDOM_SIZE) == 0) {
        s->hello_retry_request = SSL_HRR_PENDING;
        hrr = 1;
        if (!PACKET_forward(pkt, SSL3_RANDOM_SIZE)) {
        }
    } else {
        if (!PACKET_copy_bytes(pkt, s->s3->server_random, SSL3_RANDOM_SIZE)) {
        }
    }

    // 获取服务端生成的sessionId
    if (!PACKET_get_length_prefixed_1(pkt, &session_id)) {
    }

    // 获取服务端选择的加密算法
    if (!PACKET_get_bytes(pkt, &cipherchars, TLS_CIPHER_LEN)) {
    
    }

    // 获取服务端选择的压缩方式
    if (!PACKET_get_1(pkt, &compression)) {
    }


}
```
客户端获取了服务端的"选择"后,机修处于读状态机,到下一轮transition阶段
```c
// ssl/statem/statem_clnt.c
static int ossl_statem_client13_read_transition(SSL *s, int mt)
{

    case TLS_ST_CR_SRVR_HELLO:
        if (mt == SSL3_MT_ENCRYPTED_EXTENSIONS) {
            // 将握手阶段变为 TLS_ST_CR_ENCRYPTED_EXTENSIONS
            st->hand_state = TLS_ST_CR_ENCRYPTED_EXTENSIONS;
            return 1;
        }
        break;
}
// 这里暂时略过非主线的 TLS_ST_CR_ENCRYPTED_EXTENSIONS 处理,假设处理完

```
