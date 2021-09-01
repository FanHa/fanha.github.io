## 简介
redis 用一个sds结构包裹代替裸的C字符串

### 结构
redis根据字符串长度的大小,用途,异化了一系列类似结构的struct header来包裹实际存储的字符串
```c
// src/sds.h
typedef char *sds;

/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; // 当前字符串实际用到了多长,便于O(1)找到字符长度
    uint8_t alloc; // 当前结构分配的空间可能会大于len,减少字符串变化时频繁的内存空间分配
    unsigned char flags; // 类型
    char buf[]; // 实际存储字符的地方
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```
结构里有了len和alloc后就有了快速获得字符串长度,和当前结构剩余可用空间等一些常见方法的可能
```c
// src/sds.h
static inline size_t sdslen(const sds s) {
    // 前面结构可以看出 flags 都在实际存储字符串的buf的前面一个位置
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return SDS_TYPE_5_LEN(flags);
        case SDS_TYPE_8:
            // SDS_HDR 宏根据不同的传入数字,取不同的结构hdr
            return SDS_HDR(8,s)->len;
        case SDS_TYPE_16:
            return SDS_HDR(16,s)->len;
        case SDS_TYPE_32:
            return SDS_HDR(32,s)->len;
        case SDS_TYPE_64:
            return SDS_HDR(64,s)->len;
    }
    return 0;
}

static inline size_t sdsavail(const sds s) {
    // 同样用-1 取出实际字符串前面的 flags得到数据结构类型
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5: {
            return 0;
        }
        case SDS_TYPE_8: {
            // 同样用宏快速取出具体的机构位置
            SDS_HDR_VAR(8,s);
            return sh->alloc - sh->len;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            return sh->alloc - sh->len;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            return sh->alloc - sh->len;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            return sh->alloc - sh->len;
        }
    }
    return 0;
}

// 一些常用的通过实际字符串初始位置 快速取对应的包裹结构的宏
#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));
#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))
```

### 创建字符串
```c
// src/sds.c
sds sdsnew(const char *init) {
    // 裸的c字符串是没有直接获取长度的属性的,需要计算并保存
    size_t initlen = (init == NULL) ? 0 : strlen(init);
    return sdsnewlen(init, initlen);
}

sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    sds s;
    char type = sdsReqType(initlen); // 根据字符串实际长度选择合适的header结构存储
    // ...
    int hdrlen = sdsHdrSize(type); // 不同header结构的len不同
    unsigned char *fp; /* flags pointer. */

    sh = s_malloc(hdrlen+initlen+1); // 分配空间(header结构空间,字符串实际长度空间 + 1个结尾符号\0)
    if (sh == NULL) return NULL;
    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
        memset(sh, 0, hdrlen+initlen+1);
    s = (char*)sh+hdrlen; // 存储字符串的地址
    fp = ((unsigned char*)s)-1; // 存储sds类型的地址
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s); // 根据类型取初header结构的地址
            sh->len = initlen; // 初始化header结构的len
            sh->alloc = initlen; // 刚开始,已分配和实际len是一致的
            *fp = type; // 设置header类型
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
    }
    if (initlen && init)
        memcpy(s, init, initlen); //将原始字符串内容复制到新的存储空间
    s[initlen] = '\0'; // 这里意思是原有字符串没有结尾符号?
    return s;
}
```

### len 与 alloc 改动
sds header 将字符串空间 和 实际分配空间分离后,提供一些接口分别改这些值,这样使用方可以根据需要来决定什么时候做真正的内存空间增加和减少操作
```c
// src/sds.c
// 更新字符串长度,直接取出当前字符串的长度,更新到信息里,不管实际分配空间是多少(以后再在合适的时间去具体操作空间)
void sdsupdatelen(sds s) {
    size_t reallen = strlen(s);
    sdssetlen(s, reallen);
}

// 清理整个字符串也是直接将len置换为0,将字符串第一个字符置换为结尾符,不去执行具体的空间回收操作
void sdsclear(sds s) {
    sdssetlen(s, 0);
    s[0] = '\0';
}

// 应用代码判断改变字符串后,原先的存储空间不够了,需要重新分配更多的空间
sds sdsMakeRoomFor(sds s, size_t addlen) {
    void *sh, *newsh;
    size_t avail = sdsavail(s); // 得到当前结构里还剩下多少剩余空间
    size_t len, newlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK; // 以前的header类型
    int hdrlen;

    if (avail >= addlen) return s; // 当header结构里剩余可用空间大于想要新增的空间时,不需要多做什么

    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype); // 取到原有的header结构地址
    newlen = (len+addlen);
    // 新增空间并不是需要多少加多少,而是成倍数加
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;

    type = sdsReqType(newlen); // 根据新长度决定新的header类型

    // ...

    hdrlen = sdsHdrSize(type);
    if (oldtype==type) { // 类型没变时,直接在原空间上扩增
        newsh = s_realloc(sh, hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        // 类型变了时,需要重新初始化存储空间,并把以前的内容复制过来
        newsh = s_malloc(hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, newlen);
    return s;
}

// 减少sds多余的空间操作
sds sdsRemoveFreeSpace(sds s) {
    void *sh, *newsh;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen, oldhdrlen = sdsHdrSize(oldtype);
    size_t len = sdslen(s);
    size_t avail = sdsavail(s);
    sh = (char*)s-oldhdrlen;

    // 已经没有多余的空间了
    if (avail == 0) return s;

    // 根据当前的实际len决定新的sds结构采用什么类型
    type = sdsReqType(len);
    hdrlen = sdsHdrSize(type);

    if (oldtype==type || type > SDS_TYPE_8) {
        // 类型不变或者 类型不是最基本(最小)的SDS_TYPE_8,调用realloc ,让系统自己去决定是重新分配空间,还是沿用原空间并裁减
        newsh = s_realloc(sh, oldhdrlen+len+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+oldhdrlen;
    } else {
        // 类型变化,或类型是SDS_TYPE_8基本(最小)类型,重新分配空间,复制以前的内容到新空间(猜想是为了减少碎片)
        newsh = s_malloc(hdrlen+len+1);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, len); // 经过操作后,实际分配的存户空间=字符串占用空间了
    return s;
}
```