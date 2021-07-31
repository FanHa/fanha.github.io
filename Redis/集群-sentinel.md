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
    int tilt;           // ?? TODO
    int running_scripts;    // ?? TODO
    mstime_t tilt_start_time;       // ?? TODO
    mstime_t previous_time;         // TODO
    list *scripts_queue;            // todo
    char *announce_ip;  // 我与其他sentinel交互用的ip
    int announce_port;  // 我与其他sentinel交互用的port
    unsigned long simfailure_flags; // todo
    int deny_scripts_reconfig;// todo
} sentinel;

// “我”所监管的redis实例结构(前面的 dict *masters)
typedef struct sentinelRedisInstance {
    int flags;      /* See SRI_... defines */
    char *name;     /* Master name from the point of view of this sentinel. */
    char *runid;    /* Run ID of this instance, or unique ID if is a Sentinel.*/
    uint64_t config_epoch;  /* Configuration epoch. */
    sentinelAddr *addr; /* Master host. */
    instanceLink *link; /* Link to the instance, may be shared for Sentinels. */
    mstime_t last_pub_time;   /* Last time we sent hello via Pub/Sub. */
    mstime_t last_hello_time; /* Only used if SRI_SENTINEL is set. Last time
                                 we received a hello from this Sentinel
                                 via Pub/Sub. */
    mstime_t last_master_down_reply_time; /* Time of last reply to
                                             SENTINEL is-master-down command. */
    mstime_t s_down_since_time; /* Subjectively down since time. */
    mstime_t o_down_since_time; /* Objectively down since time. */
    mstime_t down_after_period; /* Consider it down after that period. */
    mstime_t info_refresh;  /* Time at which we received INFO output from it. */
    dict *renamed_commands;     /* Commands renamed in this instance:
                                   Sentinel will use the alternative commands
                                   mapped on this table to send things like
                                   SLAVEOF, CONFING, INFO, ... */

    /* Role and the first time we observed it.
     * This is useful in order to delay replacing what the instance reports
     * with our own configuration. We need to always wait some time in order
     * to give a chance to the leader to report the new configuration before
     * we do silly things. */
    int role_reported;
    mstime_t role_reported_time;
    mstime_t slave_conf_change_time; /* Last time slave master addr changed. */

    /* Master specific. */
    dict *sentinels;    /* Other sentinels monitoring the same master. */
    dict *slaves;       /* Slaves for this master instance. */
    unsigned int quorum;/* Number of sentinels that need to agree on failure. */
    int parallel_syncs; /* How many slaves to reconfigure at same time. */
    char *auth_pass;    /* Password to use for AUTH against master & replica. */
    char *auth_user;    /* Username for ACLs AUTH against master & replica. */

    /* Slave specific. */
    mstime_t master_link_down_time; /* Slave replication link down time. */
    int slave_priority; /* Slave priority according to its INFO output. */
    mstime_t slave_reconf_sent_time; /* Time at which we sent SLAVE OF <new> */
    struct sentinelRedisInstance *master; /* Master instance if it's slave. */
    char *slave_master_host;    /* Master host as reported by INFO */
    int slave_master_port;      /* Master port as reported by INFO */
    int slave_master_link_status; /* Master link status as reported by INFO */
    unsigned long long slave_repl_offset; /* Slave replication offset. */
    /* Failover */
    char *leader;       /* If this is a master instance, this is the runid of
                           the Sentinel that should perform the failover. If
                           this is a Sentinel, this is the runid of the Sentinel
                           that this Sentinel voted as leader. */
    uint64_t leader_epoch; /* Epoch of the 'leader' field. */
    uint64_t failover_epoch; /* Epoch of the currently started failover. */
    int failover_state; /* See SENTINEL_FAILOVER_STATE_* defines. */
    mstime_t failover_state_change_time;
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

### initSentinel
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

### sentinel 的 cron
当程序知道自己是sentinel时,会在这个周期性调度里执行`sentinelTimer`
```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    // ...
    // #ref(sentinelTimer)
    if (server.sentinel_mode) sentinelTimer();
    // ...
}
```

### sentinelTimer
// sentinel的周期性执行逻辑
```c
// sentinel.c
void sentinelTimer(void) {
    sentinelCheckTiltCondition();
    sentinelHandleDictOfRedisInstances(sentinel.masters);
    sentinelRunPendingScripts();
    sentinelCollectTerminatedScripts();
    sentinelKillTimedoutScripts();

    /* We continuously change the frequency of the Redis "timer interrupt"
     * in order to desynchronize every Sentinel from every other.
     * This non-determinism avoids that Sentinels started at the same time
     * exactly continue to stay synchronized asking to be voted at the
     * same time again and again (resulting in nobody likely winning the
     * election because of split brain voting). */
    server.hz = CONFIG_DEFAULT_HZ + rand() % CONFIG_DEFAULT_HZ;
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


