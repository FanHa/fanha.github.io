### 不同持久化策略
+ RDB 保存数据集结果快照到某个文件;
+ AOF 以日志方式记录写操作;
### 入口
Redis 的serverCron中与持久化相关的代码
代码中可以看出,无论是执行aof 还是 rdb 都是用另外的子进程执行,且保证同一时间只有一个子进程在执行,避免并发带来的麻烦
```c
//server.c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    // ... 
    // 如果aof_rewrite_scheduled开关置为1,则执行aof持久化
    // 这个开关用来在一个持久化进程执行时,收到另外的持久化需求,则把这些需求scheduled,等待正在持久化的进程结束(即xxx_child_pid == -1)时再执行
    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1 &&
        server.aof_rewrite_scheduled)
    {
        rewriteAppendOnlyFileBackground();
    }
    // 
    // 如果有rdb子进程标记或 aof子进程标记为真,则检测是否子进程已完成,并且此次serverCron 不再发起新的 aof 或 rdb请求
    if (server.rdb_child_pid != -1 || server.aof_child_pid != -1 ||
        ldbPendingChildren())
    {
        int statloc;
        pid_t pid;

        if ((pid = wait3(&statloc,WNOHANG,NULL)) != 0) {

            // 
            if (pid == -1) {
            // 检测到 rdb子进程已完成    
            } else if (pid == server.rdb_child_pid) {
                backgroundSaveDoneHandler(exitcode,bysignal);
                if (!bysignal && exitcode == 0) receiveChildInfo();
            // 检测到 aof 子进程已完成
            } else if (pid == server.aof_child_pid) {
                backgroundRewriteDoneHandler(exitcode,bysignal);
                if (!bysignal && exitcode == 0) receiveChildInfo();
            } else {

            }
            updateDictResizePolicy();
            closeChildInfoPipe();
        }
    } else {
        for (j = 0; j < server.saveparamslen; j++) {
            struct saveparam *sp = server.saveparams+j;
            // 判断是否达到了触发rdb 持久化的阈值;
            // redis可以配置不同的rdb持久化策略,比如改动了多少次,距离上次持久化多长事件等等
            if (server.dirty >= sp->changes &&
                server.unixtime-server.lastsave > sp->seconds &&
                (server.unixtime-server.lastbgsave_try >
                 CONFIG_BGSAVE_RETRY_DELAY ||
                 server.lastbgsave_status == C_OK))
            {
                // 开启rdb持久化进程
                rdbSaveBackground(server.rdb_filename,rsiptr);
                break;
            }
        }

        // 执行AOF持久化
        if (server.aof_state == AOF_ON &&
            server.rdb_child_pid == -1 &&
            server.aof_child_pid == -1 &&
            server.aof_rewrite_perc &&
            server.aof_current_size > server.aof_rewrite_min_size)
        {

            long long growth = (server.aof_current_size*100/base) - 100;
            if (growth >= server.aof_rewrite_perc) {
                // 开启aof持久化进程
                rewriteAppendOnlyFileBackground();
            }
        }
    }
```

#### aof
```c
//aof.c
int rewriteAppendOnlyFileBackground(void) {
    
    if ((childpid = fork()) == 0) {
        // 子进程开写改动到磁盘
        if (rewriteAppendOnlyFile(tmpfile) == C_OK) {
           //...
        } else {

        }
    } else {
        //父进程负责把各种全局信息完善,供外面的循环判断现在的子进程状态和采取的相应措施   
        server.aof_rewrite_scheduled = 0;
        server.aof_rewrite_time_start = time(NULL);
        server.aof_child_pid = childpid;

        return C_OK;
    }
    return C_OK; /* unreached */
}
```

#### rdb
```c
int rdbSaveBackground(char *filename, rdbSaveInfo *rsi) {

    if ((childpid = fork()) == 0) {
        //子进程负责具体的rdb持久化
        retval = rdbSave(filename,rsi);

    } else {
        // 父进程记录各种全局信息,供外面的循环判断现在的子进程状态和采取的相应措施 
        server.stat_fork_time = ustime()-start;
        server.stat_fork_rate = (double) zmalloc_used_memory() * 1000000 / server.stat_fork_time / (1024*1024*1024); /* GB per second. */
        
        serverLog(LL_NOTICE,"Background saving started by pid %d",childpid);
        server.rdb_save_time_start = time(NULL);
        server.rdb_child_pid = childpid;
        server.rdb_child_type = RDB_CHILD_TYPE_DISK;
        updateDictResizePolicy();
        return C_OK;
    }
    return C_OK; /* unreached */
}
```
