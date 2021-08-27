## 版本
+ Github:redis/redis 
+ 分支:6.0 
+ 版本号:6.0.7

## redis数据类型(外部)
+ 常用的 string,list,set,zset,hash
+ 特殊的 module,stream
```c
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */

#define OBJ_MODULE 5    /* Module object. */
#define OBJ_STREAM 6    /* Stream object. */
```

## Redis数据结构
Redis的数据都保存在一个redisObject的结构中,
```c
// src/server.h
typedef struct redisObject {
    unsigned type:4;    // type就是前面的数据类型
    unsigned encoding:4;
    unsigned lru:LRU_BITS;
    int refcount;
    void *ptr;  // 真实数据通过指针ptr指向不同类型的数据
} robj;
```

```c
// src/object.c
// robj 数据通过createObject创建,type即为要创建的数据结构类型,ptr就是指向实际数据的指针
robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));
    o->type = type;
    o->encoding = OBJ_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;

    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }
    return o;
}
```
### string
使用最基本的命令 “set a b”,则会创建一个基本的string数据类型
```c
// src/t_string.c
void setCommand(client *c) {
    int j;
    robj *expire = NULL;
    int unit = UNIT_SECONDS;
    int flags = OBJ_SET_NO_FLAGS;

    for (j = 3; j < c->argc; j++) {
        // 一些set衍生命令的判断
        // ...
    }
    // 创建一个数据结构来编码压缩存放值
    c->argv[2] = tryObjectEncoding(c->argv[2]);
    setGenericCommand(c,flags,c->argv[1],c->argv[2],expire,unit,NULL,NULL);
}
```
```c
robj *tryObjectEncoding(robj *o) {
    long value;
    sds s = o->ptr;
    size_t len;

    len = sdslen(s);
    if (len <= 20 && string2l(s,len,&value)) {
        // 当字符串短到可以用一个long结构表示时,转为long
        if ((server.maxmemory == 0 ||
            !(server.maxmemory_policy & MAXMEMORY_FLAG_NO_SHARED_INTEGERS)) &&
            value >= 0 &&
            value < OBJ_SHARED_INTEGERS)
        {   
            // 字符串转变为一个long后,如果满足条件,可以进一步把这个这个数字保存在一个共享空间中,不必为每个相同的数字保留一份独立的数据空间
            decrRefCount(o);
            incrRefCount(shared.integers[value]);
            return shared.integers[value];
        } else {
            if (o->encoding == OBJ_ENCODING_RAW) {
                // 如果本身的encoding是OBJ_ENCODING_RAW,
                // 更改相应的编码信息属性
                sdsfree(o->ptr);
                o->encoding = OBJ_ENCODING_INT;
                o->ptr = (void*) value;
                return o;
            } else if (o->encoding == OBJ_ENCODING_EMBSTR) {    
                // 如果本身的encoding属性是OBJ_ENCODING_EMBSTR
                // 新建一个String结构
                decrRefCount(o);
                return createStringObjectFromLongLongForValue(value);
            }
        }
    }

    // 如果字符串不满足转变成long类型,但还是比较短,满足EmbeddedString的条件
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT) {
        robj *emb;

        if (o->encoding == OBJ_ENCODING_EMBSTR) return o;
        emb = createEmbeddedStringObject(s,sdslen(s));
        decrRefCount(o);
        return emb;
    }

    // 字符串实在太长,就不挣扎了,trim一下空格就好
    trimStringObjectIfNeeded(o);

    /* Return the original object. */
    return o;
}

```

### list
使用命令“lpush a b”,如果redis本身不存在a,则会创建一个list
```c
// src/object.c
robj *createQuicklistObject(void) {
    // 先创建一个内部数据结构quicklist ,然后把该list作为参数,创建一个包裹该list的robj
    quicklist *l = quicklistCreate();
    robj *o = createObject(OBJ_LIST,l);
    o->encoding = OBJ_ENCODING_QUICKLIST; // OBJ_ENCODING_QUICKLIST是list在redis内部的表示名称
    return o;
}
```

#### quiklist
```c
// src/quicklist.h
// quicklist也是一个包裹,负责记录一些有关list的整体信息,具体list内的信息是通过quicklistNode节点保存的
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned long len;          /* number of quicklistNodes */
    int fill : QL_FILL_BITS;              /* fill factor for individual nodes */
    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;

quicklist *quicklistCreate(void) {
    struct quicklist *quicklist;
    quicklist = zmalloc(sizeof(*quicklist));
    quicklist->head = quicklist->tail = NULL;
    quicklist->len = 0;
    quicklist->count = 0;
    quicklist->compress = 0;
    quicklist->fill = -2;
    quicklist->bookmark_count = 0;
    return quicklist;
}
```

#### 
已有list后向list通过`listTypePush`增加element
```c
// src/t_list.c
void listTypePush(robj *subject, robj *value, int where) {
    if (subject->encoding == OBJ_ENCODING_QUICKLIST) {
        // 参数where用来表示是从list的head加入还是tail加入
        int pos = (where == LIST_HEAD) ? QUICKLIST_HEAD : QUICKLIST_TAIL;
        value = getDecodedObject(value);
        size_t len = sdslen(value->ptr);
        // 调用quicklistPush 往robj里的quiklist结构增加element
        quicklistPush(subject->ptr, value->ptr, len, pos);
        decrRefCount(value);
    } else {
        serverPanic("Unknown list encoding");
    }
}
```

```c
// src/quicklist.c
void quicklistPush(quicklist *quicklist, void *value, const size_t sz,
                   int where) {
    // 根据 ‘where’字段调用是从队列尾还是队列头增加一个新节点
    if (where == QUICKLIST_HEAD) {
        quicklistPushHead(quicklist, value, sz);
    } else if (where == QUICKLIST_TAIL) {
        quicklistPushTail(quicklist, value, sz);
    }
}

// 从队列头增加新节点
int quicklistPushHead(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *orig_head = quicklist->head;
    
    if (likely(
            _quicklistNodeAllowInsert(quicklist->head, quicklist->fill, sz))) {
        // 判断quicklist队列头的那个quicklistNode是否还能插入值,
        // 如果还未满则在队列头的node缩代表的ziplist里插入值
        quicklist->head->zl =
            ziplistPush(quicklist->head->zl, value, sz, ZIPLIST_HEAD);
        // 更新quicklistNode的信息
        quicklistNodeUpdateSz(quicklist->head);
    } else {
        // 如果quicklist队列头的node已经满了(或者不存在),
        // 则需要新建一个quicklistNode,然后把值插入这个新的quicklistNode内部的ziplist;
        quicklistNode *node = quicklistCreateNode();
        // 新建一个ziplist,
        // 将quicklistNode的zl指针指向这个新增的ziplist,
        // 将要插入的值插入ziplist;
        node->zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_HEAD);
        // 更新quicklistNode信息
        quicklistNodeUpdateSz(node);
        // 将quicklistNode插入quicklist的队列头
        _quicklistInsertNodeBefore(quicklist, quicklist->head, node);
    }
    // 更新quicklist的信息
    quicklist->count++;
    quicklist->head->count++;
    return (orig_head != quicklist->head);
}

int quicklistPushTail(quicklist *quicklist, void *value, size_t sz) {
    // 与quicklistPushHead类似,只是从另一段新增节点
}
```

#### quicklistNode 与 ziplist
由上面的插入节点逻辑可知redis的quicklist不是一个一维的队列,而是一个二维队列,即一个quicklist由若干个quicklistNode节点而成,每个quicklistNode包含了一个ziplist,实际往队列里push的值都是保存为ziplist的节点;

```c
// src/quicklist.h
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;  // 用一个字符串指针来表示ziplist,是因为ziplist并不是一个传统意义上有定义的C结构,而是把ziplist这个队列的所有内容压缩成了一个字符串,然后通过一系列“ziplist”操作接口来实现ziplist的增和减;
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;
```

### set
创建一个set的基本命令
```c
// src/t_set.c
void saddCommand(client *c) {
    robj *set;
    int j, added = 0;

    set = lookupKeyWrite(c->db,c->argv[1]);
    if (set == NULL) {
        // 创建set接口
        set = setTypeCreate(c->argv[2]->ptr);
        dbAdd(c->db,c->argv[1],set);
    } else {
        if (set->type != OBJ_SET) {
            addReply(c,shared.wrongtypeerr);
            return;
        }
    }

    for (j = 2; j < c->argc; j++) {
        // 往set里新增元素的接口
        if (setTypeAdd(set,c->argv[j]->ptr)) added++;
    }

    server.dirty += added;
    addReplyLongLong(c,added);
}
```

#### setTypeCreate
```c
// src/t_set.c
robj *setTypeCreate(sds value) {
    // 根据不同形式的值创建intSet 或 普通set
    if (isSdsRepresentableAsLongLong(value,NULL) == C_OK)
        return createIntsetObject();
    return createSetObject();
}
```
+ 普通set
```c
// src/object.c
robj *createSetObject(void) {
    // 实际保存set内容的是个dict结构,调用dictCreate初始化Set,
    // setDictType是个数据结构,保存了与set相关的操作函数信息
    dict *d = dictCreate(&setDictType,NULL);
    robj *o = createObject(OBJ_SET,d);
    o->encoding = OBJ_ENCODING_HT;
    return o;
}
```
```c
// src/dict.h
// dict 是个封装的保存hash数据的结构
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2]; // hash表并不是一层不变的,随着冲突增加需要Rehash,用两个表方便Rehash操作
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; // 猜想是redis支持多线程后需要记录正在使用该表的“用户”数量
} dict;

```
```c
// dictType用来给不同类型但都用到dict的数据一个SAPI,反转控制,解耦合
// src/server.c
dictType setDictType = {
    dictSdsHash,               /* hash function */
    NULL,                      /* key dup */
    NULL,                      /* val dup */
    dictSdsKeyCompare,         /* key compare */
    dictSdsDestructor,         /* key destructor */
    NULL                       /* val destructor */
};
```
```c
// src/dict.c
// 创建dict结构,分配空间
dict *dictCreate(dictType *type,
        void *privDataPtr)
{   
    dict *d = zmalloc(sizeof(*d));

    _dictInit(d,type,privDataPtr);
    return d;
}

// 初始化dict结构
int _dictInit(dict *d, dictType *type,
        void *privDataPtr)
{
    _dictReset(&d->ht[0]);
    _dictReset(&d->ht[1]);
    d->type = type;
    d->privdata = privDataPtr;
    d->rehashidx = -1;
    d->iterators = 0;
    return DICT_OK;
}
```


+ Intset
```c
// Intset
robj *createIntsetObject(void) {
    // 实际保存Intset的是个intset结构
    intset *is = intsetNew();
    robj *o = createObject(OBJ_SET,is);
    o->encoding = OBJ_ENCODING_INTSET;
    return o;
}
```

#### setTypeAdd 往set里加入元素
```c
// src/t_set.c
int setTypeAdd(robj *subject, sds value) {
    long long llval;
    if (subject->encoding == OBJ_ENCODING_HT) {
        // 普通Set情况
        dict *ht = subject->ptr;
        // 找到要插入的值的位置,如果值已经存在,这个函数会返回空
        dictEntry *de = dictAddRaw(ht,value,NULL);
        if (de) {
            // Set的元素值其实是给dict的dict表设一个Key,并不需要设置这个key的值;
            dictSetKey(ht,de,sdsdup(value));
            dictSetVal(ht,de,NULL);
            return 1;
        }
    } else if (subject->encoding == OBJ_ENCODING_INTSET) {
        // 如果set本身是个Intset的情况
        if (isSdsRepresentableAsLongLong(value,&llval) == C_OK) {
            // 如果新插入的值也是Int型
            uint8_t success = 0;
            subject->ptr = intsetAdd(subject->ptr,llval,&success);
            if (success) {
                // 如果IntSet里面包含的元素过多,也需要转成普通的Set
                if (intsetLen(subject->ptr) > server.set_max_intset_entries)
                    setTypeConvert(subject,OBJ_ENCODING_HT);
                return 1;
            }
        } else {
            // 如果新插入的值不是Int型
            // 需要将整个set转成dict类型
            setTypeConvert(subject,OBJ_ENCODING_HT);

            // 按普通类型的Set插入新值
            serverAssert(dictAdd(subject->ptr,sdsdup(value),NULL) == DICT_OK);
            return 1;
        }
    } else {
        serverPanic("Unknown set encoding");
    }
    return 0;
}

```

+ dictAddRaw 找到要插入set的值的写入点(entry)
```c
// src/dict.c
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    long index;
    dictEntry *entry;
    dictht *ht;
    // 如果该dict处于Rehashing状态,则调用_dictRehashStep函数尝试Rehash
    if (dictIsRehashing(d)) _dictRehashStep(d);

    // 当要set的值已经存在时,返回null
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;

    // rehashing时要操作的区域在ht[1]
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    // 分配空间
    entry = zmalloc(sizeof(*entry));
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    // TODO 为什么这里调用了这个dictSetKey,外面还会再调用一次?
    dictSetKey(d, entry, key);
    return entry;
}

static void _dictRehashStep(dict *d) {
    // 如果当前没有人(或线程)在使用这个表,才会真正调用dictRehash
    if (d->iterators == 0) dictRehash(d,1);
}
```

### Hash
创建一个Hash结构的基本命令
```c
// src/t_hash.c
void hsetCommand(client *c) {
    int i, created = 0;
    robj *o;
    // ...
    // 查找命令中的Hash结构的key,不存在则创建
    if ((o = hashTypeLookupWriteOrCreate(c,c->argv[1])) == NULL) return;
    hashTypeTryConversion(o,c->argv,2,c->argc-1);

    // 往Hash结构里写入键值对
    for (i = 2; i < c->argc; i += 2)
        created += !hashTypeSet(o,c->argv[i]->ptr,c->argv[i+1]->ptr,HASH_SET_COPY);

    // ...
}

robj *hashTypeLookupWriteOrCreate(client *c, robj *key) {
    robj *o = lookupKeyWrite(c->db,key);
    if (o == NULL) {
        // 如果不存在当前键的数据,则需新建一个Hash结构
        o = createHashObject();
        dbAdd(c->db,key,o);
    } else {
        if (o->type != OBJ_HASH) {
            // 如果存在,但结构不是Hash,则报错
            addReply(c,shared.wrongtypeerr);
            return NULL;
        }
    }
    return o;
}
```

#### Hash结构的创建与写入
+ createHashObject(创建)
```c
// src/object.c
robj *createHashObject(void) {
    // redis对外的Hash结构的内部实现其实是个ziplist
    // 注:在刚刚开始是以ziplist形式存放,但随着Hash里的元素的增多,会相应变换数据存放方式(dict);
    unsigned char *zl = ziplistNew();
    robj *o = createObject(OBJ_HASH, zl);
    o->encoding = OBJ_ENCODING_ZIPLIST;
    return o;
}
```
+ HashTypeSet(写入)
```c
// src/t_hash.c
int hashTypeSet(robj *o, sds field, sds value, int flags) {
    int update = 0;

    if (o->encoding == OBJ_ENCODING_ZIPLIST) {
        unsigned char *zl, *fptr, *vptr;
        // 当Hash结构内部的编码方式还是ziplist形式时(通常是因为元素还比较少,直接用压缩的链表也不比用个真正的hash结构差)
        zl = o->ptr;
        fptr = ziplistIndex(zl, ZIPLIST_HEAD);
        if (fptr != NULL) {
            // 寻找链表中是否存在要设置的key
            fptr = ziplistFind(fptr, (unsigned char*)field, sdslen(field), 1);
            if (fptr != NULL) {
                // 要设置的key已经存在了,
                // 则把指针指向key的‘next’值(链表版Hash就是“键->值->键->值”这样顺序的存下来的),key的next就是指向的值
                vptr = ziplistNext(zl, fptr);
                serverAssert(vptr != NULL);
                // 设置update=1表明这次操作是更新值,不是新写入值
                update = 1;

                // 删除原值
                zl = ziplistDelete(zl, &vptr);

                // 插入新值
                zl = ziplistInsert(zl, vptr, (unsigned char*)value,
                        sdslen(value));
            }
        }

        if (!update) {
            // 如果此次操作不是更新值,需要依次在ziplist链表尾插入键和值
            zl = ziplistPush(zl, (unsigned char*)field, sdslen(field),
                    ZIPLIST_TAIL);
            zl = ziplistPush(zl, (unsigned char*)value, sdslen(value),
                    ZIPLIST_TAIL);
        }
        o->ptr = zl;

        // 检测链表ziplist的元素数量,超过某个设定值时转为传统的HashTable结构
        if (hashTypeLength(o) > server.hash_max_ziplist_entries)
            hashTypeConvert(o, OBJ_ENCODING_HT);
    } else if (o->encoding == OBJ_ENCODING_HT) {
        // 当hash结构已经是使用dict结构编码时,调用dict数据结构的操作函数做查找和插入
        dictEntry *de = dictFind(o->ptr,field);
        if (de) {
            sdsfree(dictGetVal(de));
            if (flags & HASH_SET_TAKE_VALUE) {
                dictGetVal(de) = value;
                value = NULL;
            } else {
                dictGetVal(de) = sdsdup(value);
            }
            update = 1;
        } else {
            sds f,v;
            if (flags & HASH_SET_TAKE_FIELD) {
                f = field;
                field = NULL;
            } else {
                f = sdsdup(field);
            }
            if (flags & HASH_SET_TAKE_VALUE) {
                v = value;
                value = NULL;
            } else {
                v = sdsdup(value);
            }
            dictAdd(o->ptr,f,v);
        }
    } else {
        serverPanic("Unknown hash encoding");
    }
    // ...
    return update;
}

```

### zset
创建一个zset(有序集合)的命令“zadd key 100 xixixixi”
```c
// src/t_zset.c
void zaddGenericCommand(client *c, int flags) {
    static char *nanerr = "resulting score is not a number (NaN)";
    robj *key = c->argv[1];
    robj *zobj;
    sds ele;
    double score = 0, *scores = NULL;
    int j, elements;
    int scoreidx = 0;

    zobj = lookupKeyWrite(c->db,key);
    if (zobj == NULL) {
        if (xx) goto reply_to_client; 
        // 当zset结构不存在时,根据设定的值来决定创建一个zset结构或一个ziplist伪装的压缩版本的zset结构
        if (server.zset_max_ziplist_entries == 0 ||
            server.zset_max_ziplist_value < sdslen(c->argv[scoreidx+1]->ptr))
        {
            zobj = createZsetObject();
        } else {
            zobj = createZsetZiplistObject();
        }
        dbAdd(c->db,key,zobj);
    } else {
        // 出错处理
    }

    for (j = 0; j < elements; j++) {
        double newscore;
        score = scores[j];
        int retflags = flags;

        ele = c->argv[scoreidx+1+j*2]->ptr;
        // 将命令的值和score加入到zset中
        int retval = zsetAdd(zobj, score, ele, &retflags, &newscore);
        if (retval == 0) {
            addReplyError(c,nanerr);
            goto cleanup;
        }
        // 统计信息
        if (retflags & ZADD_ADDED) added++;
        if (retflags & ZADD_UPDATED) updated++;
        if (!(retflags & ZADD_NOP)) processed++;
        score = newscore;
    }
    server.dirty += (added+updated);

reply_to_client:
    // ...

cleanup:
    // ...

```

#### createZsetObject
```c
// src/object.c
robj *createZsetObject(void) {
    zset *zs = zmalloc(sizeof(*zs));
    robj *o;
    // 初始化存放数据的普通dict结构
    zs->dict = dictCreate(&zsetDictType,NULL);
    // 这个结构应该是用来保存顺序的
    zs->zsl = zslCreate();
    o = createObject(OBJ_ZSET,zs);
    o->encoding = OBJ_ENCODING_SKIPLIST;
    return o;
}
```

```c
// src/server.h
// zset结构包裹了一个普通dict结构(用来存放元素的值,快速查找并保证值不重复)
// 一个zskiplist结构(跳跃表), 用来存放元素的值和score
typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level; // 跳跃表的层数
} zskiplist;

typedef struct zskiplistNode {
    sds ele;    // 节点元素,即zset里的元素
    double score;   // 节点的排序分
    struct zskiplistNode *backward; // 链的上一个节点位置
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 当前层的下一个节点位置
        unsigned long span; // 当前层到下一个节点的间隔
    } level[];
} zskiplistNode;
```


#### createZsetZiplistObject
```c
// src/object.c
// 一般刚开始zset里面元素少,直接用个ziplist(压缩链表)保存
robj *createZsetZiplistObject(void) {
    unsigned char *zl = ziplistNew();
    robj *o = createObject(OBJ_ZSET,zl);
    o->encoding = OBJ_ENCODING_ZIPLIST;
    return o;
}
```
#### zsetAdd
```c
// src/t_zset.c
int zsetAdd(robj *zobj, double score, sds ele, int *flags, double *newscore) {
    int incr = (*flags & ZADD_INCR) != 0;
    int nx = (*flags & ZADD_NX) != 0;
    int xx = (*flags & ZADD_XX) != 0;
    *flags = 0; /* We'll return our response flags. */
    double curscore;

    // zset 内的元素必须传入score值
    if (isnan(score)) {
        *flags = ZADD_NAN;
        return 0;
    }

    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        // 当zset的实际保存结构还是ziplist(一般此时zset里面的元素比较少)
        unsigned char *eptr;

        if ((eptr = zzlFind(zobj->ptr,ele,&curscore)) != NULL) {
            // 要插入的元素已经存在时的情况
            if (nx) {
                // nx不允许更新元素的score
                *flags |= ZADD_NOP;
                return 1;
            }

            // 准备好需要插入元素的顺序分score
            if (incr) {
                score += curscore;
                if (isnan(score)) {
                    *flags |= ZADD_NAN;
                    return 0;
                }
                if (newscore) *newscore = score;
            }

            // 调用ziplist的操作函数删除旧值,插入新值
            if (score != curscore) {
                zobj->ptr = zzlDelete(zobj->ptr,eptr);
                zobj->ptr = zzlInsert(zobj->ptr,ele,score);
                *flags |= ZADD_UPDATED;
            }
            return 1;
        } else if (!xx) {
            // 没有找到相同的元素的情况下,插入新值
            zobj->ptr = zzlInsert(zobj->ptr,ele,score);
            // 判断插入新值后的ziplist是否“超载”了,超载就需要转变成skiplist
            if (zzlLength(zobj->ptr) > server.zset_max_ziplist_entries ||
                sdslen(ele) > server.zset_max_ziplist_value)
                zsetConvert(zobj,OBJ_ENCODING_SKIPLIST);
            if (newscore) *newscore = score;
            *flags |= ZADD_ADDED;
            return 1;
        } else {
            *flags |= ZADD_NOP;
            return 1;
        }
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        zset *zs = zobj->ptr;
        zskiplistNode *znode;
        dictEntry *de;
        // 从dict结构里找寻是否已经包含要插入的元素
        de = dictFind(zs->dict,ele);
        if (de != NULL) {
            // nx参数 不允许更新元素的score
            if (nx) {
                *flags |= ZADD_NOP;
                return 1;
            }
            curscore = *(double*)dictGetVal(de);

            /* Prepare the score for the increment if needed. */
            if (incr) {
                score += curscore;
                if (isnan(score)) {
                    *flags |= ZADD_NAN;
                    return 0;
                }
                if (newscore) *newscore = score;
            }

            /* Remove and re-insert when score changes. */
            if (score != curscore) {
                znode = zslUpdateScore(zs->zsl,curscore,ele,score);
                /* Note that we did not removed the original element from
                 * the hash table representing the sorted set, so we just
                 * update the score. */
                dictGetVal(de) = &znode->score; /* Update score ptr. */
                *flags |= ZADD_UPDATED;
            }
            return 1;
        } else if (!xx) {
            // zset不存在要插入的值,
            // 调用zskiplist的插入方法zslInsert插入值和score,
            // 调用dictAdd方法插入值和score结构保证值不重复
            // 
            ele = sdsdup(ele);
            znode = zslInsert(zs->zsl,score,ele);
            serverAssert(dictAdd(zs->dict,ele,&znode->score) == DICT_OK);
            *flags |= ZADD_ADDED;
            if (newscore) *newscore = score;
            return 1;
        } else {
            *flags |= ZADD_NOP;
            return 1;
        }
    } else {
        // ...
    }
    return 0; /* Never reached. */
}

```
