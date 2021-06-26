## 版本
+ Github:redis/redis 
+ 分支:6.0 
+ 版本号:6.0.7

## 创建cluster流程
- 首先是有n台独立已经运行的redis服务,并通过配置将自己初始化成为了一个只有一台实例的cluster;
- 然后是有一个组织者(这个组织者可以是人,也可以是某个其他服务或脚本),知道有这n台独立运行的redis的信息;
- 组织者决定哪些redis实例负责哪些slot,哪些redis实例是master,哪些redis实例是slave等等,并把这些信息通过命令发给到实例;
    - 官网创建集群脚本 "./redis-cli.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005"
- 实例接受命令开始自发的更新集群信息,组成一个更大的cluster

## clusterInit
```c
// src/server.c
// redis 启动时根据配置需要初始化cluster
void initServer(void) {
    // ...
    if (server.cluster_enabled) clusterInit();
    // ...
}
```
```c
// src/cluster.c
void clusterInit(void) {
    int saveconf = 0;
    // 初始化cluster结构(clusterState)
    server.cluster = zmalloc(sizeof(clusterState));
    // ...

    /* Load or create a new nodes configuration. */
    if (clusterLoadConfig(server.cluster_configfile) == C_ERR) {
        // 在初始化时,redis本身就是一个特殊的只有一个实例的集群,设置集群信息myself并将信息加入到已知节点列表中
        myself = server.cluster->myself =
            createClusterNode(NULL,CLUSTER_NODE_MYSELF|CLUSTER_NODE_MASTER);
        
        clusterAddNode(myself);
        saveconf = 1;
    }
    // 保存本地cluster元信息到文件
    if (saveconf) clusterSaveConfigOrDie(1);

    // ...
    int port = server.tls_cluster ? server.tls_port : server.port;
    if (port > (65535-CLUSTER_PORT_INCR)) {
        // ...
        exit(1);
    }
    // 设置一个专门用来集群间信息交流的端口
    if (listenToPort(port+CLUSTER_PORT_INCR,
        server.cfd,&server.cfd_count) == C_ERR)
    {
        exit(1);
    } else {
        int j;

        for (j = 0; j < server.cfd_count; j++) {
            // 创建收到集群信息的可读事件回调clusterAcceptHandler
            // 注:一台机器可能有多个网口,每个口对应一个fd,都需要绑定回调
            if (aeCreateFileEvent(server.el, server.cfd[j], AE_READABLE,
                clusterAcceptHandler, NULL) == AE_ERR)
                    // ...
        }
    }

    // TODO ??
    server.cluster->slots_to_keys = raxNew();
    memset(server.cluster->slots_keys_count,0,
           sizeof(server.cluster->slots_keys_count));

    // 设置port信息,用于告诉别人怎么和自己交互
    myself->port = port;
    myself->cport = port+CLUSTER_PORT_INCR;
    if (server.cluster_announce_port)
        myself->port = server.cluster_announce_port;
    if (server.cluster_announce_bus_port)
        myself->cport = server.cluster_announce_bus_port;

    // ...
}

```
### clusterState 结构
```c
// src/cluster.h
typedef struct clusterState {
    clusterNode *myself;  // 自身节点的信息
    uint64_t currentEpoch; // 信息版本(越高代表越新)
    int state;            // 整个cluster的状态
    int size;             // 整个cluster中的master数量
    dict *nodes;          // 集群中所有节点的名字与信息表
    dict *nodes_black_list; // 节点黑名单
    clusterNode *migrating_slots_to[CLUSTER_SLOTS]; // 正在迁移的slot信息, 哪个slot 迁移到哪个 节点
    clusterNode *importing_slots_from[CLUSTER_SLOTS]; // 正在导入的slot信息, 哪个slot正在从哪个节点导过来
    clusterNode *slots[CLUSTER_SLOTS]; // slot与节点的对应关系
    uint64_t slots_keys_count[CLUSTER_SLOTS]; // TODO ??
    rax *slots_to_keys; // TODO ??

    // master 挂掉时,slave进行选举时用到的字段 TODO
    mstime_t failover_auth_time; /* Time of previous or next election. */
    int failover_auth_count;    /* Number of votes received so far. */
    int failover_auth_sent;     /* True if we already asked for votes. */
    int failover_auth_rank;     /* This slave rank for current auth request. */
    uint64_t failover_auth_epoch; /* Epoch of the current election. */
    int cant_failover_reason;   /* Why a slave is currently not able to
                                   failover. See the CANT_FAILOVER_* macros. */
    /* Manual failover state in common. */
    mstime_t mf_end;            /* Manual failover time limit (ms unixtime).
                                   It is zero if there is no MF in progress. */
    /* Manual failover state of master. */
    clusterNode *mf_slave;      /* Slave performing the manual failover. */
    /* Manual failover state of slave. */
    long long mf_master_offset; /* Master offset the slave needs to start MF
                                   or zero if stil not received. */
    int mf_can_start;           /* If non-zero signal that the manual failover
                                   can start requesting masters vote. */
    /* The followign fields are used by masters to take state on elections. */
    uint64_t lastVoteEpoch;     /* Epoch of the last vote granted. */
    int todo_before_sleep; /* Things to do in clusterBeforeSleep(). */

    /* Messages received and sent by type. */
    long long stats_bus_messages_sent[CLUSTERMSG_TYPE_COUNT];
    long long stats_bus_messages_received[CLUSTERMSG_TYPE_COUNT];
    long long stats_pfail_nodes;    /* Number of nodes in PFAIL status,
                                       excluding nodes without address. */
} clusterState;
```
### clusterNode 结构
```c
// src/cluster.h
// 用于保存一个具体的cluster即诶单信息
typedef struct clusterNode {
    mstime_t ctime; // 节点创建时间
    char name[CLUSTER_NAMELEN]; //节点名
    int flags;      // 节点标记
    uint64_t configEpoch; // “我”所知道的这个节点的信息 的版本
    unsigned char slots[CLUSTER_SLOTS/8]; // 节点处理的slot TODO 为啥/8??
    int numslots;   // 节点处理的slot数
    int numslaves;  // 节点的slave数
    struct clusterNode **slaves; // 指向节点的slave节点的信息结构的指针
    struct clusterNode *slaveof; // 指向节点的master节点的信息结构的指针
    mstime_t ping_sent;      // ”我“上次ping他的时间
    mstime_t pong_received;  // 上次收到”他“的pong的时间
    mstime_t data_received;  // 上次收到”他“的data的时间 ?? TODO data不是通过ping 和pong 传递么
    mstime_t fail_time;      // 设置”他“为fail状态的时间
    mstime_t voted_time;     /* Last time we voted for a slave of this master */
    mstime_t repl_offset_time;  /* Unix time we received offset for this node */
    mstime_t orphaned_time;     /* Starting time of orphaned master condition */
    long long repl_offset;      /* Last known repl offset for this node. */
    char ip[NET_IP_STR_LEN];  // ip
    int port;                   // 客户端调用的port
    int cport;                  // cluster间交互信息的port
    clusterLink *link;          // 与这个节点的tcp连接信息
    list *fail_reports;         // 其他认为这个节点已经fail的节点列表
} clusterNode;
```

## redis-cli 发送 Cluster meet到对应redis实例
脚本(redis-cli)经过自分配好每个节点的角色后,需要通知每个redis实例组成集群(通过向redis服务端口发送命令cluster meet,让所有redis实例主动加入第一个实例的集群)
```c
// src/redis-cli.c
static int clusterManagerCommandCreate(int argc, char **argv) {
        // ...
        clusterManagerNode *first = NULL;
        listRewind(cluster_manager.nodes, &li);
        while ((ln = listNext(&li)) != NULL) {
            clusterManagerNode *node = ln->value;
            if (first == NULL) {
                first = node;
                continue;
            }
            redisReply *reply = NULL;
            // 向每个redis实例发送cluster meet 命令带上第一个redis 实例的信息
            reply = CLUSTER_MANAGER_COMMAND(node, "cluster meet %s %d",
                                            first->ip, first->port);
            // ...
        }
        // ...
}
```
### cluster meet命令处理
```c
// src/cluster.c
void clusterCommand(client *c) {
    // ...
    if (c->argc == 2 && !strcasecmp(c->argv[1]->ptr,"help")) {
        // ...
    } else if (!strcasecmp(c->argv[1]->ptr,"meet") && (c->argc == 4 || c->argc == 5)) {
        /* CLUSTER MEET <ip> <port> [cport] */
        long long port, cport;

        if (getLongLongFromObject(c->argv[3], &port) != C_OK) {
            // ...
            return;
        }

        if (c->argc == 5) {
            if (getLongLongFromObject(c->argv[4], &cport) != C_OK) {
                // ...
            }
        } else {
            cport = port + CLUSTER_PORT_INCR;
        }
        // 收到cluster meet 尝试连接目标redis 实例
        if (clusterStartHandshake(c->argv[2]->ptr,port,cport) == 0 &&
            errno == EINVAL)
        {
            // ... 
        } else {
            addReply(c,shared.ok);
        }
    }
    // ...
}

int clusterStartHandshake(char *ip, int port, int cport) {
    clusterNode *n;
    char norm_ip[NET_IP_STR_LEN];
    struct sockaddr_storage sa;
    // ...
    // 如果已经在握手中了则不再握手
    if (clusterHandshakeInProgress(norm_ip,port,cport)) {
        errno = EAGAIN;
        return 0;
    }

    // 初始化clusternode结构信息,并保存,
    // 此时这个node 的link值为null,然后在clusterCron中,发现link为null,尝试连接
    n = createClusterNode(NULL,CLUSTER_NODE_HANDSHAKE|CLUSTER_NODE_MEET);
    memcpy(n->ip,norm_ip,sizeof(n->ip));
    n->port = port;
    n->cport = cport;
    clusterAddNode(n);
    return 1;
}


```
### clusterCron 连接目标机器
```c
void clusterCron(void) {
    // ...
    di = dictGetSafeIterator(server.cluster->nodes);
    server.cluster->stats_pfail_nodes = 0;
    while((de = dictNext(di)) != NULL) {
        clusterNode *node = dictGetVal(de);
        // ...
        // 与目标机器的连接不存在(初始化时)
        if (node->link == NULL) {

            clusterLink *link = createClusterLink(node); //初始化连接信息
            link->conn = server.tls_cluster ? connCreateTLS() : connCreateSocket();
            connSetPrivateData(link->conn, link);
            // 与目标机器的低层连接(tcp)交给内核,这里只关注连接完成后的回调clusterLinkConnectHandler
            if (connConnect(link->conn, node->ip, node->cport, NET_FIRST_BIND_ADDR,
                        clusterLinkConnectHandler) == -1) {
                // ...
            }
            node->link = link;
        }
    }
    // ...
}

void clusterLinkConnectHandler(connection *conn) {
    clusterLink *link = connGetPrivateData(conn);
    clusterNode *node = link->node;

    //设置连接成功后的可读事件回调
    connSetReadHandler(conn, clusterReadHandler);

    mstime_t old_ping_sent = node->ping_sent;
    // 发送ping消息(第一次是发送meet消息)
    clusterSendPing(link, node->flags & CLUSTER_NODE_MEET ?
            CLUSTERMSG_TYPE_MEET : CLUSTERMSG_TYPE_PING);
    // ...
    // 设置flag,后面就只需发送普通的ping消息了
    node->flags &= ~CLUSTER_NODE_MEET;
    // ...
}

// 这里是第一次发送消息,所以type为CLUSTER_NODE_MEET
void clusterSendPing(clusterLink *link, int type) {
    unsigned char *buf;
    clusterMsg *hdr;
    // ...
    buf = zcalloc(totlen);
    hdr = (clusterMsg*) buf;
    // ...
    // 构造meet头(type为CLUSTER_NODE_MEET)
    clusterBuildMessageHdr(hdr,type);

    // 这里目前只有自身实例的信息,直接continue略过
    while(freshnodes > 0 && gossipcount < wanted && maxiterations--) {
        dictEntry *de = dictGetRandomKey(server.cluster->nodes);
        clusterNode *this = dictGetVal(de);
        // ...
        if (this == myself) continue;
        // ...
    }

    totlen = sizeof(clusterMsg)-sizeof(union clusterMsgData);
    totlen += (sizeof(clusterMsgDataGossip)*gossipcount);
    hdr->count = htons(gossipcount);
    hdr->totlen = htonl(totlen);
    // 发送Meet消息到目标实例
    clusterSendMessage(link,buf,totlen);
    zfree(buf);
}

// TODO connSetReadHandler
```
### 目标机器接受meet消息
```c
// src/cluster.c
void clusterInit(void) {
    //...
            // 目标机器初始化时设置了集群信息端口的可读回调(tcp握手等低层信息交换)
            if (aeCreateFileEvent(server.el, server.cfd[j], AE_READABLE,
                clusterAcceptHandler, NULL) == AE_ERR)
                    // ...
            }
    // ...
}

void clusterAcceptHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    // ...
    while(max--) {
        // ...
        // 握手成功后设置,需要设置下次收到集群信息的回调(这里是指其他实例发过来的meet信息)
        if (connAccept(conn, clusterConnAcceptHandler) == C_ERR) {
            if (connGetState(conn) == CONN_STATE_ERROR)
                // ...
            connClose(conn);
            return;
        }
    }
}
static void clusterConnAcceptHandler(connection *conn) {
    clusterLink *link;
    // 初始化连接信息
    link = createClusterLink(NULL);
    link->conn = conn;
    connSetPrivateData(conn, link);

    // 创建收到集群信息的可读事件回调 clusterReadHandler
    connSetReadHandler(conn, clusterReadHandler);
}

void clusterReadHandler(connection *conn) {
    clusterMsg buf[1];
    ssize_t nread;
    clusterMsg *hdr;
    clusterLink *link = connGetPrivateData(conn);
    unsigned int readlen, rcvbuflen;

    while(1) { /* Read as long as there is data to read. */
        rcvbuflen = sdslen(link->rcvbuf);
        //...
        // 收到一个完整的Meet信息后调用clusterProcessPacket解析meet信息
        if (rcvbuflen >= 8 && rcvbuflen == ntohl(hdr->totlen)) {
            if (clusterProcessPacket(link)) {
                // ...
            } else {
                // ...
            }
        }
    }
}
int clusterProcessPacket(clusterLink *link) {
    clusterMsg *hdr = (clusterMsg*) link->rcvbuf;
    uint32_t totlen = ntohl(hdr->totlen);
    uint16_t type = ntohs(hdr->type);
    mstime_t now = mstime();


    uint16_t flags = ntohs(hdr->flags);
    uint64_t senderCurrentEpoch = 0, senderConfigEpoch = 0;
    clusterNode *sender;

    // 寻找发送者对应的本地集群信息,但因为是第一次收到对方节点的信息(meet),所以这里找不到对应本地集群节点信息
    sender = clusterLookupNode(hdr->sender);

    /* Initial processing of PING and MEET requests replying with a PONG. */
    if (type == CLUSTERMSG_TYPE_PING || type == CLUSTERMSG_TYPE_MEET) {

        if (!sender && type == CLUSTERMSG_TYPE_MEET) {
            clusterNode *node;
            // 根据发送方的的ip,port 信息初始化一个集群node信息,并加入本地集群信息表
            node = createClusterNode(NULL,CLUSTER_NODE_HANDSHAKE);
            nodeIp2String(node->ip,link,hdr->myip);
            node->port = ntohs(hdr->port);
            node->cport = ntohs(hdr->cport);
            clusterAddNode(node);
            clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG);
        }

        // 新节点信息加入到gossip区域,及时传播出去 //TODO
        if (!sender && type == CLUSTERMSG_TYPE_MEET)
            clusterProcessGossipSection(hdr,link);

        // 回应 TODO
        clusterSendPing(link,CLUSTERMSG_TYPE_PONG);
    }

    /* PING, PONG, MEET: process config information. */
    if (type == CLUSTERMSG_TYPE_PING || type == CLUSTERMSG_TYPE_PONG ||
        type == CLUSTERMSG_TYPE_MEET)
    {
        serverLog(LL_DEBUG,"%s packet received: %p",
            type == CLUSTERMSG_TYPE_PING ? "ping" : "pong",
            (void*)link->node);
        if (link->node) {
            if (nodeInHandshake(link->node)) {
                /* If we already have this node, try to change the
                 * IP/port of the node with the new one. */
                if (sender) {
                    serverLog(LL_VERBOSE,
                        "Handshake: we already know node %.40s, "
                        "updating the address if needed.", sender->name);
                    if (nodeUpdateAddressIfNeeded(sender,link,hdr))
                    {
                        clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|
                                             CLUSTER_TODO_UPDATE_STATE);
                    }
                    /* Free this node as we already have it. This will
                     * cause the link to be freed as well. */
                    clusterDelNode(link->node);
                    return 0;
                }

                /* First thing to do is replacing the random name with the
                 * right node name if this was a handshake stage. */
                clusterRenameNode(link->node, hdr->sender);
                serverLog(LL_DEBUG,"Handshake with node %.40s completed.",
                    link->node->name);
                link->node->flags &= ~CLUSTER_NODE_HANDSHAKE;
                link->node->flags |= flags&(CLUSTER_NODE_MASTER|CLUSTER_NODE_SLAVE);
                clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG);
            } else if (memcmp(link->node->name,hdr->sender,
                        CLUSTER_NAMELEN) != 0)
            {
                /* If the reply has a non matching node ID we
                 * disconnect this node and set it as not having an associated
                 * address. */
                serverLog(LL_DEBUG,"PONG contains mismatching sender ID. About node %.40s added %d ms ago, having flags %d",
                    link->node->name,
                    (int)(now-(link->node->ctime)),
                    link->node->flags);
                link->node->flags |= CLUSTER_NODE_NOADDR;
                link->node->ip[0] = '\0';
                link->node->port = 0;
                link->node->cport = 0;
                freeClusterLink(link);
                clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG);
                return 0;
            }
        }

        /* Copy the CLUSTER_NODE_NOFAILOVER flag from what the sender
         * announced. This is a dynamic flag that we receive from the
         * sender, and the latest status must be trusted. We need it to
         * be propagated because the slave ranking used to understand the
         * delay of each slave in the voting process, needs to know
         * what are the instances really competing. */
        if (sender) {
            int nofailover = flags & CLUSTER_NODE_NOFAILOVER;
            sender->flags &= ~CLUSTER_NODE_NOFAILOVER;
            sender->flags |= nofailover;
        }

        /* Update the node address if it changed. */
        if (sender && type == CLUSTERMSG_TYPE_PING &&
            !nodeInHandshake(sender) &&
            nodeUpdateAddressIfNeeded(sender,link,hdr))
        {
            clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|
                                 CLUSTER_TODO_UPDATE_STATE);
        }

        /* Update our info about the node */
        if (link->node && type == CLUSTERMSG_TYPE_PONG) {
            link->node->pong_received = now;
            link->node->ping_sent = 0;

            /* The PFAIL condition can be reversed without external
             * help if it is momentary (that is, if it does not
             * turn into a FAIL state).
             *
             * The FAIL condition is also reversible under specific
             * conditions detected by clearNodeFailureIfNeeded(). */
            if (nodeTimedOut(link->node)) {
                link->node->flags &= ~CLUSTER_NODE_PFAIL;
                clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|
                                     CLUSTER_TODO_UPDATE_STATE);
            } else if (nodeFailed(link->node)) {
                clearNodeFailureIfNeeded(link->node);
            }
        }

        /* Check for role switch: slave -> master or master -> slave. */
        if (sender) {
            if (!memcmp(hdr->slaveof,CLUSTER_NODE_NULL_NAME,
                sizeof(hdr->slaveof)))
            {
                /* Node is a master. */
                clusterSetNodeAsMaster(sender);
            } else {
                /* Node is a slave. */
                clusterNode *master = clusterLookupNode(hdr->slaveof);

                if (nodeIsMaster(sender)) {
                    /* Master turned into a slave! Reconfigure the node. */
                    clusterDelNodeSlots(sender);
                    sender->flags &= ~(CLUSTER_NODE_MASTER|
                                       CLUSTER_NODE_MIGRATE_TO);
                    sender->flags |= CLUSTER_NODE_SLAVE;

                    /* Update config and state. */
                    clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|
                                         CLUSTER_TODO_UPDATE_STATE);
                }

                /* Master node changed for this slave? */
                if (master && sender->slaveof != master) {
                    if (sender->slaveof)
                        clusterNodeRemoveSlave(sender->slaveof,sender);
                    clusterNodeAddSlave(master,sender);
                    sender->slaveof = master;

                    /* Update config. */
                    clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG);
                }
            }
        }

        /* Update our info about served slots.
         *
         * Note: this MUST happen after we update the master/slave state
         * so that CLUSTER_NODE_MASTER flag will be set. */

        /* Many checks are only needed if the set of served slots this
         * instance claims is different compared to the set of slots we have
         * for it. Check this ASAP to avoid other computational expansive
         * checks later. */
        clusterNode *sender_master = NULL; /* Sender or its master if slave. */
        int dirty_slots = 0; /* Sender claimed slots don't match my view? */

        if (sender) {
            sender_master = nodeIsMaster(sender) ? sender : sender->slaveof;
            if (sender_master) {
                dirty_slots = memcmp(sender_master->slots,
                        hdr->myslots,sizeof(hdr->myslots)) != 0;
            }
        }

        /* 1) If the sender of the message is a master, and we detected that
         *    the set of slots it claims changed, scan the slots to see if we
         *    need to update our configuration. */
        if (sender && nodeIsMaster(sender) && dirty_slots)
            clusterUpdateSlotsConfigWith(sender,senderConfigEpoch,hdr->myslots);

        /* 2) We also check for the reverse condition, that is, the sender
         *    claims to serve slots we know are served by a master with a
         *    greater configEpoch. If this happens we inform the sender.
         *
         * This is useful because sometimes after a partition heals, a
         * reappearing master may be the last one to claim a given set of
         * hash slots, but with a configuration that other instances know to
         * be deprecated. Example:
         *
         * A and B are master and slave for slots 1,2,3.
         * A is partitioned away, B gets promoted.
         * B is partitioned away, and A returns available.
         *
         * Usually B would PING A publishing its set of served slots and its
         * configEpoch, but because of the partition B can't inform A of the
         * new configuration, so other nodes that have an updated table must
         * do it. In this way A will stop to act as a master (or can try to
         * failover if there are the conditions to win the election). */
        if (sender && dirty_slots) {
            int j;

            for (j = 0; j < CLUSTER_SLOTS; j++) {
                if (bitmapTestBit(hdr->myslots,j)) {
                    if (server.cluster->slots[j] == sender ||
                        server.cluster->slots[j] == NULL) continue;
                    if (server.cluster->slots[j]->configEpoch >
                        senderConfigEpoch)
                    {
                        serverLog(LL_VERBOSE,
                            "Node %.40s has old slots configuration, sending "
                            "an UPDATE message about %.40s",
                                sender->name, server.cluster->slots[j]->name);
                        clusterSendUpdate(sender->link,
                            server.cluster->slots[j]);

                        /* TODO: instead of exiting the loop send every other
                         * UPDATE packet for other nodes that are the new owner
                         * of sender's slots. */
                        break;
                    }
                }
            }
        }

        /* If our config epoch collides with the sender's try to fix
         * the problem. */
        if (sender &&
            nodeIsMaster(myself) && nodeIsMaster(sender) &&
            senderConfigEpoch == myself->configEpoch)
        {
            clusterHandleConfigEpochCollision(sender);
        }

        /* Get info from the gossip section */
        if (sender) clusterProcessGossipSection(hdr,link);
    } else if (type == CLUSTERMSG_TYPE_FAIL) {
        clusterNode *failing;

        if (sender) {
            failing = clusterLookupNode(hdr->data.fail.about.nodename);
            if (failing &&
                !(failing->flags & (CLUSTER_NODE_FAIL|CLUSTER_NODE_MYSELF)))
            {
                serverLog(LL_NOTICE,
                    "FAIL message received from %.40s about %.40s",
                    hdr->sender, hdr->data.fail.about.nodename);
                failing->flags |= CLUSTER_NODE_FAIL;
                failing->fail_time = now;
                failing->flags &= ~CLUSTER_NODE_PFAIL;
                clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|
                                     CLUSTER_TODO_UPDATE_STATE);
            }
        } else {
            serverLog(LL_NOTICE,
                "Ignoring FAIL message from unknown node %.40s about %.40s",
                hdr->sender, hdr->data.fail.about.nodename);
        }
    } else if (type == CLUSTERMSG_TYPE_PUBLISH) {
        robj *channel, *message;
        uint32_t channel_len, message_len;

        /* Don't bother creating useless objects if there are no
         * Pub/Sub subscribers. */
        if (dictSize(server.pubsub_channels) ||
           listLength(server.pubsub_patterns))
        {
            channel_len = ntohl(hdr->data.publish.msg.channel_len);
            message_len = ntohl(hdr->data.publish.msg.message_len);
            channel = createStringObject(
                        (char*)hdr->data.publish.msg.bulk_data,channel_len);
            message = createStringObject(
                        (char*)hdr->data.publish.msg.bulk_data+channel_len,
                        message_len);
            pubsubPublishMessage(channel,message);
            decrRefCount(channel);
            decrRefCount(message);
        }
    } else if (type == CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST) {
        if (!sender) return 1;  /* We don't know that node. */
        clusterSendFailoverAuthIfNeeded(sender,hdr);
    } else if (type == CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK) {
        if (!sender) return 1;  /* We don't know that node. */
        /* We consider this vote only if the sender is a master serving
         * a non zero number of slots, and its currentEpoch is greater or
         * equal to epoch where this node started the election. */
        if (nodeIsMaster(sender) && sender->numslots > 0 &&
            senderCurrentEpoch >= server.cluster->failover_auth_epoch)
        {
            server.cluster->failover_auth_count++;
            /* Maybe we reached a quorum here, set a flag to make sure
             * we check ASAP. */
            clusterDoBeforeSleep(CLUSTER_TODO_HANDLE_FAILOVER);
        }
    } else if (type == CLUSTERMSG_TYPE_MFSTART) {
        /* This message is acceptable only if I'm a master and the sender
         * is one of my slaves. */
        if (!sender || sender->slaveof != myself) return 1;
        /* Manual failover requested from slaves. Initialize the state
         * accordingly. */
        resetManualFailover();
        server.cluster->mf_end = now + CLUSTER_MF_TIMEOUT;
        server.cluster->mf_slave = sender;
        pauseClients(now+(CLUSTER_MF_TIMEOUT*CLUSTER_MF_PAUSE_MULT));
        serverLog(LL_WARNING,"Manual failover requested by replica %.40s.",
            sender->name);
    } else if (type == CLUSTERMSG_TYPE_UPDATE) {
        clusterNode *n; /* The node the update is about. */
        uint64_t reportedConfigEpoch =
                    ntohu64(hdr->data.update.nodecfg.configEpoch);

        if (!sender) return 1;  /* We don't know the sender. */
        n = clusterLookupNode(hdr->data.update.nodecfg.nodename);
        if (!n) return 1;   /* We don't know the reported node. */
        if (n->configEpoch >= reportedConfigEpoch) return 1; /* Nothing new. */

        /* If in our current config the node is a slave, set it as a master. */
        if (nodeIsSlave(n)) clusterSetNodeAsMaster(n);

        /* Update the node's configEpoch. */
        n->configEpoch = reportedConfigEpoch;
        clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|
                             CLUSTER_TODO_FSYNC_CONFIG);

        /* Check the bitmap of served slots and update our
         * config accordingly. */
        clusterUpdateSlotsConfigWith(n,reportedConfigEpoch,
            hdr->data.update.nodecfg.slots);
    } else if (type == CLUSTERMSG_TYPE_MODULE) {
        if (!sender) return 1;  /* Protect the module from unknown nodes. */
        /* We need to route this message back to the right module subscribed
         * for the right message type. */
        uint64_t module_id = hdr->data.module.msg.module_id; /* Endian-safe ID */
        uint32_t len = ntohl(hdr->data.module.msg.len);
        uint8_t type = hdr->data.module.msg.type;
        unsigned char *payload = hdr->data.module.msg.bulk_data;
        moduleCallClusterReceivers(sender->name,module_id,type,payload,len);
    } else {
        serverLog(LL_WARNING,"Received unknown packet type: %d", type);
    }
    return 1;
}

```