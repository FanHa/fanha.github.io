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

// “我”所知道的redis实例结构
typedef struct sentinelRedisInstance {
    int flags;      // redis实例的类型和状态(master,sentinel,slave,oshotdown等等)
    char *name;     // 实例名
    char *runid;    // 实例ID
    uint64_t config_epoch;  // 信息版本epoch
    sentinelAddr *addr; /* Master host. */
    instanceLink *link; /* Link to the instance, may be shared for Sentinels. */
    mstime_t last_pub_time;   // 上一次我们向这个实力发送publish(hello)消息的时间
    mstime_t last_hello_time; // 如果这个实例是个sentinel,这个字段保存的就是上次收到这个sentinel实例发hello信息的时间
    mstime_t last_master_down_reply_time; // sentinel询问其他sentinel关于一个master是否down的回应时间
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

    // Failover相关
    char *leader;       // 该sentinel实例选的failover操盘sentinelid(由这个sentinel去执行后续的failover实际命令收发) todo 如果实例的master的情况
    uint64_t leader_epoch; // 该sentinel实例选的leader的epoch
    uint64_t failover_epoch; // failover的信息版本epoch
    int failover_state; // failover过程状态 #ref(SENTINEL_FAILOVER_STATE_XXX)
    mstime_t failover_state_change_time; // failover过程状态上次变化的时间
    mstime_t failover_start_time;   // 上一次尝试对这个实例failover的时间
    mstime_t failover_timeout;      // 设置的failover超时时间
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


## 初始化 与 实例间的信息交互
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
        // 并不是所有的服务器实例都是走配置流程加入到监管表中,大部分还是通过向目标实例发送‘info’命令,目标返回‘ta‘所知道的信息
        // 或是订阅目标实例的信息通道,目标实例会在有新消息时通过通道把信息传达给sentinel
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


### sentinel 通过周期性cron和其他实例交换信息
当程序知道自己是sentinel时,会在这个周期性调度里执行`sentinelTimer`,通过cron分批获取了整个集群的信息
- `sentinel1`与配置好的`redis实例1`建立连接,并订阅`hello`通道;
- 此时`sentinel1` 与 配置好的`redis实例1`知道对方存在;
- `sentinel2` 与同一个`redis实例1`建立连接,同样订阅`hello`通道;
- 此时`redis实例1` 与 `sentinel2` 互相知道对方存在;
- sentinel之间的感知有两种方式
    - `sentinel1` 和 `sentinel2` 会周期性的向`redis实例1` 发送`info`命令,通过返回的`redis实例1`视角的信息感知其他sentinel存在;
    - `sentinel1`, `sentinel2`与`redis实例1` 建立连接时会通过向`redis实例1`的`publish`命令向`hello`通道发送信息,这样就能通过先前订阅这个通道的回调得到对方的信息;
- 其他master,slave信息也都是通过`info`命令和`hello`通道得到(只要有一个交集实例,就能通过这个交集实例把信息传播开来)

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
    // ...
    // 与所有已知的master间的交互
    sentinelHandleDictOfRedisInstances(sentinel.masters);


    sentinelRunPendingScripts(); //todo
    sentinelCollectTerminatedScripts(); //todo
    sentinelKillTimedoutScripts(); //todo
}

```
#### sentinelHandleDictOfRedisInstances 处理已知服务实例的交互
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
        if (ri->flags & SRI_MASTER) { // 实例为master时还需要对该master的sentine和slave作消息交流
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

##### sentinelReconnectInstance 与其他服务建立进程连接(非阻塞)
- 需要建立两个连接,一个用来发送命令,一个用来订阅消息
    - 命令连接建立:这个主要利用操作系统本身提供的socket非阻塞接口,完成tcp连接,并记录在册,具体就是非阻塞的`socket`,`bind`,`connect`等.
    - 订阅链接建立:订阅目标机器的一个channel(SENTINEL_HELLO_CHANNEL),设置这个channel有消息时的回调,其他实例发布到这个channel的信息就能被回调处理了
```c
// src/sentinel.c
void sentinelReconnectInstance(sentinelRedisInstance *ri) {
    // 用来发送命令的连接
    if (link->cc == NULL) {
        // 普通的用来发送命令的连接(ping,info啥的)
    }
    // 订阅channel的连接
    if ((ri->flags & (SRI_MASTER|SRI_SLAVE)) && link->pc == NULL) { //只需要订阅master和slave的消息,sentinel的信息是通过master和slave间接知道的
        link->pc = redisAsyncConnectBind(ri->addr->ip,ri->addr->port,NET_FIRST_BIND_ADDR);
        // ...
        {
            int retval;
            // ...
            // 订阅SENTINEL_HELLO_CHANNEL的消息,并设置回调 #ref(`sentinelReceiveHelloMessages`),后续别的机器通过目标实例发布的消息,可以通过此channel收到并触发回调
            retval = redisAsyncCommand(link->pc,
                sentinelReceiveHelloMessages, ri, "%s %s",
                sentinelInstanceMapCommand(ri,"SUBSCRIBE"),
                SENTINEL_HELLO_CHANNEL);
            // ...
        }
    }
}

```
```c
// 订阅了实例的channel后收到其他服务实例发布的信息的回调处理
void sentinelReceiveHelloMessages(redisAsyncContext *c, void *reply, void *privdata) {
    // ...
    sentinelProcessHelloMessage(r->element[2]->str, r->element[2]->len);
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
        master_config_epoch = strtoull(token[7],NULL,10); // 发信息告诉‘我’,'我'的master是谁谁谁,这条信息的epoch版本

        if (!si) {  // 本地没有这个sentinel信息时
            
            // 根据信息创建一个新的sentinel信息结构
            si = createSentinelRedisInstance(token[2],SRI_SENTINEL,
                            token[0],port,master->quorum,master);
            // ...
        }

        // 如果发信息过来的sentinel 的epoch 大于我本地已知该sentinel信息的epoch,需要更新信息和epoch数值
        if (current_epoch > sentinel.current_epoch) {
            sentinel.current_epoch = current_epoch;
            sentinelFlushConfig();
        }

        // 如果发过来的master_config_epoch 大于 我本地知道的这个master 的 config_epoch,更新关于这个master的信息
        if (si && master->config_epoch < master_config_epoch) {
            master->config_epoch = master_config_epoch;
            if (master_port != master->addr->port ||
                strcmp(master->addr->ip, token[5]))
            {
                sentinelAddr *old_addr;

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
// ...
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

     // 当目标实例是slave,且它的master是客观Down状态时,需要降低信息交互间隔,加快和它的信息交换
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

    // sentinel之间不需要直接交换信息(通过对其他redis实例发送‘info’命令可以返回该redis实例已知的其他实例节点的信息,包括sentinel,master,slave)
    if ((ri->flags & SRI_SENTINEL) == 0 &&
        (ri->info_refresh == 0 ||
        (now - ri->info_refresh) > info_period))
    {
        // 向一般redis节点(master,slave)发送 `info`命令,并设置回调sentinelInfoReplyCallback来获取目标机器的信息(异步命令)
        // 目标机器会返回它已知的sentinel,master,slave的信息,‘我’就可以利用这些信息更新自己的集群信息表了
        // #ref(sentinelInfoReplyCallback)
        retval = redisAsyncCommand(ri->link->cc,
            sentinelInfoReplyCallback, ri, "%s",
            sentinelInstanceMapCommand(ri,"INFO"));
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

    // 清空原来的数据,全量更新目标实例信息
    sdsfree(ri->info);
    ri->info = sdsnew(info);

    // 一行行解析字符串,将目标服务器的信息存好
    lines = sdssplitlen(info,strlen(info),"\r\n",2,&numlines);
    for (j = 0; j < numlines; j++) {
        sentinelRedisInstance *slave;
        sds l = lines[j];

        // 当实例是master时,更新‘ta’的slave信息
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

    // 报告的角色与以前不同时需要记录
    if (role != ri->role_reported) {
        ri->role_reported_time = mstime();
        ri->role_reported = role;
        if (role == SRI_SLAVE) ri->slave_conf_change_time = mstime();
    }

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

            // todo
            sentinelCallClientReconfScript(ri->master,SENTINEL_LEADER,
                "start",ri->master->addr,ri->addr);
            sentinelForceHelloUpdateForMaster(ri->master);
        } else {
            // 还存在一些疑点的情况,比如对原master的情况不太确定,需要确认状态,解决疑惑
            // TODO ??
            mstime_t wait_time = SENTINEL_PUBLISH_PERIOD*4;

            if (!(ri->flags & SRI_PROMOTED) &&
                 sentinelMasterLooksSane(ri->master) && // 从‘我’的视角,原master看上去正常
                 sentinelRedisInstanceNoDownFor(ri,wait_time) && // 没有其他实例报告这个实例 down了
                 mstime() - ri->role_reported_time > wait_time)
            {
                // 向他发送slave of 命令,让他继续当他以前master的slave
                int retval = sentinelSendSlaveOf(ri,
                        ri->master->addr->ip,
                        ri->master->addr->port);
            }
        }
    }

    // 当一个slave实例报告‘ta’的master变化时
    if ((ri->flags & SRI_SLAVE) &&
        role == SRI_SLAVE &&
        (ri->slave_master_port != ri->master->addr->port ||
         strcasecmp(ri->slave_master_host,ri->master->addr->ip)))
    {
        mstime_t wait_time = ri->master->failover_timeout;

        if (sentinelMasterLooksSane(ri->master) && //确认原master正常
            sentinelRedisInstanceNoDownFor(ri,wait_time) && // 确认这个实例也正常
            mstime() - ri->slave_conf_change_time > wait_time)
        {
            // 向他发送slave of 命令,让他继续当他以前master的slave
            int retval = sentinelSendSlaveOf(ri,
                    ri->master->addr->ip,
                    ri->master->addr->port);

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

    // 向实例的的SENTINEL_HELLO_CHANNEL发布hello消息,告诉订阅了这个服务实例的其他sentinel,‘我’的存在,以及‘我’认为的这个实例的master信息
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

```

## failover
+ sentinel实例独立于redis服务集群(包括master和slave)运行,通过周期性的与各个服务实例交换信息来获取每个服务实例的健康状态;
+ 当确认master服务失效时,sentinel群从本来的slave中商量选出新的master服务,同时发送命令要求各个服务遵从新的`master-slave`关系;
+ 运行中的`redis-server`服务接到这些改变`master-slave`关系的命令时,对自身的做事方式做出相应改变,返回必要的信息;
+ 然后整个`redis-server`群和,`redis-sentinel`在新的关系下运行并继续检测这些服务的运行状况以便下一次变动

### 周期性执行程序里有收集信息确认实例是否健康
```c
void sentinelHandleRedisInstance(sentinelRedisInstance *ri) {

    // 确认该实例是否已经SDown(主观) #ref(sentinelCheckSubjectivelyDown)
    sentinelCheckSubjectivelyDown(ri);
    /* Only masters */
    if (ri->flags & SRI_MASTER) {
        // 如果实例是master,需要判断他是否ODown(客观) #ref(sentinelCheckObjectivelyDown)
        sentinelCheckObjectivelyDown(ri);
        // 判断是否需要启动failover
        if (sentinelStartFailoverIfNeeded(ri))
            // 确认要开始failover时,向其他sentinel广播这个master的信息
            sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_ASK_FORCED);
        // todo
        sentinelFailoverStateMachine(ri);
        // todo
        sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_NO_FLAGS);
    }
}
```
#### sentinelCheckSubjectivelyDown 
主观判断当前连接的服务进程是否down
```c
void sentinelCheckSubjectivelyDown(sentinelRedisInstance *ri) {
    mstime_t elapsed = 0;

    if (ri->link->act_ping_time)
        elapsed = mstime() - ri->link->act_ping_time;
    else if (ri->link->disconnected)
        elapsed = mstime() - ri->link->last_avail_time;

    // 判断给该实例发命令的连接是否仍活跃
    if (ri->link->cc &&
        (mstime() - ri->link->cc_conn_time) >
        SENTINEL_MIN_LINK_RECONNECT_PERIOD &&
        ri->link->act_ping_time != 0 && 
        (mstime() - ri->link->act_ping_time) > (ri->down_after_period/2) &&
        (mstime() - ri->link->last_pong_time) > (ri->down_after_period/2))
    {
        instanceLinkCloseConnection(ri->link,ri->link->cc);
    }

    // 判断用来订阅该实例的消息连接是否仍活跃
    if (ri->link->pc &&
        (mstime() - ri->link->pc_conn_time) >
         SENTINEL_MIN_LINK_RECONNECT_PERIOD &&
        (mstime() - ri->link->pc_last_activity) > (SENTINEL_PUBLISH_PERIOD*3))
    {
        instanceLinkCloseConnection(ri->link,ri->link->pc);
    }

    // 很久没有过联系了,将该实例标记为sdown
    if (elapsed > ri->down_after_period ||
        (ri->flags & SRI_MASTER &&
         ri->role_reported == SRI_SLAVE &&
         mstime() - ri->role_reported_time >
          (ri->down_after_period+SENTINEL_INFO_PERIOD*2)))
    {
       // 标记目标机器为sdown(主观)
        if ((ri->flags & SRI_S_DOWN) == 0) {
            ri->s_down_since_time = mstime();
            ri->flags |= SRI_S_DOWN;
        }
    }
}
```

#### sentinelCheckObjectivelyDown
查找其他sentinel交互过来的服务进程信息,决定是否当前连接的服务进程down确实down掉了
```c
void sentinelCheckObjectivelyDown(sentinelRedisInstance *master) {
    dictIterator *di;
    dictEntry *de;
    unsigned int quorum = 0, odown = 0;
    //如果从‘我’的视角,该实例已经SDown了,需要遍历一下其他同样监管这个实例的sentinel对该实例的看法
    // 这些“看法”信息从发送的sentinelAskMasterStateToOtherSentinels获得
    if (master->flags & SRI_S_DOWN) { 
        quorum = 1; 
        di = dictGetIterator(master->sentinels);
        // 遍历所有监管这个实例的sentinel信息,如果也认为该实例为SRI_MASTER_DOWN,则票数+1
        while((de = dictNext(di)) != NULL) {
            sentinelRedisInstance *ri = dictGetVal(de);

            if (ri->flags & SRI_MASTER_DOWN) quorum++;
        }
        dictReleaseIterator(di);
        if (quorum >= master->quorum) odown = 1; // 票数满足要求,设置Odown
    }

    if (odown) {
        if ((master->flags & SRI_O_DOWN) == 0) {
            // 设置flag为Odown
            master->flags |= SRI_O_DOWN;
            master->o_down_since_time = mstime();
        }
    } else {
        if (master->flags & SRI_O_DOWN) {
            // 不够票时,或者票数减少到不够时,需要解除该实例的odown标记
            master->flags &= ~SRI_O_DOWN;
        }
    }
}
```

#### sentinelStartFailoverIfNeeded
判断是否需要启用failover 
```c
// 判断是否需要启动failover
int sentinelStartFailoverIfNeeded(sentinelRedisInstance *master) {
    // 判断flag 是否为ODown,不是则返回false
    if (!(master->flags & SRI_O_DOWN)) return 0;

    // 判断是否已经在failover进行中,是则返回false
    if (master->flags & SRI_FAILOVER_IN_PROGRESS) return 0;

    // 距离上次启动failover刚刚过去不久 
    if (mstime() - master->failover_start_time <
        master->failover_timeout*2)
    {
        // ...
        return 0;
    }
    // 所有条件都满足,开始failover #ref(sentinelStartFailover) todo
    sentinelStartFailover(master);
    return 1;
}
// 开始failover,这个操作不需要再经过其他sentinel的同意,只需要‘我’这边得到的信息满足就开始
void sentinelStartFailover(sentinelRedisInstance *master) {

    master->failover_state = SENTINEL_FAILOVER_STATE_WAIT_START; //设置该实例的failover状态
    master->flags |= SRI_FAILOVER_IN_PROGRESS; // 设置该实例的flag为正在failover
    master->failover_epoch = ++sentinel.current_epoch; // 更新设置failoverEpoch

    master->failover_start_time = mstime()+rand()%SENTINEL_MAX_DESYNC; //更新failover开始时间
    master->failover_state_change_time = mstime(); //更新failover状态变更时间
}
```

#### sentinelAskMasterStateToOtherSentinels
向所有sentinen服务进程发送命令询问大家这个‘我’认为down掉的master在他们眼里是什么状况
```c
void sentinelAskMasterStateToOtherSentinels(sentinelRedisInstance *master, int flags) {
    dictIterator *di;
    dictEntry *de;

    di = dictGetIterator(master->sentinels);
    while((de = dictNext(di)) != NULL) {
        sentinelRedisInstance *ri = dictGetVal(de);

        /* Ask */
        ll2string(port,sizeof(port),master->addr->port);
        // 向目标sentinel实例发送“sentinel is-master-down-by-addr”命令,询问该master的运行状态
        retval = redisAsyncCommand(ri->link->cc,
                    // 设置回调sentinelReceiveIsMasterDownReply纪录结果,下一次循环时,sentinelCheckObjectivelyDown就根据已有信息决定 是否认为该master已经ODown(客观down)了
                    sentinelReceiveIsMasterDownReply, ri, 
                    "%s is-master-down-by-addr %s %s %llu %s",
                    sentinelInstanceMapCommand(ri,"SENTINEL"),
                    master->addr->ip, port,
                    sentinel.current_epoch,
                    (master->failover_state > SENTINEL_FAILOVER_STATE_NONE) ? // 如果failover已经开始了,需要带上‘我’的id
                    sentinel.myid : "*");
        if (retval == C_OK) ri->link->pending_commands++;
    }
    dictReleaseIterator(di);
}
// todo 被调用方sentinel收到 sentinel is-master-down-by-addr 命令的处理 
void sentinelCommand(client *c) {
    // ...
    if (!strcasecmp(c->argv[1]->ptr,"is-master-down-by-addr")) {
        // 发送方sentinel带上了自己的id时表明failover已经开始,需要根据epoch大小投票选出sentinel的leader(由这个leader后续去向相关的master,slave发送failover命令) #ref(sentinelVoteLeader)
        if (ri && ri->flags & SRI_MASTER && strcasecmp(c->argv[5]->ptr,"*")) {
            leader = sentinelVoteLeader(ri,(uint64_t)req_epoch,
                                            c->argv[5]->ptr,
                                            &leader_epoch);
        }

        // 返回目标master实例的down状态, 我选出的leader, 已经我认为的leader的epoch
        addReplyArrayLen(c,3);
        addReply(c, isdown ? shared.cone : shared.czero);
        addReplyBulkCString(c, leader ? leader : "*");
        addReplyLongLong(c, (long long)leader_epoch);
        if (leader) sdsfree(leader);
    }
    // ...
}
// 调用方sentinel 处理“sentinel is-master-down-by-addr”命令回复的回调
void sentinelReceiveIsMasterDownReply(redisAsyncContext *c, void *reply, void *privdata) {
    if (r->type == REDIS_REPLY_ARRAY && r->elements == 3 &&
        r->element[0]->type == REDIS_REPLY_INTEGER &&
        r->element[1]->type == REDIS_REPLY_STRING &&
        r->element[2]->type == REDIS_REPLY_INTEGER)
    {
        ri->last_master_down_reply_time = mstime(); // 更新时间信息
        if (r->element[0]->integer == 1) { 保存该sentinel是否认为这个master的down状态
            ri->flags |= SRI_MASTER_DOWN;
        } else {
            ri->flags &= ~SRI_MASTER_DOWN;
        }
        if (strcmp(r->element[1]->str,"*")) {
            // 如果该sentinel回了个投票选出的sentinelId
            // 保存该sentinel投票的sentinelId 和 epoch
            ri->leader = sdsnew(r->element[1]->str);
            ri->leader_epoch = r->element[2]->integer;
        }
    }
}


```

#### sentinelFailoverStateMachine
开始failover

### `tilt`模式
```c
// src/sentinel.c
void sentinelTimer(void) {
    // 判断是否因为时间的不同步而需要进`tilt`模式 #ref(sentinelCheckTiltCondition)
    sentinelCheckTiltCondition();
    // ...
}
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



