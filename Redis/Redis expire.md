## 版本
+ Github:redis/redis 
+ 分支:6.0 
+ 版本号:6.0.7

## 序
redis 可以为一个键设置 expire时间,到期自动删除键

## 设置键的过期时间
```c
// src/expire.c
void expireGenericCommand(client *c, long long basetime, int unit) {
    robj *key = c->argv[1], *param = c->argv[2];

    // 根据当前时间和设置的过期时间算出什么时间 `when`过期
    long long when;
    if (getLongLongFromObjectOrReply(c, param, &when, NULL) != C_OK)
        return;
    if (unit == UNIT_SECONDS) when *= 1000;
    when += basetime;

    // 在主db中寻找键是否存在
    if (lookupKeyWrite(c->db,key) == NULL) {
        addReply(c,shared.czero);
        return;
    }

    if (checkAlreadyExpired(when)) { // 如果最终决定要过期的时间已经过了,则直接走expire流程
        robj *aux;
        
        // 根据服务器的配置决定是同步还是异步删除
        int deleted = server.lazyfree_lazy_expire ? dbAsyncDelete(c->db,key) :
                                                    dbSyncDelete(c->db,key);
        serverAssertWithInfo(c,key,deleted);
        server.dirty++;

        // 一些修改(删除)操作的hook
        // redis正处在replicate 或者 aof模式下,需要把这个删除键的操作记录下
        aux = server.lazyfree_lazy_expire ? shared.unlink : shared.del;
        rewriteClientCommandVector(c,2,aux,key);
        // 触发修改键的hook
        signalModifiedKey(c,c->db,key);
        // 触发del事件的hook
        notifyKeyspaceEvent(NOTIFY_GENERIC,"del",key,c->db->id);
        addReply(c, shared.cone);
        return;
    } else {
        // 当最终决定的过期时间还没到时
        // 设置键的过期
        setExpire(c,c->db,key,when);
        addReply(c,shared.cone);
        // 触发修改键的hook
        signalModifiedKey(c,c->db,key);
        // 触发expire事件的hook
        notifyKeyspaceEvent(NOTIFY_GENERIC,"expire",key,c->db->id);
        server.dirty++;
        return;
    }
}
```

### db的同步删除与异步删除
```c
// src/lazyfree.c
// 异步删除
#define LAZYFREE_THRESHOLD 64
int dbAsyncDelete(redisDb *db, robj *key) {
    // 先删除expires db中的信息
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);

    // 释放主dict里dictEntry中的信息,但具体存放存放数据的robj 和 dictEntry本身并没有释放,后面还需要用到
    // 注:这个dictUnlink 和 后面的 dictFreeUnlinkedEntry需要配合着用
    dictEntry *de = dictUnlink(db->dict,key->ptr);
    if (de) {
        robj *val = dictGetVal(de);
        size_t free_effort = lazyfreeGetFreeEffort(val);

        if (free_effort > LAZYFREE_THRESHOLD && val->refcount == 1) {
            atomicIncr(lazyfree_objects,1);
            bioCreateBackgroundJob(BIO_LAZY_FREE,val,NULL,NULL);
            dictSetVal(db->dict,de,NULL);
        }
    }
    if (de) {
        dictFreeUnlinkedEntry(db->dict,de);
        if (server.cluster_enabled) slotToKeyDel(key->ptr);
        return 1;
    } else {
        return 0;
    }
}

// src/db.c
// 同步删除
int dbSyncDelete(redisDb *db, robj *key) {
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);
    if (dictDelete(db->dict,key->ptr) == DICT_OK) {
        if (server.cluster_enabled) slotToKeyDel(key->ptr);
        return 1;
    } else {
        return 0;
    }
}
```
### setExpire
```c
// src/db.c
void setExpire(client *c, redisDb *db, robj *key, long long when) {
    dictEntry *kde, *de;

    // 找到key在主dict里的位置
    kde = dictFind(db->dict,key->ptr);
    serverAssertWithInfo(NULL,key,kde != NULL);
    // 在expires dict 里加入key,指向主dict里的数据位置,如果已存在key则复用
    de = dictAddOrFind(db->expires,dictGetKey(kde));
    // 设置expire 的过期时间, expire dict里的dictEntry的值就是过期时间
    dictSetSignedIntegerVal(de,when);

    int writable_slave = server.masterhost && server.repl_slave_ro == 0;
    // ...主从相关
    if (c && writable_slave && !(c->flags & CLIENT_MASTER))
        rememberSlaveKeyWithExpire(db,key);
}
```

## “惰性过期” 与 周期性主动过期
### “惰性过期”
一个key已经到了过期时间,但redis并不会立刻删除他,
而是下一次查询这个key时顺带检查一下是否过期,过期了就删除
```c
// src/t_string.c
void getCommand(client *c) {
    getGenericCommand(c);
}
int getGenericCommand(client *c) {
    robj *o;
    if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.null[c->resp])) == NULL)
        return C_OK;
    // ...
}

// src/db.c
robj *lookupKeyReadOrReply(client *c, robj *key, robj *reply) {
    robj *o = lookupKeyRead(c->db, key);
    // ...
}
robj *lookupKeyRead(redisDb *db, robj *key) {
    return lookupKeyReadWithFlags(db,key,LOOKUP_NONE);
}
robj *lookupKeyReadWithFlags(redisDb *db, robj *key, int flags) {
    robj *val;
    // ...
    // 在这里判断并删除过期的key
    if (expireIfNeeded(db,key) == 1) {
        // ...
    }
    // ...
}
```
```c
// src/db.c
int expireIfNeeded(redisDb *db, robj *key) {
    // 比对db的expire dict 里的key的值是否已经过期,没过期则返回
    if (!keyIsExpired(db,key)) return 0;

    // slave redis的键的过期由master决定,本身不触发过期
    if (server.masterhost != NULL) return 1;

    // 统计信息更新
    server.stat_expiredkeys++;
    // 传播expire动作,(aof 或者 主从时都需要把expire这个操作传播出去)
    propagateExpire(db,key,server.lazyfree_lazy_expire);
    notifyKeyspaceEvent(NOTIFY_EXPIRED,
        "expired",key,db->id);
    // 根据配置决定时同步还是异步删除数据库里的信息
    int retval = server.lazyfree_lazy_expire ? dbAsyncDelete(db,key) :
                                               dbSyncDelete(db,key);
    // 触发key 的修改hook
    if (retval) signalModifiedKey(NULL,db,key);
    return retval;
}

void propagateExpire(redisDb *db, robj *key, int lazy) {
    robj *argv[2];

    argv[0] = lazy ? shared.unlink : shared.del;
    argv[1] = key;
    incrRefCount(argv[0]);
    incrRefCount(argv[1]);

    // 正在aof时把命令附在aof文件后面
    if (server.aof_state != AOF_OFF)
        feedAppendOnlyFile(server.delCommand,db->id,argv,2);
    // 将expire动作分发到slave服务器
    replicationFeedSlaves(server.slaves,db->id,argv,2);

    decrRefCount(argv[0]);
    decrRefCount(argv[1]);
}
```

### 周期性主动过期
```c
// src/ae.c
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    // ...
    // 在redis的遍历周期调用eventLoop 的 beforesleep方法
        if (eventLoop->beforesleep != NULL && flags & AE_CALL_BEFORE_SLEEP)
            eventLoop->beforesleep(eventLoop);
    // ...
}
```
```c
// redis服务init阶段注册了eventLoop的beforeSleep方法
// src/server.c
void initServer(void) {
    // ...
    aeSetBeforeSleepProc(server.el,beforeSleep);
    // ...
}
// src/ae.c
void aeSetBeforeSleepProc(aeEventLoop *eventLoop, aeBeforeSleepProc *beforesleep) {
    eventLoop->beforesleep = beforesleep;
}
```

```c
// src/server.c
void beforeSleep(struct aeEventLoop *eventLoop) {
    // ...
    // beforeSleep 方法里根据配置触发主动过期流程
    if (server.active_expire_enabled && server.masterhost == NULL)
        activeExpireCycle(ACTIVE_EXPIRE_CYCLE_FAST);

    // ...
}
```

#### 
```c
void activeExpireCycle(int type) {
    // 设置一个effort值,限制各个流程的执行时间,避免因为expire占用了命令处理主循环太多时间
    unsigned long
    effort = server.active_expire_effort-1, /* Rescale from 0 to 9. */
    config_keys_per_loop = ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP +
                           ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP/4*effort,
    config_cycle_fast_duration = ACTIVE_EXPIRE_CYCLE_FAST_DURATION +
                                 ACTIVE_EXPIRE_CYCLE_FAST_DURATION/4*effort,
    config_cycle_slow_time_perc = ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC +
                                  2*effort,
    config_cycle_acceptable_stale = ACTIVE_EXPIRE_CYCLE_ACCEPTABLE_STALE-
                                    effort;

    // ...

    int j, iteration = 0;
    int dbs_per_call = CRON_DBS_PER_CALL; // 每次触发循环最多只遍历这么多个数据库
    long long start = ustime(), timelimit, elapsed;

    // ...
    // 避免过于频繁的开启expire流程
    if (type == ACTIVE_EXPIRE_CYCLE_FAST) {
        if (!timelimit_exit &&
            server.stat_expired_stale_perc < config_cycle_acceptable_stale)
            return;

        if (start < last_fast_cycle + (long long)config_cycle_fast_duration*2)
            return;

        last_fast_cycle = start;
    }


    if (dbs_per_call > server.dbnum || timelimit_exit)
        dbs_per_call = server.dbnum;

    // 初始化各种统计信息,在expire流程中即时更新这些信息,并见机行事
    timelimit = config_cycle_slow_time_perc*1000000/server.hz/100;
    timelimit_exit = 0;
    if (timelimit <= 0) timelimit = 1;

    if (type == ACTIVE_EXPIRE_CYCLE_FAST)
        timelimit = config_cycle_fast_duration; /* in microseconds. */
    long total_sampled = 0;
    long total_expired = 0;

    for (j = 0; j < dbs_per_call && timelimit_exit == 0; j++) {
        unsigned long expired, sampled;
        redisDb *db = server.db+(current_db % server.dbnum);

        // 在这里把要处理的数db index + 1 是为了后面在处理某个db因为各种原因跳出时,下一个循环能从新的db开始
        current_db++;

        do {
            unsigned long num, slots;
            long long now, ttl_sum;
            int ttl_samples;
            iteration++;

            // 当前db 的expire里没有东西,直接break
            if ((num = dictSize(db->expires)) == 0) {
                db->avg_ttl = 0;
                break;
            }
            slots = dictSlots(db->expires);
            now = mstime();

            // 当expire dict 里的实际数据量 大大小于 dict的slot数量时(1%),直接break循环 去处理下一个db(可能样本量太小,得不偿失)
            if (num && slots > DICT_HT_INITIAL_SIZE &&
                (num*100/slots < 1)) break;

            /* The main collection cycle. Sample random keys among keys
             * with an expire set, checking for expired ones. */
            expired = 0;
            sampled = 0;
            ttl_sum = 0;
            ttl_samples = 0;

            // 限制每次循环最多只取样这么多个key
            if (num > config_keys_per_loop)
                num = config_keys_per_loop;

            long max_buckets = num*20;
            long checked_buckets = 0;

            while (sampled < num && checked_buckets < max_buckets) {
                for (int table = 0; table < 2; table++) { // redis 的dict 有两个hashTable(便于rehash)
                    // 正在rehash状态时,直接break,猜测应该时rehash操作本身就会判断是否过期
                    if (table == 1 && !dictIsRehashing(db->expires)) break; 

                    // 从db的expires_cursor 开始扫hashtable的slot
                    unsigned long idx = db->expires_cursor;
                    idx &= db->expires->ht[table].sizemask;
                    dictEntry *de = db->expires->ht[table].table[idx];
                    long long ttl;

                    checked_buckets++;
                    // 可能有多个键的hash值相同,用链表形式存在了一个slot里,需要遍历链表
                    while(de) {
                        // 当前bucket存在值时
                        dictEntry *e = de;
                        de = de->next;

                        ttl = dictGetSignedIntegerVal(e)-now;
                        // 当键确实expire时,触发expire代码(可能同步,可能异步,但作为调用方不需要清楚细节,可以认为已经expired)
                        if (activeExpireCycleTryExpire(db,e,now)) expired++;
                        if (ttl > 0) {
                            // 统计信息更新
                            ttl_sum += ttl; // 统计没有过期的键距离过期时间的和
                            ttl_samples++; // 统计没有过期的键的数量
                        }
                        // 更新样本值,下一个循环会判断这个sampled值是不是超过了最大数,超了就不再当前db里继续作为了
                        sampled++;
                    }
                }
                // db 的expire_cursor 前移,下次就可以从这里继续开始
                db->expires_cursor++;
            }
            total_expired += expired;
            total_sampled += sampled;

            // 粗略的统计db里那些有过期时间的键距离过期时间的平均值
            if (ttl_samples) {
                long long avg_ttl = ttl_sum/ttl_samples;
                if (db->avg_ttl == 0) db->avg_ttl = avg_ttl;
                db->avg_ttl = (db->avg_ttl/50)*49 + (avg_ttl/50);
            }

            // 确保循环不回占用过多时间,超时了需要直接break(整个expire操作都在redis的主命令处理循环内,这里处理太久,会影响正常命令的响应)
            if ((iteration & 0xf) == 0) {
                elapsed = ustime()-start;
                if (elapsed > timelimit) {
                    timelimit_exit = 1;
                    server.stat_expired_time_cap_reached_count++;
                    break;
                }
            }
            /* We don't repeat the cycle for the current database if there are
             * an acceptable amount of stale keys (logically expired but yet
             * not reclaimed). */
        } while (sampled == 0 ||
                 (expired*100/sampled) > config_cycle_acceptable_stale);
    }

    // 更新统计信息
    elapsed = ustime()-start;
    server.stat_expire_cycle_time_used += elapsed;
    latencyAddSampleIfNeeded("expire-cycle",elapsed/1000);
    double current_perc;
    if (total_sampled) {
        current_perc = (double)total_expired/total_sampled;
    } else
        current_perc = 0;
    server.stat_expired_stale_perc = (current_perc*0.05)+
                                     (server.stat_expired_stale_perc*0.95);
}

```
