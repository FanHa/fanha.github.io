
### schema 及 示例数据
```
// schema
sqlite> CREATE TABLE COMPANY(
   ID INT PRIMARY KEY     NOT NULL,
   NAME           TEXT    NOT NULL,
   AGE            INT     NOT NULL,
   ADDRESS        CHAR(50),
   SALARY         REAL
);
sqlite> CREATE TABLE DEPARTMENT(
   ID INT PRIMARY KEY      NOT NULL,
   DEPT           CHAR(50) NOT NULL,
   EMP_ID         INT      NOT NULL
);

// data
sqlite> select * from COMPANY;
id          name        age         address     salary
----------  ----------  ----------  ----------  ----------
1           Paul        32          California  20000.0
2           Allen       25          Texas       15000.0
3           Teddy       23          Norway      20000.0
4           Mark        25          Rich-Mond   65000.0
5           David       27          Texas       85000.0
6           Kim         22          South-Hall  45000.0

sqlite> select * from department;
id          dept        emp_id
----------  ----------  ----------
1           IT Billing  1
2           Engineerin  2
3           Finance     7
```

### Join方式
SQlite三种join方式
+ cross join
+ inner join
+ outer join

#### cross join
cross join 返回连接的两个表的笛卡尔积,即左表行数n * 右表行数m
```
// 查询语句
sqlite> SELECT EMP_ID, NAME, DEPT FROM COMPANY CROSS JOIN DEPARTMENT;

// 返回结果
EMP_ID      name        DEPT
----------  ----------  ----------
1           Paul        IT Billing
2           Paul        Engineerin
7           Paul        Finance
1           Allen       IT Billing
2           Allen       Engineerin
7           Allen       Finance
1           Teddy       IT Billing
2           Teddy       Engineerin
7           Teddy       Finance
1           Mark        IT Billing
2           Mark        Engineerin
7           Mark        Finance
1           David       IT Billing
2           David       Engineerin
7           David       Finance
1           Kim         IT Billing
2           Kim         Engineerin
7           Kim         Finance

// explain 结果
sqlite> explain SELECT EMP_ID, NAME, DEPT FROM COMPANY CROSS JOIN DEPARTMENT;
addr  opcode         p1    p2    p3    p4             p5  comment
----  -------------  ----  ----  ----  -------------  --  -------------
0     Init           0     12    0                    00  Start at 12
1     OpenRead       0     2     0     2              00  root=2 iDb=0; COMPANY
2     OpenRead       1     4     0     3              00  root=4 iDb=0; DEPARTMENT
3     Rewind         0     11    0                    00
4       Rewind         1     11    0                    00
5         Column         1     2     1                    00  r[1]=DEPARTMENT.EMP_ID
6         Column         0     1     2                    00  r[2]=COMPANY.name
7         Column         1     1     3                    00  r[3]=DEPARTMENT.DEPT
8         ResultRow      1     3     0                    00  output=r[1..3]
9       Next           1     5     0                    01
10    Next           0     4     0                    01
11    Halt           0     0     0                    00
12    Transaction    0     0     4     0              01  usesStmtJournal=0
13    Goto           0     1     0                    00
```
addr为 3-10 的OPcode是一个嵌套循环,即执行前面的 n*m 扫描,返回所有结果;  
接下来给sql加一些where 和 limit条件

---
加入条件 where name = 'paul'
+ 执行流程在外循环里加入了条件判断,过滤掉了不满足条件的'外表'的行(即company表中 name != 'paul'的行)
```
sqlite> explain SELECT EMP_ID, NAME, DEPT FROM COMPANY CROSS JOIN DEPARTMENT where name = 'paul';
addr  opcode         p1    p2    p3    p4             p5  comment
----  -------------  ----  ----  ----  -------------  --  -------------
0     Init           0     14    0                    00  Start at 14
1     OpenRead       0     2     0     2              00  root=2 iDb=0; COMPANY
2     OpenRead       1     4     0     3              00  root=4 iDb=0; DEPARTMENT
3     Rewind         0     13    0                    00
4       Column         0     1     1                    00  r[1]=COMPANY.name
5       Ne             2     12    1     (BINARY)       52  if r[1]!=r[2] goto 12
6       Rewind         1     13    0                    00
7         Column         1     2     3                    00  r[3]=DEPARTMENT.EMP_ID
8         Copy           1     4     0                    00  r[4]=r[1]
9         Column         1     1     5                    00  r[5]=DEPARTMENT.DEPT
10        ResultRow      3     3     0                    00  output=r[3..5]
11      Next           1     7     0                    01
12    Next           0     4     0                    01
13    Halt           0     0     0                    00
14    Transaction    0     0     4     0              01  usesStmtJournal=0
15    String8        0     2     0     paul           00  r[2]='paul'
16    Goto           0     1     0                    00
```

---
加入条件 where name = 'paul' and dept = 'Finance'
+ 外层循环为 addr 3-23
+ addr 4-5 过滤出name != 'paul'的外层行
+ 第一次循环到`addr-6-Once`时执行的时 `addr-7-OpenAutoIndex` 和 循环 addr 8-14
  + `addr-7-OpenAutoIndex` 创建一个临时存index的区域 
  + addr 8-14 循环整个`department`表,临时取出department.dept,department.emp_id,department的rowID组成一个类似联合索引(也是覆盖索引),存入前面的临时index,这里联合索引的第一个字段就是要查的department表的条件 dept;
+ 接下来的外层循环到达 `addr-6-Once`时会直接跳到p2 所指的 `addr-15-String8` 和内层循环 addr 16-22
  + `addr-16-SeekGE`首先把循环指针定位到前面生成的临时index中找到'finance'所在的行
  + 然后 addr 17-22 会扫描前面生成的index,把满足条件 dept = 'finance' 的行取出并放到resultRow里,因为前面生成的是联合索引,所以GT(Great than)代表匹配到的行
```
sqlite> explain SELECT EMP_ID, NAME, DEPT FROM COMPANY CROSS JOIN DEPARTMENT where name = 'paul'and dept = 'finance';
addr  opcode         p1    p2    p3    p4             p5  comment
----  -------------  ----  ----  ----  -------------  --  -------------
0     Init           0     25    0                    00  Start at 25
1     OpenRead       0     2     0     2              00  root=2 iDb=0; COMPANY
2     OpenRead       1     4     0     3              00  root=4 iDb=0; DEPARTMENT
3     Rewind         0     24    0                    00
4       Column         0     1     1                    00  r[1]=COMPANY.name
5       Ne             2     23    1     (BINARY)       52  if r[1]!=r[2] goto 23
6       Once           0     15    0                    00
7       OpenAutoindex  2     3     0     k(3,B,,)       00  nColumn=3; for DEPARTMENT
8       Rewind         1     15    0                    00
9         Column         1     1     4                    00  r[4]=DEPARTMENT.DEPT
10        Column         1     2     5                    00  r[5]=DEPARTMENT.EMP_ID
11        Rowid          1     6     0                    00  r[6]=rowid
12        MakeRecord     4     3     3                    00  r[3]=mkrec(r[4..6])
13        IdxInsert      2     3     0                    10  key=r[3]
14      Next           1     9     0                    03
15      String8        0     7     0     finance        00  r[7]='finance'
16      SeekGE         2     23    7     1              00  key=r[7]
17        IdxGT          2     23    7     1              00  key=r[7]
18        Column         2     1     8                    00  r[8]=DEPARTMENT.EMP_ID
19        Copy           1     9     0                    00  r[9]=r[1]
20        Column         2     0     10                   00  r[10]=DEPARTMENT.DEPT
21        ResultRow      8     3     0                    00  output=r[8..10]
22      Next           2     17    0                    00
23    Next           0     4     0                    01
24    Halt           0     0     0                    00
25    Transaction    0     0     4     0              01  usesStmtJournal=0
26    String8        0     2     0     paul           00  r[2]='paul'
27    Goto           0     1     0                    00
```

---
加入条件 where name = 'paul' and dept = 'Finance' limit 2
+ 与前面where条件的OPCODE差异不大
+ 在`addr-1-Integer`赋值了一个counter
+ 在`addr-20-ResultRow`每次生成一行满足条件的数据后加入了`addr-21-DecrJumZero`检测,数量达到count指定的值后跳出内外循环
```
sqlite> explain SELECT EMP_ID, NAME, DEPT FROM COMPANY CROSS JOIN DEPARTMENT where dept = 'finance' limit 2;
addr  opcode         p1    p2    p3    p4             p5  comment
----  -------------  ----  ----  ----  -------------  --  -------------
0     Init           0     25    0                    00  Start at 25
1     Integer        2     1     0                    00  r[1]=2; LIMIT counter
2     OpenRead       0     2     0     2              00  root=2 iDb=0; COMPANY
3     OpenRead       1     4     0     3              00  root=4 iDb=0; DEPARTMENT
4     Rewind         0     24    0                    00
5       Once           0     14    0                    00
6       OpenAutoindex  2     3     0     k(3,B,,)       00  nColumn=3; for DEPARTMENT
7       Rewind         1     14    0                    00
8         Column         1     1     3                    00  r[3]=DEPARTMENT.DEPT
9         Column         1     2     4                    00  r[4]=DEPARTMENT.EMP_ID
10        Rowid          1     5     0                    00  r[5]=rowid
11        MakeRecord     3     3     2                    00  r[2]=mkrec(r[3..5])
12        IdxInsert      2     2     0                    10  key=r[2]
13      Next           1     8     0                    03
14      String8        0     6     0     finance        00  r[6]='finance'
15      SeekGE         2     23    6     1              00  key=r[6]
16        IdxGT          2     23    6     1              00  key=r[6]
17        Column         2     1     7                    00  r[7]=DEPARTMENT.EMP_ID
18        Column         0     1     8                    00  r[8]=COMPANY.name
19        Column         2     0     9                    00  r[9]=DEPARTMENT.DEPT
20        ResultRow      7     3     0                    00  output=r[7..9]
21        DecrJumpZero   1     24    0                    00  if (--r[1])==0 goto 24
22      Next           2     16    0                    00
23    Next           0     5     0                    01
24    Halt           0     0     0                    00
25    Transaction    0     0     4     0              01  usesStmtJournal=0
26    Goto           0     1     0                    00
```

#### inner join
