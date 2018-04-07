## 前言
SQLite没有使用客户端-服务器模式来完成数据库操作,而是作为c语言应用内嵌的一部分编译进了程序.

## 源码版本
3.23.0

## 接口层
C程序引用了SQLite库后就可以直接通过调用接口层提供的函数来完成数据库操作,接口层的函数写在src/main.c 和 src/legacy.c

### main.c

#### sqlite全局操作接口(部分)
```c
int sqlite3_initialize(void){}//初始化SQLite,同时也保证了sqlite全局唯一

int sqlite3_shutdown(void){}//关闭SQLite

int sqlite3_config(int op, ...){}//修改SQLite配置

static int setupLookaside(sqlite3 *db, void *pBuf, int sz, int cnt){}//查寻缓存的初始化
```

#### SQLite数据库操作接口(部分)
```c
sqlite3_mutex *sqlite3_db_mutex(sqlite3 *db){}//数据库连接的互斥锁,保证一个数据库连接只被一个地方操作?

int sqlite3_db_release_memory(sqlite3 *db){}//释放数据库连接使用的内存

int sqlite3_db_cacheflush(sqlite3 *db){}//将数据库操作写回磁盘

int sqlite3_db_config(sqlite3 *db, int op, ...){}//与数据库连接相关的配置

int sqlite3_open(
  const char *zFilename, 
  sqlite3 **ppDb 
){}//打开一个数据库


```

#### 数据库应接不暇时的回调函数
当数据库连接的某些资源过于紧张时的回调函数相关
```c
//默认回调
static int sqliteDefaultBusyCallback(
  void *ptr,               /* Database connection */
  int count,               /* Number of times table has been busy */
  sqlite3_file *pFile      /* The file on which the lock occurred */
){}

//执行回调
int sqlite3InvokeBusyHandler(BusyHandler *p, sqlite3_file *pFile){}

//定义回调函数
int sqlite3_busy_handler(
  sqlite3 *db,
  int (*xBusy)(void*,int),
  void *pArg
){}

void sqlite3_progress_handler(
  sqlite3 *db, 
  int nOps,
  int (*xProgress)(void*), 
  void *pArg
){}

//定义busyCallBack的触发临界值
int sqlite3_busy_timeout(sqlite3 *db, int ms){}
```

#### 工具性函数(部分)
```c
//和其他名字带有 create_func 关键字的函数,可以对标mysql用来创建存储过程功能接口
int sqlite3CreateFunc(
  sqlite3 *db,
  const char *zFunctionName,
  int nArg,
  int enc,
  void *pUserData,
  void (*xSFunc)(sqlite3_context*,int,sqlite3_value **),
  void (*xStep)(sqlite3_context*,int,sqlite3_value **),
  void (*xFinal)(sqlite3_context*),
  FuncDestructor *pDestructor
){}

//

```


#### 钩子函数
用来注册不同操作不同阶段的钩子函数
```c
void *sqlite3_trace(sqlite3 *db, void(*xTrace)(void*,const char*), void *pArg){}

void *sqlite3_profile(
  sqlite3 *db,
  void (*xProfile)(void*,const char*,sqlite_uint64),
  void *pArg
){}

void *sqlite3_commit_hook(
  sqlite3 *db,              /* Attach the hook to this database */
  int (*xCallback)(void*),  /* Function to invoke on each commit */
  void *pArg                /* Argument to the function */
){}

void *sqlite3_update_hook(
  sqlite3 *db,              /* Attach the hook to this database */
  void (*xCallback)(void*,int,char const *,char const *,sqlite_int64),
  void *pArg                /* Argument to the function */
){

//...
```

#### WAL(Write-Ahead Log) 预写日志相关
预写日志将对数据文件的修改先顺序写到一个日志文件中,之后再把这个日志文件解析并写到真正修改的数据文件中;这个机制缓冲了对磁盘随机写的压力(顺序写磁盘相对于随机写磁盘更高速).
```c
//名字中带"wal"关键字的都是与预写日志机制相关的函数
int sqlite3_wal_checkpoint(sqlite3 *db, const char *zDb){}
//...
```


### legacy.c
legacy.c 只有一个函数用来将提交字符串SQL请求
```c
int sqlite3_exec(
  sqlite3 *db,                /* The database on which the SQL executes */
  const char *zSql,           /* The SQL to be executed */
  sqlite3_callback xCallback, /* Invoke this callback routine */
  void *pArg,                 /* First argument to xCallback() */
  char **pzErrMsg             /* Write error messages here */
){}
```
#### 