# 一、db 问题



## 1.1 like % 在前面时候为什么会匹配不到

应为索引前缀匹配，匹配不到索引

## 1.2 5.8 json 字段的查询方式

Mysql 5.7 以后提供了支持 json 的字段类型，同样提供了支持 json 操作的函数

查询：

``` sql
Column -> '$.key'  或者 column --> '$.key'

-- '' 引号内位 key 的路径

-- where 中还可以使用 JSON_CONSTANS(column, 'value', '$.key' )
```

修改:

```sql
-- 只插入，存在不会覆盖
JSON_INSERT(column, '$.key1', 'value1', '$.key2', 'value2')

-- 插入新值，如果存在就覆盖
JSON_SET(column, '$.key', 'value', '$.key2', 'value2')

-- 只覆盖存在的值
JSON_REPLACE(column, '$.key', 'value', '$.key2', 'value2')

-- 删除存在的值
JSON_REMOVE(column, '$.key', 'value', '$.key2', 'value2')
```

## 1.3 mysql 引擎是针对表的还是针对数据库的

针对表的，例如内存表的搜索引擎就是使用的 mysam


## 1.4 引擎，bin log, redo log, undo log
MySQL 自带的引擎是 MyISAM， binlog 是 MySQL 的 Server 层实现的，没有 crash-safe 的能力所以只能用于归档。而 InnoDB 是另一个公司以插件形式引入 MySQL 的，并引入了 redo log 来实现 crash-safe 能力。

1. redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。
2. redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。
3. redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

bing log 加上时间点的全量备份可以完整恢复数据库，也可以用作于主从同步。
redo log 减少磁盘 IO，保证事务持久性，防止发生故障时间点，有脏页未写入磁盘。当 mysql 重启服务时，根据 redo log 从做。
undo log 保存事务之前的数据若干个版本，可以用于回滚，同时提供多版本控制的读（MVCC）也叫非锁定读

为了让两份日志之间的记录一致，可以配置 sync_binlog 和 innodb_flush_log_at_trx_commit 都设置成 1，也就是说，一个事务完整提交前，需要等待两次刷盘，一次是 redo log（prepare 阶段），一次是 binlog。每完成一次事务需要两次刷盘，为了提升 IO 效率 mysql 对 bin log 和 redo log 进行了组提交的策略，由 bin log 做协调者，最大化刷盘收益

## 1.5 索引

InnoDB 使用 B+ 树来创建索引，使用 B+ 树作为索引的好处也就是充分利用了 B+ 树的优势，B+ 树是 B 树的变种且都是平衡树，B+ 树的数据存储在叶子节点，叶子节点，所有的叶子结点会组成一个链表。
主键索引叶子结点存储的是整行数据，也称为聚集索引。非主键索引的叶子结点存储的是主键的值，也称二级索引。基于非主键索引进行查询会多扫描一棵索引树。
索引的维护应遵循以下原则；
- 索引的主键应尽量小，以减轻非主键索引的大小
- 覆盖索引可以减少回表次数
- 最左前缀匹配创建索引解决 like 查询缓慢问题
- mysql 5.6 以后引入下推索引，优先匹配在索引中存在的字段



## 1.6 mysql乐观锁 悲观锁

悲观锁：悲观锁认为每次的操作都会有竞争问题，所以每次操作都会加锁，synchronized属于悲观锁、sql 语句中的 forupdate

乐观锁：乐观锁认为每次操作都不会产生竞争问题，操作的时候不加锁，操作提交的时候判断期间是否有其他线程修改，可以通过多版本控制，和时间戳来判断

## .MySQL存储使用的方式 为什么不用B树 二叉树  平衡二叉树  画下B+树

Mysql 使用 B+ 是因为 B+ 数的独有特性，首先 B+ 的每个结点可以有多个子节点，且每个非叶子节点和非根结点的子结点数都相同，数据都存在叶子结点，其他结点不存储数据，按照每个 mysql 页大小为 16KB 的话，每个结点使用一个页大小，这样能够充分降低树的高度，也使得查询的时候减少 IO 次数。

3.数据库事务原理  索引原理

为了保证操作数据的原子性、一致性、隔离性、持久性，简称（ACID）

为了实现这些特性，mysql 定制了4 种隔离级别，分别为：读未提交、读已提交、可重复读(mysql 默认的级别)、序列化

在可重复读的级别下, 还会产生幻读的情况，mysql 增加了多版本控制（mvcc），当前读，解决幻读的情况。

Undo log : 是为了回滚事务创建的，记录了每行数据的多个版本，回滚的时候执行，undo log 头部有两个指针，用于存放前一条 undo log 的结束位置，也是本条 undo log  的起始位置，并存放全局唯一的事务id

MVCC：实现是通过 undo log  来实现的，在事务开启时将当前所有进行中的事务全局事务 id 进行维护，在进行操作的时候如果当前数据的版本 事务id 大于当前事务的ID 此时的数据是在当前事务之后提交的，此数据对当前的事务不可见

如果当前数据版本的事务 ID 在维护的 事务 ID 列表时，此时数据对当前事务也是不可见，这两种情况需要查询 undo log 找到当前事务能看到的版本，也就是小于当前事务ID

且不在维护的 ID 列表中的数据版本

innodb默认隔离级别是RR， 是通过MVVC来实现了，读方式有两种，执行select的时候是快照读，其余是当前读，所以，mvvc不能根本上解决幻读的情况

4.为什么limit的起始值越大越慢

5.MySQL  索引优化  什么情况下不会走索引

Innodb 索引执行最左匹配规则，如果没有命中最左的列则无法命中索引，数据类型不一致也会导致无法使用索引，函数计算类的方法也会导致索引失效

6.可重复读是  怎么可以来保证不出现幻读

可重复读是在一个事务中多次读不回读到别的线程插入的数据，innodb 引入了mvcc 快照读和当前读 解决了幻读的问题

7.mybatis的流程

8.MySQL用过哪些索引

从数据结构：

B+树索引：

Hash 索引：

fullText 索引：

R-tree 索引：

从物理存储结构看：

聚簇索引、非聚簇索引

从逻辑角度：

主键索引、普通索引、单列索引、唯一索引、复合索引

9.mybatis N+1问题

N+1问题： 一个老师下面有多个学生的问题，如果我们要查询老师下的学生信息，需要先查询老师，然后遍历老师列表查询老师下的学生，这就产生了 N+1 问题。大量的遍历查询。

解决办法：

1.使用连表查询，一次查询出想要的信息

2.开启mybatis 的 enableLazyLoad 开关，但使用到老师的学员信息时再进行查询，如果不使用，先不查询。

10.mybatis 中collection和association的使用和区别

11.mybatis和ibatis区别

ibates 是Mybatis 的前身，如今 ibatis 已经更名为 Mybatis 

区别：

1.ibatis 使用时需要我们在 dao 的实现类中显示的指定用到的 xml 而 mybatis 则自动帮我们实现了 xml 文件和接口的映射

\2. 

12.mybatis缓存机制

Mybatis 有三级缓存机制，在单个sqlsession 中 同一个 sql 同样的参数会命中一级缓存。

二级缓存需要手动开启，将 cacheEnable 设置为 true。在 sqlFactory 层级同样的 sql 相同的参数会被缓存

13.MySQL使用B+树怎么保证数据分布均匀

14.MySQL的同步方式  半同步  同步 并行

为了保证 mysql 性能，业务上一般使用主从分离。主从之间的同步问题也是个大问题。

异步：mysql 5.5 之前的版本支持异步同步，从库和主库建立 io 链接将bing log 复制到从库的 ready log ，从库执行 ready log 实现异步但线程的同步。缺点是会有一定的延时

半同步：mysql 5.5 的版本实现了半同步和基于 schema 的并行同步，半同步是当主库提交后，事务提交的线程将等待至少一个半同步的从库刷盘并返回确认信息后再对客户端返回响应。

15.第一，第二，第三范式的理解和比较

所谓的范式，可以理解为设计数据库、表、的一些规范准则，在数据库概论中

1N范式：表中的每个属性，不可再分

2N范式：要求要有主键、其它数据都依赖主键，能唯一定位到一条数据

3N范式：要求减少冗余，减少同样的信息在多张表中都存在

16.


# 二、spring



## 2.1 springboot

2.1.1 springboot 如何启动的

SpringFactoriesLoader会加载classpath下所有JAR文件里面的META-INF/spring.factories文件。其中加载spring.factories文件的代码在loadFactoryNames方法里

Main 方法启动：

如果我们使用的是SpringApplication的静态run方法，那么，这个方法里面首先要创建一个SpringApplication对象实例，然后调用这个创建好的SpringApplication的实例方法。在SpringApplication实例初始化的时候，它会提前做几件事情：

•根据classpath里面是否存在某个特征类（org.springframework.web.context.ConfigurableWebApplicationContext）来决定是否应该创建一个为Web应用使用的ApplicationContext类型。

•使用SpringFactoriesLoader在应用的classpath中查找并加载所有可用的ApplicationContextInitializer。

•使用SpringFactoriesLoader在应用的classpath中查找并加载所有可用的ApplicationListener。

•推断并设置main方法的定义类。



2.2 





# 三、 其他组件

## 3.1 redis

3.1.1 存储老师的消息用的是list

string、list、hash、set和zset。

3.1.2 用 redis 实现购物车



## 3.2 websocket

websocket 是多对象的，每个用户的聊天客户端对应 java 后台的一个 websocket 对象，前后台一对一（多对多）实时连接，所以 websocket 不可能像 servlet 一样做成单例的，让所有聊天用户连接到一个 websocket对象，这样无法保存所有用户的实时连接信息。可能 spring 开发者考虑到这个问题，没有让 spring 创建管理 websocket ，而是由 java 原来的机制管理websocket ，所以用户聊天时创建的 websocket 连接对象不是 spring 创建的，spring 也不会为不是他创建的对象进行依赖注入，所以如果不用static关键字，每个 websocket 对象的 service 都是 null。



## 3.3 Es



## 3.4 多线程

3.4.1 countDownLatch 简单说下

java 1.5中，提供了一些非常有用的辅助类来帮助我们进行并发编程，比如CountDownLatch，CyclicBarrier和Semaphore，不客气地说，创建 java.util.concurrent 的目的就是要实现 Collection 框架对数据结构所执行的并发操作。通过提供一组可靠的、高性能并发构建块，开发人员可以提高并发类的线程安全、可伸缩性、性能、可读性和可靠性。

CountDownLatch类位于java.util.concurrent包下，利用它可以实现类似计数器的功能。比如有一个任务A，它要等待其他4个任务执行完毕之后才能执行，此时就可以利用CountDownLatch来实现这种功能了。



# 四、java

## 4.1 java 调优

- jps：JVM Process Status Tool,显示指定系统内所有的HotSpot虚拟机进程。
- jstat：jstat(JVM statistics Monitoring)是用于监视虚拟机运行时状态信息的命令，它可以显示出虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据。


- jmap：jmap(JVM Memory Map)命令用于生成heap dump文件，如果不使用这个命令，还阔以使用-XX:+HeapDumpOnOutOfMemoryError参数来让虚拟机出现OOM的时候·自动生成dump文件。
- jhat：jhat(JVM Heap Analysis Tool)命令是与jmap搭配使用，用来分析jmap生成的dump，jhat内置了一个微型的HTTP/HTML服务器，生成dump的分析结果后，可以在浏览器中查看。在此要注意，一般不会直接在服务器上进行分析，因为jhat是一个耗时并且耗费硬件资源的过程，一般把服务器生成的dump文件复制到本地或其他机器上进行分析。
- jstack：jstack用于生成java虚拟机当前时刻的线程快照。线程快照是当前java虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等。 线程出现停顿的时候通过jstack来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做什么事情，或者等待什么资源。 如果java程序崩溃生成core文件，jstack工具可以用来获得core文件的java stack和native stack的信息，从而可以轻松地知道java程序是如何崩溃和在程序何处发生问题。另外，jstack工具还可以附属到正在运行的java程序中，看到当时运行的java程序的java stack和native stack的信息, 如果现在运行的java程序呈现hung的状态，jstack是非常有用的。
- jinfo：jinfo(JVM Configuration info)这个命令作用是实时查看和调整虚拟机运行参数。
之前的jps -v口令只能查看到显示指定的参数，如果想要查看未被显示指定的参数的值就要使用jinfo口令













Java问题

1.

**2.hashmap put的流程**

A:hashMap 底层是数组加连表实现的，数组用来存储新put进来的值，如果出现hash冲突，则对比key值是否相同。如果相同则覆盖，如果不同则将原来数组中的值放到单向连表中，新put进来的值放到数组中。JDK1.8以后引入红黑树，防止hash集中造成的桶长度过长成为单向连表。

1. 判断数组是否为空，如果是空，则创建默认长度位 16 的数组。
2. 通过与运算计算对应 hash 值的下标，如果对应下标的位置没有元素，则直接创建一个。
3. 如果有元素，说明 hash 冲突了，则再次进行 3 种判断。
4. 1. 判断两个冲突的key是否相等，equals 方法的价值在这里体现了。如果相等，则将已经存在的值赋给变量e。最后更新e的value，也就是替换操作。
   2. 如果key不相等，则判断是否是红黑树类型，如果是红黑树，则交给红黑树追加此元素。
   3. 如果key既不相等，也不是红黑树，则是链表，那么就遍历链表中的每一个key和给定的key是否相等。如果，链表的长度大于等于8了，则将链表改为红黑树，这是Java8 的一个新的优化。
5. 最后，如果这三个判断返回的 e 不为null，则说明key重复，则更新key对应的value的值。
6. 对维护着迭代器的modCount 变量加一。
7. 最后判断，如果当前数组的长度已经大于阀值了。则重新hash。

**3.Java内存模型**

Java 为了保证程序在不同的平台，不同的操作系统中都能得到同样的效果，屏蔽了不同的操作系统、不同的硬件对内存的操作差异。

Java 内存模型 因为堆上的对象是多线程共享的，且 cpu 执行线程时需要对堆上的对象进行一级、二级、多级缓存，这样多个线程如果同时修改堆上的对象，会造成数据不一致的问题，为了解决这些问题，Java 定义了 java 内存模型，例如volit 关键字，修饰的对象，在对对象读取之前，和对象写入之后加上内存屏障，强行刷新主存中的值。这也是 hapen-before 规则，解决了多级缓存，指令重排，处理器优化，等。这些规范组成了Java 内存模型。

**4.JVM中对象都是在堆上生成的吗**

不一定，在开启逃逸分析后，Java 运行期间会进行 JIT 运行时编译，通过 profiling、逃逸分析等方法，确定方法中的局部变量时候会逃逸，如果确定不会逃逸，对象的创建会在栈上创建，好处是随着方法的调用执行完成，对象也会被销毁，减少 GC 的压力。

**5.Java类加载步骤**

Java 类加载器有启动类加载器，主要加载 jre/lib 下的一些 Java 类库。扩展类加载器主要负责加载 jre/lib/ext 目录下的 Java 类库。应用类加载器主要负责加载应用 classpath 下的 Java 文件。

Java 会通过加载阶段，加载不同类型的 Java 文件，如 jar 包，class 文件，网络传送的 java 文件等，然后通过链接阶段，对 java 文件进行验证、准备、解析。最后对Java 文件初始化。

6.JVM划分 每个模块的作用

Jvm 内存划分为 堆，主要存放新生成的对象。方法区，方法区里主要存放一些元数据，以及类的结构信息，和方法，还有运行时常量池。方法区和堆事线程共享的。

除了方法区和堆，Java 虚拟机会给每个线程分配空间，主要存放程序计数器，Java栈，本地方法栈。

**7.Stringbuffer怎么实现线程安全**

String builder 不是线程安全的，stringbuffer是线程安全的，String Buffe 底层是通过对 append()，charat()方法加上 synchronized 实现同步

**8.synchronized 使用场景,实现原理  方法前加static 和不加static的区别**

Synchronized 可以作用于方法，代码块，类。当作用于类，和静态方法，静态代码块时，对改类的所有对象起效果，其他情况对类的当前对象起效果。

Java 6 以前 synchronized 实现是通过 Javac 编译时对方法前后生成 monitorenter 在方法后生成 moniterexit 指令来控制，当有线程进入方法时会先进入 moniter 对象的 enterList 当enterList 里的线程获取到锁时，会修改 moniter 对象的 count++ 并修改 owner 为当前线程。moniter 是使用操作系统的 mutex lock 实现的，线程需要从用户态切换到核心态，开销比较大

Java 6 以后 对 synchronized 进行了优化，加入了自适应自旋锁，锁消除，锁粗化，轻量级锁，偏向锁

由于 Synchronized 为重量级锁，开销太大。Java 6 以后对 synchronized 进行了升级，当没有线程竞争的时候使用偏斜锁，使用 CAS 在对象头中 mark word 写入当前的线程id, 如果发现有其他线程竞争，当前锁会升级为轻量级锁，此时会尝试通过 CAS 对 mark word 进行修改，并进入自旋状态，默认自旋10次，如果修改成功则获得锁，如果修改不成功，则升级为重量级锁。

9.介绍下反射

反射就是在 java 运行时动态的创建对象，以及操作对象的属性，方法。

反射在开发中经常使用，我们经常使用的注解就是通过方式的方式实现注入的

10.简单说下泛型

泛型是 Java 1.5 之后提供的类型，本质就是参数化类型，也就是类型也可以像参数一样传递。泛型可以用在，类、接口、方法的创建上。

在以前不支持泛型的时候，我们把不确定的类型用 Object 来接，这样在代码运行时做强制类型的转换，可能会报出 ClassCastException 异常。泛型的好处就是把类型的检察放在了编译期，而且所有的代码检查都是隐式和自动的。提高了代码的重复利用率。

11.Java 7 和 Java 8 的区别（Java 各个版本的新特性）

Java 5: 新增枚举、泛型、自动拆装箱、可变参数、注解、增强 for 循环

Java 6: 开发代号为Mustang(野马),于2006-12-11发行。评价：鸡肋的版本，有JDBC4.0更新、Complier API、WebSevice支持的加强等更新。

Java 7: switch 支持 String 类型、try-with-resourse 、fork-join 框架 

Java 8: lambda 函数式编程、接口 default 默认方法、引用方法、stream

Java 9: 集合工厂方法、增强的 stream Api

Java 10: G1 垃圾收集器、集合类的增强API

Java 11: ZGC、安全基础设施的升级

12.hashmap的结构 1.7 和 1.8的区别

1.7 中 hashMap 出现冲突时是在冲突位置变成链表，18 中如果链表长度超过 8 链表会树化（红黑树）

同样 HashMap 扩容的时候，也有不同

1.7 中 hashMap 扩容时先扩容，然后在插入新的值，1.8 中是先插入值，然后再进行扩容，扩容完成后统一重新计算位置

1.7 中 hashMap 计算扩容后的位置是每个元素重新走一遍 put() 流程，也就是 hash() >> 扰动处理 >> (h&length-1)，尔 1.8 中是通过标志位是否为 1 来判断在原位还是原位+扩容量，降低了扩容的计算量。

1.7 中 hashMap 扩容以后链表是头插发，而 1.8 以后是通过尾插法，这样避免了扩容导致的逆序

13.sleep()和wait()的区别

Sleep() 会放弃 cpu 资源，但是不会放弃锁资源。wait() 会放弃锁资源，和 cpu 资源 

14.线程池的使用  有哪些比较重要的参数

可以在项目中配置线程池，也可以新疆一个线程池。重要的参数有，核心线程数，最大线程数，线程存活保存时间

Java 1.5 以后引入 Java.current 包，使用 ThreadPoolExecuter 创建线程池，并使用 ArrayBlockingQueue 作任务队列 

15.ArrayList和LinkedList的区别

ArrayList 底层采用数组方式实现，LinkedList 底层采用双向链表实现，所以 arrayList 查询插比较快，LinkedList 插入、删除比较快

16.最小堆是什么结构  什么是平衡树

每一个结点都大于或这等于子节点的完全二叉树为最大堆，

每一个结点都小于或这等于子节点的完全二叉树为最小堆，

完全二叉树：除了最后一层，最后一层往上的层结点数都达到最大数

平衡树：在完全二叉树的基础上，如果结点的做孩子不为空，则左孩子小于当前结点，如果右孩子不为空，右孩子大于当前结点

红黑树：平衡的不完全二叉树，如果结点的做孩子不为空，则左孩子小于当前结点，如果右孩子不为空，右孩子大于当前结点，但是左右层高不一定一样，且高度差可能大于1，为了查询，插入，删除效率，比平衡数效率要高

17.synchronized和lock的区别

Lock 是 手动获取锁，lock 需要自己释放 synchronized 不需要

实现 lock 的典型锁有 ReentrantLock 再入锁 ReentrantLock 适合高并发锁竞争激烈的场合，因为它提供了锁的一些操作，例如锁请求超时返回，以设置及公平、非公平锁，锁绑定多个条件

18 Java 代理

19 线程同步

线程同步我们可以分为资源的同步，和线程间的同步

资源的同步可以用锁解决，synchronized、lock、reentranLock、使用 java.util.concurrent.atomic 包下的 atomicInteger\ long\ array boolean 等线程同步的包装类，还有一些线程同步的 arrayblockqunue \ linkedblockQunue

线程间同步可以用使用

 countdown latch : 创建的多个线程执行完成，调用countdown() 使当前线程 await 状态，或者线程执行超时，视为线程全部执行完成，则会继续执行其他业务。

cyclicBarrier : 创建的线程到达同样的起始点，如果达到要求的数量，同时出发。

Semaphore : 规定固定长度的坑位，如果坑位满了，需要等有线程执行完成以后才能进入坑位执行。

20 atomicInteger、atomicLong、atomicBoolean、atomicArray、atomic

java.util.concurrent.atomic 包中提供了atomic 对 integer \ long\ boolean\ array 等数据类型的包装类，主要解决线程同步问题，这些数据类型是可以支持高并发下的操作。

底层实现原理是 使用了 Java 的 Unsafe 类中的 nativ 方法

21 常用的线程池

NewCachedThreadPool: 主要用来处理大量的段时间工作任务的线程池，并且该线程池尝试对使用过的线程进行缓存，60 秒后如果不进行复用便会销毁。

NewFixedThreadPool: 创建一个固定大小的线程池，线程池中最多有 n 个线程正在执行。如果空出一个线程，则会执行队列中的任务。

NewSingleThreadExector:  只有一个工作线程的线程池，且有个无界的队列。

NewSingleThreadScheduleExector\ NewThreadScheduleExecutor: 可以进行周期性的工作调度

NewWorkSetaLingPool: 

db问题



dubbo问题

1.dubbo工作原理

2.dubbo依赖的zk挂了的时候 怎么保证可用

3.讲下dubbo的工作流程 对dubbo的认识

4.为什么使用dubbo 不使用springcloud等其他RPC dubbo使用的什么序列化方式 用的什么协议 hessian协议有哪些缺点

5.注册中心用的什么 zookeeper如何解决脑裂

redis问题

1.redis分布式锁  目前的实现方式  多机版为什么不用集群方式

2.redis有哪几种类型 用过哪些

3.redis是线程安全的吗

4.redis哨兵模式和集群模式的区别

5.redis 动态字符串

6.redis 集群、用的什么模式

7.redis和数据库数据一致、缓存雪崩、缓存穿透

spring问题

1.spring aop 动态代理的两种方式  两种的区别

Spring 动态代理有两种方式，cglib代理、JDK代理

JDK代理：代理只能代理被接口修饰的类，而 cglib 没有这个要求，同时使用了asm 框架生成字节码，效率更高

cglib代理：没有接口的限制

Jdk 代理是动态生成的委托类的接口的实现类，而 cglib 代理是生成的代理类的子类

Spring 中如果 bean 实现了接口就会用 jdk 代理，否则使用 cglib 代理，

2.spring 事务原理

3.spring的事务级别

4.springmvc是线程安全的吗

5.是否读过spring的源码

6.springboot启动步骤

8.apring启动步骤

rabbitmq

1.rabbitmq交互流程 采用什么模式  如果保证高可用

2.rabbitmq和kafka的区别

3.most at once、last at once、

4.

其他

1.http 哪些情况会发生404

2.https握手流程

3.ftp是怎么实现上传的(excel异步导出功能)

4.简单介绍下七层模型和五层模型  http在哪一层  TCP在哪一层 超时一般发生在哪一层

5.一位广州的客人访问北京的服务中间经过什么流程

6.http1.0和http1.1的区别

8.sso认证过程

ES

1.倒排索引

2.多消费、丢消息、幂等性、接口幂等性