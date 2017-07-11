## Redis源码学习笔记 --- 网络IO
源码版本 3.2.9

### 概论

Redis是单线程执行,server.c中的main函数在正常运行的情况下会进入死循环.  
redis处理一个简单普通命令(如 "get X")的网络IO流程如下:  
1.redis服务器在主循环前监听某端口,注册一个读事件,配以处理tcp连接的函数;  
2.客户端发起tcp连接,服务器在主循环中检测到这个事件,触发处理tcp连接函数,在这个函数里,注册了另外的读事件,配以处理redis命令的函数;  
3.建立tcp连接完成后,客户端发送redis命令,服务器在主循环中检测到这个事件,触发处理redis命令的函数,这个函数解析,并执行命令,把要返回的结果写到相应的buffer区.  
4.每一次主循环前,程序会查看Buffer区是否有要返回的内容,并通过两种方式返回给客户端.  
    4.1. 直接返回给客户端;  
    4.2. 注册一个写事件,再下一次主循环触发写事件的函数,返回信息给客户端;


### 主循环
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

由上面的源码,服务器接受TCP连接,根据accept后的套接字fd,创建一个客户连接,然后调用aeCreateFileEvent注册一个可读的事件,并配以readQueryFromClient函数,并把新的客户连接 listAddNodeTail(server.clients,c)添加到服务器管理的客户连接队列中.这样下一次客户端传来redis命令(比如 "set a 10"),就会触发eventloop里的事件,在redis的主循环里就会调用readQueryFromClient函数来出来这次请求.

### readQueryFromClient
redis服务器与客户端建立TCP连接后,监听了客户端后续发送的请求事件,并注册此事件的执行函数为readQueryFromClient.

networking.c  

    void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
        ...
        c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
        nread = read(fd, c->querybuf+qblen, readlen);
        ...
        processInputBuffer(c);
    }
---
    void processInputBuffer(client *c) {
        ...
        while(sdslen(c->querybuf)) {
            ...
            if (c->argc == 0) {
                resetClient(c);
            } else {
                /* Only reset the client when the command was executed. */
                if (processCommand(c) == C_OK)
                    resetClient(c);
                ...
            }
        }
        server.current_client = NULL;
    }
去掉其他辅助的处理,可以看函数将套接字请求里的内容读取到了 c->querybuf中,然后 processInputBuffer(c)处理请求内容;processInputBuffer经过一些前置性的辅助处理后,最终调用processCommand(C)来处理命令;

server.c  

    int processCommand(client *c) {
        ...
        c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
        ...
        /* Exec the command */
        if (c->flags & CLIENT_MULTI &&
            c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
            c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
        {
            ...
        } else {
            call(c,CMD_CALL_FULL);
            c->woff = server.master_repl_offset;
            if (listLength(server.ready_keys))
                handleClientsBlockedOnLists();
        }
        return C_OK;
    }
---
    void call(client *c, int flags) {
        ...
        c->cmd->proc(c);
        ...
    ｝
processCommand解析命令,找到命令将调用的函数,检测排除各种可能的错误,到call(c,CMD_CALL_FULL)函数执行;call函数里,调用了之前动态查找到的命令函数.  
注意,这里call函数并没有返回结果,那么服务器怎么把命令的结果返回给客户端呢?这里以最基本的命令"get X"为例;这个命令在t_string.c文件的对应函数是void getCommand(client *c).

t_string.c

    void getCommand(client *c) {
        getGenericCommand(c);
    }
---
    int getGenericCommand(client *c) {
        robj *o;

        if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.nullbulk)) == NULL)
            return C_OK;

        if (o->type != OBJ_STRING) {
            ...
        } else {
            addReplyBulk(c,o);
            return C_OK;
        }
    }

函数将命令调用返回的结果通过addReplyBulk(c,o)交给了client结构处理.

networking.c

    void addReplyBulk(client *c, robj *obj) {
        addReplyBulkLen(c,obj);
        addReply(c,obj);
        addReply(c,shared.crlf);
    }
---
    void addReply(client *c, robj *obj) {
        if (prepareClientToWrite(c) != C_OK) return;
        ...
        if (...)) {
            if (...)
                _addReplyObjectToList(c,obj);
        } else if (...) {
            ...
            if (...) {
                ...
                if (_addReplyToBuffer(c,buf,len) == C_OK)
                ...
            }
            ...
            if (...)
            ...
        } else {
            serverPanic("Wrong obj->encoding in addReply()");
        }
    }

首先通过prepareClientToWrite(c) 把客户端加到需要回复请求的队列里 ,然后把要写的内容通过_addReplyObjectToList加到相应的buff区. 
>在这里,我本以为会注册一个写事件,这样下一次程序主循环会触发这个写事件并返回内容到客户端,但实际上源码并没有这样做,于是带着疑问又回到主循环的代码.

ae.c  

    void aeMain(aeEventLoop *eventLoop) {
        eventLoop->stop = 0;
        while (!eventLoop->stop) {
            if (eventLoop->beforesleep != NULL)
                eventLoop->beforesleep(eventLoop);
            aeProcessEvents(eventLoop, AE_ALL_EVENTS);
        }
    }

在这里发现在aeProcessEvent函数前面调用了 eventLoop->beforesleep(eventLoop),回溯到这个函数的注册和原型.

server.c

    aeSetBeforeSleepProc(server.el,beforeSleep);
    aeMain(server.el);
    aeDeleteEventLoop(server.el);
---
    void beforeSleep(struct aeEventLoop *eventLoop) {
        ...
        /* Handle writes with pending output buffers. */
        handleClientsWithPendingWrites();
    }


networking.c

    int handleClientsWithPendingWrites(void) {
        ...
        while((ln = listNext(&li))) {
            ...
            /* Try to write buffers to the client socket. */
            if (writeToClient(c->fd,c,0) == C_ERR) continue;

            /* If there is nothing left, do nothing. Otherwise install
            * the write handler. */
            if (clientHasPendingReplies(c) &&
                aeCreateFileEvent(server.el, c->fd, AE_WRITABLE,
                    sendReplyToClient, c) == AE_ERR)
            {
                freeClientAsync(c);
            }
        }
        return processed;
    }

从这个流程可以看出,redis服务器先尝试直接向客户端写,如果遇到了问题(可能某些命令需要后续的返回),才会注册一个写事件,以便再下一次主循环时触发这个事件,把内容返回给客户端.


### 总结
redis处理一个简单普通命令(如 "get X")的网络IO流程如下:  
1.redis服务器在主循环前监听某端口,注册一个读事件,配以处理tcp连接的函数;  
2.客户端发起tcp连接,服务器在主循环中检测到这个事件,触发处理tcp连接函数,在这个函数里,注册了另外的读事件,配以处理redis命令的函数;  
3.建立tcp连接完成后,客户端发送redis命令,服务器在主循环中检测到这个事件,触发处理redis命令的函数,这个函数解析,并执行命令,把要返回的结果写到相应的buffer区.  
4.每一次主循环前,程序会查看Buffer区是否有要返回的内容,并通过两种方式返回给客户端.  
    4.1. 直接返回给客户端;  
    4.2. 注册一个写事件,再下一次主循环触发写事件的函数,返回信息给客户端;

### 其他
第一次写笔记,剪辑掉了很多自认为相关性较少的内容,只保留了与网络IO相关的.以后慢慢把其他方面的内容补上,并结合更深层次的理解,把这篇也优化,表述更清晰