## 版本
+ Github:redis/redis 
+ 分支:6.0 
+ 版本号:6.0.7

## 简述
- 用独立于`redis`实例的进程`redis-sentinel`来监控`redis`主从集群,发现异常及时failover旧master,选举新master(redis的高可用)
- `redis-snetinel`本身也可以是个集群(`redis-sentinel`本身也高可用)

### setinel实例结构
```c
// src/sentinel.c
// 全局状态结构
struct sentinelState {
    char myid[CONFIG_RUN_ID_SIZE+1]; // sentinel实例ID
    uint64_t current_epoch;         // 信息版本(后续需要用这个版本比对来解决冲突)
    dict *masters;      // “我”所监管的master实例
    int tilt;           // sentinel因为系统时间的不一致而进入的一种保护模式
    int running_scripts;    // ?? TODO
    mstime_t tilt_start_time;       // 开始tilt模式的时间
    mstime_t previous_time;         // 上次执行周期性函数的时间(用来判断是否需要进入tilt模式)
    list *scripts_queue;            // todo
    char *announce_ip;  // 我与其他sentinel交互用的ip
    int announce_port;  // 我与其他sentinel交互用的port
    unsigned long simfailure_flags; // todo
    int deny_scripts_reconfig;// todo
} sentinel;

// “我”所监管的redis实例结构
typedef struct sentinelRedisInstance {
    int flags;      // redis实例的类型和状态(master,sentinel,slave,oshotdown等等)
    char *name;     // 实例名
    char *runid;    // 实例ID
    uint64_t config_epoch;  // 信息版本epoch
    sentinelAddr *addr; /* Master host. */
    instanceLink *link; /* Link to the instance, may be shared for Sentinels. */
    mstime_t last_pub_time;   // 上一次我们向这个实力发送publish(hello)消息的时间
    mstime_t last_hello_time; // 如果这个实例是个sentinel,这个字段保存的就是上次收到这个sentinel实例发hello信息的时间
    mstime_t last_master_down_reply_time; /* Time of last reply to
                                             SENTINEL is-master-down command. */
    mstime_t s_down_since_time; /* Subjectively down since time. */
    mstime_t o_down_since_time; /* Objectively down since time. */
    mstime_t down_after_period; /* Consider it down after that period. */
    mstime_t info_refresh;  // 更新信息的时间
    dict *renamed_commands;     /* Commands renamed in this instance:
                                   Sentinel will use the alternative commands
                                   mapped on this table to send things like
                                   SLAVEOF, CONFING, INFO, ... */

    int role_reported; // 上次这个实例报告的ta的角色
    mstime_t role_reported_time;
    mstime_t slave_conf_change_time; // slave上次更新master信息的时间

    /* Master specific. */
    dict *sentinels;    // 监控了这台master的sentinel们
    dict *slaves;       // 这台master的slave们
    unsigned int quorum;/* Number of sentinels that need to agree on failure. */
    int parallel_syncs; /* How many slaves to reconfigure at same time. */
    char *auth_pass;    /* Password to use for AUTH against master & replica. */
    char *auth_user;    /* Username for ACLs AUTH against master & replica. */

    // slave的信息
    mstime_t master_link_down_time; // 该slave报告的ta与ta的master断连的时间
    int slave_priority; // todo
    mstime_t slave_reconf_sent_time; /* Time at which we sent SLAVE OF <new> */
    struct sentinelRedisInstance *master; // master实例信息
    char *slave_master_host;    // 该slave报告的ta的host
    int slave_master_port;      // port
    int slave_master_link_status; // 与master的连接状态
    unsigned long long slave_repl_offset; // 与master之间的基准数据的偏移量

    /* Failover */
    char *leader;       /* If this is a master instance, this is the runid of
                           the Sentinel that should perform the failover. If
                           this is a Sentinel, this is the runid of the Sentinel
                           that this Sentinel voted as leader. */
    uint64_t leader_epoch; /* Epoch of the 'leader' field. */
    uint64_t failover_epoch; // failover的信息版本epoch
    int failover_state; // failover过程状态 #ref(SENTINEL_FAILOVER_STATE_XXX)
    mstime_t failover_state_change_time; // failover过程状态上次变化的时间
    mstime_t failover_start_time;   /* Last failover attempt start time. */
    mstime_t failover_timeout;      /* Max time to refresh failover state. */
    mstime_t failover_delay_logged; /* For what failover_start_time value we
                                       logged the failover delay. */
    struct sentinelRedisInstance *promoted_slave; /* Promoted slave instance. */
    /* Scripts executed to notify admin or reconfigure clients: when they
     * are set to NULL no script is executed. */
    char *notification_script;
    char *client_reconfig_script;
    sds info; /* cached INFO output */
} sentinelRedisInstance;

// sentinel整体状态
#define SENTINEL_FAILOVER_STATE_NONE 0  /* No failover in progress. */
#define SENTINEL_FAILOVER_STATE_WAIT_START 1  /* Wait for failover_start_time*/
#define SENTINEL_FAILOVER_STATE_SELECT_SLAVE 2 /* Select slave to promote */
#define SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE 3 /* Slave -> Master */
#define SENTINEL_FAILOVER_STATE_WAIT_PROMOTION 4 /* Wait slave to change role */
#define SENTINEL_FAILOVER_STATE_RECONF_SLAVES 5 /* SLAVEOF newmaster */
#define SENTINEL_FAILOVER_STATE_UPDATE_CONFIG 6 /* Monitor promoted slave. */
```


## 初始化
Redis sentinel 和 普通的redis程序共用一套代码(事件循环,事件处理逻辑,存储结果等),程序通过参数辨别自己是`redis`还是`sentinel`;  
```c
// server.c
int main(int argc, char **argv) {
    // ...
    server.sentinel_mode = checkForSentinelMode(argc,argv);
    // ...
    if (server.sentinel_mode) {
        initSentinelConfig();
        // 初始化sentinel #ref(initSentinel)
        initSentinel();
    }
    // ...
    initServer();
    // ...
}
```

### initSentinel sentinel本身的初始化
```c
// src/sentinel.c
void initSentinel(void) {
    unsigned int j;

    // sentinel 虽然与普通redis实例共享很多机制,但本身是一个独立的进程,且不接受一般redis实例中的命令
    dictEmpty(server.commands,NULL);

    // 初始化redis-sentinel的命令(用户客户端查看sentinel情况,手动调整sentinel设置等)
    for (j = 0; j < sizeof(sentinelcmds)/sizeof(sentinelcmds[0]); j++) {
        int retval;
        struct redisCommand *cmd = sentinelcmds+j;

        retval = dictAdd(server.commands, sdsnew(cmd->name), cmd);
        // ...

    }

    // 初始化sentinel结构
    sentinel.current_epoch = 0;
    sentinel.masters = dictCreate(&instancesDictType,NULL);
    sentinel.tilt = 0;
    sentinel.tilt_start_time = 0;
    sentinel.previous_time = mstime();
    sentinel.running_scripts = 0;
    sentinel.scripts_queue = listCreate();
    sentinel.announce_ip = NULL;
    sentinel.announce_port = 0;
    sentinel.simfailure_flags = SENTINEL_SIMFAILURE_NONE;
    sentinel.deny_scripts_reconfig = SENTINEL_DEFAULT_DENY_SCRIPTS_RECONFIG;
    memset(sentinel.myid,0,sizeof(sentinel.myid));
}

```

### sentinel配置文件的初始化
```c
// src/config.c
void loadServerConfigFromString(char *config) {
    // ...

    for (i = 0; i < totlines; i++) {
        // ...        
        else if (!strcasecmp(argv[0],"sentinel")) {
            if (argc != 1) {
                // ...
                // 解析配置文件中与sentinel相关的参数 #ref(sentinelHandleConfiguration)
                err = sentinelHandleConfiguration(argv+1,argc-1);
                if (err) goto loaderr;
            }
        } 
    }

    // ...
}

// src/sentinel.c
// 解析配置文件中与sentinel相关的参数
char *sentinelHandleConfiguration(char **argv, int argc) {
    sentinelRedisInstance *ri;

    if (!strcasecmp(argv[0],"monitor") && argc == 5) {
        /* monitor <name> <host> <port> <quorum> */
        int quorum = atoi(argv[4]);

        if (quorum <= 0) return "Quorum must be 1 or greater.";
        // 根据配置文件的配置创建要监管的redis实例结构(加到了全局变量sentinel.masters中)
        if (createSentinelRedisInstance(argv[1],SRI_MASTER,argv[2],
                                        atoi(argv[3]),quorum,NULL) == NULL)
        {
            // ...
        }
    }
    // ...
    return NULL;
}

```

### 如何告诉监管的redis实例,‘我’正在监管‘你’,‘你’需要订阅我的一些队列消息
### sentinel 的 cron
当程序知道自己是sentinel时,会在这个周期性调度里执行`sentinelTimer`
```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    // ...
    // #ref(sentinelTimer)
    if (server.sentinel_mode) sentinelTimer();
    // ...
}

// sentinel.c
// 需要周期执行的逻辑
void sentinelTimer(void) {
    // 判断是否因为时间的不同步而需要进`tilt`模式 #ref(sentinelCheckTiltCondition)
    sentinelCheckTiltCondition();
    // 与所有已知的master间的交互
    sentinelHandleDictOfRedisInstances(sentinel.masters);


    sentinelRunPendingScripts(); //todo
    sentinelCollectTerminatedScripts(); //todo
    sentinelKillTimedoutScripts(); //todo

    /* We continuously change the frequency of the Redis "timer interrupt"
     * in order to desynchronize every Sentinel from every other.
     * This non-determinism avoids that Sentinels started at the same time
     * exactly continue to stay synchronized asking to be voted at the
     * same time again and again (resulting in nobody likely winning the
     * election because of split brain voting). */
    server.hz = CONFIG_DEFAULT_HZ + rand() % CONFIG_DEFAULT_HZ;
}

```
#### sentinelCheckTiltCondition 判断是否因为时间同步问题而暂时进入`tilt`模式
```c
// src/sentinel.c
void sentinelCheckTiltCondition(void) {
    mstime_t now = mstime();
    mstime_t delta = now - sentinel.previous_time; //算出当前时间与上次执行sentinel周期函数的差值

    // 当差值小于0或者大于某个阈值时,认为不太正常,需要进入tilt模式
    if (delta < 0 || delta > SENTINEL_TILT_TRIGGER) {
        sentinel.tilt = 1;
        sentinel.tilt_start_time = mstime();
    }
    sentinel.previous_time = mstime();
}
```
#### sentinelHandleDictOfRedisInstances(sentinel.masters) 处理已知服务实例的交互
递归与所有已知的`redis-server(master)`服务及该服务已知的`redis-server(slave)`和其他`sentinel`; 
具体与每个进程(服务)交互的是`sentinelHandleRedisInstance(ri)`
```c
// src/sentinel.c
void sentinelHandleDictOfRedisInstances(dict *instances) {
    dictIterator *di;
    dictEntry *de;
    sentinelRedisInstance *switch_to_promoted = NULL;

    // 遍历所有已知服务器实例
    di = dictGetIterator(instances);
    while((de = dictNext(di)) != NULL) {
        sentinelRedisInstance *ri = dictGetVal(de);
        // 每个实例的处理 #ref(sentinelHandleRedisInstance)
        sentinelHandleRedisInstance(ri); 
        if (ri->flags & SRI_MASTER) { // 只有实例为master时才做接下来的处理
            // 处理实例的slave #ref(sentinelHandleDictOfRedisInstances)
            sentinelHandleDictOfRedisInstances(ri->slaves);
            // 处理实例的sentinel #ref(sentinelHandleDictOfRedisInstances)
            sentinelHandleDictOfRedisInstances(ri->sentinels);
            // todo
            if (ri->failover_state == SENTINEL_FAILOVER_STATE_UPDATE_CONFIG) {
                switch_to_promoted = ri;
            }
        }
    }
    if (switch_to_promoted) // todo
        sentinelFailoverSwitchToPromotedSlave(switch_to_promoted);
    dictReleaseIterator(di);
}

void sentinelHandleRedisInstance(sentinelRedisInstance *ri) {
    // 取得与服务实体的连接 #ref(sentinelReconnectInstance)
    sentinelReconnectInstance(ri);

    // 发送周期性的问候信息来掌握服务(包括redis-server.master,redis-server.slave,其他sentinel)的信息 #ref(sentinelSendPeriodicCommands)
    sentinelSendPeriodicCommands(ri);

    // 确认该服务是否已经Down(主观) #ref(sentinelCheckSubjectivelyDown) todo
    sentinelCheckSubjectivelyDown(ri);
    /* Only masters */
    if (ri->flags & SRI_MASTER) {
        // 判断该实例是否已经触及“客观Down” todo
        sentinelCheckObjectivelyDown(ri);
        // 判断是否需要开始failover todo
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
```c
// src/sentinel.c
void sentinelReconnectInstance(sentinelRedisInstance *ri) {
    if (ri->link->disconnected == 0) return;
    if (ri->addr->port == 0) return; /* port == 0 means invalid address. */
    instanceLink *link = ri->link;
    mstime_t now = mstime();

    if (now - ri->link->last_reconn_time < SENTINEL_PING_PERIOD) return;
    ri->link->last_reconn_time = now;

    /* Commands connection. */
    if (link->cc == NULL) {
        link->cc = redisAsyncConnectBind(ri->addr->ip,ri->addr->port,NET_FIRST_BIND_ADDR);
        if (!link->cc->err && server.tls_replication &&
                (instanceLinkNegotiateTLS(link->cc) == C_ERR)) {
            sentinelEvent(LL_DEBUG,"-cmd-link-reconnection",ri,"%@ #Failed to initialize TLS");
            instanceLinkCloseConnection(link,link->cc);
        } else if (link->cc->err) {
            sentinelEvent(LL_DEBUG,"-cmd-link-reconnection",ri,"%@ #%s",
                link->cc->errstr);
            instanceLinkCloseConnection(link,link->cc);
        } else {
            link->pending_commands = 0;
            link->cc_conn_time = mstime();
            link->cc->data = link;
            redisAeAttach(server.el,link->cc);
            redisAsyncSetConnectCallback(link->cc,
                    sentinelLinkEstablishedCallback);
            redisAsyncSetDisconnectCallback(link->cc,
                    sentinelDisconnectCallback);
            sentinelSendAuthIfNeeded(ri,link->cc);
            sentinelSetClientName(ri,link->cc,"cmd");

            /* Send a PING ASAP when reconnecting. */
            sentinelSendPing(ri);
        }
    }
    /* Pub / Sub */
    if ((ri->flags & (SRI_MASTER|SRI_SLAVE)) && link->pc == NULL) {
        link->pc = redisAsyncConnectBind(ri->addr->ip,ri->addr->port,NET_FIRST_BIND_ADDR);
        if (!link->pc->err && server.tls_replication &&
                (instanceLinkNegotiateTLS(link->pc) == C_ERR)) {
            sentinelEvent(LL_DEBUG,"-pubsub-link-reconnection",ri,"%@ #Failed to initialize TLS");
        } else if (link->pc->err) {
            sentinelEvent(LL_DEBUG,"-pubsub-link-reconnection",ri,"%@ #%s",
                link->pc->errstr);
            instanceLinkCloseConnection(link,link->pc);
        } else {
            int retval;

            link->pc_conn_time = mstime();
            link->pc->data = link;
            redisAeAttach(server.el,link->pc);
            redisAsyncSetConnectCallback(link->pc,
                    sentinelLinkEstablishedCallback);
            redisAsyncSetDisconnectCallback(link->pc,
                    sentinelDisconnectCallback);
            sentinelSendAuthIfNeeded(ri,link->pc);
            sentinelSetClientName(ri,link->pc,"pubsub");
            /* Now we subscribe to the Sentinels "Hello" channel. */
            retval = redisAsyncCommand(link->pc,
                sentinelReceiveHelloMessages, ri, "%s %s",
                sentinelInstanceMapCommand(ri,"SUBSCRIBE"),
                SENTINEL_HELLO_CHANNEL);
            if (retval != C_OK) {
                /* If we can't subscribe, the Pub/Sub connection is useless
                 * and we can simply disconnect it and try again. */
                instanceLinkCloseConnection(link,link->pc);
                return;
            }
        }
    }
    /* Clear the disconnected status only if we have both the connections
     * (or just the commands connection if this is a sentinel instance). */
    if (link->cc && (ri->flags & SRI_SENTINEL || link->pc))
        link->disconnected = 0;
}

```

##### sentinelSendPeriodicCommands 发送周期性的问候和别的服务进程交换信息
```c
// sentinel.c
void sentinelSendPeriodicCommands(sentinelRedisInstance *ri) {
    mstime_t now = mstime();
    mstime_t info_period, ping_period;
    int retval;

    // 连接还未建立,立刻返回,等待下一次periodc触发
    if (ri->link->disconnected) return;

    // 有过多还没得到回应的command,也立刻返回,等待下一次periodc触发
    if (ri->link->pending_commands >=
        SENTINEL_MAX_PENDING_COMMANDS * ri->link->refcount) return;

     // 当目标实例时slave,且它的master时客观Down状态时,需要降低信息交互间隔,加快和它的信息交换
    if ((ri->flags & SRI_SLAVE) &&
        ((ri->master->flags & (SRI_O_DOWN|SRI_FAILOVER_IN_PROGRESS)) ||
         (ri->master_link_down_time != 0)))
    {
        info_period = 1000;
    } else {
        info_period = SENTINEL_INFO_PERIOD;
    }

    // 控制ping的间隔时间
    ping_period = ri->down_after_period;
    if (ping_period > SENTINEL_PING_PERIOD) ping_period = SENTINEL_PING_PERIOD;

    // sentinel之间不需要直接交换信息(通过对其他redis节点的某个通道的订阅来得到其他sentinel节点的信息)
    if ((ri->flags & SRI_SENTINEL) == 0 &&
        (ri->info_refresh == 0 ||
        (now - ri->info_refresh) > info_period))
    {
        // 向一般redis节点(master,slave)发送 `info`命令,并设置回调sentinelInfoReplyCallback来获取目标机器的信息(异步命令)
        // #ref(sentinelInfoReplyCallback)
        retval = redisAsyncCommand(ri->link->cc,
            sentinelInfoReplyCallback, ri, "%s",
            sentinelInstanceMapCommand(ri,"INFO"));
        if (retval == C_OK) ri->link->pending_commands++; //统计还未得到回应的命令+1
    }

    // 每种类型的实例都需要周期ping
    if ((now - ri->link->last_pong_time) > ping_period &&
               (now - ri->link->last_ping_time) > ping_period/2) {
        sentinelSendPing(ri);
    }

    // 周期性的publish hello信息
    if ((now - ri->last_pub_time) > SENTINEL_PUBLISH_PERIOD) {
        sentinelSendHello(ri);
    }
}
```
```c
// 发送info命令
        // ...
        retval = redisAsyncCommand(ri->link->cc,
            sentinelInfoReplyCallback, ri, "%s",
            sentinelInstanceMapCommand(ri,"INFO"));
        // ...

// 目标机器收到info命令的处理
void sentinelInfoCommand(client *c) {
    int sections = 0;
    sds info = sdsempty();

    // 服务器,客户端连接,cpu,统计信息
    info_section_from_redis("server");
    info_section_from_redis("clients");
    info_section_from_redis("cpu");
    info_section_from_redis("stats");

    if (defsections || allsections || !strcasecmp(section,"sentinel")) {
        dictIterator *di;
        dictEntry *de;
        int master_id = 0;

        if (sections++) info = sdscat(info,"\r\n");
        // 与sentinel相关的信息
        info = sdscatprintf(info,
            "# Sentinel\r\n"
            "sentinel_masters:%lu\r\n"
            "sentinel_tilt:%d\r\n"
            "sentinel_running_scripts:%d\r\n"
            "sentinel_scripts_queue_length:%ld\r\n"
            "sentinel_simulate_failure_flags:%lu\r\n",
            dictSize(sentinel.masters),
            sentinel.tilt,
            sentinel.running_scripts,
            listLength(sentinel.scripts_queue),
            sentinel.simfailure_flags);

        di = dictGetIterator(sentinel.masters);
        // “我”所知道的master信息,以及在“我”眼里他们的状态(odown,sdown 或者ok)
        while((de = dictNext(di)) != NULL) {
            sentinelRedisInstance *ri = dictGetVal(de);
            char *status = "ok";

            if (ri->flags & SRI_O_DOWN) status = "odown";
            else if (ri->flags & SRI_S_DOWN) status = "sdown";
            info = sdscatprintf(info,
                "master%d:name=%s,status=%s,address=%s:%d,"
                "slaves=%lu,sentinels=%lu\r\n",
                master_id++, ri->name, status,
                ri->addr->ip, ri->addr->port,
                dictSize(ri->slaves),
                dictSize(ri->sentinels)+1);
        }
        dictReleaseIterator(di);
    }

    addReplyBulkSds(c, info);
}
// 异步info命令的回调
void sentinelInfoReplyCallback(redisAsyncContext *c, void *reply, void *privdata) {
    sentinelRedisInstance *ri = privdata;
    instanceLink *link = c->data;
    redisReply *r;

    if (!reply || !link) return;
    link->pending_commands--;
    r = reply;
    // 更新目标服务实例的信息 #ref(sentinelRefreshInstanceInfo)
    if (r->type == REDIS_REPLY_STRING)
        sentinelRefreshInstanceInfo(ri,r->str);
}

// sentinel收到info命令的结果的处理
void sentinelRefreshInstanceInfo(sentinelRedisInstance *ri, const char *info) {
    sds *lines;
    int numlines, j;
    int role = 0;

    // 清空原来的数据,全量更新目标sentinel信息
    sdsfree(ri->info);
    ri->info = sdsnew(info);

    // 一行行解析字符串,将目标服务器的信息存好
    lines = sdssplitlen(info,strlen(info),"\r\n",2,&numlines);
    for (j = 0; j < numlines; j++) {
        sentinelRedisInstance *slave;
        sds l = lines[j];

        // 更新实例id
        if (sdslen(l) >= 47 && !memcmp(l,"run_id:",7)) {
            if (ri->runid == NULL) {
                ri->runid = sdsnewlen(l+7,40);
            } else {
                if (strncmp(ri->runid,l+7,40) != 0) {
                    sentinelEvent(LL_NOTICE,"+reboot",ri,"%@");
                    sdsfree(ri->runid);
                    ri->runid = sdsnewlen(l+7,40);
                }
            }
        }

        // 当前实例时master时,更新他传过来的slave信息
        if ((ri->flags & SRI_MASTER) &&
            sdslen(l) >= 7 &&
            !memcmp(l,"slave",5) && isdigit(l[5]))
        {
            // ...
            // 更新当前master的slave信息
            // ...
            // 如果一个slave在本地没有信息,则新建个instance保存这个新slave的信息
            if (sentinelRedisInstanceLookupSlave(ri,ip,atoi(port)) == NULL) {
                if ((slave = createSentinelRedisInstance(NULL,SRI_SLAVE,ip,
                            atoi(port), ri->quorum, ri)) != NULL)
                {
                    // todo
                    sentinelEvent(LL_NOTICE,"+slave",slave,"%@");
                    sentinelFlushConfig();
                }
            }
        }

        // 这个消息只有slave会带,表示该slave与“ta”的master断连的时间
        if (sdslen(l) >= 32 &&
            !memcmp(l,"master_link_down_since_seconds",30))
        {
            ri->master_link_down_time = strtoll(l+31,NULL,10)*1000;
        }

        // 该实例的角色(master or slave)
        if (sdslen(l) >= 11 && !memcmp(l,"role:master",11)) role = SRI_MASTER;
        else if (sdslen(l) >= 10 && !memcmp(l,"role:slave",10)) role = SRI_SLAVE;

        if (role == SRI_SLAVE) {
            // 实例时slave时更新ta的master信息
            if (sdslen(l) >= 12 && !memcmp(l,"master_host:",12)) {
                if (ri->slave_master_host == NULL ||
                    strcasecmp(l+12,ri->slave_master_host))
                {
                    sdsfree(ri->slave_master_host);
                    ri->slave_master_host = sdsnew(l+12); //更新host信息
                    ri->slave_conf_change_time = mstime();
                }
            }

            if (sdslen(l) >= 12 && !memcmp(l,"master_port:",12)) {
                int slave_master_port = atoi(l+12);

                if (ri->slave_master_port != slave_master_port) {
                    ri->slave_master_port = slave_master_port; // 更新port信息
                    ri->slave_conf_change_time = mstime();
                }
            }

            /* master_link_status:<status> */
            if (sdslen(l) >= 19 && !memcmp(l,"master_link_status:",19)) { // 更新与master的连接状态是信息
                ri->slave_master_link_status =
                    (strcasecmp(l+19,"up") == 0) ?
                    SENTINEL_MASTER_LINK_STATUS_UP :
                    SENTINEL_MASTER_LINK_STATUS_DOWN;
            }

            if (sdslen(l) >= 15 && !memcmp(l,"slave_priority:",15))
                ri->slave_priority = atoi(l+15); // TODO 这个priority的作用

            if (sdslen(l) >= 18 && !memcmp(l,"slave_repl_offset:",18))
                ri->slave_repl_offset = strtoull(l+18,NULL,10); //更新repl_offset
        }
    }
    ri->info_refresh = mstime(); //更新信息时间
    sdsfreesplitres(lines,numlines);

    /* ---------------------------- Acting half -----------------------------
     * Some things will not happen if sentinel.tilt is true, but some will
     * still be processed. */

    /* Remember when the role changed. */
    if (role != ri->role_reported) {
        ri->role_reported_time = mstime();
        ri->role_reported = role;
        if (role == SRI_SLAVE) ri->slave_conf_change_time = mstime();
        // 如果实例宣称的角色与ta上次宣称的角色不同时,不要引起重视,发送sentinelEvent // TODO
        sentinelEvent(LL_VERBOSE,
            ((ri->flags & (SRI_MASTER|SRI_SLAVE)) == role) ?
            "+role-change" : "-role-change",
            ri, "%@ new reported role is %s",
            role == SRI_MASTER ? "master" : "slave",
            ri->flags & SRI_MASTER ? "master" : "slave");
    }

    // 当处在tilt(系统现在不是很正常)模式时,返回
    if (sentinel.tilt) return;

    // 当一个slave变成了master时
    if ((ri->flags & SRI_SLAVE) && role == SRI_MASTER) {
        // 正常的slave 提升 为master 流程(各种信息已经知道了)
        if ((ri->flags & SRI_PROMOTED) &&
            (ri->master->flags & SRI_FAILOVER_IN_PROGRESS) &&
            (ri->master->failover_state ==
                SENTINEL_FAILOVER_STATE_WAIT_PROMOTION))
        {
            // 更新原master的信息
            ri->master->config_epoch = ri->master->failover_epoch;
            ri->master->failover_state = SENTINEL_FAILOVER_STATE_RECONF_SLAVES;
            ri->master->failover_state_change_time = mstime();
            sentinelFlushConfig();
            // 发布slave被提升的消息
            sentinelEvent(LL_WARNING,"+promoted-slave",ri,"%@");

            // simfailure todo??
            if (sentinel.simfailure_flags &
                SENTINEL_SIMFAILURE_CRASH_AFTER_PROMOTION)
                sentinelSimFailureCrash();
            // todo
            sentinelEvent(LL_WARNING,"+failover-state-reconf-slaves",
                ri->master,"%@");
            sentinelCallClientReconfScript(ri->master,SENTINEL_LEADER,
                "start",ri->master->addr,ri->addr);
            sentinelForceHelloUpdateForMaster(ri->master);
        } else {
            // 还存在一些疑点的情况,比如对原master的情况不太确定,需要确认状态,解决疑惑
            // TODO ??
            mstime_t wait_time = SENTINEL_PUBLISH_PERIOD*4;

            if (!(ri->flags & SRI_PROMOTED) &&
                 sentinelMasterLooksSane(ri->master) &&
                 sentinelRedisInstanceNoDownFor(ri,wait_time) &&
                 mstime() - ri->role_reported_time > wait_time)
            {
                int retval = sentinelSendSlaveOf(ri,
                        ri->master->addr->ip,
                        ri->master->addr->port);
                if (retval == C_OK)
                    sentinelEvent(LL_NOTICE,"+convert-to-slave",ri,"%@");
            }
        }
    }

    // 当一个slave实例宣称的master变化时
    if ((ri->flags & SRI_SLAVE) &&
        role == SRI_SLAVE &&
        (ri->slave_master_port != ri->master->addr->port ||
         strcasecmp(ri->slave_master_host,ri->master->addr->ip)))
    {
        mstime_t wait_time = ri->master->failover_timeout;

        /* Make sure the master is sane before reconfiguring this instance
         * into a slave. */
        if (sentinelMasterLooksSane(ri->master) &&
            sentinelRedisInstanceNoDownFor(ri,wait_time) &&
            mstime() - ri->slave_conf_change_time > wait_time)
        {
            // todo ?? 为什么发送的slaveof 的目标master依然是原master的信息
            int retval = sentinelSendSlaveOf(ri,
                    ri->master->addr->ip,
                    ri->master->addr->port);
            if (retval == C_OK)
                sentinelEvent(LL_NOTICE,"+fix-slave-config",ri,"%@");
        }
    }

    /* Detect if the slave that is in the process of being reconfigured
     * changed state. */
     // TODO ??
    if ((ri->flags & SRI_SLAVE) && role == SRI_SLAVE &&
        (ri->flags & (SRI_RECONF_SENT|SRI_RECONF_INPROG)))
    {
        /* SRI_RECONF_SENT -> SRI_RECONF_INPROG. */
        if ((ri->flags & SRI_RECONF_SENT) &&
            ri->slave_master_host &&
            strcmp(ri->slave_master_host,
                    ri->master->promoted_slave->addr->ip) == 0 &&
            ri->slave_master_port == ri->master->promoted_slave->addr->port)
        {
            ri->flags &= ~SRI_RECONF_SENT;
            ri->flags |= SRI_RECONF_INPROG;
            sentinelEvent(LL_NOTICE,"+slave-reconf-inprog",ri,"%@");
        }

        /* SRI_RECONF_INPROG -> SRI_RECONF_DONE */
        if ((ri->flags & SRI_RECONF_INPROG) &&
            ri->slave_master_link_status == SENTINEL_MASTER_LINK_STATUS_UP)
        {
            ri->flags &= ~SRI_RECONF_INPROG;
            ri->flags |= SRI_RECONF_DONE;
            sentinelEvent(LL_NOTICE,"+slave-reconf-done",ri,"%@");
        }
    }
}

```
```c
// ping目标机器
int sentinelSendPing(sentinelRedisInstance *ri) {
    // ping 和 pong 也是异步的,发送ping后,设置回调,sentinelPingReplyCallback,等待pong #ref(sentinelPingReplyCallback)
    int retval = redisAsyncCommand(ri->link->cc,
        sentinelPingReplyCallback, ri, "%s",
        sentinelInstanceMapCommand(ri,"PING"));
    if (retval == C_OK) {
        // 与ping相关的状态信息更新
        ri->link->pending_commands++;
        ri->link->last_ping_time = mstime();
        if (ri->link->act_ping_time == 0)
            ri->link->act_ping_time = ri->link->last_ping_time;
        return 1;
    } else {
        return 0;
    }
}
// 收到pong回应更新一些时间信息
void sentinelPingReplyCallback(redisAsyncContext *c, void *reply, void *privdata) {
    sentinelRedisInstance *ri = privdata;
    instanceLink *link = c->data;
    redisReply *r;

    if (!reply || !link) return;
    link->pending_commands--;
    r = reply;

    if (r->type == REDIS_REPLY_STATUS ||
        r->type == REDIS_REPLY_ERROR) {
        if (strncmp(r->str,"PONG",4) == 0 ||
            strncmp(r->str,"LOADING",7) == 0 ||
            strncmp(r->str,"MASTERDOWN",10) == 0)
        {
            link->last_avail_time = mstime();
            link->act_ping_time = 0; /* Flag the pong as received. */
        } else {
            //...
        }
    }
    link->last_pong_time = mstime();
}
```
```c
// 向实例发送异步`Publish`消息
int sentinelSendHello(sentinelRedisInstance *ri) {
    char ip[NET_IP_STR_LEN];
    char payload[NET_IP_STR_LEN+1024];
    int retval;
    char *announce_ip;
    int announce_port;
    // 通过已知信息找到这个实例的master信息
    sentinelRedisInstance *master = (ri->flags & SRI_MASTER) ? ri : ri->master;
    sentinelAddr *master_addr = sentinelGetCurrentMasterAddress(master);

    if (ri->link->disconnected) return C_ERR;

    // 准备当前sentinel自己的信息
    if (sentinel.announce_ip) {
        announce_ip = sentinel.announce_ip;
    } else {
        if (anetSockName(ri->link->cc->c.fd,ip,sizeof(ip),NULL) == -1)
            return C_ERR;
        announce_ip = ip;
    }
    if (sentinel.announce_port) announce_port = sentinel.announce_port;
    else if (server.tls_replication && server.tls_port) announce_port = server.tls_port;
    else announce_port = server.port;

    // 向目标实例发送异步的publish消息,并设置回调 #ref(sentinelPublishReplyCallback)
    snprintf(payload,sizeof(payload),
        "%s,%d,%s,%llu," /* Info about this sentinel. */
        "%s,%s,%d,%llu", /* Info about current master. */
        announce_ip, announce_port, sentinel.myid,
        (unsigned long long) sentinel.current_epoch,
        /* --- */
        master->name,master_addr->ip,master_addr->port,
        (unsigned long long) master->config_epoch);
    retval = redisAsyncCommand(ri->link->cc,
        sentinelPublishReplyCallback, ri, "%s %s %s",
        sentinelInstanceMapCommand(ri,"PUBLISH"),
        SENTINEL_HELLO_CHANNEL,payload); // SENTINEL_HELLO_CHANNEL是一个内定的Hello channel, payload就是前面组装好的master信息和sentinel信息
    if (retval != C_OK) return C_ERR;
    ri->link->pending_commands++;
    return C_OK;
}

// 实例收到`Publish`消息的处理
void sentinelPublishCommand(client *c) {
    // 处理消息
    sentinelProcessHelloMessage(c->argv[2]->ptr,sdslen(c->argv[2]->ptr));
    // 回复
    addReplyLongLong(c,1);
}
void sentinelProcessHelloMessage(char *hello, int hello_len) {
    /* Format is composed of 8 tokens:
     * 0=ip,1=port,2=runid,3=current_epoch,4=master_name,
     * 5=master_ip,6=master_port,7=master_config_epoch. */
    int numtokens, port, removed, master_port;
    uint64_t current_epoch, master_config_epoch;
    char **token = sdssplitlen(hello, hello_len, ",", 1, &numtokens);
    sentinelRedisInstance *si, *master;

    if (numtokens == 8) {
        master = sentinelGetMasterByName(token[4]);
        if (!master) goto cleanup; /* Unknown master, skip the message. */

        // 根据发来的信息找到‘我’是否曾有这个sentinel实例的信息
        port = atoi(token[1]);
        master_port = atoi(token[6]);
        si = getSentinelRedisInstanceByAddrAndRunID(
                        master->sentinels,token[0],port,token[2]);
        current_epoch = strtoull(token[3],NULL,10); // 发信息的sentinel的epoch版本
        master_config_epoch = strtoull(token[7],NULL,10); // 发信息告诉‘我’,我的master是谁谁谁,这条信息的epoch版本

        if (!si) {
            // 本地找不到这个ip这个id的sentinel信息时,需要清空这个id的以往信息 TODO
            removed = removeMatchingSentinelFromMaster(master,token[2]);
            if (removed) {
                // TODO,即时传播这个变动
                sentinelEvent(LL_NOTICE,"+sentinel-address-switch",master,
                    "%@ ip %s port %d for %s", token[0],port,token[2]);
            } else {
                /* Check if there is another Sentinel with the same address this
                 * new one is reporting. What we do if this happens is to set its
                 * port to 0, to signal the address is invalid. We'll update it
                 * later if we get an HELLO message. */
                sentinelRedisInstance *other =
                    getSentinelRedisInstanceByAddrAndRunID(
                        master->sentinels, token[0],port,NULL);
                if (other) {
                    sentinelEvent(LL_NOTICE,"+sentinel-invalid-addr",other,"%@");
                    other->addr->port = 0; /* It means: invalid address. */
                    sentinelUpdateSentinelAddressInAllMasters(other);
                }
            }

            // 根据信息创建一个新的sentinel信息结构
            si = createSentinelRedisInstance(token[2],SRI_SENTINEL,
                            token[0],port,master->quorum,master);

            if (si) {
                if (!removed) sentinelEvent(LL_NOTICE,"+sentinel",si,"%@");
                /* The runid is NULL after a new instance creation and
                 * for Sentinels we don't have a later chance to fill it,
                 * so do it now. */
                si->runid = sdsnew(token[2]);
                sentinelTryConnectionSharing(si);
                if (removed) sentinelUpdateSentinelAddressInAllMasters(si);
                sentinelFlushConfig();
            }
        }

        // 如果发信息过来的sentinel 的epoch 大于我本地已知该sentinel信息的epoch,需要更新信息和epoch数值
        if (current_epoch > sentinel.current_epoch) {
            sentinel.current_epoch = current_epoch;
            sentinelFlushConfig();
            // todo sentinelEvent
            sentinelEvent(LL_WARNING,"+new-epoch",master,"%llu",
                (unsigned long long) sentinel.current_epoch);
        }

        // 如果发过来的master_config_epoch 大于 我本地知道的这个master 的 config_epoch,更新关于这个master的信息
        if (si && master->config_epoch < master_config_epoch) {
            master->config_epoch = master_config_epoch;
            if (master_port != master->addr->port ||
                strcmp(master->addr->ip, token[5]))
            {
                sentinelAddr *old_addr;

                sentinelEvent(LL_WARNING,"+config-update-from",si,"%@");
                sentinelEvent(LL_WARNING,"+switch-master",
                    master,"%s %s %d %s %d",
                    master->name,
                    master->addr->ip, master->addr->port,
                    token[5], master_port);

                old_addr = dupSentinelAddr(master->addr);
                sentinelResetMasterAndChangeAddress(master, token[5], master_port);
                sentinelCallClientReconfScript(master,
                    SENTINEL_OBSERVER,"start",
                    old_addr,master->addr);
                releaseSentinelAddr(old_addr);
            }
        }

        // 更新sentinel的hello统计时间信息
        if (si) si->last_hello_time = mstime();
    }

cleanup:
    sdsfreesplitres(token,numtokens);
}

// sentinelPublishReplyCallback publish消息的回调
void sentinelPublishReplyCallback(redisAsyncContext *c, void *reply, void *privdata) {
    sentinelRedisInstance *ri = privdata;
    instanceLink *link = c->data;
    redisReply *r;

    if (!reply || !link) return;
    link->pending_commands--;
    r = reply;
    // 更新pub时间信息
    if (r->type != REDIS_REPLY_ERROR)
        ri->last_pub_time = mstime();
}
```
### sentinelEvent 向所有订阅了自己的服务发送事件消息
```c
// level 参数
#define LL_DEBUG 0
#define LL_VERBOSE 1
#define LL_NOTICE 2
#define LL_WARNING 3
#define LL_RAW (1<<10) /* Modifier to log without timestamp */

// type参数是消息类型 
// ri参数 这个事件是针对哪个服务实例的(target)
// fmt...参数 具体消息内容

void sentinelEvent(int level, char *type, sentinelRedisInstance *ri,
                   const char *fmt, ...) {
    va_list ap;
    char msg[LOG_MAX_LEN];
    robj *channel, *payload;

    // 具体消息中以 %@ 开头时发布的消息需该target及其master信息
    if (fmt[0] == '%' && fmt[1] == '@') {
        sentinelRedisInstance *master = (ri->flags & SRI_MASTER) ?
                                         NULL : ri->master;

        if (master) {
            snprintf(msg, sizeof(msg), "%s %s %s %d @ %s %s %d",
                sentinelRedisInstanceTypeStr(ri),
                ri->name, ri->addr->ip, ri->addr->port,
                master->name, master->addr->ip, master->addr->port);
        } else {
            snprintf(msg, sizeof(msg), "%s %s %s %d",
                sentinelRedisInstanceTypeStr(ri),
                ri->name, ri->addr->ip, ri->addr->port);
        }
        fmt += 2;
    } else {
        msg[0] = '\0';
    }

    // 消息格式化
    if (fmt[0] != '\0') {
        va_start(ap, fmt);
        vsnprintf(msg+strlen(msg), sizeof(msg)-strlen(msg), fmt, ap);
        va_end(ap);
    }

    // 发布消息到参数指定的channel
    if (level != LL_DEBUG) {
        channel = createStringObject(type,strlen(type));
        payload = createStringObject(msg,strlen(msg));
        // todo 
        pubsubPublishMessage(channel,payload);
        decrRefCount(channel);
        decrRefCount(payload);
    }

    // todo
    if (level == LL_WARNING && ri != NULL) {
        sentinelRedisInstance *master = (ri->flags & SRI_MASTER) ?
                                         ri : ri->master;
        if (master && master->notification_script) {
            sentinelScheduleScriptExecution(master->notification_script,
                type,msg,NULL);
        }
    }
}

```

### 订阅sentinel事件消息

### sentinel事件消息处理


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


