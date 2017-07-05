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

aeApiPoll(eventLoop, tvp)通过更底层的异步IO方式(目前Redis有4种可选IO:epoll,evport,kqueue,select,打算再在另一个笔记里理清不同IO方式的优劣__TODO__)取得此次循环将要处理的事件,并将事件添加到eventloop->fired[]里，并通过循环每个事件,执行事件里事先注册的 rFileProc 或者 wFileProc函数，完成一次循环
>注:这里的读和写,是指redis读取客户端发送的命令,和往客户端写客户端请求的数据.  

笔记到这里有几个疑问:1,eventloop里的事件从哪里来?2,读取了事件里的内容(比如执行"get a"),执行的程序做了什么,或把?3,写事件把要返回给客户端的内容写到了哪里,怎么传回客户端