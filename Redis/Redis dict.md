## 版本
+ Github:redis/redis 
+ 分支:6.0 
+ 版本号:6.0.7

## 数据结构
dict是redis的内部数据结构,对Hash表的一层封装,redis的set(zset)和全局键值的存放内部实现都是使用的dict
```c
// src/dict.h
typedef struct dict {
    dictType *type; // 类型
    void *privdata; // ?
    dictht ht[2]; // 两个Hash结构,只有一个结构提供服务,
                  // dict的Hash不是固定的,Hash表里的值少时,Hash值的“slot”数量较少,减少空间,
                  // 随着Hash表里保存的值越来越多,需要增加Hash值的“slot”的数量并重新Hash,这个过程是增量分阶段完成的,
                  // 两个表就可以一边ReHash到新结构,一边继续用旧结构提供服务,知道ReHash完成,切换服务到新结构
    long rehashidx; // Rehash过程一定到了哪个index了,-1代表不在rehash状态
    unsigned long iterators; // 正在使用当前dict结构的iterotor数量
} dict;

//dict 类型, 通过封装一系列的hash基本操作方法来实现类似其他面向对象语言(如C++)的多态
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;

// Hash表
typedef struct dictht {
    dictEntry **table; // 指针数组,数组的每一个值都是指向一个dictEntry的指针
    unsigned long size; // Hash表的“slot”的个数
    unsigned long sizemask; // 用来计算Hash值的掩码
    unsigned long used; // Hash表已使用的“slot”的个数,当used 和 size的比例达到某个值时会出发ReHash,即需要扩大或缩小size,节约存储空间
} dictht;

// 存放Hash表的key的结构
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
```

## 基本操作
### dictCreate
```c
// src/dict.c
dict *dictCreate(dictType *type,
        void *privDataPtr)
{   
    // 分配空间
    dict *d = zmalloc(sizeof(*d));
    // 初始化dict
    _dictInit(d,type,privDataPtr);
    return d;
}

int _dictInit(dict *d, dictType *type,
        void *privDataPtr)
{   
    // 初始化两个Hash表
    _dictReset(&d->ht[0]);
    _dictReset(&d->ht[1]);
    // 将用来区别不同类型的dict的type结构设置为属性(实现多态)
    d->type = type;
    d->privdata = privDataPtr; // TODO ?
    d->rehashidx = -1;
    d->iterators = 0;
    return DICT_OK;
}

static void _dictReset(dictht *ht)
{
    ht->table = NULL;
    ht->size = 0; // Hash表的“slot”的个数
    ht->sizemask = 0; // 用来计算Hash值的掩码
    ht->used = 0; // Hash表已使用的“slot”的个数,当used 和 size的比例达到某个值时会出发ReHash,即需要扩大或缩小size,节约存储空间
}
```

### dictAdd
```c
// src/dict.c
int dictAdd(dict *d, void *key, void *val)
{   
    // dict里的key 和 val是分开存放的
    dictEntry *entry = dictAddRaw(d,key,NULL);

    if (!entry) return DICT_ERR;
    // 存放val
    dictSetVal(d, entry, val);
    return DICT_OK;
}

dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    long index;
    dictEntry *entry;
    dictht *ht;

    // 当dict正在ReHash时,先执行reHash的步骤
    if (dictIsRehashing(d)) _dictRehashStep(d);

    // 查找当前key是否已存在,不存在则返回一个index,
    // 后面可以用这个index找到dict预备给当前key的位置,并把新建的保存这个key的数据结构插入这个位置
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;

    // Hash表正在Rehash时,要插入的值往ht[1]里走;
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];

    // 初始化一个存放键值对的entry
    entry = zmalloc(sizeof(*entry));

    // 放在同Hash值的slot的最前面
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    // 将Hash表 key的Hash值所代表的slot 的entry->key属性设置为*key
    dictSetKey(d, entry, key);
    return entry;
}
```

#### dictEntry 存放键值对
```c
// src/dict.h
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next; //当不同键的Hash值一样时需要用链表串起来
} dictEntry;
```

#### _dictKeyIndex 判断hash表中是否已存在key
```c
// src/dict.c
// 第一个参数d是要搜寻的dict,第二个是要查找的key,第三个是key的hash值(存在key不同,hash值相同的情况)
static long _dictKeyIndex(dict *d, const void *key, uint64_t hash, dictEntry **existing)
{
    unsigned long idx, table;
    dictEntry *he;
    if (existing) *existing = NULL;

    // 判断当前的dict结构需不需要扩张,比如dict里的键多了需要rehash就是在这里改变rehashidx属性的值
    if (_dictExpandIfNeeded(d) == DICT_ERR)
        return -1;
    // 当Hash结构正处在“Rehash”状态时,两个ht都需要遍历,如果没有处在“Rehash”状态,则在遍历完table0后会break
    for (table = 0; table <= 1; table++) {
        idx = hash & d->ht[table].sizemask;
        // 判断当前表的Hash值所在的slot是否已经已存在key
        he = d->ht[table].table[idx];
        while(he) {
            // 如果当前slot已存在key,还需要再比对可以是否与参数中的key相同,两者都形同才能认为是当前dict已经存在了参数key
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                if (existing) *existing = he;
                return -1;
            }
            // key的hash值存在,但key不存在的情形,需要继续遍历当前slot
            he = he->next;
        }
        // 当当前dict并没有在Rehash阶段,不需要再去搜索ht[1]了
        if (!dictIsRehashing(d)) break;
    }
    // 返回当前key在dict的idx索引值
    return idx;
}
```

#### dictSetKey 往entry结构里写入key
```c
// src/dict.h
#define dictSetKey(d, entry, _key_) do { \
    if ((d)->type->keyDup) \
        (entry)->key = (d)->type->keyDup((d)->privdata, _key_); \
    else \
        (entry)->key = (_key_); \
} while(0)
```

#### dictSetVal 往entry结构里写入val
```c
// src/dict.h
#define dictSetVal(d, entry, _val_) do { \
    if ((d)->type->valDup) \
        (entry)->v.val = (d)->type->valDup((d)->privdata, _val_); \
    else \
        (entry)->v.val = (_val_); \
} while(0)
```

#### _dictRehashStep ReHash表
```c
// src/dict.c
// 当前Hash结构没有在使用时才执行Rehash
static void _dictRehashStep(dict *d) {
    if (d->iterators == 0) dictRehash(d,1);
}

// rehash是增量的,第二个参数n控制一次Rehash最多移动n个slot内的元素(一个slot可能含多个元素).
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    if (!dictIsRehashing(d)) return 0;

    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;
        // Rehash从地位slot往高位slot推进,直到找到存在键值对的slot
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        // 将该slot下所有的entry都移动到d->ht[1]
        while(de) {
            uint64_t h;
            nextde = de->next;
            // 重新计算Hash值
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    // 判断是否Rehash完了整个表
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```

#### _dictExpandIfNeeded 判断当前Hash结构是否不要扩张
```c
// src/dict.c
static int _dictExpandIfNeeded(dict *d)
{
    // Hash表已经在“Rehash”状态中了
    if (dictIsRehashing(d)) return DICT_OK;

    // Hash结构是空时,需要初始化
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    // Hash表中的元素数量超过了某个值时,需要扩张hash表
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2);
    }
    return DICT_OK;
}

int dictExpand(dict *d, unsigned long size)
{
    dictht n; /* the new hash table */
    // 为了计算方便取个合适的resize
    unsigned long realsize = _dictNextPower(size);

    // 初始化Hashtable
    n.size = realsize;
    n.sizemask = realsize-1; 
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    n.used = 0;

    // 第一次初始化Hash表时,ht[0]还是空的,直接把初始化的Hash表放在ht[0]下
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }

    // 已有ht[0]时,进入Rehash状态
    d->ht[1] = n;
    d->rehashidx = 0;
    return DICT_OK;
}
```

