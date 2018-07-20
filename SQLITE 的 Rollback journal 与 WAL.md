sqlite 有两种处理事务的方式`rollback journal` 与 `WAL(write-ahead log)`

- rollback journal 在事务前将数据备份在一个新文件中,事务提交时替代原数据文件的内容
- WAL 将更改内容写到一个新文件中,事务提交也只是在WAL文件里加一个标记,然后在"适当的时候(checkpoints)"将数据写到数据原文件里