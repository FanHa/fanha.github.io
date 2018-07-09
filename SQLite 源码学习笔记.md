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


## Sql语句的执行流程
sqlite3_exec()函数执行整个sql语句调用,其中最关键的两个阶段
```c
//将字符串的sql语句(zSql)消化解析，并将解析后的语句信息放在pStmt里
rc = sqlite3_prepare_v2(db, zSql, -1, &pStmt, &zLeftover);

//将已经经过解析的pstmt按既定规则执行
while( 1 ){
  //...
  rc = sqlite3_step(pStmt);
  //...
}
```

### sql prepare阶段
sqlite3_prepare_v2函数经过一层层封装,最红调用的是sqlite3Prepare(),这里把一条sql语句拆解成了若干个可执行的步骤(step),供后续参照执行.
```c
int sqlite3_prepare_v2(
  sqlite3 *db,              /* Database handle. */
  const char *zSql,         /* UTF-8 encoded SQL statement. */
  int nBytes,               /* Length of zSql in bytes. */
  sqlite3_stmt **ppStmt,    /* OUT: A pointer to the prepared statement */
  const char **pzTail       /* OUT: End of parsed string */
){
  //...
  rc = sqlite3LockAndPrepare(db,zSql,nBytes,SQLITE_PREPARE_SAVESQL,0,ppStmtpzTail);
  //...
}

static int sqlite3LockAndPrepare(
  sqlite3 *db,              /* Database handle. */
  const char *zSql,         /* UTF-8 encoded SQL statement. */
  int nBytes,               /* Length of zSql in bytes. */
  u32 prepFlags,            /* Zero or more SQLITE_PREPARE_* flags */
  Vdbe *pOld,               /* VM being reprepared */
  sqlite3_stmt **ppStmt,    /* OUT: A pointer to the prepared statement */
  const char **pzTail       /* OUT: End of parsed string */
){
  //...
  rc = sqlite3Prepare(db, zSql, nBytes, prepFlags, pOld, ppStmt, pzTail);
  //...
}

static int sqlite3Prepare(
  sqlite3 *db,              /* Database handle. */
  const char *zSql,         /* UTF-8 encoded SQL statement. */
  int nBytes,               /* Length of zSql in bytes. */
  u32 prepFlags,            /* Zero or more SQLITE_PREPARE_* flags */
  Vdbe *pReprepare,         /* VM being reprepared */
  sqlite3_stmt **ppStmt,    /* OUT: A pointer to the prepared statement */
  const char **pzTail       /* OUT: End of parsed string */
){
  //...
  //解析sql语句,并将结果放在sParse中
  sqlite3RunParser(&sParse, zSqlCopy, &zErrMsg);
  //...
}
```

#### Parser
Parser主要由一个while(1)循环逐字节解析sql语句,主要分为两个阶段:
1.sqlite3GetToken函数取得一个token,记录下相应的tokentType;
2.调用sqlite3Parser函数根据tokenType(及lastTokenType)做对应的token操作
```c
//tokenize.c
int sqlite3RunParser(Parse *pParse, const char *zSql, char **pzErrMsg){
  //...
   while( 1 ){
    if( zSql[0]!=0 ){
      n = sqlite3GetToken((u8*)zSql, &tokenType);
      //...
    }else{
      //...
    }
    if( tokenType>=TK_SPACE ){
      //...
    }else{
      pParse->sLastToken.z = zSql;
      pParse->sLastToken.n = n;
      sqlite3Parser(pEngine, tokenType, pParse->sLastToken, pParse);
      lastTokenParsed = tokenType;
      zSql += n;
      if( pParse->rc!=SQLITE_OK || db->mallocFailed ) break;
    }
  }
}
```

##### sqlite3GetToken
tokenize首先通过宏和一个数组标记每一种字符所代表的类型
```c
//tokenize.c

#define CC_X          0    /* The letter 'x', or start of BLOB literal */
#define CC_KYWD       1    /* Alphabetics or '_'.  Usable in a keyword */
#define CC_ID         2    /* unicode characters usable in IDs */
#define CC_DIGIT      3    /* Digits */
#define CC_DOLLAR     4    /* '$' */
#define CC_VARALPHA   5    /* '@', '#', ':'.  Alphabetic SQL variables */
#define CC_VARNUM     6    /* '?'.  Numeric SQL variables */
#define CC_SPACE      7    /* Space characters */
#define CC_QUOTE      8    /* '"', '\'', or '`'.  String literals, quoted ids */
#define CC_QUOTE2     9    /* '['.   [...] style quoted ids */
#define CC_PIPE      10    /* '|'.   Bitwise OR or concatenate */
#define CC_MINUS     11    /* '-'.  Minus or SQL-style comment */
#define CC_LT        12    /* '<'.  Part of < or <= or <> */
#define CC_GT        13    /* '>'.  Part of > or >= */
#define CC_EQ        14    /* '='.  Part of = or == */
#define CC_BANG      15    /* '!'.  Part of != */
#define CC_SLASH     16    /* '/'.  / or c-style comment */
#define CC_LP        17    /* '(' */
#define CC_RP        18    /* ')' */
#define CC_SEMI      19    /* ';' */
#define CC_PLUS      20    /* '+' */
#define CC_STAR      21    /* '*' */
#define CC_PERCENT   22    /* '%' */
#define CC_COMMA     23    /* ',' */
#define CC_AND       24    /* '&' */
#define CC_TILDA     25    /* '~' */
#define CC_DOT       26    /* '.' */
#define CC_ILLEGAL   27    /* Illegal character */

static const unsigned char aiClass[] = {
#ifdef SQLITE_ASCII
/*         x0  x1  x2  x3  x4  x5  x6  x7  x8  x9  xa  xb  xc  xd  xe  xf */
/* 0x */   27, 27, 27, 27, 27, 27, 27, 27, 27,  7,  7, 27,  7,  7, 27, 27,
/* 1x */   27, 27, 27, 27, 27, 27, 27, 27, 27, 27, 27, 27, 27, 27, 27, 27,
/* 2x */    7, 15,  8,  5,  4, 22, 24,  8, 17, 18, 21, 20, 23, 11, 26, 16,
/* 3x */    3,  3,  3,  3,  3,  3,  3,  3,  3,  3,  5, 19, 12, 14, 13,  6,
/* 4x */    5,  1,  1,  1,  1,  1,  1,  1,  1,  1,  1,  1,  1,  1,  1,  1,
/* 5x */    1,  1,  1,  1,  1,  1,  1,  1,  0,  1,  1,  9, 27, 27, 27,  1,
/* 6x */    8,  1,  1,  1,  1,  1,  1,  1,  1,  1,  1,  1,  1,  1,  1,  1,
/* 7x */    1,  1,  1,  1,  1,  1,  1,  1,  0,  1,  1, 27, 10, 27, 25, 27,
/* 8x */    2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,
/* 9x */    2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,
/* Ax */    2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,
/* Bx */    2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,
/* Cx */    2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,
/* Dx */    2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,
/* Ex */    2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,
/* Fx */    2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2,  2
}
```


*z是指向sql语句字符的指针,通过switch得到对应的tokenType,并返回该token的长度,如检测到 z
所指向的字符对应的aiClass是 CC_SPACE,case内部的循环会接着判断接下来的字符是不是也是'space'类型,直到循环结束,将tokenType 赋值为TK_SPACE,并返回该token长度(便于sqlite3GetToken外部的循环将sql指针向前推进,进行下一个token的解析)
```c
int sqlite3GetToken(const unsigned char *z, int *tokenType){
  int i, c;
  switch( aiClass[*z] ){ 
    case CC_SPACE: {
      testcase( z[0]==' ' );
      testcase( z[0]=='\t' );
      testcase( z[0]=='\n' );
      testcase( z[0]=='\f' );
      testcase( z[0]=='\r' );
      for(i=1; sqlite3Isspace(z[i]); i++){}
      *tokenType = TK_SPACE;
      return i;
    }
    //...
}
```
##### sqlite3Parser 
//TODO 先了解 yacc 和 bison等相关原理再来

### sql step执行阶段
sqlite3_step函数经过层层封装最终调用sqlite3VdbeExec对已经prepare的sql,按规则执行既定的step.
```c
//vdbeapi.c
int sqlite3_step(sqlite3_stmt *pStmt){
  //...
  while( (rc = sqlite3Step(v))==SQLITE_SCHEMA
         && cnt++ < SQLITE_MAX_SCHEMA_RETRY ){}
  //...

}

static int sqlite3Step(Vdbe *p){
  //...
  rc = sqlite3VdbeExec(p);
  //...
}
```

#### sqlite3VdbeExec 
sqlite3VdbeExec()函数主要由一个大的case语句来将prepare解析好的执行步骤分发给具体的执行函数;
```c
int sqlite3VdbeExec(
  Vdbe *p                    /* The VDBE */
){
  Op *aOp = p->aOp;          /* Copy of p->aOp */
  Op *pOp = aOp;             /* Current operation */
  //...
  switch( pOp->opcode ){
    //...
    case OP_Insert: 
    case OP_InsertInt: {
      //Do something
    }
    //...
}
```

官网有一条例子可窥  https://www.sqlite.org/opcode.html  
  $ sqlite3 ex1.db  
  sqlite> explain delete from tbl1 where two<20;  
```
  addr  opcode         p1    p2    p3    p4             p5  comment      
  ----  -------------  ----  ----  ----  -------------  --  -------------
  0     Init           0     12    0                    00  Start at 12  
  1     Null           0     1     0                    00  r[1]=NULL    
  2     OpenWrite      0     2     0     3              00  root=2 iDb=0; tbl1
  3     Rewind         0     10    0                    00               
  4       Column         0     1     2                    00  r[2]=tbl1.two
  5       Ge             3     9     2     (BINARY)       51  if r[2]>=r[3] goto 9
  6       Rowid          0     4     0                    00  r[4]=rowid   
  7       Once           0     8     0                    00               
  8       Delete         0     1     0     tbl1           02               
  9     Next           0     4     0                    01               
  10    Noop           0     0     0                    00               
  11    Halt           0     0     0                    00               
  12    Transaction    0     1     1     0              01  usesStmtJournal=0
  13    TableLock      0     2     1     tbl1           00  iDb=0 root=2 write=1
  14    Integer        20    3     0                    00  r[3]=20      
  15    Goto           0     1     0                    00
```

sql 语句"delete from tbl1 where two<20" 经过prepare解析阶段成了如上得opcode(s),step会从第0条Init 开始执行,Init 得代码主要只是一个入口,表明stmt从p2表示得opcodeAddr开始执行,例子中p2值为12, 既从addr=12得Transaction执行
```c
// vdbe.c
jump_to_p2:
  pOp = &aOp[pOp->p2 - 1];
  break;


// ...
case OP_Init: {          /* jump */
  // ...
  goto jump_to_p2;
}
```
Transaction如其名表示事务(sql语句含delete可能得多条数据,在sql执行时是一个循环一条条delete得,存在失败可能需要回滚),依次往下执行 `13-TableLock`, `14-Integer`, `15-Goto`

```c
// vdbe.c
case OP_Transaction: {
  // 主要做一些事务的初始化
}

case OP_TableLock: {
  // ...
  // 给需要执行操作的表上锁(这里的锁不是常见的读写锁,是sqlite shared-cache特性的锁,读写锁是在后面的OP_Openwrite上加的) 
  // TODO: 什么是shared-cache
  rc = sqlite3BtreeLockTable(db->aDb[p1].pBt, pOp->p2, isWriteLock);
  // ...
}

case OP_Integer: {
  // 将整数20存入一个变量(这里是存到一个register里)以便后面用来比较
  pOut = out2Prerelease(p, pOp);
  pOut->u.i = pOp->p1;
  break;
}

case OP_Goto: {
jump_to_p2_and_check_for_interrupt:
  //...
  // 跳转到p2所指的opcodeAddr(同前面的jumb_to_p2)是同一段代码
}
```

接下来开始对表进入写操作阶段, 首先打开表文件 `2-OpenWrite`, 然后设立一个循环标记`3-Rewind`
```c
// vdbe.c
case OP_OpenWrite:
// ...
// 初始化内存空间
pCur = allocateCursor(p, pOp->p1, nField, iDb, CURTYPE_BTREE);
// ...
// 初始化指向数据(以BTREE形式存放)的指针,在这个函数里会给表加上读写锁
rc = sqlite3BtreeCursor(pX, p2, wrFlag, pKeyInfo, pCur->uc.pCursor);
// ...

case OP_Rewind: {        /* jump */
  // ...
  // 将cursor移动到表数据记录的第一条,如果为空就直接跳到p2 所指的addr,再这个例子中p2值为10
  rc = sqlite3BtreeFirst(pCrsr, &res);
  // ...
  if( res ) goto jump_to_p2;
  break;
}
```


 然后进入一个循环
 `4-Column`,`5-Ge`,`6-RowId`,`7-Once`,`8-Delete`,`9-Next`遍历表的行,找出`two<20` 的行并删除, 循环结束且没有出错step到`11-Halt`

```c
// vdbe.c
case OP_Column: {
  // ...
  // 将当前游标(cursor)指向的数据行加载到内存以便作处理
  memset(&sMem, 0, sizeof(sMem));
  rc = sqlite3VdbeMemFromBtree(pC->uc.pCursor, 0, aOffset[0], &sMem);

  // ...
  // 最终行中特定的列的数据去取出,放到p3指向的register,此例为r[2]
  // 此时register里已经有了r[2] = 当前行的two列指向的值,
  // 和r[3] = 20 ,即需要和"two"进行比较的值
  // 可以继续前进进入下一个OP 即OP_Ge
op_column_out:
  UPDATE_MAX_BLOBSIZE(pDest);
  REGISTER_TRACE(pOp->p3, pDest);
  break;
}

case OP_Eq:
case OP_Ne:
case OP_Lt:
case OP_Le:
case OP_Gt:  
case OP_Ge: {
  // 先从p1, p3中取出要比较的值和信息 
  pIn1 = &aMem[pOp->p1];
  pIn3 = &aMem[pOp->p3];
  flags1 = pIn1->flags;
  flags3 = pIn3->flags;
  // ...
  // 比较r[p1] 和 r[p3] 的值, 这一层封装了不同类型的比较, 得到的是两个值的相对关系, 
  // res2再根据 OPCode的值和 res来决定比对是否满足,并跳到p2所指的OPcode address
  // 和人一般的思维过程有点不同, 但这样做是为了方便把对两个值的比较过程本身和比较结果解耦, 
  // 这样方便在sqlite3MemCompare里加入对不同数据类型的比较,
  // 甚至通过 p4传入回调(文档上说二进制)来比较高级数据类型的参数
  res = sqlite3MemCompare(pIn3, pIn1, pOp->p4.pColl);
  // ...
compare_op:
  if( res<0 ){                        /* ne, eq, gt, le, lt, ge */
    static const unsigned char aLTb[] = { 1,  0,  0,  1,  1,  0 };
    res2 = aLTb[pOp->opcode - OP_Ne];
  }else if( res==0 ){
    static const unsigned char aEQb[] = { 0,  1,  0,  1,  0,  1 };
    res2 = aEQb[pOp->opcode - OP_Ne];
  }else{
    static const unsigned char aGTb[] = { 1,  0,  1,  0,  0,  1 };
    res2 = aGTb[pOp->opcode - OP_Ne];
  }
  // ...
  if( res2 ){
    // 比较结果不满足既定要求时, 跳到p2所指的 OPCode Address, 本例是9-Next
    goto jump_to_p2;
  }
  // 满足比较要求时接着执行下一个 OPcode, 本例是6-RowId
  break;

}

case OP_Rowid: {                 /* out2 */
  // ...
  // 这里封装了一层, 将pOut指向p2 所指向的register, 本例是register[4]
  pOut = out2Prerelease(p, pOp);

  // 将游标指向p1现在所在行
  pC = p->apCsr[pOp->p1];

  // 取出当前行的key,即rowId
  v = sqlite3BtreeIntegerKey(pC->uc.pCursor);
  // ...
  // 将rowId 赋值给 p2
  pOut->u.i = v;
  break;
}


case OP_Halt: {
  //这里做一些检测和记录后跳转到vdbe_return(即返回执行结果).
  goto vdbe_return;
}

// ...
vdbe_return:
  testcase( nVmStep>0 );
  p->aCounter[SQLITE_STMTSTATUS_VM_STEP] += (int)nVmStep;
  sqlite3VdbeLeave(p);
  assert( rc!=SQLITE_OK || nExtraDelete==0 
       || sqlite3_strlike("DELETE%",p->zSql,0)!=0 
  );
  return rc;
```