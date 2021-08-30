## 版本
+ Github:redis/redis 
+ 分支:6.0 
+ 版本号:6.0.7

## 数据结构
```c
// 客户数据结构
typedef struct client {
    dict *pubsub_channels;  // 该client订阅的channel们
    list *pubsub_patterns;  // 该client订阅的patter channel

}
// 全局服务器数据结构
struct redisServer {
    dict *pubsub_channels;  // channel 被哪些client订阅了
    list *pubsub_patterns;  // pattern 列表
    dict *pubsub_patterns_dict;  // patterns channel 被哪些client订阅了
}
// pattern格式
typedef struct pubsubPattern {
    client *client;
    robj *pattern;
} pubsubPattern;
```

## Subscribe 订阅
客户端订阅channel是需要与服务端建立长连接的

### subscribe 指定channel名订阅 
```c
// src/pubsub.c
void subscribeCommand(client *c) {
    int j;

    for (j = 1; j < c->argc; j++)
        pubsubSubscribeChannel(c,c->argv[j]); // 一个命令可以订阅多个channel
    c->flags |= CLIENT_PUBSUB; // 设置这个客户端的flag
}

int pubsubSubscribeChannel(client *c, robj *channel) {
    dictEntry *de;
    list *clients = NULL;
    int retval = 0;

    if (dictAdd(c->pubsub_channels,channel,NULL) == DICT_OK) { 纪录该客户端订阅了这个channel
        retval = 1;
        incrRefCount(channel);
        de = dictFind(server.pubsub_channels,channel); // 搜寻这个channel是否被“其他”client订阅过
        if (de == NULL) {
            clients = listCreate();
            dictAdd(server.pubsub_channels,channel,clients); // 没有订阅过需要新建一个list用来保存订阅这个channel的client
            incrRefCount(channel);
        } else {
            clients = dictGetVal(de);
        }
        listAddNodeTail(clients,c); // 纪录channel被client订阅
    }
    // ...
}
```
### psubscribe 模式channel 订阅 todo
```c
// src/pubsub.c
void psubscribeCommand(client *c) {
    int j;

    for (j = 1; j < c->argc; j++)
        pubsubSubscribePattern(c,c->argv[j]);
    c->flags |= CLIENT_PUBSUB;
}
int pubsubSubscribePattern(client *c, robj *pattern) {
    dictEntry *de;
    list *clients;
    int retval = 0;

    if (listSearchKey(c->pubsub_patterns,pattern) == NULL) {
        retval = 1;
        pubsubPattern *pat;
        listAddNodeTail(c->pubsub_patterns,pattern); // 将新的pattern加入到该client的pubsub_patterns list中
        incrRefCount(pattern);
        // 新建一个pubsubPattern实例
        pat = zmalloc(sizeof(*pat));
        pat->pattern = getDecodedObject(pattern);
        pat->client = c;
        listAddNodeTail(server.pubsub_patterns,pat); // 将pattern实例添加到服务器数据结构中的patter list中
        de = dictFind(server.pubsub_patterns_dict,pattern);
        if (de == NULL) {
            clients = listCreate();
            dictAdd(server.pubsub_patterns_dict,pattern,clients); // 添加被订阅的pattern
            incrRefCount(pattern);
        } else {
            clients = dictGetVal(de);
        }
        listAddNodeTail(clients,c); // 把client添加进订阅该pattern的list中
    }
    // ...
}
```

## Publish 发布
```c
// src/pubsub.c
void publishCommand(client *c) {
    int receivers = pubsubPublishMessage(c->argv[1],c->argv[2]); // 找到订阅了要发布的channel的client们,并将消息发送出去
    // ...
    addReplyLongLong(c,receivers); // 回复给消息发送者消息发给了哪些client们
}

int pubsubPublishMessage(robj *channel, robj *message) {
    int receivers = 0;
    dictEntry *de;
    dictIterator *di;
    listNode *ln;
    listIter li;

    de = dictFind(server.pubsub_channels,channel);
    if (de) {
        list *list = dictGetVal(de);
        listNode *ln;
        listIter li;

        listRewind(list,&li);
        while ((ln = listNext(&li)) != NULL) { // 遍历所有订阅了这个channel的client,将消息发送给她们
            client *c = ln->value;
            addReplyPubsubMessage(c,channel,message);
            receivers++;
        }
    }
    di = dictGetIterator(server.pubsub_patterns_dict);
    if (di) {
        channel = getDecodedObject(channel);
        while((de = dictNext(di)) != NULL) { // 遍历所有被订阅的模式(pattern channel),如果满足,则也需要把消息发送个这些client
            robj *pattern = dictGetKey(de);
            list *clients = dictGetVal(de);
            if (!stringmatchlen((char*)pattern->ptr,
                                sdslen(pattern->ptr),
                                (char*)channel->ptr,
                                sdslen(channel->ptr),0)) continue; // 匹配channel名是否满足pattern格式

            listRewind(clients,&li);
            while ((ln = listNext(&li)) != NULL) { // channel名满足pattern格式,需要把消息发送给所有订阅了该pattern的client
                client *c = listNodeValue(ln);
                addReplyPubsubPatMessage(c,pattern,channel,message);
                receivers++;
            }
        }
        decrRefCount(channel);
        dictReleaseIterator(di);
    }
    return receivers;
}

```
