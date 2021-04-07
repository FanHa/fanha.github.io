## 版本
+ Github:redis/redis 
+ 分支:6.0 
+ 版本号:6.0.7

## 简介
redis 中的一些耗时耗资源的任务会由其他线程来执行(Background I/O)
## BIO结构

```c
// src/bio.h
// 目前的异步BIO任务类型主要是“关闭文件”,“异步AOF”,和 “内存释放”
#define BIO_CLOSE_FILE    0 /* Deferred close(2) syscall. */
#define BIO_AOF_FSYNC     1 /* Deferred AOF fsync. */
#define BIO_LAZY_FREE     2 /* Deferred objects freeing. */
#define BIO_NUM_OPS       3

// src/bio.c
static pthread_t bio_threads[BIO_NUM_OPS]; // BIO_NUM_OPS 个线程 分别处理不同类型的任务
static pthread_mutex_t bio_mutex[BIO_NUM_OPS]; // BIO_NUM_OPS个线程锁
static pthread_cond_t bio_newjob_cond[BIO_NUM_OPS];
static pthread_cond_t bio_step_cond[BIO_NUM_OPS];
static list *bio_jobs[BIO_NUM_OPS]; // BIO_NUM_OPS 个线程队列

// src/adlist.h 
// BIO任务队列
typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```

## 调用接口加入异步BIO任务(lazy_free举例)
异步空间时会调用bioCreateBackgroundJob新增一个bio任务
```c
// src/lazyfree.c
int dbAsyncDelete(redisDb *db, robj *key) {
    // ...
    bioCreateBackgroundJob(BIO_LAZY_FREE,val,NULL,NULL);
    // ...
}
```
```c
// src/bio.c
void bioCreateBackgroundJob(int type, void *arg1, void *arg2, void *arg3) {
    struct bio_job *job = zmalloc(sizeof(*job));

    job->time = time(NULL);
    job->arg1 = arg1;
    job->arg2 = arg2;
    job->arg3 = arg3;
    pthread_mutex_lock(&bio_mutex[type]); // 线程锁
    listAddNodeTail(bio_jobs[type],job); // 将job添加到相应类型的任务队列里
    bio_pending[type]++;
    pthread_cond_signal(&bio_newjob_cond[type]); // 防止惊群
    pthread_mutex_unlock(&bio_mutex[type]); // 解锁
}

// src/adlist.c
list *listAddNodeTail(list *list, void *value)
{
    listNode *node;

    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;
    node->value = value;
    if (list->len == 0) {
        list->head = list->tail = node;
        node->prev = node->next = NULL;
    } else {
        node->prev = list->tail;
        node->next = NULL;
        list->tail->next = node;
        list->tail = node;
    }
    list->len++;
    return list;
}
```

## 异步BIO任务的初始化
```c
// src/server.c
int main(int argc, char **argv) {
    // ...
    InitServerLast();
    // ...
}

void InitServerLast() {
    bioInit();
    // ...
}
```
```c
// src/bio.c
void bioInit(void) {
    pthread_attr_t attr;
    pthread_t thread;
    size_t stacksize;
    int j;

    // 初始化每种BIO任务类型的队列
    for (j = 0; j < BIO_NUM_OPS; j++) {
        pthread_mutex_init(&bio_mutex[j],NULL);
        pthread_cond_init(&bio_newjob_cond[j],NULL);
        pthread_cond_init(&bio_step_cond[j],NULL);
        bio_jobs[j] = listCreate();
        bio_pending[j] = 0;
    }

    // ... 
    // 创建每种BIO任务类型的后端处理线程
    for (j = 0; j < BIO_NUM_OPS; j++) {
        void *arg = (void*)(unsigned long) j;
        if (pthread_create(&thread,&attr,bioProcessBackgroundJobs,arg) != 0) {
            serverLog(LL_WARNING,"Fatal: Can't initialize Background Jobs.");
            exit(1);
        }
        bio_threads[j] = thread;
    }
}
```
### bioProcessBackgroundJobs
```c
// src/bio.c
void *bioProcessBackgroundJobs(void *arg) {
    struct bio_job *job;
    unsigned long type = (unsigned long) arg;
    sigset_t sigset;

    // ...
    pthread_mutex_lock(&bio_mutex[type]); // 锁住队列(应该防止在取出任务前的临界阶段就被其他线程如住线程写入数据)
    while(1) {
        listNode *ln;

        // 当亲啊类型BIO任务队列里没有任务,等信号
        if (listLength(bio_jobs[type]) == 0) {
            pthread_cond_wait(&bio_newjob_cond[type],&bio_mutex[type]); // 等信号时当前任务类型的互斥锁bio_mutex[type]会解封
            continue;
        }
        // 取出队列最前面的任务
        ln = listFirst(bio_jobs[type]);
        job = ln->value;
        // 取出任务后可以解线程锁了
        pthread_mutex_unlock(&bio_mutex[type]);

        // 根据不同任务类型执行操作
        if (type == BIO_CLOSE_FILE) {
            close((long)job->arg1);  //关闭文件
        } else if (type == BIO_AOF_FSYNC) {
            // redis数据刷盘
            redis_fsync((long)job->arg1);
        } else if (type == BIO_LAZY_FREE) {
            // 释放空间
            // 释放空间也有不同的类型,如object, dict, list, 在释放了空间后还需要同步更新外层包裹结构的信息
            if (job->arg1)
                lazyfreeFreeObjectFromBioThread(job->arg1);
            else if (job->arg2 && job->arg3)
                lazyfreeFreeDatabaseFromBioThread(job->arg2,job->arg3);
            else if (job->arg3)
                lazyfreeFreeSlotsMapFromBioThread(job->arg3);
        } else {
            serverPanic("Wrong job type in bioProcessBackgroundJobs().");
        }
        zfree(job);

        // 删除队列里的节点需要设置当前类型任务的互斥锁, 这个锁会在下个循环解除
        pthread_mutex_lock(&bio_mutex[type]);
        listDelNode(bio_jobs[type],ln);
        bio_pending[type]--;

        // 通知等待锁的用户
        pthread_cond_broadcast(&bio_step_cond[type]);
    }
}

```