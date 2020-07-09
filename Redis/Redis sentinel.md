### 入口
Redis sentinel 和 普通的redis程序共用一套代码(事件循环,事件处理逻辑,存储结果等),程序通过参数辨别自己是`redis`还是`sentinel`;  
```c
// server.c
int main(int argc, char **argv) {
    // ...
    server.sentinel_mode = checkForSentinelMode(argc,argv);
    // ...
    if (server.sentinel_mode) {
        initSentinelConfig();
        initSentinel();
    }
    // ...
    initServer();
    // ...
}
```

初始化阶段创建了一个`TimeEvent`,程序会在接下来周期性调用`serverCron`回调来进行一些幕后的操作
```c
void initServer(void) {
    // ...
    if (aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
        // ...
    }
    // ...
}
```

当程序直到自己是sentinel时,会在这个周期性调度里执行`sentinelTimer`
```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    // ...
    if (server.sentinel_mode) sentinelTimer();
    // ...
}
```

### sentinelTimer
```c
// sentinel.c
void sentinelTimer(void) {
    // 与当前sentinel已知的服务器(包括redis-server和其他sentinel)交互
    sentinelHandleDictOfRedisInstances(sentinel.masters);
}

```

#### sentinelHandleDictOfRedisInstances(sentinel.masters)
递归与所有已知的`redis-server(master)`服务及该服务已知的`redis-server(slave)`和其他`sentinel`; 
具体与每个进程(服务)交互的是`sentinelHandleRedisInstance(ri)`
```c
void sentinelHandleDictOfRedisInstances(dict *instances) {
    dictIterator *di;
    dictEntry *de;
    sentinelRedisInstance *switch_to_promoted = NULL;

    /* There are a number of things we need to perform against every master. */
    di = dictGetIterator(instances);
    while((de = dictNext(di)) != NULL) {
        sentinelRedisInstance *ri = dictGetVal(de);

        sentinelHandleRedisInstance(ri);
        if (ri->flags & SRI_MASTER) {
            sentinelHandleDictOfRedisInstances(ri->slaves);
            sentinelHandleDictOfRedisInstances(ri->sentinels);
            if (ri->failover_state == SENTINEL_FAILOVER_STATE_UPDATE_CONFIG) {
                switch_to_promoted = ri;
            }
        }
    }
    if (switch_to_promoted)
        sentinelFailoverSwitchToPromotedSlave(switch_to_promoted);
    dictReleaseIterator(di);
}
```

#### sentinelHandleRedisInstance

```c
void sentinelHandleRedisInstance(sentinelRedisInstance *ri) {
    // 取得与服务实体的连接
    sentinelReconnectInstance(ri);

    // 发送周期性的问候信息来掌握服务(包括redis-server.master,redis-server.slave,其他sentinel)的信息
    sentinelSendPeriodicCommands(ri);

    // 确认该服务是否已经Down(主观)
    sentinelCheckSubjectivelyDown(ri);
    /* Only masters */
    if (ri->flags & SRI_MASTER) {
        // 判断redis-server.master是否仍有效,及无效时与其他sentinel讨论商量怎么failover
        sentinelCheckObjectivelyDown(ri);
        if (sentinelStartFailoverIfNeeded(ri))
            sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_ASK_FORCED);
        sentinelFailoverStateMachine(ri);
        sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_NO_FLAGS);
    }
}
```

+ 整个`sentinelHandleRedisInstance`函数基于`异步非阻塞`机制;
+ 可以想象整个流程分为若干步骤,执行每个步骤就是把这个步骤要执行的内容丢入redis封装的一层async执行,然后返回;
+ 然后下一次循环执行到同一个地方时判断上一个步骤是否完成.如果没完成,仍然返回,如果已完成,则进行下一个步骤;
+ 这样从看代码的角度,这里时一步一步的执行下来,但在实际运行中,整个`sentinelHandleRedisInstance`可能被调度了很多次,才把整个与服务交换信息并作出决策的逻辑循环走完;

##### sentinelReconnectInstance 与其他服务进程连接(非阻塞)
这个主要利用操作系统本身提供的socket非阻塞接口,完成tcp连接,并记录在册,具体就是非阻塞的`socket`,`bind`,`connect`等.

##### sentinelSendPeriodicCommands 发送周期性的问候和别的服务进程交换信息
```c
// sentinel.c
void sentinelSendPeriodicCommands(sentinelRedisInstance *ri) {
    // ...
    // 向所有已知的非sentinel进程发送info命令,并设置回调sentinelInfoReplyCallback,将返回的info信息记录,这里用到了redis本身自带的事件异步机制,见后文 __redisAsyncCommand()
    if ((ri->flags & SRI_SENTINEL) == 0 &&
        (ri->info_refresh == 0 ||
        (now - ri->info_refresh) > info_period))
    {
        retval = redisAsyncCommand(ri->link->cc,
            sentinelInfoReplyCallback, ri, "%s",
            sentinelInstanceMapCommand(ri,"INFO"));
        if (retval == C_OK) ri->link->pending_commands++;
    }

    // 对所有服务进程发送ping来获取"是否可达"信息,同样通过下面的redisAsyncCommand机制
    if ((now - ri->link->last_pong_time) > ping_period &&
               (now - ri->link->last_ping_time) > ping_period/2) {
        sentinelSendPing(ri);
    }

    /* PUBLISH hello messages to all the three kinds of instances. */
    if ((now - ri->last_pub_time) > SENTINEL_PUBLISH_PERIOD) {
        sentinelSendHello(ri);
    }
}
```

```c
// async.c
static int __redisAsyncCommand(redisAsyncContext *ac, redisCallbackFn *fn, void *privdata, const char *cmd, size_t len) {
    // ...
    // 这个宏即把要发送的命令加入写队列,redis的主循环会把写队列发送给相应的服务进程;
    _EL_ADD_WRITE(ac);
}
```

### failover 机制
+ sentinel群独立于redis服务进程群(包括master和slave)运行,通过周期性的与各个服务进程交换信息来获取每个服务进程的健康状态;
+ 当确认master服务失效时,sentinel群从本来的slave中商量选出新的master服务,同时发送命令要求各个服务遵从新的`master-slave`关系;
+ 运行中的`redis-server`服务接到这些改变`master-slave`关系的命令时,对自身的做事方式做出相应改变,返回必要的信息;
+ 然后整个`redis-server`群和,`redis-sentinel`在新的关系下运行并继续检测这些服务的运行状况以便下一次变动

```c
    // 确认该服务是否已经Down(主观)
    sentinelCheckSubjectivelyDown(ri);
    /* Only masters */
    if (ri->flags & SRI_MASTER) {
        // 判断redis-server.master是否仍有效,及无效时与其他sentinel讨论商量怎么failover
        sentinelCheckObjectivelyDown(ri);
        if (sentinelStartFailoverIfNeeded(ri))
            sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_ASK_FORCED);
        sentinelFailoverStateMachine(ri);
        sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_NO_FLAGS);
    }
```
#### sentinelCheckSubjectivelyDown 
主观判断当前连接的服务进程是否down

#### sentinelCheckObjectivelyDown
查找其他sentinel交互过来的服务进程信息,决定是否当前连接的服务进程down确实down掉了

#### sentinelStartFailoverIfNeeded
判断是否需要启用failover 

#### sentinelAskMasterStateToOtherSentinels
向所有sentinen服务进程发送命令询问大家现在严重的master服务进程时个什么状况

#### sentinelFailoverStateMachine
开始failover


