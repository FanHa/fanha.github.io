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

```c
// src/ziplist.c
unsigned char *ziplistNew(void) {
    // 没用用上C语言的结构,直接通过字符串和位移来表示一个list,省空间
    unsigned int bytes = ZIPLIST_HEADER_SIZE+ZIPLIST_END_SIZE;
    unsigned char *zl = zmalloc(bytes);
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    ZIPLIST_LENGTH(zl) = 0;
    zl[bytes-1] = ZIP_END;
    return zl;
}
```

```c
// src/ziplist.c
// 往ziplist 里 push一个值其实就是在字符串里进行操作
unsigned char *ziplistPush(unsigned char *zl, unsigned char *s, unsigned int slen, int where) {
    unsigned char *p;
    p = (where == ZIPLIST_HEAD) ? ZIPLIST_ENTRY_HEAD(zl) : ZIPLIST_ENTRY_END(zl);
    return __ziplistInsert(zl,p,s,slen);
}

unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen;
    unsigned int prevlensize, prevlen = 0;
    size_t offset;
    int nextdiff = 0;
    unsigned char encoding = 0;
    long long value = 123456789; /* initialized to avoid warning. Using a value
                                    that is easy to see if for some reason
                                    we use it uninitialized. */
    zlentry tail;

    /* Find out prevlen for the entry that is inserted. */
    if (p[0] != ZIP_END) {
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
    } else {
        unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
        if (ptail[0] != ZIP_END) {
            prevlen = zipRawEntryLength(ptail);
        }
    }

    /* See if the entry can be encoded */
    if (zipTryEncoding(s,slen,&value,&encoding)) {
        /* 'encoding' is set to the appropriate integer encoding */
        reqlen = zipIntSize(encoding);
    } else {
        /* 'encoding' is untouched, however zipStoreEntryEncoding will use the
         * string length to figure out how to encode it. */
        reqlen = slen;
    }
    /* We need space for both the length of the previous entry and
     * the length of the payload. */
    reqlen += zipStorePrevEntryLength(NULL,prevlen);
    reqlen += zipStoreEntryEncoding(NULL,encoding,slen);

    /* When the insert position is not equal to the tail, we need to
     * make sure that the next entry can hold this entry's length in
     * its prevlen field. */
    int forcelarge = 0;
    nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;
    if (nextdiff == -4 && reqlen < 4) {
        nextdiff = 0;
        forcelarge = 1;
    }

    /* Store offset because a realloc may change the address of zl. */
    offset = p-zl;
    zl = ziplistResize(zl,curlen+reqlen+nextdiff);
    p = zl+offset;

    /* Apply memory move when necessary and update tail offset. */
    if (p[0] != ZIP_END) {
        /* Subtract one because of the ZIP_END bytes */
        memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

        /* Encode this entry's raw length in the next entry. */
        if (forcelarge)
            zipStorePrevEntryLengthLarge(p+reqlen,reqlen);
        else
            zipStorePrevEntryLength(p+reqlen,reqlen);

        /* Update offset for tail */
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

        /* When the tail contains more than one entry, we need to take
         * "nextdiff" in account as well. Otherwise, a change in the
         * size of prevlen doesn't have an effect on the *tail* offset. */
        zipEntry(p+reqlen, &tail);
        if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
        }
    } else {
        /* This element will be the new tail. */
        ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
    }

    /* When nextdiff != 0, the raw length of the next entry has changed, so
     * we need to cascade the update throughout the ziplist */
    if (nextdiff != 0) {
        offset = p-zl;
        zl = __ziplistCascadeUpdate(zl,p+reqlen);
        p = zl+offset;
    }

    /* Write the entry */
    p += zipStorePrevEntryLength(p,prevlen);
    p += zipStoreEntryEncoding(p,encoding,slen);
    if (ZIP_IS_STR(encoding)) {
        memcpy(p,s,slen);
    } else {
        zipSaveInteger(p,value,encoding);
    }
    ZIPLIST_INCR_LENGTH(zl,1);
    return zl;
}

```
#### ziplist 的压缩约解压 todo

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
    if (added) {
        // todo??
        signalModifiedKey(c,c->db,c->argv[1]);
        notifyKeyspaceEvent(NOTIFY_SET,"sadd",c->argv[1],c->db->id);
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
            // Set的元素值其实是给dict的hash表设一个Key,并不需要设置这个key的值;
            dictSetKey(ht,de,sdsdup(value));
            dictSetVal(ht,de,NULL);
            return 1;
        }
    } else if (subject->encoding == OBJ_ENCODING_INTSET) {
        if (isSdsRepresentableAsLongLong(value,&llval) == C_OK) {
            uint8_t success = 0;
            subject->ptr = intsetAdd(subject->ptr,llval,&success);
            if (success) {
                /* Convert to regular set when the intset contains
                 * too many entries. */
                if (intsetLen(subject->ptr) > server.set_max_intset_entries)
                    setTypeConvert(subject,OBJ_ENCODING_HT);
                return 1;
            }
        } else {
            /* Failed to get integer from object, convert to regular set. */
            setTypeConvert(subject,OBJ_ENCODING_HT);

            /* The set *was* an intset and this value is not integer
             * encodable, so dictAdd should always work. */
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

    /* Get the index of the new element, or -1 if
     * the element already exists. */
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;

    /* Allocate the memory and store the new entry.
     * Insert the element in top, with the assumption that in a database
     * system it is more likely that recently added entries are accessed
     * more frequently. */
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    entry = zmalloc(sizeof(*entry));
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    /* Set the hash entry fields. */
    dictSetKey(d, entry, key);
    return entry;
}

static void _dictRehashStep(dict *d) {
    // 如果当前没有人(或线程)在使用这个表,才会真正调用dictRehash
    // TODO Rehash
    if (d->iterators == 0) dictRehash(d,1);
}
```


