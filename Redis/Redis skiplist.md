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
    double score;   // 元素Score
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
/* Create a new skiplist. */
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
    for (i = zsl->level-1; i >= 0; i--) { // 从当前最高层开始
        /* store rank that is crossed to reach the insert position */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    
    level = zslRandomLevel(); // 生成一个随机的层数
    if (level > zsl->level) { // 当随机层数大于当前skiplist的最高层数时
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
    x = zslCreateNode(level,score,ele);
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    /* increment span for untouched levels */
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```
