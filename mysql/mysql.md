# 06全局锁和表锁：给表加个字段怎么有这么多阻碍？

锁的作用：处理数据库并发问题。

Mysql三类锁：全局锁、表级锁、行锁。

### 1、全局锁

##### （1）什么是全局锁？

全局锁是对整个数据库实例加锁。

##### （2）加全局锁的方法（命令）？

`Flush tables with read lock(FTWRL)`

##### （3）加全局锁对数据库的影响

整个数据库处于只读状态，其他线程的语句会被阻塞(失去cpu执行权）：数据更新语句（数据的增删改）、数据定义语句（建表、修改表结构等）、更新类事务的提交语句。

##### （4）全局锁的使用场景

做数据库备份，把整个数据库中的所有表查询出来存成文本格式

不支持事务的引擎上用全局锁

##### （5）全局锁的作用

在数据库做逻辑备份时，为了避免其他操作更新数据，给数据库加全局锁，使数据库处于只读状态，其他线程语句处于阻塞状态。

#####  （6）整个据库处于只读状态的缺点-----业务停止

主库做备份，处于只读状态时，备份期间不能执行更新，业务基本上停止；

从库做备份，处于只读状态时，备份期间主库依然在执行业务，但此时，从库不能执行主库生成的binlog日志，造成主库和从库数据不一致；

从库是建立一个和主库完全一样的数据库环境；

主库存储的是实时的业务数据，有一个Binlog日志，用来记录所有执行过的sql语句；

从库复制主库binlog日志中的sql语句，再执行一次。

##### （7）做逻辑备份的方法

###### 1）只适用于支持事务的存储引擎InnoDB——使用参数single-transaction

在备份之前，逻辑备份工具mysqldump用参数single-transaction，启动一个事务，事务的可重复性隔离级别确保数据的**一致性**，同时，MVCC确保备份过程中，数据正常更新。

启动事务——开始备份——因为事务的可重复性隔离级别的作用，备份在事务执行过程期间，所看到的数据都是一致的，同时，多版本并发控制可以保障数据的更新。

###### 2）对于不支持事务的引擎，只能用FTWRL方法

##### （8）readonly和FTWRL的区别（为什么不用readonly使数据库处于只读状态）

功能：readonly除了能使数据库处于只读状态，还可以用来做其他的逻辑；

异常：执行FTWRL命令后，客户端发生异常时，msyql会自动释放全局锁，这个数据库回到可以正常更新的状态；

而执行readonly命令发生异常，数据库一直保持readonly状态，长时间处于不更新的状态。

### 2、表级锁

两种表锁：表锁和元数据锁（meta data lock MDL)

使用场景：不支持行锁的引擎myisam

##### （1）表锁

###### 1）加表锁语法

###### `lock tables…read/write`

###### 2）释放锁语法

unlock tables   或客户端断开连接自动释放

###### 3）作用

举例：某一个线程A中执行了lock tables t1 read,t2 writer，意思是t1这个表只能读，t2这个表只能写；其他线程再想要写t1、读t2是不可以的，并且线程A也只能读t1，写t2。

处理并发，并发时限制别的线程对表的读写，也限制本线程对表的操作

###### 4）使用场景

并发控制，一张表上任何时刻只能有一个更新在执行

##### （2）元数据锁MDL

使用表时，自动加MDL锁。

###### 1）作用

保证读写正确（一个线程查询表中的数据，得到结果，而查询期间，另一个线程删除了这个表的一行，此时，查到的数据和目前的表结果不一致）

###### 2）使用场景

当对一个表做增删改查（DML）操作时，加MDL读锁（此刻进程对表执行读操作，别的进程可以读，但不可写）；

当对表进行结构变更（DDL）操作时，加MDL写锁（此刻进程对表执行写操作，别的进程不可读也不可写）。

DML：数据操纵语言，insert/delete/update/select（删除表中的某一行数据）

DDL：数据定义语言，create/alter/drop(库、表、视图等)（修改表，增加或删除一个字段（列），会改变表结构）

######  3）读写关系

读读共享：可以有多个进程对同一张表进行DML操作

读写互斥、写写互斥

session1:begin:--开启事务

​                select * from mylock;--加MDL读锁

​                select id from mylock;--加MDL读锁，可以执行

session2:alter table mylock add f int;--DDL操作，要加写锁，读写互斥，这个修改被阻塞

session1:commit;--提交事务 或者 rollback 释放读锁

session2:query ok,0 rows affected  --因为读锁释放了，此时修改可以进行，修改完成

###### 4）如何安全的给小表加字段？

在alter table表中设定等待时间。

# 07行锁功过：怎么减少行锁对性能的影响

使用场景：InnoDB

## 1、什么是行锁

行锁是对数据表中的**行**数据加锁

行锁分为共享锁（s锁）和排他锁(x锁)

## 2、两阶段锁原则

两阶段协议是指锁的操作分为两个阶段，加锁阶段和解锁阶段，在InnoDB引擎中，行锁在需要时加，在事务结束时释放。

## 3、在两阶段协议原则下，如果事务中有多行操作，需要锁多行，这些行的加锁（执行）顺序是怎么样的？

要把最可能造成锁冲突、最可能影响并发度的锁往后放。

## 4、死锁和死锁检测

##### （1）什么是死锁？

两个或两个以上进程在执行过程中，因为争夺资源造成互相等待的现象（循环等待），没有外力的推动下，都没法进行下去。

事务A等待事务B释放id=2的行锁，事务B等待事务A释放id=1的行锁，互相等待释放资源，进入死锁。

##### （2）解决死锁的策略

一、进入等待，直到超时；innodb_lock_wait_timeout=50s设置超时时间。

二、发起死锁检测，innodb_deadlock_detect=on开启死锁检测。

发起死锁检测后，主动回滚锁链条中的某一个事务，让其他事务能够继续执行。

##### （3）怎么解决热点行更新导致的性能问题？

很多个（1000）并发线程要更新同一行（热点行），此时，对于每个新来的线程，都要进行死锁检测，判断自己的加入是否会造成死锁。但是检测的这个期间会消耗大量的CPU资源。怎么解决？

一、临时关掉死锁检测，但是要保证这个业务一定不会出现死锁。

二、控制并发度。

