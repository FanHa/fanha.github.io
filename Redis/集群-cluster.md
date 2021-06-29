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
- 集群初始化流程
    - 收到cluster meet命令处理,设置好要加入的集群的目标机器信息
    - clusterCron 根据前面设置好的目标机器信息连接目标机器
    - 目标机器接收meet消息,将机器加入本地集群信息,并pong回自己视角的集群信息(当机器多时,gossip机制,返回部分)
    - 接收目标机器的pong,更新本地集群信息
    - 集群根据已知信息相互ping,pong,发送gossip消息,使得集群中的机器最终都对整个集群信息有了解 TODO
        - ping TODO
        - pong TODO
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