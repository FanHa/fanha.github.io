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

        /* If releasing the object is too much work, do it in the background
         * by adding the object to the lazy free list.
         * Note that if the object is shared, to reclaim it now it is not
         * possible. This rarely happens, however sometimes the implementation
         * of parts of the Redis core may call incrRefCount() to protect
         * objects, and then call dbDelete(). In this case we'll fall
         * through and reach the dictFreeUnlinkedEntry() call, that will be
         * equivalent to just calling decrRefCount(). */
        if (free_effort > LAZYFREE_THRESHOLD && val->refcount == 1) {
            atomicIncr(lazyfree_objects,1);
            bioCreateBackgroundJob(BIO_LAZY_FREE,val,NULL,NULL);
            dictSetVal(db->dict,de,NULL);
        }
    }

    /* Release the key-val pair, or just the key if we set the val
     * field to NULL in order to lazy free it later. */
    if (de) {
        dictFreeUnlinkedEntry(db->dict,de);
        if (server.cluster_enabled) slotToKeyDel(key->ptr);
        return 1;
    } else {
        return 0;
    }
}
```
### setExpire