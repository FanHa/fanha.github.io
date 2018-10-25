### multi使用方法
+ multi
+ command1
+ command2
+ ...
+ exec

### multi
multi命令判断当前连接是否已在multi模式中;
将当前连接得标记设为`CLIENT_MULTI`
```c
// multi.c
void multiCommand(client *c) {
    if (c->flags & CLIENT_MULTI) {
        addReplyError(c,"MULTI calls can not be nested");
        return;
    }
    c->flags |= CLIENT_MULTI;
    addReply(c,shared.ok);
}
```

### multi后的processCommand
所有命令都会经过processCommand,如果当前连接在`CLIENT_MULTI`中,则会将当前命令`queueMultiCommand(c)`,不会立即执行
```c
// server.c
int processCommand(client *c) {
    // ...
    if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
    {
        queueMultiCommand(c);
        addReply(c,shared.queued);
    } else {
        call(c,CMD_CALL_FULL);
        c->woff = server.master_repl_offset;
        if (listLength(server.ready_keys))
            handleClientsBlockedOnKeys();
    }
    return C_OK;
}
```

### exec
从前面的queueMultiCommand的队列中取出multi中的命令,依次执行
```c
void execCommand(client *c) {
    // ...
    for (j = 0; j < c->mstate.count; j++) {

        // 当redis处在master-slave运行时,需要原子性的复制
        if (!must_propagate && !(c->cmd->flags & (CMD_READONLY|CMD_ADMIN))) {
            execCommandPropagateMulti(c);
            must_propagate = 1;
        }

        call(c,server.loading ? CMD_CALL_NONE : CMD_CALL_FULL);
    }
}
```

### 其他
+ 因为redis的主循环时单线程的,所以在exec multi队列里的命令时,会依次执行,不会有其他命令干扰,所以不需要担心隔离的问题;
+ 从代码看redis只是顺序执行multi队列里的命令,没有中途出错的恢复,回滚等操作;
+ multi 并不符合一般认知中对于事务的定义,只是保证一系列能不被其他命令干扰的顺序执行,执行成功与否不在乎;

