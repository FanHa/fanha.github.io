## Redis源码学习笔记 --- 网络IO
源码版本 3.2.8

### 一、概论
Redis是单线程执行,server.c中的main函数在正常运行的情况下会进入死循环

server.c

    aeSetBeforeSleepProc(server.el,beforeSleep);
    aeMain(server.el);
    aeDeleteEventLoop(server.el);

---
    void aeMain(aeEventLoop *eventLoop) {
        eventLoop->stop = 0;
        while (!eventLoop->stop) {
            if (eventLoop->beforesleep != NULL)
                eventLoop->beforesleep(eventLoop);
            aeProcessEvents(eventLoop, AE_ALL_EVENTS);
        }
    }

这里aeProcessEvents就是真正的执行函数

ae.c
> /* Process every pending time event, then every pending file event
>  (that may be registered by time event callbacks just processed).
>  Without special flags the function sleeps until some file event
>  fires, or when the next time event occurs (if any).
> 
>  * If flags is 0, the function does nothing and returns.
>  * if flags has AE_ALL_EVENTS set, all the kind of events are processed.
>  * if flags has AE_FILE_EVENTS set, file events are processed.
>  * if flags has AE_TIME_EVENTS set, time events are processed.
>  * if flags has AE_DONT_WAIT set the function returns ASAP until all
> the events that's possible to process without to wait are processed.
> 
> The function returns the number of events >processed. */    
>   int aeProcessEvents(aeEventLoop *eventLoop, int flags)

从函数的注释中可以看出，每一次循环会先执行所有的TIME EVENT,然后执行所有的FILE EVENT,此次笔记主要关注redis接收请求和回复请求的网络IO,所以只关注FILE EVENT,FILE EVENT的执行流程如下  
ae.c  

    numevents = aeApiPoll(eventLoop, tvp);
    for (j = 0; j < numevents; j++) {
        aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
        int mask = eventLoop->fired[j].mask;
        int fd = eventLoop->fired[j].fd;
        int rfired = 0;
        
        if (fe->mask & mask & AE_READABLE) {
            rfired = 1;
            fe->rfileProc(eventLoop,fd,fe->clientData,mask);
        }
        if (fe->mask & mask & AE_WRITABLE) {
            if (!rfired || fe->wfileProc != fe->rfileProc)
                fe->wfileProc(eventLoop,fd,fe->clientData,mask);
        }
         processed++;
    }

aeApiPoll(eventLoop, tvp)通过更底层的异步IO方式(目前Redis有4种可选IO:epoll,evport,kqueue,select,打算再在另一个笔记里理清不同IO方式的优劣__TODO__)取得此次循环将要处理的事件,并将事件添加到eventloop->fired[]里，并通过循环每个事件,执行事件里事先注册的 rFileProc 或者 wFileProc函数，完成一次循环。
>注:这里的读和写,是指redis读取客户端发送的命令,和往客户端写客户端请求的数据.  

笔记到这里有几个疑问:1,eventloop里的事件从哪里来?2,读取了事件里的内容(比如执行"get a"),执行的程序做了什么,或把?3,写事件把要返回给客户端的内容写到了哪里,怎么传回客户端


### EVENTLOOP
前面知道,aeApiPoll通过eventloop里的信息来循环调用读，写事件，那eventloop里的事件是什么时候，从哪里来的呢？把代码回到server.c,找到在调用循环前的initServer();

server.c  

    initServer();
---
    void initServer(void) {
        ...
        server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);
        ...
        /* Create an event handler for accepting new connections in TCP and Unix domain sockets. */
        for (j = 0; j < server.ipfd_count; j++) {
            if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
                acceptTcpHandler,NULL) == AE_ERR)
                {
                    serverPanic(
                    "Unrecoverable error creating server.ipfd file event.");
                }
        }
        ...
    }

由此可以看到在initServer()调用封装的aeApiCreate创建更底层IO方式epoll,evport,kqueue,select创建了eventloop,并通过aeCreateFileEvent()函数创建对新的TCP连接的可读监听;即当一个客户发送与服务器开放的端口建立的TCP连接的请求,服务器收到了客户端发送的字符串,此时,会把这个这个事件,和事件的内容(字符串),存在eventloop里,并指定了以后循环此事件的函数acceptTcpHandler.

ae.c

    aeEventLoop *aeCreateEventLoop(int setsize) {
        ...
        if (aeApiCreate(eventLoop) == -1) goto err;
        ...
    }
---
    int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
    {
        if (fd >= eventLoop->setsize) {
            errno = ERANGE;
            return AE_ERR;
        }
        aeFileEvent *fe = &eventLoop->events[fd];

        if (aeApiAddEvent(eventLoop, fd, mask) == -1)
            return AE_ERR;
        fe->mask |= mask;
        if (mask & AE_READABLE) fe->rfileProc = proc;
        if (mask & AE_WRITABLE) fe->wfileProc = proc;
        fe->clientData = clientData;
        if (fd > eventLoop->maxfd)
            eventLoop->maxfd = fd;
        return AE_OK;
    }
- aeApiCreate是对创建底层IO方式epoll,evport,kqueue,select的一种封装(引用)__TODO__;
- aeCreateFileEvent通过判断mask(这里是READABLE),指定rfileProc为acceptTcpHandler.

我们再次回到开始循环处理事件的地方  
ae.c  

    numevents = aeApiPoll(eventLoop, tvp);
    for (j = 0; j < numevents; j++) {
        aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
        int mask = eventLoop->fired[j].mask;
        int fd = eventLoop->fired[j].fd;
        int rfired = 0;
        
        if (fe->mask & mask & AE_READABLE) {
            rfired = 1;
            fe->rfileProc(eventLoop,fd,fe->clientData,mask);
        }
        if (fe->mask & mask & AE_WRITABLE) {
            if (!rfired || fe->wfileProc != fe->rfileProc)
                fe->wfileProc(eventLoop,fd,fe->clientData,mask);
        }
         processed++;
    }
aeApiPoll取出了客户端发送的TCP连接请求这个事件和字符串内容后,调用了rfileProc也就是acceptTcpHandler函数,来建立TCP连接,并做后续的处理.

networking.c

    void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
        int cport, cfd, max = MAX_ACCEPTS_PER_CALL;
        ...

        while(max--) {
            cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
            
            ...

            acceptCommonHandler(cfd,0,cip);
        }
    }
---

    static void acceptCommonHandler(int fd, int flags, char *ip) {
        client *c;
        if ((c = createClient(fd)) == NULL) {
            ...
        }
        ...
    }
---
    client *createClient(int fd) {
        client *c = zmalloc(sizeof(client));

        /* passing -1 as fd it is possible to create a non connected client.
         * This is useful since all the commands needs to be executed
         * in the context of a client. When commands are executed in other
         * contexts (for instance a Lua script) we need a non connected client.     */
        if (fd != -1) {
            anetNonBlock(NULL,fd);
            anetEnableTcpNoDelay(NULL,fd);
            if (server.tcpkeepalive)
                anetKeepAlive(NULL,fd,server.tcpkeepalive);
            if (aeCreateFileEvent(server.el,fd,AE_READABLE,
                readQueryFromClient, c) == AE_ERR)
            {
                close(fd);
                zfree(c);
                return NULL;
            }
        }

        selectDb(c,0);
        c->id = server.next_client_id++;
        c->fd = fd;
        c->name = ...

        ...

        if (fd != -1) listAddNodeTail(server.clients,c);
        initClientMultiState(c);
        return c;
    }

由上面的源码,服务器接受TCP连接,根据accept后的套接字fd,创建一个客户连接,然后调用aeCreateFileEvent注册一个可读的事件,并配以readQueryFromClient函数,并把新的客户连接 listAddNodeTail(server.clients,c)添加到服务器管理的客户连接队列中.这样下一次客户端传来redis命令(比如 "set a 10"),就会触发eventloop里的事件,在redis的主循环里就会调用readQueryFromClient函数来出来这次请求
