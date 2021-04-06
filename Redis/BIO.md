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