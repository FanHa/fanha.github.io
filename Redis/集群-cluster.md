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
// 用于保存一个具体的cluster节点信息
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

## redis-cli 组织和通知各实例来建立集群
- 集群初始化流程
    - redis-cli分配好节点角色后,通知集群中的每个实例,让他们主动加入一个共同的节点(cluster meet命令)
    - 收到cluster meet命令处理,设置好要加入的集群的目标机器信息
    - clusterCron 根据前面设置好的目标机器信息连接目标机器
    - 目标机器接收meet消息,将机器加入本地集群信息,并pong回自己视角的集群信息(当机器多时,gossip机制,返回部分)
    - 接收目标机器的pong,更新本地集群信息
    - 集群根据已知信息相互ping,pong,发送gossip消息,使得集群中的机器最终都对整个集群信息有了解
    - 设置集群中各个节点的slot和master-slave关系(cluster addslots 和 cluster replicate命令)
    - 节点收到cluster addslots 和 cluster replicate 命令后开始更新自己的职责信息
    - 通过集群信息传播slot设置和replicate设置

### 发送 Cluster meet到对应redis实例
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

### 收到cluster meet命令处理,设置好要加入的集群的目标机器信息
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
### clusterCron 根据前面设置好的目标机器信息连接目标机器
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
```
### 目标机器接收meet消息,将机器加入本地集群信息,并pong回自己视角的集群信息(当机器多时,gossip机制,返回部分)
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
            clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG); // TODO
        }

        // 至此,当前节点已经有了发送方的节点信息,需要把当前节点所知道的集群信息发返回去
        // 当已知集群节点数量很多时会触发gossip机制,选一些节点信息;
        clusterSendPing(link,CLUSTERMSG_TYPE_PONG);
    }
    // ...
    return 1;
}

```
### 接受目标机器的pong, 接收目标机器的pong,更新本地集群信息
节点接收并处理目标机器pong信息同样走到了 clusterProcessPacket 方法
```c
int clusterProcessPacket(clusterLink *link) {
    clusterMsg *hdr = (clusterMsg*) link->rcvbuf;
    uint32_t totlen = ntohl(hdr->totlen);
    uint16_t type = ntohs(hdr->type);
    mstime_t now = mstime();

    uint16_t flags = ntohs(hdr->flags);
    uint64_t senderCurrentEpoch = 0, senderConfigEpoch = 0;
    clusterNode *sender;
    // ...
    sender = clusterLookupNode(hdr->sender);
    if (sender) sender->data_received = now;

    if (type == CLUSTERMSG_TYPE_PING || type == CLUSTERMSG_TYPE_PONG ||
        type == CLUSTERMSG_TYPE_MEET)
    {

        if (sender) {
            int nofailover = flags & CLUSTER_NODE_NOFAILOVER;
            sender->flags &= ~CLUSTER_NODE_NOFAILOVER;
            sender->flags |= nofailover;
        }

        // 收到pong消息更新对应节点连接的一些统计信息
        if (link->node && type == CLUSTERMSG_TYPE_PONG) {
            link->node->pong_received = now;
            link->node->ping_sent = 0;

            if (nodeTimedOut(link->node)) {
                link->node->flags &= ~CLUSTER_NODE_PFAIL;
                clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|
                                     CLUSTER_TODO_UPDATE_STATE);
            } else if (nodeFailed(link->node)) {
                clearNodeFailureIfNeeded(link->node);
            }
        }

        // 收到pong消息需要处理对方发来的关于集群的gossip消息,并更新自己本地的集群信息
        if (sender) clusterProcessGossipSection(hdr,link);
    }
    return 1;
}

```
### 节点clusterCron 周期发送ping消息,传播集群信息
```c
// src/cluster.c
void clusterCron(void) {
    // ...
    // 减少ping的频率
    if (!(iteration % 10)) {
        int j;

        for (j = 0; j < 5; j++) { //随机选几个知道的node发送ping
            de = dictGetRandomKey(server.cluster->nodes);
            clusterNode *this = dictGetVal(de);

            // 剔除掉没有完全建立连接的,
            if (this->link == NULL || this->ping_sent != 0) continue;
            if (this->flags & (CLUSTER_NODE_MYSELF|CLUSTER_NODE_HANDSHAKE))
                continue;
            // 剔除掉ping了还没回的
            if (min_pong_node == NULL || min_pong > this->pong_received) {
                min_pong_node = this;
                min_pong = this->pong_received;
            }
        }
        if (min_pong_node) {
            // Ping,会在这个clusterSendPing方法里带上一些gossip消息
            clusterSendPing(min_pong_node->link, CLUSTERMSG_TYPE_PING);
        }
    }
    // ...
}

void clusterSendPing(clusterLink *link, int type) {
    unsigned char *buf;
    clusterMsg *hdr;
    int gossipcount = 0; /* Number of gossip sections added so far. */
    int wanted; /* Number of gossip sections we want to append if possible. */
    int totlen; /* Total packet length. */
    int freshnodes = dictSize(server.cluster->nodes)-2;
    int pfail_wanted = server.cluster->stats_pfail_nodes;

 
    buf = zcalloc(totlen);
    hdr = (clusterMsg*) buf;

    if (link->node && type == CLUSTERMSG_TYPE_PING)
        link->node->ping_sent = mstime();
    clusterBuildMessageHdr(hdr,type);

    int maxiterations = wanted*3;
    while(freshnodes > 0 && gossipcount < wanted && maxiterations--) {
        // 随机选一些节点(gossip),将信息添加进要发送的包信息中
        dictEntry *de = dictGetRandomKey(server.cluster->nodes);
        clusterNode *this = dictGetVal(de);

        // 自己的信息不需要再加在gossipSecetion里了
        if (this == myself) continue;

        // 已经被自己标记为 CLUSTER_NODE_PFAIL 在后面做特殊处理再发
        if (this->flags & CLUSTER_NODE_PFAIL) continue;

        // 其他一些不需要发的例外情况
        if (this->flags & (CLUSTER_NODE_HANDSHAKE|CLUSTER_NODE_NOADDR) ||
            (this->link == NULL && this->numslots == 0))
        {
            freshnodes--; /* Tecnically not correct, but saves CPU. */
            continue;
        }

        // 因为是随机选的,难免选到相同节点,略郭
        if (clusterNodeIsInGossipSection(hdr,gossipcount,this)) continue;

        // 将该节点信息加入gossip待发送区域
        clusterSetGossipEntry(hdr,gossipcount,this);
        freshnodes--;
        gossipcount++;
    }

    // 当发现自己的cluster里有PFail(疑似fail)的节点时,遍历找到所有疑似Fail的节点,加入到gossip待发送区域中
    if (pfail_wanted) {
        dictIterator *di;
        dictEntry *de;

        di = dictGetSafeIterator(server.cluster->nodes);
        // 遍历找到所有当前节点认为疑似Fail的节点,加入到gossip待发送区域中
        while((de = dictNext(di)) != NULL && pfail_wanted > 0) {
            clusterNode *node = dictGetVal(de);
            if (node->flags & CLUSTER_NODE_HANDSHAKE) continue;
            if (node->flags & CLUSTER_NODE_NOADDR) continue;
            if (!(node->flags & CLUSTER_NODE_PFAIL)) continue;
            clusterSetGossipEntry(hdr,gossipcount,node);
            freshnodes--;
            gossipcount++;
            pfail_wanted--;
        }
        dictReleaseIterator(di);
    }
    // ...
    clusterSendMessage(link,buf,totlen);
    zfree(buf);
}

// 将gossip信息加入到要发送的信息头中
void clusterSetGossipEntry(clusterMsg *hdr, int i, clusterNode *n) {
    clusterMsgDataGossip *gossip;
    gossip = &(hdr->data.ping.gossip[i]);
    memcpy(gossip->nodename,n->name,CLUSTER_NAMELEN);
    gossip->ping_sent = htonl(n->ping_sent/1000);
    gossip->pong_received = htonl(n->pong_received/1000);
    memcpy(gossip->ip,n->ip,sizeof(n->ip));
    gossip->port = htons(n->port);
    gossip->cport = htons(n->cport);
    gossip->flags = htons(n->flags);
    gossip->notused1 = 0;
}
```
### 集群中的节点都设置了集群消息的回调,收到ping或pong消息,更新本地集群信息,并视情况回pong信息,帮助对方更新集群信息 
```c
// src/cluster.c
int clusterProcessPacket(clusterLink *link) {
    clusterMsg *hdr = (clusterMsg*) link->rcvbuf;
    uint32_t totlen = ntohl(hdr->totlen);
    uint16_t type = ntohs(hdr->type);
    mstime_t now = mstime();

    uint16_t flags = ntohs(hdr->flags);
    uint64_t senderCurrentEpoch = 0, senderConfigEpoch = 0;
    clusterNode *sender;
    sender = clusterLookupNode(hdr->sender);

    /* Initial processing of PING and MEET requests replying with a PONG. */
    if (type == CLUSTERMSG_TYPE_PING || type == CLUSTERMSG_TYPE_MEET) {
        serverLog(LL_DEBUG,"Ping packet received: %p", (void*)link->node);
        // 收到ping消息需同样需要调用 clusterSendPing 发送一个pong消息,也会带上自己生成的gossip信息
        clusterSendPing(link,CLUSTERMSG_TYPE_PONG);
    }

    if (type == CLUSTERMSG_TYPE_PING || type == CLUSTERMSG_TYPE_PONG ||
        type == CLUSTERMSG_TYPE_MEET)
    {
        // 如果是一个pong信息,我们先更新这个信息发送方的连接信息
        if (link->node && type == CLUSTERMSG_TYPE_PONG) {
            link->node->pong_received = now;
            link->node->ping_sent = 0;
            // ...
        }

        // 无论最后收到的是ping 还是 pong,最后都会走到这里,调用clusterProcessGossipSection,根据收到的信息更新本地的gossip信息
        if (sender) clusterProcessGossipSection(hdr,link);
    } 
    // ...
    return 1;
}
```
#### clusterProcessGossipSection 处理gossip信息
对收到的ping和pong消息里待的gossip信息做处理
```c
void clusterProcessGossipSection(clusterMsg *hdr, clusterLink *link) {
    uint16_t count = ntohs(hdr->count);
    clusterMsgDataGossip *g = (clusterMsgDataGossip*) hdr->data.ping.gossip;
    clusterNode *sender = link->node ? link->node : clusterLookupNode(hdr->sender);

    // 遍历gossip信息中的每个node
    while(count--) {
        uint16_t flags = ntohs(g->flags);
        clusterNode *node;
        sds ci;
        node = clusterLookupNode(g->nodename);
        if (node) { //如果我们本地已经知道这个node 
            // 更新本地信息

        } else {
            // 本地没有这个node的信息,则新建关于这个node 的 ClusterNode结构,加入到本地cluster信息中
            if (sender &&
                !(flags & CLUSTER_NODE_NOADDR) &&
                !clusterBlacklistExists(g->nodename))
            {
                clusterNode *node;
                node = createClusterNode(g->nodename, flags);
                memcpy(node->ip,g->ip,NET_IP_STR_LEN);
                node->port = ntohs(g->port);
                node->cport = ntohs(g->cport);
                clusterAddNode(node);
            }
        }

        g++;
    }
}

```
### redis-cli设置集群中各个节点的slot和master-slave关系(cluster addslots命令 和 cluster replicate命令)
```c
src/redis-cli.c
        // cluster meet 相关
        // cluster 命令后需要等集群里的节点互相认识了后再发设置slot 和 master-slave 更有效率
        sleep(1);
        clusterManagerWaitForClusterJoin();

        // 遍历所有节点,根据既定的配置,向每个节点发送他的职责
        listRewind(cluster_manager.nodes, &li);
        while ((ln = listNext(&li)) != NULL) {
            clusterManagerNode *node = ln->value;
            if (!node->dirty) continue;
            char *err = NULL;
            int flushed = clusterManagerFlushNodeConfig(node, &err);
            if (!flushed && !node->replicate) {
                // ...
            }
        }
        // ...
```
```c
// src/redis-cli.c
static int clusterManagerFlushNodeConfig(clusterManagerNode *node, char **err) {
    if (!node->dirty) return 0;
    redisReply *reply = NULL;
    int is_err = 0, success = 1;
    if (err != NULL) *err = NULL;
    if (node->replicate != NULL) {
        // 当前节点是slave时,发送cluter replicate
        reply = CLUSTER_MANAGER_COMMAND(node, "CLUSTER REPLICATE %s",
                                        node->replicate);
        // ...
    } else {
        // 当前节点时master时发送cluster addslots
        int added = clusterManagerAddSlots(node, err);
        if (!added || *err != NULL) success = 0;
    }
    node->dirty = 0;
cleanup:
    if (reply != NULL) freeReplyObject(reply);
    return success;
}
```

### 节点收到cluster replicate 命令 和 cluster addslots命令的处理
#### 处理cluster replicate 命令
```c
// src/cluster.c
void clusterCommand(client *c) {
    //...
    if (!strcasecmp(c->argv[1]->ptr,"replicate") && c->argc == 3) {
        /* CLUSTER REPLICATE <NODE ID> */
        clusterNode *n = clusterLookupNode(c->argv[2]->ptr);

        // 不允许已经是master 且已经分配的slot的node直接通过命令方式成为别的node的slave
        if (nodeIsMaster(myself) &&
            (myself->numslots != 0 || dictSize(server.db[0].dict) != 0)) {
            addReplyError(c,
                "To set a master the node must be empty and "
                "without assigned slots.");
            return;
        }

        // 设置信息,后续在gossip中会把这个信息传播出去,让别的节点知道
        clusterSetMaster(n);
        addReply(c,shared.ok);
    } 
    // ...
}
```
#### 处理cluster addslots 命令
```c
void clusterCommand(client *c) {
    // ...
    if ((!strcasecmp(c->argv[1]->ptr,"addslots") ||
               !strcasecmp(c->argv[1]->ptr,"delslots")) && c->argc >= 3)
    {
        
        int del = !strcasecmp(c->argv[1]->ptr,"delslots");

        for (j = 2; j < c->argc; j++) {
            if (del && server.cluster->slots[slot] == NULL) {
                // ...
            } else if (!del && server.cluster->slots[slot]) {
                // 如果已知要add的slot 已经有主了,则不能设置
                // 但因为现在还在初始化阶段,节点的slots值都是空
                return;
            }
            if (slots[slot]++ == 1) {
                // 去重
                return;
            }
        }
        for (j = 0; j < CLUSTER_SLOTS; j++) {
            if (slots[j]) {
                int retval;
                // 在cluster中加入这个slot信息的归属

                retval = del ? clusterDelSlot(j) :
                               clusterAddSlot(myself,j);
                serverAssertWithInfo(c,NULL,retval == C_OK);
            }
        }
        zfree(slots);
        clusterDoBeforeSleep(CLUSTER_TODO_UPDATE_STATE|CLUSTER_TODO_SAVE_CONFIG);
        addReply(c,shared.ok);
    } 
}
```
### 传播slot设置和replicate设置
上一步设置的slot 和 replicate 信息每个节点都只知道自己的,需要传播到整个集群,还是通过前面clusterCron里的ping 和 pong来交换和设置
```c
int clusterProcessPacket(clusterLink *link) {  
    // ...
    if (type == CLUSTERMSG_TYPE_PING || type == CLUSTERMSG_TYPE_PONG ||
        type == CLUSTERMSG_TYPE_MEET)
    {
        if (sender) {
            // 更新replicate信息
            if (!memcmp(hdr->slaveof,CLUSTER_NODE_NULL_NAME,
                sizeof(hdr->slaveof)))
            {
                /* Node is a master. */
                clusterSetNodeAsMaster(sender);
            } else {
                /* Node is a slave. */
                // 收到的sender信息表示自己是slave
                // 找到sender的master
                clusterNode *master = clusterLookupNode(hdr->slaveof);

                if (nodeIsMaster(sender)) { // 如果sender本来是master需要清空他的slot信息
                    // 更新sender的标记
                    clusterDelNodeSlots(sender);
                    sender->flags &= ~(CLUSTER_NODE_MASTER|
                                       CLUSTER_NODE_MIGRATE_TO);
                    sender->flags |= CLUSTER_NODE_SLAVE;
                }

                if (master && sender->slaveof != master) {
                    // sender本身是别的master的slave时,需要更新sender的master信息
                    if (sender->slaveof)
                        clusterNodeRemoveSlave(sender->slaveof,sender);
                    // 更新sender指定的master的信息
                    clusterNodeAddSlave(master,sender);
                    sender->slaveof = master;
                }
            }
        }
        // 更新slot信息 TODO
        clusterNode *sender_master = NULL; /* Sender or its master if slave. */
        int dirty_slots = 0; /* Sender claimed slots don't match my view? */

        if (sender) {
            // sender报告的slot和我们已知的信息不同时,设置dirty_slots,后续用来更新信息
            sender_master = nodeIsMaster(sender) ? sender : sender->slaveof;
            if (sender_master) {
                dirty_slots = memcmp(sender_master->slots,
                        hdr->myslots,sizeof(hdr->myslots)) != 0;
            }
        }

        // 根据sender发来的信息版本(configEpoch),更新本地有关sender的信息
        if (sender && nodeIsMaster(sender) && dirty_slots)
            clusterUpdateSlotsConfigWith(sender,senderConfigEpoch,hdr->myslots);

        if (sender && dirty_slots) {
            int j;
            for (j = 0; j < CLUSTER_SLOTS; j++) {
                if (bitmapTestBit(hdr->myslots,j)) { 
                    if (server.cluster->slots[j] == sender ||
                        server.cluster->slots[j] == NULL) continue; //要设置的slot本来就属于sender 或没有归属,continue
                    if (server.cluster->slots[j]->configEpoch >
                        senderConfigEpoch) // 要设置的slot本来属于其他node,且configEpoch信息大于sender
                    {

                        // 发送一条‘CLUSTERMSG_TYPE_UPDATE’集群信息给sender,告诉现在的最新情况(并强制让sender更新slot信息 TODO)
                        clusterSendUpdate(sender->link,
                            server.cluster->slots[j]);


                        break;
                    }
                }
            }
        }

        // 当sender的configEpoch 和 我们本地的configEpoch相同时代表冲突;
        // (每次更新信息后configEpoch都递增,正常不应该出现一个带了更新信息的configEpoch和我们本地的configEpoch相同)
        if (sender &&
            nodeIsMaster(myself) && nodeIsMaster(sender) &&
            senderConfigEpoch == myself->configEpoch)
        {
            // 解决configEpoch冲突的问题
            clusterHandleConfigEpochCollision(sender);
        }
    }
}

void clusterHandleConfigEpochCollision(clusterNode *sender) {
    // ...
    // configEpoch相同就比nodeId大小
    // sender的小就直接略过
    if (memcmp(sender->name,myself->name,CLUSTER_NAMELEN) <= 0) return;
    
    // 否则就增加本地集群currentEpoch ,防止后面继续冲突
    server.cluster->currentEpoch++;
    myself->configEpoch = server.cluster->currentEpoch;
    
}
```

## 集群建立后的命令处理
集群建立后任意节点都能接收来自客户端的请求,先判断自己能不能处理,如果不能,则通过“重定向”让客户端再连接到真正读取数据的节点进行操作
### 重定向
```c
// src/server.c
int processCommand(client *c) {

    // ...
    // 如果开启了cluster_enabled,再处理命令前需要现判定不不需要重定向
    if (server.cluster_enabled &&
        !(c->flags & CLIENT_MASTER) &&
        !(c->flags & CLIENT_LUA &&
          server.lua_caller->flags & CLIENT_MASTER) &&
        !(c->cmd->getkeys_proc == NULL && c->cmd->firstkey == 0 &&
          c->cmd->proc != execCommand))
    {
        int hashslot;
        int error_code;
        // 根据命令的键算出来的slot寻找能处理这个命令的节点
        clusterNode *n = getNodeByQuery(c,c->cmd,c->argv,c->argc,
                                        &hashslot,&error_code);
        if (n == NULL || n != server.cluster->myself) { //当能处理的节点不是自己时需要返回重定向信息,客户端根据重定向的信息向对应节点再发起请求处理
            clusterRedirectClient(c,n,hashslot,error_code);
            return C_OK;
        }
    }
    // ...
}
```

## 集群建立后管理员主动更改集群设置
- 一般主动更改设置的原因:
    - 增加冗余节点,直接向要新节点发送cluster meet, 目标随便选个在集群中的节点,然后等待集群信息传播就好(但目前实际这个节点没有分配任何slot,也不是任何其他节点的slave,只是混了进来跟着吆喝)
    - 更改slots分配

### 更改slot设置(cluster setslots命令)
- 设置slots迁出方的slots为Migrating状态
- 设置slots迁入方的slots为importing状态
- 获取要迁出的slots包含了哪些键
- 向迁出方发送migrate命令 (to 迁入方)
- 向迁入方和迁出方发送cluster setslots命令保存状态

#### 迁出方 cluster setslots migrate(importing) 命令
```c
// src/cluster.c
void clusterCommand(client *c) {
    //...
    // migrating 和 importing子命令都只是给目标机器设置了slot的状态,具体的键值对的转移需要另外的命令
    if (!strcasecmp(c->argv[1]->ptr,"setslot") && c->argc >= 4) {
        int slot;
        clusterNode *n;

        if (!strcasecmp(c->argv[3]->ptr,"migrating") && c->argc == 5) {
            // ...
            server.cluster->migrating_slots_to[slot] = n;
        } else if (!strcasecmp(c->argv[3]->ptr,"importing") && c->argc == 5) {
            // ...
            server.cluster->importing_slots_from[slot] = n;
        }
    }
    // ...
}
```
#### 询问迁出方要迁出的slot包含了哪些key
使用”CLUSTER GETKEYSINSLOT slot“命令

#### 向迁出方发送migrate命令
migrate命令指定了目标节点的地址和要迁移的key
```c
// src/cluster.c
// 这是个大同步操作,所有同步完成(或失败)后才返回,是否会阻塞redis接受别的请求?? TODO
void migrateCommand(client *c) {
    migrateCachedSocket *cs;
    // ... 参数处理

try_again:
    write_error = 0;

    // 与目标node建立实例
    cs = migrateGetSocket(c,c->argv[1],c->argv[2],timeout);
    if (cs == NULL) {
        // ...
    }
    // 初始化一个rio结构,用来写入要传输的数据(sdsempty创建一块空区域用来存数据)
    rioInitWithBuffer(&cmd,sdsempty());

    for (j = 0; j < num_keys; j++) { // 遍历所有要迁移的键
        // ...

        if (server.cluster_enabled) // 集群特有的,为每一个键值对发送restore-asking命令
            serverAssertWithInfo(c,NULL,
                rioWriteBulkString(&cmd,"RESTORE-ASKING",14));
        else
            // ...

        // 将要迁移的键值信息写入&cmd结构中
        createDumpPayload(&payload,ov[j],kv[j]);
        serverAssertWithInfo(c,NULL,
            rioWriteBulkString(&cmd,payload.io.buffer.ptr,
                               sdslen(payload.io.buffer.ptr)));
        // ...
    }

    errno = 0;
    {   
        // 所以要迁移的信息准备好后,发送给目标node实例
        sds buf = cmd.io.buffer.ptr;
        size_t pos = 0, towrite;
        int nwritten = 0;

        while ((towrite = sdslen(buf)-pos) > 0) {
            towrite = (towrite > (64*1024) ? (64*1024) : towrite);
            nwritten = connSyncWrite(cs->conn,buf+pos,towrite,timeout);
            if (nwritten != (signed)towrite) {
                write_error = 1;
                goto socket_err;
            }
            pos += nwritten;
        }
    }
    // 发送的每一条restore命令都有一个回应
    for (j = 0; j < num_keys; j++) {
        if (connSyncReadLine(cs->conn, buf2, sizeof(buf2), timeout) <= 0) {
            // ...
        }
        if ((password && buf0[0] == '-') ||
            (select && buf1[0] == '-') ||
            buf2[0] == '-')
        {
            // ...
        } else {
            if (!copy) {
                // 已经迁移成功的键值,直接删掉本地信息就行
                dbDelete(c->db,kv[j]);
                // ...
            }
        }
    }
}

```
#### 目标机器接受restore-asking命令
restore 和 restore-asking命令公用restoreCommand处理逻辑
```c
// src/server.c
{"restore",restoreCommand,-4,
     "write use-memory @keyspace @dangerous",
     0,NULL,1,1,1,0,0,0},

    {"restore-asking",restoreCommand,-4,
    "write use-memory cluster-asking @keyspace @dangerous",
    0,NULL,1,1,1,0,0,0},
```
```c
// src/cluster.c
void restoreCommand(client *c) {
    long long ttl, lfu_freq = -1, lru_idle = -1, lru_clock = -1;
    rio payload;
    int j, type, replace = 0, absttl = 0;
    robj *obj;

    // ... 参数解析

    // 将rio结构的读写区域指向传过来的键值对
    rioInitWithBuffer(&payload,c->argv[3]->ptr);

    // 写入本地db
    dbAdd(c->db,key,obj);
    // ...
    // 将操作成功的信息写入回复队列
    addReply(c,shared.ok);
    server.dirty++;
}
```
#### cluster setslots [slots] node 命令收尾
```c
void clusterCommand(client *c) {

    if (!strcasecmp(c->argv[1]->ptr,"setslot") && c->argc >= 4) {
        
        int slot;
        clusterNode *n;
        // ...
        if (!strcasecmp(c->argv[3]->ptr,"node") && c->argc == 5) {
            clusterNode *n = clusterLookupNode(c->argv[4]->ptr);

            // 清掉migrating_slot 状态
            if (countKeysInSlot(slot) == 0 &&
                server.cluster->migrating_slots_to[slot])
                server.cluster->migrating_slots_to[slot] = NULL;

            // 清掉importing 状态
            if (n == myself &&
                server.cluster->importing_slots_from[slot])
            {
                // 如果slot本来状态是import,需要更新下configEpoch,这样以后传播slot设置时,当前版本更高,可以在集群中替换掉旧版本关于当前slot的信息
                if (clusterBumpConfigEpochWithoutConsensus() == C_OK) {
                    // ...
                }
                server.cluster->importing_slots_from[slot] = NULL;
            }
            // 设置本地slot状态
            clusterDelSlot(slot);
            clusterAddSlot(n,slot);
        }
        // ...
        addReply(c,shared.ok);
    }
}
```

### key正在转移时的客户端命令处理
key正在转移时,源方和目标方的对这个key所在的slot都有标记(migrating,importing标记),处理key前会先查这个标记决定是否要重定向
```c
int processCommand(client *c) {
    if (server.cluster_enabled &&
        !(c->flags & CLIENT_MASTER) &&
        !(c->flags & CLIENT_LUA &&
          server.lua_caller->flags & CLIENT_MASTER) &&
        !(c->cmd->getkeys_proc == NULL && c->cmd->firstkey == 0 &&
          c->cmd->proc != execCommand))
    {
        int hashslot;
        int error_code;
        // 找到处理当前命令的节点,error_code 会返回重定向类型(move,ask等)
        clusterNode *n = getNodeByQuery(c,c->cmd,c->argv,c->argc,
                                        &hashslot,&error_code);
        if (n == NULL || n != server.cluster->myself) {
            // ... 当不是本季时,返回重定向消息,客户端根据消息再见机行事(error_code就是重定向的类型,比如move,ask等)
            clusterRedirectClient(c,n,hashslot,error_code);
            return C_OK;
        }
    }
}
```
 
## failover 和 选举
### 节点什么时候认为另一个节点已经fail了?
#### 一手信息,自己与另一个节点ping-pong中发现ping不通了
节点间`ping`来验证对应节点的状态
```c
// src/cluster.c
// cron里周期性的选个节点发送ping消息
void clusterCron(void) {
    if (!(iteration % 10)) {
        int j;

        // 随机选一些(5个)节点做比较
        for (j = 0; j < 5; j++) {
            de = dictGetRandomKey(server.cluster->nodes);
            clusterNode *this = dictGetVal(de);

            // 选出最久没有收到过pong信息的节点
            if (min_pong_node == NULL || min_pong > this->pong_received) {
                min_pong_node = this;
                min_pong = this->pong_received;
            }
        }
        if (min_pong_node) {
            // 向选中的节点发送一个寻常ping
            clusterSendPing(min_pong_node->link, CLUSTERMSG_TYPE_PING);
        }
    }
    // ...
    orphaned_masters = 0;
    max_slaves = 0;
    this_slaves = 0;
    di = dictGetSafeIterator(server.cluster->nodes);
    // 检验每一个节点的状态
    while((de = dictNext(di)) != NULL) { //
        clusterNode *node = dictGetVal(de);
        now = mstime(); /* Use an updated time at every iteration. */

        // 距离上次ping对方已经很久了,且一直没有收到回应,释放连接(下一次cron看到没有连接了又会尝试重新连接)
        mstime_t ping_delay = now - node->ping_sent;
        mstime_t data_delay = now - node->data_received;
        if (node->link && /* is connected */
            now - node->link->ctime >
            server.cluster_node_timeout && /* was not already reconnected */
            node->ping_sent && /* we already sent a ping */
            node->pong_received < node->ping_sent && /* still waiting pong */
            /* and we are waiting for the pong more than timeout/2 */
            ping_delay > server.cluster_node_timeout/2 &&
            /* and in such interval we are not seeing any traffic at all. */
            data_delay > server.cluster_node_timeout/2)
        {
            /* Disconnect the link, it will be reconnected automatically. */
            freeClusterLink(node->link);
        }

        // 判断最后一次与目标节点有联系的时间
        mstime_t node_delay = (ping_delay < data_delay) ? ping_delay :
                                                          data_delay;
        // 最后一次联系时间大于timeout时,我们需要在本地将目标机器的状态设置为`CLUSTER_NODE_PFAIL`(可能fail了)
        if (node_delay > server.cluster_node_timeout) {
            if (!(node->flags & (CLUSTER_NODE_PFAIL|CLUSTER_NODE_FAIL))) {
                // ...
                node->flags |= CLUSTER_NODE_PFAIL;
                update_state = 1;
            }
        }
    }
    dictReleaseIterator(di);
    // ...
}
```
#### 节点收到`ping`时更新本地该节点的信息,并`pong`回一些知道的gossip信息
```c
// src/cluster.c
int clusterProcessPacket(clusterLink *link) {
    
    uint16_t type = ntohs(hdr->type);
    mstime_t now = mstime();
    sender = clusterLookupNode(hdr->sender);
    // 更新sender节点data_received的时间
    if (sender) sender->data_received = now;

    if (type == CLUSTERMSG_TYPE_PING || type == CLUSTERMSG_TYPE_MEET) {
        //...
        // 返回一个pong信息(这个pong消息会带上一些已知的gossip消息,包括“我”认为哪个节点可能fail了)
        clusterSendPing(link,CLUSTERMSG_TYPE_PONG);
    }
    if (type == CLUSTERMSG_TYPE_PING || type == CLUSTERMSG_TYPE_PONG ||
        type == CLUSTERMSG_TYPE_MEET)
    {
        // 更新sender节点的flags,后面gossip传播的也时这个状态
        if (sender) {
            int nofailover = flags & CLUSTER_NODE_NOFAILOVER;
            sender->flags &= ~CLUSTER_NODE_NOFAILOVER;
            sender->flags |= nofailover;
        }
    }
}
```
#### 节点收到`pong`消息
```c
// src/cluster.c
int clusterProcessPacket(clusterLink *link) {
    
    uint16_t type = ntohs(hdr->type);
    mstime_t now = mstime();
    sender = clusterLookupNode(hdr->sender);
    // 更新sender节点data_received的时间
    if (sender) sender->data_received = now;

    if (type == CLUSTERMSG_TYPE_PING || type == CLUSTERMSG_TYPE_PONG ||
        type == CLUSTERMSG_TYPE_MEET)
    {
        // 更新sender节点的flags,后面gossip传播的也时这个状态
        if (sender) {
            int nofailover = flags & CLUSTER_NODE_NOFAILOVER;
            sender->flags &= ~CLUSTER_NODE_NOFAILOVER;
            sender->flags |= nofailover;
        }

        // 更新pong_received的时间
        if (link->node && type == CLUSTERMSG_TYPE_PONG) {
            link->node->pong_received = now;
            link->node->ping_sent = 0;

            // ... 收到了目标节点的pong消息,可以大小对该节点的`CLUSTER_NODE_PFAIL`状态的疑虑了
            if (nodeTimedOut(link->node)) {
                link->node->flags &= ~CLUSTER_NODE_PFAIL;
                clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|
                                     CLUSTER_TODO_UPDATE_STATE);
            } else if (nodeFailed(link->node)) {
                clearNodeFailureIfNeeded(link->node);
            }
        }
        // ...
        // 处理sender发过来的gossip消息
        if (sender) clusterProcessGossipSection(hdr,link);
    }
}

void clusterProcessGossipSection(clusterMsg *hdr, clusterLink *link) {
    uint16_t count = ntohs(hdr->count);
    clusterMsgDataGossip *g = (clusterMsgDataGossip*) hdr->data.ping.gossip;
    clusterNode *sender = link->node ? link->node : clusterLookupNode(hdr->sender);

    while(count--) {
        uint16_t flags = ntohs(g->flags);
        clusterNode *node;
        sds ci;

        node = clusterLookupNode(g->nodename);
        if (node) {
            if (sender && nodeIsMaster(sender) && node != myself) {
                // 听到关于某个节点可能失败,或已经失败
                if (flags & (CLUSTER_NODE_FAIL|CLUSTER_NODE_PFAIL)) {
                    // 更新收到的关于该节点的”投诉“(即有哪些节点都报告了同一个节点的fail信息),
                    if (clusterNodeAddFailureReport(node,sender)) {

                    }
                    // 当对该节点的”投诉“超过某个值时,需要正式标记他“fail” TODO
                    markNodeAsFailingIfNeeded(node);
                } else { // 听说某个节点很正常,需要消除掉同一个sender以前报告这个节点的“fail”信息
                    if (clusterNodeDelFailureReport(node,sender)) {
                        
                    }
                }
            }

            /* If from our POV the node is up (no failure flags are set),
             * we have no pending ping for the node, nor we have failure
             * reports for this node, update the last pong time with the
             * one we see from the other nodes. */
            if (!(flags & (CLUSTER_NODE_FAIL|CLUSTER_NODE_PFAIL)) &&
                node->ping_sent == 0 &&
                clusterNodeFailureReportsCount(node) == 0)
            {
                mstime_t pongtime = ntohl(g->pong_received);
                pongtime *= 1000; /* Convert back to milliseconds. */

                /* Replace the pong time with the received one only if
                 * it's greater than our view but is not in the future
                 * (with 500 milliseconds tolerance) from the POV of our
                 * clock. */
                if (pongtime <= (server.mstime+500) &&
                    pongtime > node->pong_received)
                {
                    node->pong_received = pongtime;
                }
            }

            /* If we already know this node, but it is not reachable, and
             * we see a different address in the gossip section of a node that
             * can talk with this other node, update the address, disconnect
             * the old link if any, so that we'll attempt to connect with the
             * new address. */
            if (node->flags & (CLUSTER_NODE_FAIL|CLUSTER_NODE_PFAIL) &&
                !(flags & CLUSTER_NODE_NOADDR) &&
                !(flags & (CLUSTER_NODE_FAIL|CLUSTER_NODE_PFAIL)) &&
                (strcasecmp(node->ip,g->ip) ||
                 node->port != ntohs(g->port) ||
                 node->cport != ntohs(g->cport)))
            {
                if (node->link) freeClusterLink(node->link);
                memcpy(node->ip,g->ip,NET_IP_STR_LEN);
                node->port = ntohs(g->port);
                node->cport = ntohs(g->cport);
                node->flags &= ~CLUSTER_NODE_NOADDR;
            }
        } else {
            /* If it's not in NOADDR state and we don't have it, we
             * add it to our trusted dict with exact nodeid and flag.
             * Note that we cannot simply start a handshake against
             * this IP/PORT pairs, since IP/PORT can be reused already,
             * otherwise we risk joining another cluster.
             *
             * Note that we require that the sender of this gossip message
             * is a well known node in our cluster, otherwise we risk
             * joining another cluster. */
            if (sender &&
                !(flags & CLUSTER_NODE_NOADDR) &&
                !clusterBlacklistExists(g->nodename))
            {
                clusterNode *node;
                node = createClusterNode(g->nodename, flags);
                memcpy(node->ip,g->ip,NET_IP_STR_LEN);
                node->port = ntohs(g->port);
                node->cport = ntohs(g->cport);
                clusterAddNode(node);
            }
        }

        /* Next node */
        g++;
    }
}

```

## 自由slave
## configEpoch 与 冲突解决
