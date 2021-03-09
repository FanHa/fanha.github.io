## 版本
+ Github:redis/redis 
+ 分支:6.0 
+ 版本号:6.0.7

## 用途
为了解决list结构的查找慢的问题,在list上封装一层skiplist,具体到外部结构就是zset


## 结构
```c
// src/server.h
// zset 结构包括两部分,dict 部分,和zsl部分
typedef struct zset {
    dict *dict; // dict部分用来保证zset中每个元素的score值唯一
    zskiplist *zsl; // szl部分用来存放zset中元素的值
} zset;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail; // 头尾节点
    unsigned long length; // 节点总数
    int level; // 当前跳跃表最高层级,跳跃表的每一层都是一个zskiplist
} zskiplist;

typedef struct zskiplistNode {
    sds ele;    // 元素值
    double score;   // 元素Score,用于排序
    struct zskiplistNode *backward; // 链表的下一个节点
    struct zskiplistLevel { // 层级信息,包含当前层级节点的上一个节点指针 和 间隔宽度(span)
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;
```

## 操作方法
### 初始化
```c
// src/t_zset.c
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1;
    zsl->length = 0;
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) { // 头节点的每一层都初始化
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```

### 插入
```c
// src/t_zset.c
// 这里不考虑element值重复的问题,由调用方保证
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) { // 从当前最高层开始, 找到每一层的“上一个节点”
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0))) // 每一层都顺着链表找到要插入节点的位置
        {
            rank[i] += x->level[i].span; // 累加i层的span
            x = x->level[i].forward; // 还没有找到要插入的位置,x指向链表当前节点的forward节点
        }
        update[i] = x; // 找到了待插入节点i层的位置,将要插入节点i层的“上一个节点”位置保存
    }
    
    level = zslRandomLevel(); // 生成一个随机的层数
    if (level > zsl->level) { // 当随机层数大于当前skiplist的最高层数时
        for (i = zsl->level; i < level; i++) { // 处理ziplist的header中大于level的层数
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length; // 将header中大于当前level的层数的span置换为zsl的长度
        }
        zsl->level = level; // 更新当前zsl的最高level层数
    }
    x = zslCreateNode(level,score,ele); // 创建节点
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward; // 依次将新节点的每一层的forward指向前面已经保存好的update结构中每一层的forward
        update[i]->level[i].forward = x; // update结构中每一层的forward指向新节点

        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]); // 更新新节点每一层的span
        update[i]->level[i].span = (rank[0] - rank[i]) + 1; // 更新新节点的“上一个节点”每一层的span
    }

    // 更新新节点的中那些还没有用到的层数的上一个节点的span,因为新增了一个节点,所以直接+1就行
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    // 更新新节点的backward节点
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}

// 生成随机层数
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF)) // 通过ZSKIPLIST_P 来决定有多大概率 层数+1 ,这里目前版本是写死的0.25,表明层数n+1 是 层数n 节点的1/4;
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}

// src/server.h
#define ZSKIPLIST_MAXLEVEL 32 /* Should be enough for 2^64 elements */
#define ZSKIPLIST_P 0.25      /* Skiplist P = 1/4 */
```


