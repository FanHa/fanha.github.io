## 版本
+ Github:redis/redis 
+ 分支:6.0 
+ 版本号:6.0.7

## 用途
为了节省空间,redis的list结构有时不使用C本身的结构,而使用字符串存储list的数据,然后提供一系列方法封装了对该“list”元素的增删改查操作

## 数据结构
```c
// src/ziplist.c
unsigned char *ziplistNew(void) {
    // 没有用上C语言的结构,直接通过字符串和位移来表示一个list,省空间
    // 数据结构 [byte][TailOffset][zlen][zdata][end]

    // 初始化时没有数据,字符串长度bytes就等于HeaderSize + EndSize
    unsigned int bytes = ZIPLIST_HEADER_SIZE+ZIPLIST_END_SIZE;
    // 分配空间
    unsigned char *zl = zmalloc(bytes);
    // 在字符串的byte位置写入byte值
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    // 因为list中目前没有元素,直接在TailOffset位置直接写入HeaderSize
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    // 数据长度初始化为0
    ZIPLIST_LENGTH(zl) = 0;
    // end位置为0
    zl[bytes-1] = ZIP_END;
    return zl;
}
```

## 数据操作方法
### push
```c
// src/ziplist.c
// 往ziplist 里 push一个值其实就是在字符串里进行操作
unsigned char *ziplistPush(unsigned char *zl, unsigned char *s, unsigned int slen, int where) {
    unsigned char *p;
    // 找到需要插入值的位置(offset)
    p = (where == ZIPLIST_HEAD) ? ZIPLIST_ENTRY_HEAD(zl) : ZIPLIST_ENTRY_END(zl);
    return __ziplistInsert(zl,p,s,slen);
}

unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    // 初始化当前字符串总长度curlen, 需要为新元素新增的空间reqlen
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen;
    unsigned int prevlensize, prevlen = 0;
    size_t offset;
    int nextdiff = 0;
    unsigned char encoding = 0;
    long long value = 123456789; 
    zlentry tail;

    // 前面ziplist字符串结构 的 zdata部分是由一个个的enrty来表示具体的元素,
    // [byte][TailOffset][zlen][entry]...[entry][end]
    // 每个enrty的结构[prevlen][len][data]
    if (p[0] != ZIP_END) { // 要插入元素的位置不在list尾部时算出prevlen的方式 TODO 这里的prevlen和很多其他位置的占位长度其实是可变的
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
    } else {
        // 要插入的元素在list尾部时算出prevlen的方式
        unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
        if (ptail[0] != ZIP_END) {
            prevlen = zipRawEntryLength(ptail);
        }
    }

    // 尝试对要保存的元素是否压缩,比如一个字符串“1234567”可压缩保存成一个int型,节约空间
    if (zipTryEncoding(s,slen,&value,&encoding)) {
        reqlen = zipIntSize(encoding);
    } else {
        reqlen = slen;
    }
    // 数据大小+prevlen+数据长度 = 新增元素需要的空间
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
