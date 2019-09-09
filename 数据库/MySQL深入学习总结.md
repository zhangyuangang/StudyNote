# **MySQL简介**

- 关于MySQL发音的官方答案：
    The official way to pronounce “MySQL” is “My Ess Que Ell” (not “my sequel”), but we do not mind if you pronounce it as “my sequel” or in some other localized way.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MySQL 可以分为 Server 层和存储引擎层两部分。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Server层包括连接器、查询缓存、分析器、优化器、执行器等**，涵盖MySQL的大多数核心服务功能，以及所有的内置函数（如日期、时间、数学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**存储引擎层负责数据的存储和提取。**其架构模式是插件式的，支持 InnoDB、MyISAM、Memory 等多个存储引擎。现在最常用的存储引擎是InnoDB，它从MySQL 5.5.5版本开始成为了默认存储引擎。**create table 语句中使用 engine=memory，来指定使用内存引擎创建表。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;现在最常用的存储引擎是InnoDB，它从MySQL 5.5.5版本开始成为了默认存储引擎。create table 语句中使用 engine=memory, 来指定使用内存引擎创建表。

# **查询语句执行过程**

![clipboard.png](/img/bVbxve2)

### **连接器**
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;第一步，连接器连接到数据库，连接器负责跟客户端建立连接、获取权限、维持和管理连接。
> 连接命令一般是这么写的：mysql -h$ip -P$port -u$user -p$password
>
> 账号密码错误会报错：Access denied for user

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;连接完成后，如果没有后续的动作，这个连接就处于空闲状态，可以在**show processlist**命令中看到它。文本中这个图是show processlist的结果，其中的Command列显示为"Sleep"的这一行，就表示现在系统里面有一个空闲连接。

![clipboard.png](/img/bVbxve5)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;客户端如果太长时间没动静，连接器就会自动将它断开。这个时间是由参数**wait timeout**控制的，默认值是8小时。

> 断开后再执行sql会报错：Lost connection to MySQL server during query

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;建立连接的过程通常是比较复杂的，所以建议在使用中要尽量减少建立连接的动作，也就是**尽量使用长连接。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;但是 MySQL 在执行过程中临时使用的内存是管理在连接对象里面的。这些资源会在连接断开的时候才释放。所以如果长连接累积下来，可能导致内存占用太大，被系统强行杀掉（OOM），从现象看就是 MySQL 异常重启。

怎么解决这个问题呢？可以考虑以下两种方案。

1. 定期断开长连接。使用一段时间，或者程序里面判断执行过一个占用内存的大查询后，断开连接，之后要查询再重连。
2. **如果用的是MySQL 5.7或更新版本，可以在每次执行一个比较大的操作后，通过执行mysql reset connection来重新初始化连接资源。这个过程不需要重连和重新做权限验证但是会将连接恢复到刚刚创建完时的状态。**

### **查询缓存**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;第二步，查询语句会先查询缓存，之前执行过的语句及其结果可能会以 key-value 对的形式，被直接缓存在内存中。key 是查询的语句，value 是查询的结果。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;但是**查询缓存利大于弊**，因为查询缓存的失效非常频繁，只要有对一个表的更新，这个表上所有的查询缓存都会被清空。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;除非是静态配置表才适合用查询缓存。**可以将参数 query_cache_type 设置成DEMAND，这样对于默认的 SQL 语句都不使用查询缓存。SQL_CACHE 显式指定使用查询缓存。**

> select SQL_CACHE * from T where ID=10；

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;但是，**MySQL 8.0版本彻底删除了查询缓存功能。**

### **分析器**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;第三步，分析语句，先是词法分析，找出select，表名，列名等关键字；然后是语法分析，判断语法是否正确。**表名列名不对的sql，会在语法分析时报错。**

> 语法错误：ERROR 1064 (42000): You have an error in your SQL syntax;

### **优化器**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;第四步，**决定使用哪个索引，join的时候决定各个表的连接顺序。**

### **执行器**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;第五步，**先判断对当前表是否有权限（如果命中查询缓存，会在返回结果时验证权限）。**

> ERROR 1142 (42000): SELECT command denied to user 'b'@'localhost' for table 'T'

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如：select * from T where ID=10; 执行过程

1. 调用 InnoDB 引擎接口取这个表的第一行，判断 ID 值是不是 10，如果不是则跳过，如果是则将这行存在结果集中；
2. 调用引擎接口取“下一行”，重复相同的判断逻辑，直到取到这个表的最后一行。
3. 执行器将上述遍历过程中所有满足条件的行组成的记录集作为结果集返回给客户端。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;慢查询日志中有一行 rows_examined 字段，表示这个语句执行过程中扫描了多少行。这个值就是在执行器每次调用引擎获取数据行的时候累加的。但是引擎扫描行数跟 rows_examined 并不是完全相同的。

### **查询的数据如何返回**

- 对一个200G的大表做全表扫描，而内存只有16G，会不会把数据库主机的内存用光了？

  实际上，MySQL不是取到全部数据再返回客户端。取数据和发数据的流程是这样的：

  1. 获取一行，写到 net_buffer 中。这块内存的大小是由参数 net_buffer_length 定义的，默认是 16k。
  2. 重复获取行，直到 net_buffer 写满，调用网络接口发出去。
  3. 如果发送成功，就清空 net_buffer，然后继续取下一行，并写入 net_buffer。
  4. 如果发送函数返回 EAGAIN 或 WSAEWOULDBLOCK，就表示本地网络栈（socket send buffer）写满了，进入等待。直到网络栈重新可写，再继续发送。

- MySQL 客户端发送请求后，接收服务端返回结果的方式有两种：

  1. 一种是本地缓存，也就是在本地开一片内存，先把结果存起来。如果用 API 开发，对应的就是 mysql_store_result 方法。

  2. 另一种是不缓存，读一个处理一个。如果用 API 开发，对应的就是 mysql_use_result 方法。

     >  **MySQL 客户端默认采用第一种方式，而如果加上–quick 参数，就会使用第二种不缓存的方式。**
     >
     > 采用不缓存的方式时，如果本地处理得慢，就会导致服务端发送结果被阻塞，因此会让服务端变慢。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**MySQL 是“边读边发的”。这就意味着，如果客户端接收得慢，会导致 MySQL 服务端由于结果发不出去，这个事务的执行时间变长。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**对于正常的线上业务来说，如果一个查询的返回结果不会很多的话，都建议使用 mysql_store_result 这个接口，直接把查询结果保存到本地内存。**

# **更新语句执行过程**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;更新语句同样会走连接器，查询缓存（清空该表缓存），分析器，优化器这一套流程，与查询流程不一样的是，更新流程还涉及两个重要的日志模块，redo log（重做日志）和 binlog（归档日志）。

### **重做日志：redo log**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果每一次的更新操作都需要写进磁盘，然后磁盘也要找到对应的那条记录，然后再更新，整个过程 IO 成本、查找成本都很高。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MySQL采用了WAL技术，全称是 Write-Ahead Logging，的关键点就是**先写日志，再写磁盘。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;具体来说，当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 redo log里面，并更新内存，这个时候更新就算完成了。同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;但是如果 InnoDB 的 redo log 写满了。这时候系统会停止所有更新操作，把 checkpoint 往前推进(对应的所有脏页都 flush 到磁盘上)，redo log 留出空间可以继续写。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;一旦一个查询请求需要在执行过程中先 flush 掉一个脏页时，这个查询就可能要比平时慢了。由于刷脏页的逻辑会占用 IO 资源并可能影响到了更新语句，要尽量避免这种情况，就要合理地设置 innodb_io_capacity 的值，**并且平时要多关注脏页比例，不要让它经常接近 75%。**脏页比例是通过 Innodb_buffer_pool_pages_dirty/Innodb_buffer_pool_pages_total 得到的，具体的命令参考下面代码：

> mysql> select VARIABLE_VALUE into @a from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty';
> select VARIABLE_VALUE into @b from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_total';
> select @a/@b;
>
> 在 InnoDB 中，innodb_flush_neighbors 参数就是用来控制这个行为的，值为 1 的时候会有“连坐”机制，值为 0 时表示不找邻居，自己刷自己的。固态硬盘建议设置为0。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;InnoDB 的 redo log 是可以配置的固定大小，比如可以配置为一组 4 个文件，每个文件的大小是 1GB，总共就可以记录 4GB 的操作。从头开始写，写到末尾就又回到开头循环写，如下面这个图所示。**如果redo log 设置的太小，磁盘压力很小，但是数据库出现间歇性的性能下跌。**
![clipboard.png](/img/bVbxvfg)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;write pos 是当前记录的位置，一边写一边后移，写到第 3 号文件末尾后就回到 0 号文件开头。checkpoint 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;write pos 和 checkpoint 之间的是还空着的部分，可以用来记录新的操作。如果 write pos 追上 checkpoint，这时候就得停下来先擦掉一些记录，把 checkpoint 推进一下。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**有了 redo log，InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为crash-safe。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**redo log buffer **：插入数据的过程中，生成的日志都得先保存起来，但又不能在还没 commit 的时候就直接写到 redo log 文件里。所以，redo log buffer 就是一块内存，用来先存 redo 日志的。也就是说，在执行第一个 insert 的时候，数据的内存被修改了，在执行 commit 的时候 redo log buffer 才写入了日志。

为了控制 redo log 的写入策略，innodb_flush_log_at_trx_commit 参数，它有三种可能取值：
1. 设置为 0 的时候，表示每次事务提交时都只是把 redo log 留在 redo log buffer 中 ;
2. 设置为 1 的时候，表示每次事务提交时都将 redo log 直接持久化到磁盘；
3. 设置为 2 的时候，表示每次事务提交时都只是把 redo log 写到 page cache。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;InnoDB 有一个后台线程，每隔 1 秒，就会把 redo log buffer 中的日志，调用 write 写到文件系统的 page cache，然后调用 fsync 持久化到磁盘。也就是说，**一个没有提交的事务的 redo log，也是可能已经持久化到磁盘的。**

还有两种场景也会把没有提交的redo log 写到硬盘。

1. **redo log buffer 占用的空间即将达到 innodb_log_buffer_size 一半的时候，后台线程会主动写盘。**注意，由于这个事务并没有提交，所以这个写盘动作只是 write，而没有调用 fsync，也就是只留在了文件系统的 page cache。

2. **并行的事务提交的时候，顺带将这个事务的 redo log buffer 持久化到磁盘。**假设一个事务 A 执行到一半，另一个事务B提交，事务B要把 redo log buffer 里的日志全部持久化到磁盘。


### **归档日志：binlog**

redo log 是 InnoDB 引擎特有的日志，而 Server 层也有自己的日志，称为 binlog（归档日志）。

binlog 的三种格式对比：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**statement：记录到 binlog 里的是语句原文，最后会有 COMMIT；可能会导致主备不一致，因为limit 、等sql 执行时可能主备优化器选择的索引不一样，排序也不一样。now()执行的结果也不一样。**
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**row ：记录了操作的事件每一条数据的变化情况，最后会有一个 XID event。缺点是太占空间。**
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**mixed：同时使用两种格式，由数据库判断具体某条sql使用哪种格式。但是有选择错误的情况。**

这两种日志有以下三点不同。

1. **redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。**
2. **redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。**
3. **redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。**

redo log 和 binlog 是怎么关联起来的?
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;它们有一个共同的数据字段，叫 XID。崩溃恢复的时候，会按顺序扫描 redo log：
- 如果碰到既有 prepare、又有 commit 的 redo log，就直接提交；
- 如果碰到只有 parepare、而没有 commit 的 redo log，就拿着 XID 去 binlog 找对应的事务。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;处于 prepare 阶段的 redo log 加上完整 binlog，重启也能恢复，因为 binlog 完整了，那么从库就同步过去了，为了保证主从一致，有完整的 binlog 就算成功。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**事务执行过程中，先把日志写到 binlog cache，事务提交的时候，再把 binlog cache 写到 binlog 文件中。**

write 和 fsync 的时机，是由参数 sync_binlog 控制的： 

1. sync_binlog=0 的时候，表示每次提交事务都只 write，不 fsync； 
2. sync_binlog=1 的时候，表示每次提交务都会执行 fsync；
3. sync_binlog=N(N>1) 的时候，表示每次提交事务都 write，但累积 N 个事务后才 fsync。

> 比较常见的是将其设置为 100~1000 中的某个数值。对应的风险是：如果主机发生异常重启，会丢失最近 N 个事务的 binlog 日志。

### **更新语句执行过程**

比如：update T set c=c+1 where ID=2;

1. 执行器先找引擎取 ID=2 这一行。如果 ID=2 这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。
2. 执行器拿到引擎给的行数据，把这个值加上 1，得到新的一行数据，再调用引擎接口写入这行新数据。
3. **引擎将这行新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，此时 redo log 处于 prepare 状态。然后告知执行器执行完成了，随时可以提交事务。**
4. **执行器生成这个操作的 binlog，并把 binlog 写入磁盘。**
5. **执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交（commit）状态，更新完成。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这里给出这个 update 语句的执行流程图，图中浅色框表示是在 InnoDB 内部执行的，深色框表示是在执行器中执行的。**其实就是把redo log 和binlog 做两阶段提交，为了让两份日志之间的逻辑一致。**
![clipboard.png](/img/bVbxvfi)

### 备份恢复
**保存一定时间的binlog，同时系统会定期做整库备份。**

当需要恢复到指定的某一秒时，

1. 首先，找到最近的一次全量备份，如果运气好，可能就是昨天晚上的一个备份，从这个备份恢复到临时库
2. 然后，从备份的时间点开始，将备份的 binlog 依次取出来，重放到指定的那个时刻。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**redo log 用于保证 crash-safe 能力。innodb_flush_log_at_trx_commit 这个参数设置成 1 的时候，表示每次事务的 redo log 都直接持久化到磁盘。**这个参数建议设置成 1，这样可以保证 MySQL 异常重启之后数据不丢失。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**binlog用于备份恢复和从库同步。sync_binlog 这个参数设置成 1 的时候，表示每次事务的 binlog 都持久化到磁盘。**这个参数也建议设置成 1，这样可以保证 MySQL 异常重启之后 binlog 不丢失。

### **主备同步**

1. 在备库 B 上通过 change master 命令，设置主库 A 的 IP、端口、用户名、密码，以及要从哪个位置开始请求 binlog，这个位置包含文件名和日志偏移量。
2. 在备库 B 上执行 start slave 命令，这时候备库会启动两个线程，就是图中的 io_thread 和 sql_thread。其中 io_thread 负责与主库建立连接。
3. 主库 A 校验完用户名、密码后，开始按照备库 B 传过来的位置，从本地读取 binlog，发给 B。
4. 备库 B 拿到 binlog 后，写到本地文件，称为中转日志（relay log）。
5. sql_thread 读取中转日志，解析出日志里的命令，并执行。
![clipboard.png](/img/bVbxvfj)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;一主一备结构，需要注意主备切换，备库设置只读，避免切换bug造成双写不一致问题（设置 readonly 对超级用户是无效的，同步更新的线程有超级权限，所以还能写入同步数据）。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;双主结构，要避免循环更新问题，因为MySQL 在 binlog 中记录了这个命令第一次执行时所在实例的 server id。所以可以规定两个库的 server id 必须不同，每个库在收到从自己的主库发过来的日志后，先判断 server id，如果跟自己的相同，表示这个日志是自己生成的，就直接丢弃这个日志。

### **主备延迟**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**可以在备库上执行 show slave status 命令，它的返回结果里面会显示 seconds_behind_master，用于表示当前备库延迟了多少秒。**每个事务的 binlog 里面都有一个时间字段，用于记录主库上写入的时间； 备库取出当前正在执行的事务的时间字段的值，计算它与当前系统时间的差值，得到 seconds_behind_master。

主备延迟最直接的表现是，备库消费中转日志（relay log）的速度，比主库生产 binlog 的速度要慢。

主备延迟的来源

1. 有些部署条件下，备库所在机器的性能要比主库所在的机器性能差。

2. 考虑到主备切换，主备机器一般都一样了，但是还可能备库读的压力太大，

   > 一主多从，或者通过binlog输出到外部系统(比如Hadoop)，让外部系统提供部分统计查询能力。

3. **大事务，如果事务执行十分钟，那就会导致主从延迟十分钟。**

### **主备复制策略**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**在官方的 5.6 版本之前，MySQL 只支持单线程复制**，由此在主库并发高、TPS 高时就会出现严重的主备延迟问题。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;并行复制策略有按表并行分发策略，按行并行分发策略，但是按行分发在决定线程分发的时候，需要消耗更多的计算资源。这两个方案其实都有一些约束条件：
1. 要能够从 binlog 里面解析出表名、主键值和唯一索引的值。也就是说，主库的 binlog 格式必须是 row；
2. 表必须有主键；
3. 不能有外键。表上如果有外键，级联更新的行不会记录在 binlog 中，这样冲突检测就不准确。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**官方 MySQL5.6 版本，支持了并行复制，只是支持的粒度是按库并行。**相比于按表和按行分发，这个策略有两个优势：

1. 构造 hash 值的时候很快，只需要库名；而且一个实例上 DB 数也不会很多，不会出现需要构造 100 万个项这种情况。
2. 不要求 binlog 的格式。因为 statement 格式的 binlog 也可以很容易拿到库名。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**MariaDB 的并行复制策略，伪模拟主库并发度**，主库 redo log 组提交 (group commit) 优化，同一组提交会记录commit_id，备库把同一个commit_id分发到多个worker执行。

官方的 MySQL5.7 版本，由参数 slave-parallel-type 来控制并行复制策略：

1. 配置为 DATABASE，表示使用 MySQL 5.6 版本的按库并行策略； 
2. 配置为 LOGICAL_CLOCK，表示的就是类似 MariaDB 的策略。不过，MySQL 5.7 这个策略，针对并行度做了优化。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**MySQL 5.7.22 版本里，MySQL 增加了一个新的并行复制策略，基于 WRITESET 的并行复制。**对于事务涉及更新的每一行，计算出这一行的 hash 值，组成集合 writeset。如果两个事务没有操作相同的行，也就是说它们的 writeset 没有交集，就可以并行。

### **读写分离**

读写分离有两种方案：

1. 客户端直连方案，因为少了一层 proxy 转发，所以查询性能稍微好一点儿，并且整体架构简单，排查问题更方便。但是这种方案，由于要了解后端部署细节，所以在出现主备切换、库迁移等作的时候，客户端都会感知到，并且需要调整数据库连接信息。 可能会觉得这样客户端也太麻烦了，信息大量冗余，架构很丑。其实也未必，一般采用这样的架构，一定会伴随一个负责管理后端的组件，比如 Zookeeper，尽量让业务端只专注于业务逻辑开发。
2. 带 proxy 的架构，对客户端比较友好。客户端不需要关注后端细节，连接维护、后端信息维护等工作，都是由 proxy 完成的。但这样的话，对后端维护团队的要求会更高。而且，proxy 也需要有高可用架构。因此，带 proxy 架构的整体就相对比较复杂。

​		**主从延迟的情况下怎么办？**

1. 强制走主库方案；对于必须要拿到最新结果的请求，强制将其发到主库上。
2. sleep 方案；主库更新后，读从库之前先 sleep 一下。因为大多数情况下主备延迟在 1 秒之内。
3. 判断主备无延迟方案； 每次从库执行查询请求前，先判断 seconds_behind_master 是否已经等于 0。如果还不等于 0 ，那就必须等到这个参数变为 0 才能执行查询请求。
4. 配合 semi-sync 方案；半同步复制：
   1. 事务提交的时候，主库把 binlog 发给从库；
   2. 从库收到 binlog 以后，发回给主库一个 ack，表示收到了； 
   3. 主库收到这个 ack 以后，才能给客户端客户端返回“事务完成”的确认。
5. 等主库位点方案；
6. 等 GTID 方案。

# **隔离级别**

### **数据库特性**

**ACID（Atomicity、Consistency、Isolation、Durability，即原子性、一致性、、隔离性、持久性）。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当数据库上有多个事务同时执行的时候，就可能出现**脏读（dirty read）、不可重复读（non-repeatable read）、幻读（phantom read）**的问题，为了解决这些问题，就有了“隔离级别”的概念。

- 脏读：指的是一个事务的读操作读到了另一个未提交的事务修改的值。

- 不可重复读：指的是一个事务读了同一个值两次，但是两次的值不同，因为中间另一个事务修改了这个值。

- 幻读：仍然指的是一个事务中读了两次，结果不同，但是与不可重复读不同的是，这里不同是因为别的事物做了插入操作，而是读的条件是一个范围的条件，这样第二次会多读到一条数据。

  > 不可重复读重点在于update和delete，而幻读的重点在于insert。

### **幻读问题——间隙锁**
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;即使把所有的记录都加上锁，还是阻止不了新插入的记录，也就是说行锁解决不了幻读问题，行锁只能锁住行，但是新插入记录这个动作，要更新的是记录之间的“间隙”。因此，为了解决幻读问题，InnoDB 只好引入新的锁，也就是间隙锁 (Gap Lock)。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当执行 select * from t where d=5 for update 的时候，就不止是给数据库中已有的 6 个记录加上了行锁，还同时加了 7 个间隙锁。这样就确保了无法再插入新的记录。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;间隙锁和行锁合称 next-key lock，每个 next-key lock 是前开后闭区间。也就是说，表 t 初始化以后，如果用 select * from t for update 要把整个表所有记录锁起来，就形成了 7 个 next-key lock，分别是(-∞,0]、(0,5]、(5,10]、(10,15]、(15,20]、(20, 25]、(25, +supremum]。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**间隙锁和 next-key lock 的引入，解决了幻读的问题，但同时也带来了一些“困扰”。间隙锁的引入，可能会导致同样的语句锁住更大的范围，这其实是影响了并发度的。**

### **隔离级别**
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SQL 标准的事务隔离级别包括：读未提交read uncommitted）、读提交（read committed）、可重复读（repeatable read）和串行化（serializable ）。**隔离级别越高，效率越低。**

- 读未提交是指，一个事务还没提交时，它做的变更就能被别的事务看到。
- 读提交是指，一个事务提交之后，它做的变更才会被其他事务看到。
- 可重复读是指，一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。当然在可重复读隔离级别下，未提交变更对其他事务也是不可见的。
- 串行化，顾名思义是对于同一行记录，“写”会加“写锁”，“读” 会加“读锁”。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行。

**在实现上，数据库里面会创建一个视图，访问的时候以视图的逻辑结果为准。**

1. “可重复读”隔离级别下：这个视图是在事务启动时创建的，整个事务存在期间都用这个视图。

2. “读提交”隔离级别下：这个视图是在每个 SQL 语句开始执行的时候创建的。

3. “读未提交”隔离级别下：直接返回记录上的最新值，没有视图概念

4. “串行化”隔离级别下：直接用加锁的方式来避免并行访问。

在 MySQL 里，有两个“视图”的概念：

- 一个是 view。它是一个用查询语句定义的虚拟表，在调用的时候执行查询语句并生成结果。创建视图的语法是 create view … ，而它的查询方法与表一样。
- 另一个是 InnoDB 在实现 MVCC 时用到的一致性读视图，即 consistent read view，用于支持 RC（Read Committed，读提交）和 RR（Repeatable Read，可重复读）隔离级别的实现。
**MySQL 默认隔离级别是可重复读，Oracle 默认隔离级别是“读提交”。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;将启动参数 **transaction-isolation** 的值设置成 READ-UNCOMMITTED、READ-COMMITTED、REPEATABLE-READ 、SERIALIZABLE。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可以用 show variables 来查看当前的值。

### **事务隔离的实现——undo log**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;每条记录在更新的时候都会同时记录一条回滚操作。同一条记录在系统中可以存在多个版本，这就是数据库的（MVCC）。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**MVCC的全称是“多版本并发控制”。**为了查询一些正在被另一个事务更新的行，并且可以看到它们被更新之前的值，不用等待另一个事务释放锁。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**InnoDB会给数据库中的每一行增加三个字段，它们分别是DB_TRX_ID（事务版本号）、DB_ROLL_PTR（创建时间）、DB_ROW_ID（唯一id）。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**InnoDB 里面每个事务有一个唯一的事务 ID，叫作 transaction id。它是在事务开始的时候向 InnoDB 的事务系统申请的，是按申请顺序严格递增的。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**InnoDB 利用了“所有数据都有多个版本”的这个特性，实现了“秒级创建快照”的能力。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;B+Tree叶结点上，始终存储的是最新的数据（可能是还未提交的数据）。而旧版本数据，通过UNDO记录存储在回滚段（Rollback Segment）里。每一条记录都会维护一个ROW HEADER元信息，存储有创建这条记录的事务ID，一个指向UNDO记录的指针。**通过最新记录和UNDO信息，可以还原出旧版本的记录。**

假设一个值从 1 被按顺序改成了 2、3、4，在回滚日志里面就会有类似下面的记录。
![clipboard.png](/img/bVbxvfn)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当前值是 4，但是在查询这条记录的时候，不同时刻启动的事务会有不同的 read-view。同一条记录在系统中可以存在多个版本，就是数据库的多版本并发控制（MVCC）。对于 read-view A，要得到 1，就必须将当前值依次执行图中所有的回滚操作得到。这些回滚信息记录在undo log 里。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当系统里没有比这个回滚日志更早的 read-view 的时候会删除老的undo log。

### **避免长事务**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**尽量不要使用长事务，长事务意味着系统里面会存在很老的事务视图。会有很大的undo log日志占用空间。而且长事务还会占据锁资源，也可能拖垮整个库。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可以在 **information_schema 库的innodb_trx** 这个表中查询长事务，比如下面这个语句，用于查找持续时间超过 60s 的事务。可以监控这个表，设置长事务阈值报警或者直接kill。

> select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可以通过 SET MAX_EXECUTION_TIME 命令来控制每个语句执行的最长时间，避免单个语句意外执行太长时间。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;确认是否有不必要的只读事务。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果使用的是 MySQL 5.6 或者更新版本，把 innodb_undo_tablespaces 设置成 2或更大的值）。如果真的出现大事务导致undo log过大，这样设置后清理起来更方便。

# **索引**

### **常见索引模型**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Hash表 + 链表，查询新增都很快，但是只适用于只有等值查询的场景，不能区间查询， Memcached 及其他一些 NoSQL 引擎在用。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;有序数组，等值查询和范围查询场景中的性能就都非常优秀，二分查找O(log(N))，但是更新的效率很低，所以只适用于静态存储引擎。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;平衡二叉树，更新和查询都比较快。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;还有跳跃表，LSM树等。

### **B+ 树**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;为了让一个查询尽量少地读磁盘，就需要使用多叉树。MySQL采用的是B+树，由于索引不止存在内存中，还要写到磁盘上。二叉树的树高太高，100万数据，就有20层，在机械硬盘时代，从磁盘随机读一个数据块需要 10 ms 左右的寻址时间。就要花费200ms的寻址时间，就太慢了。MySQL  B+树 的一层节点数量在1200左右，只需要1-3次磁盘IO就可以了，因为InnoDB存储引擎的最小储存单元页（Page），一个页的大小是16K。一般来说主键id为bigint类型，长度8字节，指针6字节，那么16284/14 = 1170。所以一次IO最多读取1170个节点。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;相对于B树，B+树把所有的数据都放在了叶子节点上，这样虽然每次都需要查询叶子节点，但也不过两三层，如果干节点也放数据，那干节点就变大了，一次就读取不了1200节点了，层高会变大很多。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;并且MySQL把B+树的所有叶子节点的数据用指针连起来了，这样做区间查询是非常快的。

### **主键索引和非主键索引**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**主键索引的叶子节点存的是整行数据。**在 InnoDB 里，主键索引也被称为聚簇索引（clustered index）。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**非主键索引的叶子节点内容是主键的值。**在 InnoDB 里，非主键索引也被称为二级索引（secondary index）。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**查询语句，如果走主键索引，会直接得到数据，如果走非主键索引，查到主键后，还需要回主键索引再查一次数据。这个过程称为回表。（覆盖索引不需要回表）**
![clipboard.png](/img/bVbxvfo)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;分为聚簇索引和非聚簇索引的原因：更新数据的时候，由于数据的地址变了，需要更改索引，但是由于数据只跟主键索引绑定，索引只需要更新聚簇索引，当然还有被更新列涉及到的索引也要更新。如果所有所有都跟数据绑定，虽然省掉了回表的过程，但是每次更新，需要更新所有的索引，得不偿失。

### **索引维护**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;B+ 树为了维护索引有序性，在插入新值的时候需要做必要的维护。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;比如按顺序插入1-499,501-1000，索引都在一页，再插入一个500，根据 B+ 树的算法，这时候需要申请一个新的数据页，然后挪动部分数据(501到1000的数据)过去。这个过程称为**页分裂**。在这种情况下，性能自然会受影响。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**除了影响性能外，页分裂操作还影响数据页的利用率。原本放在一个页的数据，现在分到两个页中，整体空间利用率降低大约 50%。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当然有分裂就有合并。当相邻两个页由于删除了数据，利用率很低之后，会将数据页做合并。合并的过程，可以认为是分裂过程的逆过程。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;所以一般建表规范都要求用自增主键，避免页分裂，当然也有特殊情况，使用别的字段当做主键。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;并且索引可能因为删除，或者页分裂等原因，导致数据页有空洞，**重建索引**的过程会创建一个新的索引，把数据按顺序插入，这样页面的利用率最高，也就是索引更紧凑、更省空间。

> alter table T drop index k;
> alter table T add index(k);

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;但是**不能重建主键索引**，不论是删除主键还是创建主键，都会将整个表重建。可以使用 alter table T engine=InnoDB 重建表。

### **覆盖索引**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果执行的语句是 select ID from T where k between 3 and 5，这时只需要查 ID 的值，而 ID 的值已经在 k 索引树上了，因此可以直接提供查询结果，不需要回表。也就是说，在这个查询里面，索引 k 已经“覆盖了”查询需求，称为覆盖索引。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;由于覆盖索引可以减少树的搜索次数，显著提升查询性能，所以使用覆盖索引是一个常用的性能优化手段。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果有根据身份证号查询市民信息的需求，只要在身份证号字段上建立索引就够了。如果现在有一个高频请求，要根据市民的身份证号查询他的姓名，再建立一个（身份证号、姓名）的联合索引就是覆盖索引，省去了回表环节。

### **最左前缀原则**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果为每一种查询都设计一个索引，索引是不是太多了。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;B+ 树这种索引结构，可以利用索引的“最左前缀”，来定位记录。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;为了直观地说明这个概念，用（name，age）这个联合索引来分析。
![clipboard.png](/img/bVbxvfp)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可以看到，索引项是按照索引定义里面出现的字段顺序排序的。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当逻辑需求是查到所有名字是“张三”的人时，可以快速定位到 ID4，然后向后遍历得到所有需要的结果。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果要查的是所有名字第一个字是“张”的人，SQL 语句的条件是"where name like ‘张 %’"。这时，也能够用上这个索引，查找到第一个符合条件的记录是 ID3，然后向后遍历，直到不满足条件为止。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可以看到，不只是索引的全部定义，只要满足最左前缀，就可以利用索引来加速检索。这个**最左前缀可以是联合索引的最左 N 个字段，也可以是字符串索引的最左 M 个字符。**

### **前缀索引**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**使用前缀索引，定义好长度，就可以做到既节省空间，又不用额外增加太多的查询成本。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在建立索引时关注的是区分度，区分度越高越好。因为区分度越高，意味着重复的键值越少。因此，可以通过统计索引上有多少个不同的值来判断要使用多长的前缀。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可以使用下面这个语句，算出这个列上有多少个不同的值：

> select count(distinct email) as L from SUser;

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用前缀索引就用不上覆盖索引对查询性能的优化了，这是在选择是否使用前缀索引时需要考虑的一个因素。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;那么对于身份证号，一共 18 位，其中前 6 位是地址码，所以同一个县的人的身份证号前 6 位一般会是相同的。该怎么存储，怎么设计索引呢？

  1. 第一种方式是使用倒序存储。身份证号的最后 6 位没有地址码这样的重复逻辑。

     > select field_list from t where id_card = reverse('input_id_card_string');
     >
     > select field_list from t where id_card = reverse('input_id_card_string');


3. 第二种方式是使用 hash 字段。在表上再创建一个整数字段，来保存身份证的校验码，同时在这个字段上创建索引。

   > alter table t add id_card_crc int unsigned, add index(id_card_crc);
   >
   > 然后每次插入新记录的时候，都同时用 crc32() 这个函数得到校验码填到这个新字段。由于校验码可能存在冲突，所以查询语句 where 部分要判断 id_card 的值是否精确相同。
   >
   > select field_list from t where id_card_crc=crc32('input_id_card_string') and id_card='input_id_card_string'

### **索引下推**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;最左前缀的时候，那些不符合最左前缀的部分，会怎么样呢？

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果现在有一个需求：检索出表中“名字第一个字是张，而且年龄是 10 岁的所有男孩”。那么，SQL 语句是这么写的：

> mysql> select * from tuser where name like '张 %' and age=10 and ismale=1;

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这个语句在搜索索引树的时候，只能用 “张”，找到第一个满足条件的记录 ID3。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;然后需要判断其他条件是否满足。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在 MySQL 5.6 之前，只能从 ID3 开始一个个回表。到主键索引上找出数据行，再对比字段值。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**而 MySQL 5.6 引入的索引下推优化（index condition pushdown)，可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数。**
![clipboard.png](/img/bVbxvfw)

### **change buffer**
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当需要更新一个数据页时，如果数据页在内存中就直接更新，而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InooDB 会将这些更新操作缓存在 change buffer 中，这样就不需要从磁盘中读入这个数据页了。在下次查询需要访问这个数据页的时候，将数据页读入内存，然后执行 change buffer 中与这个页有关的操作。通过这种方式就能保证这个数据逻辑的正确性。**虽然是只更新内存，但是在事务提交的时候，把 change buffer 的操作也记录到 redo log 里了，所以崩溃恢复的时候，change buffer 也能找回来。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;需要说明的是，虽然名字叫作 change buffer，实际上它是可以持久化的数据。也就是说，change buffer 在内存中有拷贝，也会被写入到磁盘上。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;将 change buffer 中的操作应用到原数据页，得到最新结果的过程称为 merge。除了访问这个数据页会触发 merge 外，系统有后台线程会定期 merge。在数据库正常关闭（shutdown）的过程中，也会执行 merge 操作。

​&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;显然，如果能够将更新操作先记录在 change buffer，减少读磁盘，语句的执行速度会得到明显的提升。而且，数据读入内存是需要占用 buffer pool 的，所以这种方式能够避免占用内存，提高内存利用率。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**唯一索引的更新就不能使用 change buffer，实际上也只有普通索引可以使用。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;change buffer 用的是 buffer pool 里的内存，因此不能无限增大。change buffer 的大小，可以通过参数 innodb_change_buffer_max_size 来动态设置。这个参数设置为 50 的时候，表示 change buffer 的大小最多只能占用 buffer pool 的 50%。

​&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果要在这张表中插入一个新记录 (4,400) 的话，InnoDB 的处理流程是怎样的。

第一种情况是，这个记录要更新的目标页在内存中。这时，InnoDB 的处理流程如下： 

- 对于唯一索引来说，找到 3 和 5 之间的位置，判断到没有冲突，插入这个值，语句执行结束； 

- 对于普通索引来说，找到 3 和 5 之间的位置，插入这个值，语句执行结束。

  > 这个判断只会耗费微小的 CPU 时间。不是重点

第二种情况是，这个记录要更新的目标页不在内存中。这时，InnoDB 的处理流程如下：

- 对于唯一索引来说，需要将数据页读入内存，判断到没有冲突，插入这个值，语句执行结束；

-  **对于普通索引来说，则是将更新记录在 change buffer，语句执行就结束了。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;将数据从磁盘读入内存涉及随机 IO 的访问，是数据库里面成本最高的操作之一。change buffer 因为减少了随机磁盘访问，所以对更新性能的提升是会很明显的。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**change buffer 适用于写多读少的业务，比如账单类、日志类的系统。因为会记录很多change buffer（写的时候） 才会merge（读的时候）**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;反过来，读多写少的业务，几乎每次把更新记录在change buffer 后，就会立即出发merge，这样随机访问 IO 的次数不会减少，反而增加了change buffer 的维护代价。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;所以，对于身份证号这类字段，如果业务已经保证不会写入重复数据，不需要数据库做约束，加普通索引比加主键索引要好，如果所有的更新后面，都马上伴随着对这个记录的查询，那么应该关闭 change buffer。而在其他情况下，change buffer 都能提升更新性能。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在实际使用中，可以发现，**普通索引和 change buffer 的配合使用，对于数据量大的表的更新优化还是很明显的，特别是在使用机械硬盘时。**

**change buffer 和 redo log 对比**

> insert into t(id,k) values(id1,k1),(id2,k2);

这条更新语句做了如下操作：

1. Page 在内存中，直接更新内存；
2. Page 没有在内存中，就在内存的 change buffer 区域，记录下“要往 Page 插入一行”这个信。
3. 将上述两个动作记入 redo log 中。

后续的更新操作

1. Page 在内存中，会直接从内存返回。
2. Page 不在内容中，需要把 Page 从磁盘读入内存中，然后应用 change buffer 里面的操作日志，生成一个正确的版本并返回结果。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;所以，如果要简单地对比这两个机制在提升更新性能上的收益的话，**redo log 主要节省的是随机写磁盘的 IO 消耗（转成顺序写），而 change buffer 主要节省的则是随机读磁盘的 IO 消耗。**

### **优化器如何选择索引**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;优化器结合是否扫描行数、是否使用临时表、是否排序等因素进行综合判断。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MySQL 在真正开始执行语句之前，并不能精确地知道满足条件的记录有多少条，而只能根据统计信息来估算记录数。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这个统计信息就是索引的“区分度”。显然，一个索引上不同的值越多，这个索引的区分度就越好。而一个索引上不同的值的个数，称之为“基数”（cardinality）。也就是说，这个基数越大，索引的区分度越好。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**可以使用 show index 方法，看到一个索引的基数。**
![clipboard.png](/img/bVbxvfx)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MySQL 采样统计的方法获得基数，InnoDB 默认会选择 N 个数据页，统计这些页面上的不同值，得到一个平均值，然后乘以这个索引的页面数，就得到了这个索引的基数。当变更的数据行数超过 1/M 的时候，会自动触发重新做一次索引统计。**analyze table t 命令，可以用来重新统计索引信息。**

在 MySQL 中，有两种存储索引统计的方式，可以通过设置参数 innodb_stats_persistent 的值来选择：

- 设置为 on 的时候，表示统计信息会持久化存储。这时，默认的 N 是 20，M 是 10。

- 设置为 off 的时候，表示统计信息只存储在内存中。这时，默认的 N 是 8，M 是 16。

其实索引统计只是一个输入，对于一个具体的语句来说，优化器还要判断，执行这个语句本身要扫描多少行。

![clipboard.png](/img/bVbxvfy)
rows 这个字段表示的是预计扫描行数。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;少数情况下优化器会选错索引，**第一种方法可以采用 force index 强行选择一个索引。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;但其实使用 force index 最主要的问题还是变更的及时性。因为选错索引的情况还是比较少出现的，所以开发的时候通常不会先写上 force index。而是等到线上出现问题的时候，才会再去修改 SQL 语句、加上 force index。但是修改之后还要测试和发布，对于生产系统来说，这个过程不够敏捷。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;所以，数据库的问题最好还是在数据库内部来解决。既然优化器放弃了使用索引 a，说明 a 还不够合适，所以**第二种方法就是，可以考虑修改语句，引导 MySQL 使用期望的索引**。比如，在这个例子里，显然把“order by b limit 1” 改成 “order by b,a limit 1” ，语义的逻辑是相同的。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;之前优化器选择使用索引 b，是因为它认为使用索引 b 可以避免排序（b 本身是索引，已经是有序的了，如果选择索引 b 的话，不需要再做排序，只需要遍历），所以即使扫描行数多，也判定为代价更小。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;现在 order by b,a 这种写法，要求按照 b,a 排序，就意味着使用这两个索引都需要排序。因此，扫描行数成了影响决策的主要条件，于是此时优化器选了只需要扫描 1000 行的索引 a。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当然，这种修改并不是通用的优化手段，可能修改语义这件事儿不太好，可以用 limit 100 让优化器意识到，使用 b 索引代价是很高的。其实是根据数据特征诱导了一下优化器，也不具备通用性。

> select * from  (select * from t where (a between 1 and 1000)  and (b between 50000 and 100000) order by b limit 100)alias limit 1;

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**第三种方法是：在有些场景下，可以新建一个更合适的索引，来提供给优化器做选择，或删掉误用的索引。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器就决定放弃走树搜索功能。**

1. 条件字段函数操作

   > select count(*) from tradelog where month(t_modified)=7;
   >
   > 同理 where id+1=1000  也不会用索引，改成 where id =1000 - 1 会用索引。

2. 隐式类型转换

   > select * from tradelog where tradeid=110717;   （tradeid 是varchar）
   >
   > 等同于 select * from tradelog where  CAST(tradid AS signed int) = 110717;

3. 隐式字符编码转换

   > select * from trade_detail where tradeid=$L2.tradeid.value; 
   >
   > $L2.tradeid.value 的字符集是 utf8mb4。字符集 utf8mb4 是 utf8 的超集，所以当这两个类型的字符串在做比较的时候，MySQL 内部的操作是，先把 utf8 字符串转成 utf8mb4 字符集，再做比较。
   >
   > 相当于 select * from trade_detail  where CONVERT(traideid USING utf8mb4)=$L2.tradeid.value; 

# **全局锁和表锁**

### **全局锁**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**顾名思义，全局锁就是对整个数据库实例加锁。**MySQL 提供了一个加全局读锁的方法，命令是 Flush tables with read lock (FTWRL)。当需要**让整个库处于只读状态的时候**，可以使用可以使用这个命令，之后其他线程的以下语句会被阻塞：数据更新语句（数据的增删改）、数据定义语句（包括建表、修改表结构等）和更新类事务的提交语句。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**全局锁的典型使用场景是，做全库逻辑备份。**也就是把整库每个表都 select 出来存成文本。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通过 FTWRL 确保不会有其他线程对数据库做更新，然后对整个库做备份。在备份过程中整个库完全处于只读状，这是很危险的。但是不加锁，备份的数据会有不一致的问题。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**可以拿到一个一致性视图来备份，官方自带的逻辑备份工具是 mysqldump。当 mysqldump 使用参数–single-transaction 的时候，导数据之前就会启动一个事务，来确保拿到一致性视图。**而由于 MVCC 的支持，这个过程中数据是可以正常更新的。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;那为什么还需要FTWRL呢，因为一致性读是好，**但前提是引擎要支持这个隔离级别**。对于 MyISAM 这种不支持事务的引擎，就需要使用 FTWRL 命令了。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;既然要全库只读，为什么不使用 set global readonly=true 的方式呢？确实 readonly 方式也可以让全库进入只读状态，但还是建议用 FTWRL 方式，主要有两个原因：

- 在有些系统中，readonly 的值会被用来做其他逻辑，比如用来判断一个库是主库还是备库。因此，修改 global 变量的方式影响面更大，不建议使用。
- **在异常处理机制上有差异。如果执行 FTWRL 命令之后由于客户端发生异常断开，那么 MySQL 会自动释放这个全局锁，整个库回到可以正常更新的状态。而将整个库设置为 readonly 之后，如果客户端发生异常，则数据库就会一直保持 readonly 状态，这样会导致整个库长时间处于不可写状态，风险较高。**

### **表级锁**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MySQL 里面表级别的锁有两种：一种是表锁，一种是元数据锁（meta data lock，MDL)。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**表锁的语法是 lock tables … read/write。**与 FTWRL 类似，**可以用 unlock tables 主动释放锁，也可以在客户端断开的时候自动释放。**需要注意，lock tables 语法除了会限制别的线程的读写外，也限定了本线程接下来的操作对象。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;对于 InnoDB 这种支持行锁的引擎，一般不使用 lock tables 命令来控制并发，毕竟锁住整个表的影响面还是太大。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**另一类表级的锁是 MDL（metadata lock)。**MDL 不需要显式使用，在访问一个表的时候会被自动加上。MDL 的作用是，保证读写的正确性。可以想象一下，如果一个查询正在遍历一个表中的数据，而执行期间另一个线程对这个表结构做变更，删了一列，那么查询线程拿到的结果跟表结构对不上，肯定是不行的。

​&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;因此，在 MySQL 5.5 版本中引入了 MDL，**当对一个表做增删改查操作的时候，加 MDL 读锁；当要对表做结构变更操作的时候，加 MDL 写锁。**

- 读锁之间不互斥，因此可以有多个线程同时对一张表增删改查。
- 读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。

### **安全的给表增加字段**

​&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;有几个请求在读写表，会加上MDL读锁，然后修改表字段的请求会被blocked，请求MDL写锁，这个时候，后面的全部读写请求都会被MDL写锁 blocked，如果查询语句频繁，而且客户端有重试机制，也就是说超时后会再起一个新 session 再请求的话，这个库的线程很快就会爆满。

那么如何安全的给表加字段呢？

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;首先要解决长事务，事务不提交，就会一直占着 MDL 锁。在 MySQL 的 information_schema 库的 innodb_trx 表中，可以查到当前执行中的事务。如果要做 DDL 变更的表刚好有长事务在执行，要考虑先暂停 DDL，或者 kill 掉这个长事务。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;其次，在 alter table 语句里面设定等待时间，如果在这个指定的等待时间里面能够拿到 MDL 写锁最好，拿不到也不要阻塞后面的业务语句，先放弃。之后开发人员或者 DBA 再通过重试命令重复这个过程。

> ALTER TABLE tbl_name NOWAIT add column ...
> ALTER TABLE tbl_name WAIT N add column ... 

### **行锁**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**MyISAM 引擎就不支持行锁。**不支持行锁意味着并发控制只能使用表锁，对于这种引擎的表，同一张表上任何时刻只能有一个更新在执行，这就会影响到业务并发度。InnoDB 是支持行锁的，这也是 MyISAM 被 InnoDB 替代的重要原因之一。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**在 InnoDB 事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。这个就是两阶段锁协议。**

​&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**如果事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放。**

### **死锁**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当并发系统中不同线程出现循环资源依赖，涉及的线程都在等待别的线程释放资源时，就会导致这几个线程都进入无限等待的状态，称为死锁。这里用数据库中的行锁举个例子。
![clipboard.png](/img/bVbxvfA)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这时候，事务 A 在等待事务 B 释放 id=2 的行锁，而事务 B 在等待事务 A 释放 id=1 的行锁。 事务 A 和事务 B 在互相等待对方的资源释放，就是进入了死锁状态。当出现死锁以后，有两种策略：

- 一种策略是，直接进入等待，直到超时。这个超时时间可以通过参数 innodb_lock_wait_timeout 来设置。

  > 设置时间长，等待时间太长；设置时间短，有的长事务，不是死锁的也会结束。

- 另一种策略是，发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行。将参数 innodb_deadlock_detect 设置为 on，表示开启这个逻辑。

  > 每个新来的被堵住的线程，都要判断会不会由于自己的加入导致了死锁，这是一个时间复杂度是 O(n) 的操作。会耗费大量的CPU资源。

#### **慢SQL问题排查**

使用 show processlist 命令查看 Waiting for table metadata lock 的示意图。
![clipboard.png](/img/bVbxvfF)
这个状态表示的是，现在有一个线程正在表 t 上请求或者持有 MDL 写锁，把 select 语句堵住了。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通过查询 sys.schema_table_lock_waits 这张表，就可以直接找出造成阻塞的 process id，把这个连接用 kill 命令断开即可。

通过 sys.innodb_lock_waits 查行锁

> select * from t sys.innodb_lock_waits where locked_table=`'test'.'t'`\G
> ![clipboard.png](/img/bVbxvfI)
> 这个信息很全，4 号线程是造成堵塞的罪魁祸首。而干掉这个罪魁祸首的方式，就是 KILL QUERY 4 或 KILL 4。实际上，这里 KILL 4 才有效。

# **其他**

### **count(*) 语句分析**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MyISAM 引擎把一个表的总行数存在了磁盘上，因此执行 count(*) 的时候会直接返回这个数，效率很高；

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;InnoDB 引擎就麻烦了，执行 count(*) 的时候，需要把数据一行一行地从引擎里面读出来，然后累积计数。因为多版本并发控制（MVCC）的原因，InnoDB 表“应该返回多少行”也是不确定的。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;count() 是一个聚合函数，对于返回的结果集，一行行地判断，如果 count 函数的参数不是 NULL，累计值就加 1，否则不加。最后返回累计值。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;所以，count(*)、count(主键 id) 和 count(1) 都表示返回满足条件的结果集的总行数；而 count(字段），则表示返回满足条件的数据行里面，参数“字段”不为 NULL 的总个数。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**按照效率排序的话，count(字段) < count(主键id) < count(1) < count(\*)，所以建议，尽量使用count(\*)。**

### **order by 语句分析**

​&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MySQL 会给每个线程分配一块内存用于**快速排序**，称为 **sort_buffer**。

​&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;explain 结果里的 Extra 这个字段中的“Using filesort”表示的就是需要排序。

​&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;sort_buffer_size，就是 MySQL 为排序开辟的内存（sort_buffer）的大小。如果要排序的数据量小于 sort_buffer_size，排序就在内存中完成。但如果排序数据量太大，内存放不下，则不得不利用磁盘临时文件辅助排序。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**建立联合索引，甚至覆盖索引，可以避免排序过程。**

### **join 语句分析**

​&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;直接使用 join 语句，MySQL 优化器可能会选择表 t1 或 t2 作为驱动表，改用 straight_join 让 MySQL 使用固定的连接方式执行查询，这样优化器只会按照指定的方式去 join。

> select * from t1 straight_join t2 on (t1.a=t2.a);

![clipboard.png](/img/bVbxvfJ)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在这条语句里，**被驱动表 t2 的字段 a 上有索引，join 过程用上了这个索引，因此效率是很高的。称之为“Index Nested-Loop Join”，简称 NLJ。**

​&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**如果被驱动表 t2 的字段 a 上没有索引，那每次到 t2 去匹配的时候，就要做一次全表扫描。这个效率很低。这个算法叫做“Simple Nested-Loop Join”的算法，简称 BNL。**

​&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;所以在判断要不要使用 join 语句时，就是看 explain 结果里面，Extra 字段里面有没有出现“Block Nested Loop”字样。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在决定哪个表做驱动表的时候，应该是两个表按照各自的条件过滤，过滤完成之后，计算参与 join 的各个字段的总数据量，数据量小的那个表，就是“小表”，应该作为驱动表。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Multi-Range Read 优化，这个优化的主要目的是尽量使用顺序读盘。因为大多数的数据都是按照主键递增顺序插入得到的，所以可以认为，如果按照主键的递增顺序查询的话，对磁盘的读比较接近顺序读，能够提升读性能。**

> select * from t1 where a>=1 and a<=100;

![clipboard.png](/img/bVbxvfN)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Batched Key Access(BKA) 算法。这个 BKA 算法，其实就是对 NLJ 算法的优化。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;NLJ 算法执行的逻辑是：从驱动表 t1，一行行地取出 a 的值，再到被驱动表 t2 去做 join。也就是说，对于表 t2 来说，每次都是匹配一个值。这时，MRR 的优势就用不上了。

​&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;既然如此，就把表 t1 的数据取出来一部分，先放到一个临时内存。这个临时内存就是 join_buffer。

### **自增主键**

![clipboard.png](/img/bVbxvfO)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;表定义里面出现了一个 AUTO_INCREMENT=2，表示下一次插入数据时，如果需要自动生成自增值，会生成 id=2。

实际上，表的结构定义存放在后缀名为.frm 的文件中，但是并不会保存自增值。

- MyISAM 引擎的自增值保存在数据文件中。
- InnoDB 引擎的自增值，其实是保存在了内存里，MySQL 8.0 版本后，才有了“自增值持久化”的能力。
  - MySQL 5.7 及之前的版本，自增值保存在内存里，并没有持久化。每次重启后，第一次打开表的时候，都会去找自增值的最大值 max(id)，然后将 max(id)+1  作为这个表当前的自增值。﻿
  - MySQL 8.0 版本，将自增值的变更记录在了 redo log 中，重启的时候依靠 redo log 恢复重启之前的值。

**自增值修改机制**

- 如果插入数据时 id 字段指定为 0、null 或未指定值，那么就把这个表当前的 AUTO_INCREMENT 值填到自增字段； 
- 如果插入数据时 id 字段指定了具体的值 X ，就直接使用语句里指定的值 Y。
  - 如果 X < Y，那么这个表的自增值不变；
  - 如果 X≥Y，就需要把当前自增值修改为新的自增值。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**新的自增值生成算法是：从 auto_increment_offset 开始，以 auto_increment_increment 为步长，持续叠加，直到找到第一个大于 X 的值，作为新的自增值。**

**自增值的修改时机**

1. 执行器调用 InnoDB 引擎接口写入一行，传入的这一行的值(0,1,1); 
2. InnoDB 发现用户没有指定自增 id 的值，获取表 t 当前的自增值 2； 
3. 将传入的行的值改成 (2,1,1); 
4. 将表的自增值改成 3； 
5. 继续执行插入数据操作，由于已经存在 c=1 的记录，所以报 Duplicate key error，语句返回。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;所以，sql执行报错了，自增值已经改变了，**唯一键冲突是导致自增主键 id 不连续的第一种原因。同样地，事务回滚也会产生类似的现象，这就是第二种原因。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**批量插入的时候，由于系统预先不知道要申请多少个自增 id，所以就先申请一个，然后两个，然后四个，直到够用。这是主键 id 出现自增 id 不连续的第三种原因。**

### **自增id用完怎么办**

**1、主键id**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**再申请下一个 id 时，得到的值保持不变。**所以到最大值之后，再申请id，由于id不变，所以插入会报主键冲突，如果数据量比较大，主键id应该用 bigint unsigned。默认是无符号整型 (unsigned int) ，4 个字节2<sup>32</sup>-1（4294967295）。

**2、系统row_id**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**如果创建的 InnoDB 表没有指定主键，那么 InnoDB 会创建一个不可见的，长度为 6 个字节的 row_id。**InnoDB 维护了一个全局的 dict_sys.row_id 值，所有无主键的 InnoDB 表，每插入一行数据，都把当前的 dict_sys.row_id 值作为要插入数据的 row_id，然后把 dict_sys.row_id 的值加 1。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;实际上，在代码实现时 row_id 是一个长度为 8 字节的无符号长整型 (bigint unsigned)。但是，InnoDB 在设计时，给 row_id 留的只是 6 个节的长度，这样写到数据表中时只放了最后 6 个字节，所以 row_id 能写到数据表中的值，就有两个特征：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**2<sup>48</sup>-1到 2<sup>64</sup> 之间，row_id 会是0，2<sup>64</sup> 之后会从0开始。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**在 InnoDB 逻辑里，申请到 row_id=N 后，就将这行数据写入表中；如果表中已经存在 row_id=N 的行，新写入的行就会覆盖原有的行。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**覆盖数据，就意味着数据丢失，影响的是数据可靠性；报主键冲突，是插入失败，影响的是可用性。而一般情况下，可靠性优先于可用性。**

**3、Xid**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;redo log 和 binlog 相配合的时候，提到了有一个共同的字段叫作 Xid。它在 MySQL 中是用来对应事务的。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MySQL 内部维护了一个全局变量 global_query_id，每次执行语句的时候将它赋值给 Query_id，然后给这个变量加 1。如果当前语句是这个事务执行的第一条语句，那么 MySQL 还会同时把 Query_id 赋值给这个事务的 Xid。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**而 global_query_id 是一个纯内存变量，重启之后就清零了。所以就知道了，在同一个数据库实例中，不同事务的 Xid 也是有可能相同的。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**但是 MySQL 重启之后会重新生成新的 binlog 文件，这就保证了，同一个 binlog 文件里，Xid 一定是惟一的。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**但是 global_query_id 定义的长度是 8 个字节，这个自增值的上限是 2<sup>64</sup>-1。理论上也是可能重复的。**

**4、trx_id**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**Xid 是由 server 层维护的。InnoDB 内部使用 Xid，就是为了能够在 InnoDB 事务和 server 之间做关联。但是，InnoDB 自己的 trx_id，是另外维护的。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;InnoDB 内部维护了一个 max_trx_id 全局变量，每次需要申请一个新的 trx_id 时，就获得 max_trx_id 的当前值，然后并将 max_trx_id 加 1。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**InnoDB 数据可见性的核心思想是：每一行数据都记录了更新它的 trx_id，当一个事务读到一行数据的时候，判断这个数据是否可见的方法，就是通过事务的一致性视图与这行数据的 trx_id 做对比。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**对于正在执行的事务，可以从 information_schema.innodb_trx 表中看到事务的 trx_id。**

   > ​		update 和 delete 语句除了事务本身，还涉及到标记删除旧数据，也就是要把数据放到 purge 队列里等待后续物理删除，这个操作也会把 max_trx_id+1， 因此在一个事务中至少加 2；
   >
   > ​		InnoDB 的后台操作，比如表的索引信息统计这类操作，也是会启动内部事务的，因此你可能看到，trx_id 值并不是按照加 1 递增的。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**只读事务会分配一个特殊的，比较大的id，**把当前事务的 trx 变量的指针地址转成整数，再加上 2<sup>48</sup>，使用这个算法，就可以保证以下两点：

   1. 因为同一个只读事务在执行期间，它的指针地址是不会变的，所以不论是在 innodb_trx 还是在 innodb_locks 表里，同一个只读事务查出来的 trx_id 就会是一样的。
   2. 如果有并行的多个只读事务，每个事务的 trx 变量的指针地址肯定不同。这样，不同的并发只读事务，查出来的 trx_id 就是不同的。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;加上2<sup>48</sup>是为了保证只读事务显示的 trx_id 值比较大，正常情况下就会区别于读写事务的 id。理论情况下也可能只读事务与读写事务相等，但是没有影响。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;max_trx_id 会持久化存储，重启也不会重置为 0，那么从理论上讲，只要一个 MySQL 服务跑得足够久，就**可能出现 max_trx_id 达到 2<sup>48</sup>-1 的上限，然后从 0 开始的情况。当达到这个状态后，MySQL 就会持续出现一个脏读的 bug。因为后续的trx_id肯定比末尾那些trx_id大，能看到这些数据。**	

**5、thread_id**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;系统保存了一个全局变量 thread_id_counter，每新建一个连接，就将 thread_id_counter 赋值给这个新连接的线程变量。定义的大小是 4 个字节，因此达到 232-1 后，它就会重置为 0，然后继续增加。但是，在 show processlist 里不会看到两个相同的 thread_id。因为 MySQL 设计了一个唯一数组的逻辑，给新线程分配 thread_id 的时候，逻辑代码是这样的：

   > do {
   > &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;new_id= thread_id_counter++;
   > } while (!thread_ids.insert_unique(new_id).second);

### **误删数据怎么办**

  1. delete 语句误删数据行：Flashback工具过闪回把数据恢复回来。 原理是修改 binlog 的内容，拿回原库重放。而能够使用这个方案的前提是，需要确保 binlog_format=row 和 binlog_row_image=FULL。

     > 如何预防：把 sql_safe_updates 参数设置为 on。，delete 或者 update 语句必须有where条件，否则执行会报错。

2. 误删库 / 表：全量备份，加增量日志，在应用日志的时候，需要跳过 12 点误操作的那个语句的 binlog：
	
   1. 如果原实例没有使用 GTID 模式，只能在应用到包含 12 点的 binlog 文件的时候，先用–stop-position 参数执行到误操作之前的日志，然后再用–start-position 从误操作之后的日志继续执行；
	
   2. 如果实例使用了 GTID 模式，就方便多了。假设误操作命令的 GTID 是 gtid1，那么只需要执行 set gtid_next=gtid1;begin;commit; 先把这个 GTID 加到临时实例的 GTID 集合，之后按顺序执行 binlog 的时候，就会自动跳过误操作的语句。
	
      > 如何加速恢复：使用 mysqlbinlog 命令时，加上一个–database 参数，用来指定误删表所在的库。
      > 在 start slave 之前，先通过执行﻿ ﻿change replication filter replicate_do_table = (tbl_name) 命令，就可以让临时库只同步误操作的表；

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**延迟复制备库**，一般的主备复制结构存在的问题是，如果主库上有个表被误删了，这个命令很快也会被发给所有从库，进而导致所有从库的数据表也都一起被误删了。延迟复制的备库是一种特殊的备库，通过 CHANGE MASTER TO MASTER_DELAY = N 命令，可以指定这个备库持续保持跟主库有 N 秒的延迟。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;比如把 N 设置为 3600，这就代表了如果主库上有数据被误删了，并且在 1 小时内发现了这个误操作命令，这个命令就还没有在这个延迟复制的备库执行。这时候到这个备库上执行 stop slave，再通过之前介绍的方法，跳过误操作命令，就可以恢复出需要的数据。

**预防误删库 / 表的方法，制定操作规范。这样做的目的，是避免写错要删除的表名。**

1. 在删除数据表之前，必须先对表做改名操作。然后，观察一段时间，确保对业务无影响以后再删除这张表。
2. 改表名的时候，要求给表名加固定的后缀（比如加_to_be_deleted)，然后删除表的动作必须通过管理系统执行。并且，管理系删除表的时候，只能删除固定后缀的表。

### **删除数据，为何表文件大小不变**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**delete 命令其实只是把记录的位置，或者数据页标记为了“可复用”，但磁盘文件的大小是不会变的。**也就是说，通过 delete 命令是不能回收表空间的。这些可以复用，而没有被使用的空间，看起来就像是“空洞”。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;实际上，不止是删除数据会造成空洞，插入数据也会。如果数据是随机插入的，就可能造成索引的数据页分裂。更新索引上的值，可以理解为删除一个旧的值，再插入一个新值。不难理解，这也是会造成空洞的。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;也就是说，**经过大量增删改的表，都是可能是存在空洞的。所以，如果能够把这些空洞去掉，就能达到收缩表空间的目的。而重建表，就可以达到这样的目的。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**使用 alter table A engine=InnoDB 命令来重建表。MySQL 会自动完成转存数据、交换表名、删除旧表的操作。**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;重建表的时候，InnoDB 不会把整张表占满，每个页留了 1/16 给后续的更新用。也就是说，其实重建表之后不是“最”紧凑的。

### **怎么复制一张表**

**1、mysqldump 方法**

使用 mysqldump 命令将数据导出成一组 INSERT 语句。你可以使用下面的命令：

> mysqldump -h$host -P$port -u$user --add-locks=0 --no-create-info --single-transaction  --set-gtid-purged=OFF db1 t --where="a>900" --result-file=/client_tmp/t.sql

然后可以通过下面这条命令，将这些 INSERT 语句放到 db2 库里去执行。

> mysql -h127.0.0.1 -P13000  -uroot db2 -e "source /client_tmp/t.sql"

**2、导出 CSV 文件**

直接将结果导出成.csv 文件。MySQL 提供了下面的语法，用来将查询结果导出到服务端本地目录：

> select * from db1.t where a>900 into outfile '/server_tmp/t.csv';

然后用下面的 load data 命令将数据导入到目标表 db2.t 中。

> load data infile '/server_tmp/t.csv' into table db2.t;

**3、物理拷贝方法**

直接拷贝文件是不行的，需要在数据字典中注册。

MySQL 5.6 版本引入了可传输表空间(transportable tablespace) 的方法，，可以通过导出 + 导入表空间的方式，实现物理拷贝表的功能。
