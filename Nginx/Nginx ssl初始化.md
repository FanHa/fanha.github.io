
nginx 调用openssl的api SSL_do_handshake 时,里面的最主要的逻辑是s->handshake_func,nginx在什么时候怎样初始化这个方法?

```c
// nginx/src/http/ngx_http_request.c
ngx_int_t
ngx_ssl_handshake(ngx_connection_t *c)
{
    n = SSL_do_handshake(c->ssl->connection);
}
```

###  SSL_do_handshake
```c
// openssl/ssl/ssl_lib.c
int SSL_do_handshake(SSL *s)
{
    int ret = 1;
    // 这里调用了传入的s参数的方法 handshake_func,往前推溯这个s的来源
    ret = s->handshake_func(s);

    return ret;
}
```

```c
static void
ngx_http_ssl_handshake(ngx_event_t *rev)
{
    sscf = ngx_http_get_module_srv_conf(hc->conf_ctx,ngx_http_ssl_module);

    // 参数 c 在这里ngx_ssl_create_connection 有处理,查看ngx_ssl_create_connection详情
    if (ngx_ssl_create_connection(&sscf->ssl, c, NGX_SSL_BUFFER)
        != NGX_OK)
    {

    }
    rc = ngx_ssl_handshake(c);

```
```c
// nginx/src/event/ngx_event_openssl.c
ngx_int_t
ngx_ssl_create_connection(ngx_ssl_t *ssl, ngx_connection_t *c, ngx_uint_t flags)
{
    ngx_ssl_connection_t  *sc;
    sc->connection = SSL_new(ssl->ctx);

    // 这里区分两种情形,一种是ssl的发起者,一种是ssl的接收者,把参数里的handshake_func设置成了不同的回调函数
    if (flags & NGX_SSL_CLIENT) {
        SSL_set_connect_state(sc->connection);

    } else {
        SSL_set_accept_state(sc->connection);
    }
    c->ssl = sc;
}
```
```c
// openssl/ssl/ssl_lib.c
void SSL_set_connect_state(SSL *s)
{
    s->server = 0;
    s->shutdown = 0;
    ossl_statem_clear(s);

    // 设置s->handshake_func 为 s->method里的ssl_connect,向前推溯下这个 s 的初始化的地方
    s->handshake_func = s->method->ssl_connect;
    clear_ciphers(s);
}
void SSL_set_accept_state(SSL *s)
{
    s->server = 1;
    s->shutdown = 0;
    ossl_statem_clear(s);
    // 设置s->handshake_func 为 s->method里的ssl_accept,向前推溯下这个 s 的初始化的地方

    s->handshake_func = s->method->ssl_accept;
    clear_ciphers(s);
}
```
```c
//  nginx/src/event/ngx_event_openssl.c
ngx_int_t
ngx_ssl_create_connection(ngx_ssl_t *ssl, ngx_connection_t *c, ngx_uint_t flags)
{
    ngx_ssl_connection_t  *sc;

    // sc->connection 的初始化是SSL_new以ssl->ctx 决定的,接下来查看SSL_new的详情
    sc->connection = SSL_new(ssl->ctx);

    if (flags & NGX_SSL_CLIENT) {
        SSL_set_connect_state(sc->connection);

    }
}
```
```c
// openssl/ssl/ssl_lib.c
SSL *SSL_new(SSL_CTX *ctx)
{
    SSL *s;
    
    // s->method 由传入的参数 ctx->method决定,因此接下来推溯上一段代码的ssl->ctx 的由来
    s->method = ctx->method;
}
```
```c
// nginx/src/http/ngx_http_request.c
static void
ngx_http_ssl_handshake(ngx_event_t *rev)
{
    // 这里sscf由ngx_http_get_module_src_conf得到,继续推溯ngx_http_get_module_srv_conf
    sscf = ngx_http_get_module_srv_conf(hc->conf_ctx, ngx_http_ssl_module);
    if (ngx_ssl_create_connection(&sscf->ssl, c, NGX_SSL_BUFFER)!= NGX_OK)
    {
        ngx_http_close_connection(c);
        return;
    }
}
```
```c
// nginx/src/http/ngx_http_config.h
// 这里ngx_http_get_module_srv_conf是个宏,解开来其实是上个代码段的hc->conf_ctx->srv_conf[ngx_http_ssl_model.ctx_index]
// 继续推溯上个代码段里的hc->conf_ctx->srv_conf[ngx_http_ssl_module.ctx_index]
#define ngx_http_get_module_srv_conf(r, module)  (r)->srv_conf[module.ctx_index]
```
```c
// nginx/src/http/modules/ngx_http_ssl_module.c
static char *
ngx_http_ssl_merge_srv_conf(ngx_conf_t *cf, void *parent, void *child)
{
    // 在http_ssl模块的钩子函数中找到了ngx_ssl_create函数创建ssl
    if (ngx_ssl_create(&conf->ssl, conf->protocols, conf) != NGX_OK) {
        return NGX_CONF_ERROR;
    }
}
```
```c
// nginx/src/event/ngx_event_openssl.c
ngx_int_t
ngx_ssl_create(ngx_ssl_t *ssl, ngx_uint_t protocols, void *data)
{
    // 这里ssl->的ctx 由SSL_CTX_new 以 SSLv23_method() 为参数创建,这两个函数都是openssl的api.所以其实推溯了这么久的这个handshanke_func并不是由使用方(即nginx)设置的回调.而是openssl库提供了,猜测应该是与不同版本兼容相关的吧.
    ssl->ctx = SSL_CTX_new(SSLv23_method());
}
```


