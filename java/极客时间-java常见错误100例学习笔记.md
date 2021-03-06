[TOC]

# 01 | 使用了并发工具类库，线程安全就高枕无忧了吗？

## 1. 使用ThreadLocal后没有及时清空导致数据错乱

spring boot/spring mvc程序运行在 Tomcat 中，执行程序的线程是 Tomcat 的工作线程，而 Tomcat 的工作线程是基于线程池的。

**使用类似 ThreadLocal 工具来存放一些数据时，需要特别注意在代码运行完后，显式地去清空设置的数据。**

## 2. ConcurrentHashMap 只能保证提供的原子性读写操作是线程安全的
ConcurrentHashMap 对外提供的方法或能力的限制：
- 使用了 ConcurrentHashMap，不代表对它的多个操作之间的状态是一致的，是没有其他线程在操作它的，如果需要确保需要手动加锁。
- 诸如 size、isEmpty 和 containsValue 等聚合方法，在并发情况下可能会反映 ConcurrentHashMap 的中间状态。因此在并发情况下，这些方法的返回值只能用作参考，而不能用于流程控制
- 诸如 putAll 这样的聚合方法也不能确保原子性，在 putAll 的过程中去获取数据可能会获取到部分数据。

## 3. 没有充分了解并发工具的特性，从而无法发挥其威力
ConcurrentHashMap的computeIfAbsent方法：可以用于使用 Map 来统计 Key 出现次数的场景
LongAdder类的使用

## 4. 没有认清并发工具的使用场景，因而导致性能问题
CopyOnWriteArrayList适用于读多的情况，因为每次add都会重新建一个List，成本很高

# 02 | 代码加锁：不要让“锁”事成为烦心事

## 1. 加锁前要清楚锁和被保护的对象是不是一个层面的
静态字段属于类，类级别的锁才能保护；而非静态字段属于类实例，实例级别的锁就可以保护。

## 2. 加锁要考虑锁的粒度和场景问题
即使我们确实有一些共享资源需要保护，也要尽可能降低锁的粒度，仅对必要的代码块甚至是需要保护的资源本身加锁。
——不要在所有业务代码的方法上加synchronized关键字，要看需求，否则会损失性能

## 3. 如果精细化考虑了锁应用范围后，性能还无法满足需求的话，我们就要考虑另一个维度的粒度问题了，即：区分读写场景以及资源的访问冲突，考虑使用悲观方式的锁还是乐观方式的锁。
作者分享的观点：
- 对于读写比例差异明显的场景，考虑使用 ReentrantReadWriteLock 细化区分读写锁，来提高性能。
- 如果你的 JDK 版本高于 1.8、共享资源的冲突概率也没那么大的话，考虑使用 StampedLock 的乐观读的特性，进一步提高性能。
- JDK 里 ReentrantLock 和 ReentrantReadWriteLock 都提供了公平锁的版本，在没有明确需求的情况下不要轻易开启公平锁特性，在任务很轻的情况下开启公平锁可能会让性能下降上百倍。

## 4. 多把锁要小心死锁问题
业务逻辑中有多把锁时要考虑死锁问题，通常的规避方案是，避免无限等待和循环等待。
**如果业务逻辑中锁的实现比较复杂的话，要仔细看看加锁和释放是否配对，是否有遗漏释放或重复释放的可能性；并且对于分布式锁要考虑锁自动超时释放了，而业务逻辑却还在进行的情况下，如果别的线线程或进程拿到了相同的锁，可能会导致重复执行。**
**如果你的业务代码涉及复杂的锁操作，强烈建议 Mock 相关外部接口或数据库操作后对应用代码进行压测，通过压测排除锁误用带来的性能问题和死锁问题。**


# 03 | 线程池：业务代码最常用也最容易犯错的组件

## 1. 线程池的声明需要手动进行

**不建议使用 Executors 提供的两种快捷的线程池（newFixedThreadPool和newCachedThreadPool）**
FixedThreadPool会创建LinkedBlockingQueue对象，队列默认长度是Integer.MAX_VALUE，虽然线程数量固定，但任务较多且执行较慢的情况下，该队列可能会导致OOM。

阿里巴巴java开发规范原文：
```
线程池不允许使用Executors去创建，而是通过ThreadPoolExecutor的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。
说明：Executors返回的线程池对象的弊端如下：
1） FixedThreadPool和SingleThreadPool： 允许的请求队列长度为Integer.MAX_VALUE，可能会堆积大量的请求，从而导致OOM。 
2） CachedThreadPool： 允许的创建线程数量为Integer.MAX_VALUE，可能会创建大量的线程，从而导致OOM。
```

- 需要根据自己的场景、并发情况来评估线程池的几个核心参数，包括核心线程数、最大线程数、线程回收策略、工作队列的类型，以及拒绝策略，确保线程池的工作行为符合需求，一般都需要设置有界的工作队列和可控的线程数。
- 任何时候，都应该为自定义线程池指定有意义的名称，以方便排查问题。
- 用一些监控手段来观察线程池的状态。

## 2. 线程池线程管理策略详解

注意一定要弄清楚线程池默认的行为模式：核心线程、最大线程、队列容量的含义
我们也可以通过一些手段来改变这些默认工作行为，比如：
- 声明线程池后立即调用 prestartAllCoreThreads 方法，来启动所有核心线程；
- 传入 true 给 allowCoreThreadTimeOut 方法，来让线程池在空闲的时候同样回收核心线程。

我们有没有办法让线程池更激进一点，优先开启更多的线程，而把队列当成一个后备方案呢？
**——可以自行实现试试。**

## 3. 务必确认清楚线程池本身是不是复用的

$\color{red}{注意使用工具库时，看看工具库的代码实现，是否真的如同我们所思考的那样。}$

## 4. 需要仔细斟酌线程池的混用策略
**要根据任务的“轻重缓急”来指定线程池的核心参数，包括线程数、回收策略和任务队列：**
- 对于执行比较慢、数量不大的 IO 任务，或许要考虑更多的线程数，而不需要太大的队列。
- 而对于吞吐量较大的计算型任务，线程数量不宜过多，可以是 CPU 核数或核数 *2

盲目复用线程池混用线程的问题在于，别人定义的线程池属性不一定适合你的任务，而且混用会相互干扰。

## 5. Java 8 的 parallel stream 功能，可以让我们很方便地并行处理集合中的元素，其背后是共享同一个 ForkJoinPool，默认并行度是 CPU 核数 -1。
$\color{red}{共享同一个 ForkJoinPool!!!}$
对于 CPU 绑定的任务来说，使用这样的配置比较合适，但如果集合操作涉及同步 IO 操作的话（比如数据库操作、外部服务调用等），建议自定义一个 ForkJoinPool（或普通线程池）。


# 04 | 连接池：别让连接池帮了倒忙
## 1. 注意鉴别客户端 SDK 是否基于连接池
**使用连接池务必确保复用**
因为 TCP 基于字节流，在多线程的情况下对同一连接进行复用，可能会产生线程安全问题。

### 使用JedisPool而不是Jedis，Jedis基于Connection，不是线程安全的


## 2. 使用连接池务必确保复用
池一定是用来复用的，否则其使用代价会比每次创建单一对象更大。对连接池来说更是如此
### apache HttpClient：CloseableHttpClient 是内部带有连接池的 API，其背后是连接池，最佳实践一定是复用。

## 3. 连接池的配置不是一成不变的
最大连接数不是设置得越大越好。
连接池最大连接数设置得太小，很可能会因为获取连接的等待时间太长，导致吞吐量低下，甚至超时无法获取连接。
对类似数据库连接池的重要资源进行持续检测，并设置一半的使用量作为报警阈值，出现预警后及时扩容。
**要强调的是，修改配置参数务必验证是否生效，并且在监控系统中确认参数是否生效、是否合理!!!**


# 加餐1 | 带你吃透课程中Java 8的那些重要知识点（上）
## Lambda 表达式
Lambda 表达式如何匹配 Java 的类型系统呢？
——函数式接口是一种只有单一抽象方法的接口，使用 @FunctionalInterface 来描述，可以隐式地转换成 Lambda 表达式。


## 使用 Java 8 简化代码
- 使用 Stream 简化集合操作
- 使用 Optional 简化判空逻辑
- JDK8 结合 Lambda 和 Stream 对各种类的增强

## 并行流
常见的5种多线程执行实现
### 1. 使用线程
调用CountDownLatch阻塞主线程
```
IntStream.rangeClosed(1, threadCount).mapToObj(i -> new Thread(() -> { //手动把taskCount分成taskCount份，每一份有一个线程执行 IntStream.rangeClosed(1, taskCount / threadCount).forEach(j -> increment(atomicInteger)); //每一个线程处理完成自己那部分数据之后，countDown一次 countDownLatch.countDown(); })).forEach(Thread::start);
```
### 2. 使用线程池
比如JDK自己提供的线程池，规定好线程池数量
```
//初始化一个线程数量=threadCount的线程池 ExecutorService executorService = Executors.newFixedThreadPool(threadCount); //所有任务直接提交到线程池处理 
IntStream.rangeClosed(1, taskCount).forEach(i -> executorService.execute(() -> increment(atomicInteger)));
```

### 3. 使用 ForkJoinPool 而不是普通线程池执行任务。
ForkJoinPool 和传统的 ThreadPoolExecutor 区别在于，前者对于 n 并行度有 n 个独立队列，后者是共享队列。如果有大量执行耗时比较短的任务，ThreadPoolExecutor 的单队列就可能会成为瓶颈。这时，使用 ForkJoinPool 性能会更好。
**ForkJoinPool 更适合大任务分割成许多小任务并行执行的场景，而 ThreadPoolExecutor 适合许多独立任务并发执行的场景。**
```
ForkJoinPool forkJoinPool = new ForkJoinPool(threadCount); //所有任务直接提交到线程池处理 
forkJoinPool.execute(() -> IntStream.rangeClosed(1, taskCount).parallel().forEach(i -> increment(atomicInteger)));
```

### 4. 直接使用并行流
并行流使用公共的 ForkJoinPool，也就是 ForkJoinPool.commonPool()。
```
System.setProperty("java.util.concurrent.ForkJoinPool.common.parallelism", String.valueOf(threadCount));
IntStream.rangeClosed(1, taskCount).parallel().forEach(i -> increment(atomicInteger));
```
### 5. 使用 CompletableFuture 来实现
使用 CompletableFuture 来实现。CompletableFuture.runAsync 方法可以指定一个线程池，一般会在使用 CompletableFuture 的时候用到：
```
CompletableFuture.runAsync(() -> IntStream.rangeClosed(1, taskCount).parallel().forEach(i -> increment(atomicInteger)), forkJoinPool).get();
```

3是完全自定义一个ForkJoinPool，4是使用公共的ForkJoinPool，只不过设置了更大的并行度，5是演示CompletableFuture可以使用自定义的ForkJoinPool。

如果你的程序对性能要求特别敏感，建议通过性能测试根据场景决定适合的模式。一般而言，使用线程池（第二种）和直接使用并行流（第四种）的方式在业务代码中比较常用。但需要注意的是，我们通常会重用线程池，而不会像 Demo 中那样在业务逻辑中直接声明新的线程池，等操作完成后再关闭。


一定是先运行 stream 方法再运行 forkjoin 方法，对公共 ForkJoinPool 默认并行度的修改才能生效。

建议：**设置 ForkJoinPool 公共线程池默认并行度的操作，应该放在应用启动时设置。**

# 加餐2 | 带你吃透课程中Java 8的那些重要知识点（下）

## Stream 操作详解
### 创建流

### filter

### map

### flatMap

### sorted

### distinct

### skip & limit

### collect

### groupBy

### partitionBy


# 05 | HTTP调用：你考虑到超时、重试、并发了吗？
## 1. 配置连接超时和读取超时参数的学问

### 连接超时参数和连接超时的误区有这么两个：
- 连接超时配置得特别长，比如 60 秒。
- 排查连接超时问题，却没理清连的是哪里。


### 读取超时参数和读取超时则会有更多的误区，此处将其归纳为如下三个
- 认为出现了读取超时，服务端的执行就会中断。
- 认为读取超时只是 Socket 网络层面的概念，是数据传输的最长耗时，故将其配置得非常短，比如 100 毫秒。
- 认为超时时间越长任务接口成功率就越高，将读取超时参数配置得太长。

## 2. Feign 和 Ribbon 配合使用，你知道怎么配置超时吗？

### 结论一，默认情况下 Feign 的读取超时是 1 秒，如此短的读取超时算是坑点一。

### 结论二，也是坑点二，如果要配置 Feign 的读取超时，就必须同时配置连接超时，才能生效。

### 结论三，单独的超时可以覆盖全局超时，这符合预期，不算坑：

### 结论四，除了可以配置 Feign，也可以配置 Ribbon 组件的参数来修改两个超时时间。这里的坑点三是，参数首字母要大写，和 Feign 的配置不同。

### 结论五，同时配置 Feign 和 Ribbon 的超时，以 Feign 为准。

## 3. 你是否知道 Ribbon 会自动重试请求呢？

## 4. 并发限制了爬虫的抓取能力

# 06 | 20%的业务代码的Spring声明式事务，可能都没处理正确

## 1. 小心 Spring 的事务可能没有生效

### @Transactional 生效原则 1：除非特殊配置（比如使用 AspectJ 静态织入实现 AOP），否则只有定义在 public 方法上的 @Transactional 才能生效。

### @Transactional 生效原则 2：必须通过代理过的类从外部调用目标方法才能生效。

**强烈建议你在开发时打开相关的 Debug 日志，以方便了解 Spring 事务实现的细节，并及时判断事务的执行情况。**

## 2. 事务即便生效也不一定能回滚

通过 AOP 实现事务处理可以理解为，使用 try…catch…来包裹标记了 @Transactional 注解的方法，当方法出现了异常并且满足一定条件的时候，在 catch 里面我们可以设置事务回滚，没有异常则直接提交事务。
“一定条件”：
- 1. 只有异常传播出了标记了 @Transactional 注解的方法，事务才能回滚
- 2. 默认情况下，出现 RuntimeException（非受检异常）或 Error 的时候，Spring 才会回滚事务。

针对情况2（默认值只能捕获RuntimeException），解决方法：
方案1：如果希望自己捕获异常进行处理
```
try {
    ...
} catch (Exception ex) {
    ...
    TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
}
```

方案2：声明事务时指定
```
@Transactional(rollbackFor = Exception.class)
public void methodXxx(String name) {

}
```

## 3. 请确认事务传播配置是否符合自己的业务逻辑
出了异常事务不一定回滚，这里说的却是不出异常，事务也不一定可以提交。
如果方法涉及多次数据库操作，并希望将它们作为独立的事务进行提交或回滚—— @Transactional 注解的 Propagation 属性。



# 07 | 数据库索引：索引并不是万能药

## 1. InnoDB 是如何存储数据的？
InnoDB 采用页而不是行的粒度来保存数据，即数据被分成若干页，以页为单位保存在磁盘中。InnoDB 的页大小，一般是 16KB。
各个数据页组成一个双向链表，每个数据页中的记录按照主键顺序组成**单向链表**

页1 --->  页2 ---> 页3 ...
    <---     <---      ...

页内部：单向链表

## 2. 聚簇索引和二级索引
聚簇索引的数据的物理存放顺序与索引顺序是一致的，即：只要索引是相邻的，那么对应的数据一定也是相邻地存放在磁盘上的。如果主键不是自增id，那么可以想象，它会干些什么，不断地调整数据的物理地址、分页，当然也有其他一些措施来减少这些操作，但却无法彻底避免。但，如果是自增的，那就简单了，它只需要一页一页地写，索引结构相对紧凑，磁盘碎片少，效率也高。


为了实现非主键字段的快速搜索，就引出了二级索引，也叫作非聚簇索引、辅助索引。二级索引，也是利用的 B+ 树的数据结构
这次二级索引的叶子节点中保存的不是实际数据，而是主键，获得主键值后去聚簇索引中获得数据行。这个过程就叫作回表。

## 3. 考虑额外创建二级索引的代价
- 维护代价
  - 页分裂和合并
  - 如何设置合理的合并阈值，来平衡页的空闲率和因为再次页分裂产生的代价，参考[14.8.12 Configuring the Merge Threshold for Index Pages](https://dev.mysql.com/doc/refman/5.7/en/index-page-merge-threshold.html)
- 空间代价
- 回表代价
  - 注意有一种情况：如果我们需要查询的是索引列索引或联合索引能覆盖的数据，那么查询索引本身已经“覆盖”了需要的数据，不再需要回表查询。——**索引覆盖**

### 关于索引的最佳实践
1. 无需一开始就建立索引，可以等到业务场景明确后，或者是数据量超过 1 万、查询变慢后，再针对需要查询、排序或分组的字段创建索引。
2. 尽量索引轻量级的字段，比如能索引 int 字段就不要索引 varchar 字段。
3. 尽量不要在 SQL 语句中 SELECT *，而是 SELECT 必要的字段，甚至可以考虑使用联合索引来包含我们要搜索的字段，既能实现索引加速，又可以避免回表的开销。

## 4. 不是所有针对索引列的查询都能用上索引
### 索引只能匹配列前缀
- 索引 B+ 树中行数据按照索引值排序，只能根据前缀进行比较。如果要按照后缀搜索也希望走索引的话，并且永远只是按照后缀搜索的话，可以把数据反过来存，用的时候再倒过来。
```
EXPLAIN SELECT * FROM person WHERE NAME LIKE 'name123%' LIMIT 100
```
可以，而'%name123'不行

### 条件涉及函数操作无法走索引
WHERE后面包含函数，有计算逻辑的不行

### 联合索引只能匹配左边的列

## 5. 数据库基于成本决定是否走索引
- MySQL 选择索引，并不是按照 WHERE 条件中列的顺序进行的；
- 即便列有索引，甚至有多个可能的索引方案，MySQL 也可能不走索引。

查看mysql对于表的统计信息：
``` 
SHOW TABLE STATUS LIKE 'person'
```

人工干预，强制走索引：
```
EXPLAIN SELECT * FROM person FORCE INDEX(name_score) WHERE NAME >'name84059' AND create_time>'2020-01-24 05:00:00' 
```

在 MySQL 5.6 及之后的版本中，我们可以使用 optimizer trace 功能查看优化器生成执行计划的整个过程。有了这个功能，我们不仅可以了解优化器的选择过程，更可以了解每一个执行环节的成本，然后依靠这些信息进一步优化查询。

在尝试通过索引进行 SQL 性能优化的时候，务必通过执行计划或实际的效果来确认索引是否能有效改善性能问题，否则增加了索引不但没解决性能问题，还增加了数据库增删改的负担。如果对 EXPLAIN 给出的执行计划有疑问的话，你还可以利用 optimizer_trace 查看详细的执行计划做进一步分析。




# 08 | 判等问题：程序里如何确定你就是你？

## 1. 注意 equals 和 == 的区别

比较值的内容，除了基本类型只能使用 == 外，其他类型都需要使用 equals。

### 例外1：Integer- 缓存
Integer 对象，默认缓存[-128, 127]的数值，可以用JVM参数控制
```
-XX:AutoBoxCacheMax=1000
```

只需要记得比较 Integer 的值请使用 equals，而不是 ==

### 例外2：String intern方法
字符串常量池机制：当代码中出现双引号形式创建字符串对象时，JVM 会先对这个字符串进行检查，如果字符串常量池中存在相同内容的字符串对象的引用，则将这个引用返回；否则，创建新的字符串对象，然后将这个引用放入字符串常量池，并返回该引用。这种机制，就是字符串驻留或池化。

虽然使用 new 声明的字符串调用 intern 方法，也可以让字符串进行驻留，但在业务代码中滥用 intern，可能会产生性能问题。

**没事别轻易用 intern，如果要用一定要注意控制驻留的字符串的数量，并留意常量表的各项指标。**


## 2. 实现一个 equals 没有这么简单

实现时注意：
- 考虑到性能，可以先进行指针判等，如果对象是同一个那么直接返回 true；
- 需要对另一方进行判空，空对象和自身进行比较，结果一定是 fasle；
- 需要判断两个对象的类型，如果类型都不同，那么直接返回 false；
- 确保类型相同的情况下再进行类型强制转换，然后逐一判断所有字段。

## 3. hashCode 和 equals 要配对实现
散列表需要使用 hashCode 来定位元素放到哪个桶。  

实现这两个方法也有简单的方式，一是后面要讲到的 Lombok 方法，二是使用 IDE 的代码生成功能。

## 4. 注意 compareTo 和 equals 的逻辑一致性
对于自定义的类型，如果要实现 Comparable，请记得 equals、hashCode、compareTo 三者逻辑一致。

## 5. 小心 Lombok 生成代码的“坑”

Lombok 的 @Data 注解会帮我们实现 equals 和 hashcode 方法，但是有继承关系时，Lombok 自动生成的方法可能就不是我们期望的了。
Lombok的@Data注解，equal和hashCode方法忽视某些字段的方法：字段上添加`@EqualsAndHashCode.Exclude`

@EqualsAndHashCode 默认实现没有使用父类属性。
使用callSuper即可覆盖默认情况，即
```
@Data
@EqualsAndHashCode(callSuper = true)
class Employee extends Person {
```


# 09 | 数值计算：注意精度、舍入和溢出问题

## 1. “危险”的 Double

float/double类型有精度丢失问题

使用 BigDecimal 表示和计算浮点数，且务必**使用字符串的构造方法来初始化 BigDecimal**。double可以使用`Double.toString()`方法
如果一定要用 Double 来初始化 BigDecimal 的话，可以使用 `BigDecimal.valueOf` 方法，以确保其表现和字符串形式的构造方法一致，这也是官方文档更推荐的方式

## 2. 考虑浮点数舍入和格式化的方式

浮点数的舍入、字符串格式化也要通过 BigDecimal 进行。

## 3. 用 equals 做判等，就一定是对的吗？
BigDecimal 的 equals 方法的注释中说明了原因，equals 比较的是 BigDecimal 的 value 和 scale

**如果我们希望只比较 BigDecimal 的 value，可以使用 compareTo 方法**

BigDecimal 的 equals 和 hashCode 方法会同时考虑 value 和 scale，如果结合 HashSet 或 HashMap 使用的话就可能会出现麻烦。比如：
```
Set<BigDecimal> hashSet1 = new HashSet<>();
hashSet1.add(new BigDecimal("1.0"));
System.out.println(hashSet1.contains(new BigDecimal("1")));//返回false
```

解决方案：
- 方案1：使用 TreeSet 替换 HashSet。TreeSet 不使用 hashCode 方法，也不使用 equals 比较元素，而是使用 compareTo 方法，所以不会有问题。
- 方案2：把 BigDecimal 存入 HashSet 或 HashMap 前，先使用 stripTrailingZeros 方法去掉尾部的零，比较的时候也去掉尾部的 0，确保 value 相同的 BigDecimal，scale 也是一致的

## 4. 小心数值溢出问题
不管是 int 还是 long，所有的基本数值类型都有超出表达范围的可能性。
基本数据类型的加减乘除计算，溢出时不会抛出异常。
解决：
- 方案1：考虑使用 Math 类的 addExact、subtractExact 等 xxExact 方法进行数值运算，这些方法可以在数值溢出时主动抛出异常。
- 方案2：使用大数类 BigInteger。


# 10 | 集合类：坑满地的List列表操作
## 1. 使用 Arrays.asList 把数据转换为 List 的三个坑
### 1.1 不能直接使用 Arrays.asList 来转换基本类型数组
```
int[] arr1 = {1, 2, 3};
List list1 = Arrays.stream(arr1).boxed().collect(Collectors.toList());
log.info("list:{} size:{} class:{}", list1, list1.size(), list1.get(0).getClass());

Integer[] arr2 = {1, 2, 3};
List list2 = Arrays.asList(arr2);
log.info("list:{} size:{} class:{}", list2, list2.size(), list2.get(0).getClass());
``` 
若arr1直接传入Arrays.asList，将会丢失数据，原因：Arrays.asList泛型只能把 int 装箱为 Integer，不可能把 int 数组装箱为 Integer 数组。

### 1.2 Arrays.asList 返回的 List 不支持增删操作
Arrays.asList 返回的 List 是Arrays类内部的ArrayList，其增删方法直接抛出异常：`UnsupportedOperationException`

### 1.3 对原始数组的修改会影响到我们获得的那个 List。
**这个很重要！！！**
解决：重新new一个ArrayList

## 2. 使用 List.subList 进行切片操作居然会导致 OOM？

subList 方法可以看到获得的 List 其实是内部类 SubList，并不是普通的 ArrayList，在初始化的时候传入了 this。
SubList 初始化的时候，并没有把原始 List 中的元素复制到独立的变量中保存。我们可以认为 SubList 是原始 List 的视图，并不是独立的 List。双方对元素的修改会相互影响，而且 SubList 强引用了原始的 List，所以**大量保存这样的 SubList 会导致 OOM。**

解决：
- 1、不直接使用subList方法返回的SubList，而是new ArrayList，构建独立的List
- 2、对于 Java 8 使用 Stream 的 skip 和 limit API 来跳过流中的元素，以及限制流中元素的个数，达到subList的效果
```
List subList = list.stream().skip(1).limit(3).collect(Collectors.toList());
```

## 3. 一定要让合适的数据结构做合适的事情

### 误区1：使用数据结构不考虑平衡时间和空间
要对大 List 进行单值搜索的话，可以考虑使用 HashMap，其中 Key 是要搜索的值，Value 是原始对象，会比使用 ArrayList 有非常明显的性能优势

在应用内存吃紧的情况下，我们需要考虑是否值得使用更多的内存消耗来换取更高的性能。这里我们看到的是平衡的艺术，空间换时间，还是时间换空间，只考虑任何一个方面都是不对的。

如果业务代码中有频繁的大 ArrayList 搜索，使用 HashMap 性能会好很多。类似，如果要对大 ArrayList 进行去重操作，也不建议使用 contains 方法，而是可以考虑使用 HashSet 进行去重。

平衡的艺术，空间换时间，还是时间换空间，只考虑任何一个方面都是不对的



# 11 | 空值处理：分不清楚的null和恼人的空指针

## 1. 修复和定位恼人的空指针问题

NullPointerException 最可能出现的场景归为以下 5 种：
- 参数值是 Integer 等包装类型，使用时因为自动拆箱出现了空指针异常；
- 字符串比较出现空指针异常；
- 诸如 ConcurrentHashMap 这样的容器不支持 Key 和 Value 为 null，强行 put null 的 Key 或 Value 会出现空指针异常；
- A 对象包含了 B，在通过 A 对象的字段获得 B 之后，没有对字段判空就级联调用 B 的方法出现空指针异常；
- 方法或远程服务返回的 List 不是空而是 null，没有进行判空就直接调用 List 的方法出现空指针异常。

**阿里开源的 Java 故障诊断 Arthas**

修复思路如下：
- 对于 Integer 的判空，可以使用 Optional.ofNullable 来构造一个 Optional，然后使用 orElse(0)
- 对于 String 和字面量的比较，可以把字面量放在前面
- 对于 ConcurrentHashMap，既然其 Key 和 Value 都不支持 null，修复方式就是不要把 null 存进去。
- 对于类似 fooService.getBarService().bar().equals(“OK”) 的级联调用，需要判空的地方有很多， 可以使用**Optional**类简化
  - 改为如下：
   ```
            Optional.ofNullable(fooService)
                .map(FooService::getBarService)
                .filter(barService -> "OK".equals(barService.bar()))
                .ifPresent(result -> log.info("OK"));
   ```
- 对于 rightMethod 返回的 List，由于不能确认其是否为 null，所以在调用 size 方法获得列表大小之前，同样可以使用 Optional.ofNullable 包装一下返回值，然后通过.orElse(Collections.emptyList()) 实现在 List 为 null 的时候获得一个空的 List

**使用判空方式或 Optional 方式来避免出现空指针异常，不一定是解决问题的最好方式，空指针没出现可能隐藏了更深的 Bug。**

## 2. POJO 中属性的 null 到底代表了什么？

注意字符串格式化时可能会把 null 值格式化为 null 字符串。
数据库字段允许保存 null，会进一步增加出错的可能性和复杂度。

尽量不要：使用一个POJO同时扮演 DTO 和数据库 Entity
可以巧妙使用 Optional 来区分客户端不传值和传 null 值
```
@Datapublic class UserDto { 
  private Long id; 
  private Optional name; 
  private Optional age;
}
```

## 3. 小心 MySQL 中有关 NULL 的三个坑
- MySQL 中 sum 函数没统计到任何记录时，会返回 null 而不是 0，可以使用 IFNULL 函数把 null 转换为 0；
- MySQL 中 count 字段不统计 null 值，COUNT(*) 才是统计所有记录数量的正确方式。
- MySQL 中 =NULL 并不是判断条件而是赋值，对 NULL 进行判断只能使用 IS NULL 或者 IS NOT NULL。


## 4. 总结
业务系统最基本的标准是不能出现未处理的空指针异常，因为它往往代表了业务逻辑的中断，所以建议每天查询一次生产日志来排查空指针异常，有条件的话建议订阅空指针异常报警，以便及时发现及时处理。


# 12 | 异常处理：别让自己在出问题的时候变为瞎子

## 1. 捕获和处理异常容易犯的错
- 错误1：不在业务代码层面考虑异常处理，仅在框架层面粗犷捕获和处理异常。
  - 解决：
    - 如果异常上升到最上层逻辑还是无法处理的话，可以以统一的方式进行异常转换，比如通过 @RestControllerAdvice + @ExceptionHandler，来捕获这些“未处理”异常：
      - 对于自定义的业务异常：记录warn级别日志，返回合适的API包装体
      - 对于无法处理的系统异常：记录Error 级别日志，转换为普适的“服务器忙，请稍后再试”异常信息，同样以 API 包装体返回给调用方。
- 错误2：捕获了异常后直接生吞
- 错误3：丢弃异常的原始信息
- 错误4：抛出异常时不指定任何消息


**如果你捕获了异常打算处理的话，除了通过日志正确记录异常原始信息外，通常还有三种处理模式：**
- 转换，即转换新的异常抛出。
- 重试，即重试之前的操作。
- 恢复，即尝试进行降级处理，或使用默认值来替代原始数据。

## 2. 小心 finally 中的异常
修复方法：
- 1. finally 代码块自己负责异常捕获和处理
- 2. 可以把 try 中的异常作为主异常抛出，使用**addSuppressed**方法把 finally 中的异常附加到主异常上
  - 使用实现了 AutoCloseable 接口的资源，务必使用 try-with-resources 模式来使用资源，确保资源可以正确释放，也同时确保异常可以正确处理。

## 2. 千万别把异常定义为静态变量
**把异常定义为静态变量会导致异常信息固化**，这就和异常的栈一定是需要根据当前调用来动态获取相矛盾。
解决：改一下 Exceptions 类的实现，通过不同的方法把每一种异常都 new 出来抛出即可

## 3. 提交线程池的任务出了异常会怎么样？

解决：
- 1. 以 execute 方法提交到线程池的异步任务，最好在任务内部做好异常处理；
- 2. 设置自定义的异常处理程序作为保底，比如:
  - 2.1 在声明线程池时自定义线程池的未捕获异常处理程序
    ```
    new ThreadFactoryBuilder()
    .setNameFormat(prefix+"%d")
    .setUncaughtExceptionHandler((thread, throwable)-> log.error("ThreadPool {} got exception", thread, throwable))
    .get()
    ```
  - 2.2 或者设置全局的默认未捕获异常处理程序
    ```
    static {
        Thread.setDefaultUncaughtExceptionHandler((thread, throwable)-> log.error("Thread {} got exception", thread, throwable));
    }
    ```


线程池 ExecutorService 的 execute 方法提交任务到线程池处理，如果出现异常会导致线程退出，控制台输出中可以看到异常信息；
把 execute 方法改为 submit，线程还会退出，异常信息会被生吞，只有在调用 get 方法获取 FutureTask 结果的时候，才会以 ExecutionException 的形式重新抛出异常。
解决：把 submit 返回的 Future 放到了 List 中，随后遍历 List 来捕获所有任务的异常。这么做确实合乎情理。既然是以 submit 方式来提交任务，那么我们应该关心任务的执行结果，否则应该以 execute 来提交任务

# 13 | 日志：日志记录真没你想象的那么简单
常见容易出错的地方：
- 日志框架多，不同类库使用不同日志框架，如何兼容
- 配置复杂，且容易出错
  - 重复记录日志的问题、同步日志的性能问题、异步记录的错误配置问题
- 日志记录本身就有些误区，比如没考虑到日志内容获取的代价、胡乱使用日志级别等

### slf4j
- 统一的日志门面API
- 桥接功能
- 适配功能

**可以使用 log4j-over-slf4j 来实现 Log4j 桥接到 SLF4J，也可以使用 slf4j-log4j12 实现 SLF4J 适配到 Log4j，但是它不能同时使用它们，否则就会产生死循环。jcl 和 jul 也是同样的道理。**

业务系统使用最广泛的是 Logback 和 Log4j，同一人开发的。Logback可以认为是 Log4j 的改进版本，更推荐使用。

如果程序启动时出现 SLF4J 的错误提示，那很可能是配置出现了问题，可以使用 Maven 的 dependency:tree 命令梳理依赖关系。

## 1. 为什么我的日志会重复记录？
案例1：logger 配置继承关系导致日志重复记录。
案例2：错误配置 LevelFilter 造成日志重复记录。


## 2. 使用异步日志改善性能的坑
小技巧：EvaluatorFilter（求值过滤器），用于判断日志是否符合某个条件。配合使用标记和 EvaluatorFilter，实现日志的按标签过滤，是一个不错的小技巧。

FileAppender 继承自 OutputStreamAppender，**在追加日志的时候，是直接把日志写入 OutputStream 中，属于同步记录日志。**

**使用 Logback 提供的 AsyncAppender 即可实现异步的日志记录。**

### 关于 AsyncAppender 异步日志的坑，这些坑可以归结为三类：
- 记录异步日志撑爆内存；
- 记录异步日志出现日志丢失；
- 记录异步日志出现阻塞。

## 3. 使用日志占位符就不需要进行日志级别判断了？
使用{}占位符语法不能通过延迟参数值获取，来解决日志数据获取的性能问题

SLF4J 的 API 还不支持 lambda，因此需要使用 Log4j2 日志 API，把 Lombok 的 @Slf4j 注解替换为 @Log4j2 注解，这样就可以提供一个 lambda 表达式作为提供参数数据的方法。


# 加餐3 | 定位应用问题，排错套路很重要

## 1. 在不同环境排查问题，有不同的方式
- 本地开发环境：任何工具
- 测试环境：可以采用jvirtualvm, arthas等工具复现问题，排查原因
- 生产环境：权限管控严格，无法附加进程

### 生产问题的排查很大程度依赖监控
- 1. 日志
  - 确保错误、异常信息可以被完整地记录到文件日志中
  - 确保生产上程序的日志级别是 INFO 以上
    - 记录日志要使用合理的日志优先级，DEBUG 用于开发调试、INFO 用于重要流程信息、WARN 用于需要关注的问题、ERROR 用于阻断流程的错误。
- 2. 监控
  - 开发和运维团队做好充足的监控，而且是多个层次的监控。
    - 主机层面
    - 网络层面
    - 所有的中间件和存储都要做好监控
      - 如Prometheus
    - 应用层面
      - 如Micrometer
- 3. 快照
  - 应用进程在某一时刻的快照
  - 通常情况下，我们会为生产环境的 Java 应用设置 -XX:+HeapDumpOnOutOfMemoryError 和 -XX:HeapDumpPath=…这 2 个 JVM 参数，用于在出现 OOM 时保留堆快照。

## 2. 分析定位问题的套路
首先要定位问题出在哪个层次上：java应用自身？还是外部？
- 异常信息分析
- 资源消耗型的问题：指标监控配合显性问题点

程序问题主要来自3方面：
- 1. 程序发布后的 Bug，回滚后可以立即解决。
- 2. 外部因素。这类问题的排查方式，按照主机层面的问题、中间件或存储（统称组件）的问题分为两类。
  - 主机层面的问题，可以使用工具排查：
    - CPU 相关问题，可以使用 top、vmstat、pidstat、ps 等工具排查；
    - 内存相关问题，可以使用 free、top、ps、vmstat、cachestat、sar 等工具排查；
    - IO 相关问题，可以使用 lsof、iostat、pidstat、sar、iotop、df、du 等工具排查；
    - 网络相关问题，可以使用 ifconfig、ip、nslookup、dig、ping、tcpdump、iptables 等工具排查。
  - 组件的问题，可以从以下几个方面排查：
    - 排查组件所在主机是否有问题；
    - 排查组件进程基本情况，观察各种监控指标；
    - 查看组件的日志输出，特别是错误日志；
    - 进入组件控制台，使用一些命令查看其运作情况。
- 3. 因为系统资源不够造成系统假死的问题，通常需要先通过重启和扩容解决问题，之后再进行分析，不过最好能留一个节点作为现场。
  - 一般体现在 CPU 使用高、内存泄漏或 OOM 的问题、IO 问题、网络相关问题这四个方面
    - 3.1 CPU使用高，分析流程：
      - 在 Linux 服务器上运行 top -Hp pid 命令，来查看进程中哪个线程 CPU 使用高；
      - 输入大写P，将线程按CPU使用率排序，并把明显占用 CPU 的线程 ID 转换为 16 进制；
      - 最后，在 jstack 命令输出的线程栈中搜索这个线程 ID，定位出问题的线程当时的调用栈。
        - 若没有现场，则用排查法，CPU使用高，一般是：
          - 突发压力
          - GC
          - 程序循环逻辑或是不正常的业务处理
    - 3.2 内存泄露或是OOM
      - 堆转储后使用 MAT 分析
        - 注意：ava 进程对内存的使用不仅仅是堆区，还包括线程使用的内存（线程个数 * 每一个线程的线程栈）和元数据区。
    - 3.3 IO 相关的问题
      - 除非是代码问题引起的资源不释放等问题，否则通常都不是由 Java 进程内部因素引发的。
    - 3.4 网络相关的问题
      - 一般也是由外部因素引起的。
      - 工具：ping, wireshark, tcpdump

## 3. 分析和定位问题需要注意的九个点
- 1. 考虑“鸡”和“蛋”的问题：考虑是结果还是原因
- 2. 考虑通过分类寻找规律
- 3. 分析问题需要根据调用拓扑来，不能想当然。
- 4. 考虑资源限制类问题。
- 5. 考虑资源相互影响。
- 6. 排查网络问题要考虑三个方面，到底是客户端问题，还是服务端问题，还是传输问题。
- 7. 快照类工具和趋势类工具需要结合使用。
  - jstat、top、各种监控曲线是趋势类工具，可以让我们观察各个指标的变化情况，定位大概的问题点；而 jstack 和分析堆快照的 MAT 是快照类工具，用于详细分析某一时刻应用程序某一个点的细节
- 8. 不要轻易怀疑监控
- 9. 如果因为监控缺失等原因无法定位到根因的话，相同问题就有再出现的风险


# 14 | 文件IO：实现高效正确的文件读写并非易事

## 1. 文件读写需要确保字符编码一致
FileReader 是以当前机器的默认字符集来读取文件的，如果希望指定字符集的话，需要直接使用 InputStreamReader 和 FileInputStream。
使用 JDK1.7 推出的 Files 类的 readAllLines 方法，可以很方便地用一行代码完成文件内容读取，但这种方式有个问题是，读取超出内存大小的大文件时会出现 OOM。
——解决方案就是 File 类的 lines 方法。


## 2. 使用 Files 类静态方法进行文件操作注意释放文件句柄
Files.lines 方法返回的是 Stream
通常会认为静态方法的调用不涉及资源释放，因为方法调用结束自然代表资源使用完成，由 API 释放资源，但对于 Files 类的一些返回 Stream 的方法并不是这样。这，是一个很容易被忽略的严重问题。

其实，在JDK 文档中有提到，**注意使用 try-with-resources 方式来配合，确保流的 close 方法可以调用释放资源。**

## 3. 注意读写文件要考虑设置缓冲区

**在进行文件 IO 处理的时候，使用合适的缓冲区可以明显提高性能。**

BufferedInputStream 和 BufferedOutputStream在内部实现了一个默认 8KB 大小的缓冲区，但在使用这两个类时建议再使用一个缓冲进行读写，不要因为它们实现了内部缓冲就进行逐字节的操作。
——在实际代码中每次需要读取的字节数很可能不是固定的，有的时候读取几个字节，有的时候读取几百字节，这个时候有一个固定大小较大的缓冲，也就是使用 BufferedInputStream 和 BufferedOutputStream 做为后备的稳定的二次缓冲，就非常有意义了。

**对于类似的文件复制操作，如果希望有更高性能，可以使用 FileChannel 的 transfreTo 方法进行流的复制。**

# 15 | 序列化：一来一回你还是原来的你吗？

## 1. 序列化和反序列化需要确保算法一致
以redis为例：
使用 RedisTemplate  StringRedisTemplate 进行序列化的算法并不一样，反序列化时也不能混用。
另外，可以自定义RedisTemplate key和value的序列化方式，达到自己想要的效果。

## 2. 注意 Jackson JSON 反序列化对额外字段的处理
通过设置 JSON 序列化工具 Jackson 的 activateDefaultTyping 方法，可以在序列化数据时写入对象类型。其实，Jackson 还有很多参数可以控制序列化和反序列化，是一个功能强大而完善的序列化工具。

Jackson 针对序列化和反序列化有大量的细节功能特性，我们可以参考 Jackson 官方文档来了解这些特性，详见SerializationFeature、DeserializationFeature和[MapperFeature](https://fasterxml.github.io/jackson-databind/javadoc/2.10/com/fasterxml/jackson/databind/MapperFeature.html)。

忽略多余字段，是我们写业务代码时最容易遇到的一个配置项。Spring Boot 在自动配置时贴心地做了全局设置。如果需要设置更多的特性，可以直接修改配置文件 spring.jackson.** 或设置 Jackson2ObjectMapperBuilderCustomizer 回调接口，来启用更多设置，无需重新定义 ObjectMapper Bean。


## 3. 反序列化时要小心类的构造方法

默认情况下，在反序列化的时候，Jackson 框架只会调用无参构造方法创建对象。

如果走自定义的构造方法创建对象，需要通过 @JsonCreator 来指定构造方法，并通过 @JsonProperty 设置构造方法中参数对应的 JSON 属性名：

## 4. 枚举作为 API 接口参数或返回值的两个大坑
- 客户端和服务端的枚举定义不一致时，会出异常。
  - 要解决这个问题，可以开启 Jackson 的 read_unknown_enum_values_using_default_value 反序列化特性，也就是在枚举值未知的时候使用默认值
- 枚举序列化反序列化实现自定义的字段非常麻烦，会涉及 Jackson 的 Bug。


# 16 | 用好Java 8的日期时间类，少踩一些“老三样”的坑

## 1. 初始化日期时间


## 2. “恼人”的时区问题

要正确处理时区，在于存进去和读出来两方面：存的时候，需要使用正确的当前时区来保存，这样 UTC 时间才会正确；读的时候，也只有正确设置本地时区，才能把 UTC 时间转换为正确的当地时间。

Java 8 推出了新的时间日期类 ZoneId、ZoneOffset、LocalDateTime、ZonedDateTime 和 DateTimeFormatter

## 3. 日期时间格式化和解析

SimpleDateFormat 著名的3个坑：
- “这明明是一个 2019 年的日期，怎么使用 SimpleDateFormat 格式化后就提前跨年了”
  - SimpleDateFormat 的各种格式化模式:JDK 的文档中有说明：小写 y 是年，而大写 Y 是 week year，也就是所在的周属于哪一年。
- 定义的 static 的 SimpleDateFormat 可能会出现线程安全问题。
  - 只能在同一个线程复用 SimpleDateFormat，比较好的解决方式是，通过 ThreadLocal 来存放 SimpleDateFormat
- 当需要解析的字符串和格式不匹配的时候，SimpleDateFormat 表现得很宽容，还是能得到结果。

对于 SimpleDateFormat 的这三个坑，我们使用 Java 8 中的 DateTimeFormatter 就可以避过去。首先，使用 DateTimeFormatterBuilder 来定义格式化字符串，不用去记忆使用大写的 Y 还是小写的 Y，大写的 M 还是小写的 m。

DateTimeFormatter 是线程安全的，可以定义为 static 使用；最后，DateTimeFormatter 的解析比较严格，需要解析的字符串和格式不匹配时，会直接报错

## 4. 日期时间的计算
对日期时间做计算操作，Java 8 日期时间 API 会比 Calendar 功能强大很多。

Java 8 中有一个专门的类 Period 定义了日期间隔，通过 Period.between 得到了两个 LocalDate 的差，返回的是两个日期差几年零几月零几天。如果希望得知两个日期之间差几天，直接调用 Period 的 getDays() 方法得到的只是最后的“零几天”，而不是算总的间隔天数。



# 17 | 别以为“自动挡”就不可能出现OOM
Java 的几大内存区域始终都有 OOM 的可能。相应地，Java 程序的常见 OOM 类型，可以分为堆内存的 OOM、栈 OOM、元空间 OOM、直接内存 OOM 等。

## 1. 太多份相同的对象导致 OOM

在进行容量评估时，我们不能认为一份数据在程序内存中也是一份。

例如：
一个后台程序需要从数据库加载大量信息用于数据导出，这些数据在数据库中占用 100M 内存，但是 1GB 的 JVM 堆却无法完成导出操作。100M 的数据加载到程序内存中，变为 Java 的数据结构就已经占用了 200M 堆内存；这些数据经过 JDBC、MyBatis 等框架其实是加载了 2 份，然后领域模型、DTO 再进行转换可能又加载了 2 次；最终，占用的内存达到了 200M*4=800M。

## 2. 使用 WeakHashMap 不等于不会 OOM
Spring 提供的ConcurrentReferenceHashMap类可以使用弱引用、软引用做缓存，Key 和 Value 同时被软引用或弱引用包装，也能解决相互引用导致的数据不能释放问题。与 WeakHashMap 相比，ConcurrentReferenceHashMap 不但性能更好，还可以确保线程安全。

出现内存泄露，其实就是我们认为没有用的对象最终会被 GC，但却没有。GC 并不会回收强引用对象，我们可能经常在程序中定义一些容器作为缓存，但如果容器中的数据无限增长，要特别小心最终会导致 OOM。使用 WeakHashMap 是解决这个问题的好办法，但值得注意的是，如果强引用的 Value 有引用 Key，也无法回收 Entry。

## 3. Tomcat 参数配置不合理导致 OOM

一定要根据实际需求来修改参数配置，可以考虑预留 2 到 5 倍的量。容量类的参数背后往往代表了资源，设置超大的参数就有可能占用不必要的资源，在并发量大的时候因为资源大量分配导致 OOM。


建议你为生产系统的程序配置 JVM 参数启用详细的 GC 日志，方便观察垃圾收集器的行为，并开启 HeapDumpOnOutOfMemoryError，以便在出现 OOM 时能自动 Dump 留下第一问题现场。
对于 JDK8，你可以这么设置：
```
XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=. -XX:+PrintGCDateStamps -XX:+PrintGCDetails -Xloggc:gc.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M
```


# 18 | 当反射、注解和泛型遇到OOP时，会有哪些坑？

## 1. 反射调用方法不是以传参决定重载
反射调用方法，是以反射获取方法时传入的方法名称和参数类型来确定调用方法的。

## 2. 泛型经过类型擦除多出桥接方法的坑

使用反射查询类方法清单时，我们要注意两点：
- getMethods 和 getDeclaredMethods 是有区别的，前者可以查询到父类方法，后者只能查询到当前类。
- 反射进行方法调用要注意过滤桥接方法。


## 3. 注解可以继承吗？
子类以及子类的方法，无法自动继承父类和父类方法上的注解。
自定义注解可以通过标记元注解 @Inherited 实现注解的继承，不过这只适用于类。如果要继承定义在接口或方法上的注解，可以使用 Spring 的工具类 AnnotatedElementUtils，并注意各种 getXXX 方法和 findXXX 方法的区别，详情查看[Spring 的文档](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/core/annotation/AnnotatedElementUtils.html)。


# 19 | Spring框架：IoC和AOP是扩展的核心

## 1. 单例的 Bean 如何注入 Prototype 的 Bean？
**在为类标记上 @Service 注解把类型交由容器管理前，首先评估一下类是否有状态，然后为 Bean 设置合适的 Scope。**


单例的 Bean 如何注入 Prototype 的 Bean 这个问题:Controller 标记了 @RestController 注解，而 @RestController 注解 =@Controller 注解 +@ResponseBody 注解，又因为 @Controller 标记了 @Component 元注解，所以 @RestController 注解其实也是一个 Spring Bean.

Bean 默认是单例的，所以单例的 Controller 注入的 Service 也是一次性创建的，即使 Service 本身标识了 prototype 的范围也没用。

修复方式是，让 Service 以代理方式注入。这样虽然 Controller 本身是单例的，但每次都能从代理获取 Service。这样一来，prototype 范围的配置才能真正生效：
```
@Scope(value = ConfigurableBeanFactory.SCOPE_PROTOTYPE, proxyMode = ScopedProxyMode.TARGET_CLASS)
```

## 
如果要为单例的 Bean 注入 Prototype 的 Bean，绝不是仅仅修改 Scope 属性。由于单例的 Bean 在容器启动时就会完成一次性初始化。最简单的解决方案是，把 Prototype 的 Bean 设置为通过代理注入，也就是设置 proxyMode 属性为 TARGET_CLASS。


## 3. AOP切面顺序
如果一组相同类型的 Bean 是有顺序的，需要明确使用 @Order 注解来设置顺序。两个不同优先级切面中 @Before、@After 和 @Around 三种增强的执行顺序:
- 入操作（Around（连接点执行前）、Before），切面优先级越高，越先执行。
- 出操作（Around（连接点执行后）、After、AfterReturning、AfterThrowing），切面优先级越低，越先执行。
- 同一切面的 Around 比 After、Before 先执行。








# 加餐4 | 分析定位Java问题，一定要用好这些工具（一）

## 1. 使用 JDK 自带工具查看 JVM 情况

### jps

使用 jps 得到 Java 进程列表

### jinfo

打印 JVM 的各种参数

### jvisualvm

### jconsole

看到各个内存区的 GC 曲线图


### jstat

如果没有条件使用图形界面（毕竟在 Linux 服务器上，我们主要使用命令行工具），又希望看到 GC 趋势的话，我们可以使用 jstat 工具。

### jstack

抓取线程栈

### jcmd

查看 NMT


# 20 | Spring框架：框架帮我们做了很多工作也带来了复杂度
## 1. Feign AOP 切不到的诡异案例

虽然 LoadBalancerFeignClient 和 ApacheHttpClient 都是 feign.Client 接口的实现，但是 HttpClientFeignLoadBalancedConfiguration 的自动配置只是把前者定义为 Bean，后者是 new 出来的、作为了 LoadBalancerFeignClient 的 delegate，不是 Bean。

Spring Boot 2.x 默认使用 CGLIB 的方式，但通过继承实现代理有个问题是，无法继承 final 的类。因为，ApacheHttpClient 类就是定义为了 final：

## 2. Spring 程序配置的优先级问题



# 21 | 代码重复：搞定代码重复的三个绝招

## 1. 利用工厂模式 + 模板方法模式，消除 if…else 和重复代码

**模板方法模式**：在父类中实现了购物车处理的流程模板，然后把需要特殊处理的地方留空白也就是留抽象方法定义，让子类去实现其中的逻辑。由于父类的逻辑不完整无法单独工作，因此需要定义为抽象类。

## 2. 利用注解 + 反射消除重复代码

许多涉及类结构性的通用处理，都可以按照这个模式来减少重复代码。反射给予了我们在不知晓类结构的时候，按照固定的逻辑处理类的成员；而注解给了我们为这些成员补充元数据的能力，使得我们利用反射实现通用逻辑的时候，可以从外部获得更多我们关心的数据。

## 3. 利用属性拷贝工具消除重复代码

对于三层架构的系统，考虑到层之间的解耦隔离以及每一层对数据的不同需求，通常每一层都会有自己的 POJO 作为数据实体。

可以使用类似 BeanUtils 这种 Mapping 工具来做 Bean 的转换，copyProperties 方法还允许我们提供需要忽略的属性

# 22 | 接口设计：系统间对话的语言，一定要统一

## 1. 接口的响应要明确表示接口的处理结果

### 为了将接口设计得更合理，我们需要考虑如下两个原则：
- 对外隐藏内部实现。
- 设计接口结构时，明确每个字段的含义，以及客户端的处理方式。

### 小技巧：使用spring ResponseBodyAdvice以及自定义注解，统一包装返回值
自定义注解用于打标签

ResponseBodyAdvice则用于数据的重新封装

## 2. 要考虑接口变迁的版本控制策略

如果做大的功能调整或重构，涉及参数定义的变化或是参数废弃，导致接口无法向前兼容，这时接口就需要有版本的概念。在考虑接口版本策略设计时，我们需要注意的是，最好一开始就明确版本策略，并考虑在整个服务端统一版本策略。
- 第一，版本策略最好一开始就考虑。
  - 比如，确定是通过 URL Path 实现，是通过 QueryString 实现，还是通过 HTTP 头实现。
  - URL Path 的方式最直观也最不容易出错；QueryString 不易携带，不太推荐作为公开 API 的版本策略；HTTP 头的方式比较没有侵入性，如果仅仅是部分接口需要进行版本控制，可以考虑这种方式。
- 版本实现方式要统一。
  - 在框架层面实现统一。如果你使用 Spring 框架的话，可以按照下面的方式自定义 RequestMappingHandlerMapping 来实现。

## 3. 接口处理方式要明确同步还是异步


# 23 | 缓存设计：缓存可以锦上添花也可以落井下石

## 1. 不要把 Redis 当作数据库

Redis 的特点是，处理请求很快，但无法保存超过内存大小的数据。

把 Redis 用作缓存，我们需要注意两点：
- 从客户端的角度来说，缓存数据的特点一定是有原始数据来源，且允许丢失
- 从 Redis 服务端的角度来说，缓存系统可以保存的数据量一定是小于原始数据的。首先，我们应该限制 Redis 对内存的使用量，也就是设置 maxmemory 参数；其次，我们应该根据数据特点，明确 Redis 应该以怎样的算法来驱逐数据。

常用的数据淘汰策略有：
- allkeys-lru
- volatile-lru
- volatile-ttl
- allkeys-lfu（Redis 4.0 以上）
- volatile-lfu（Redis 4.0 以上）


## 2. 注意缓存雪崩问题

从广义上说，产生缓存雪崩的原因有两种：
- 缓存系统本身不可用，导致大量请求直接回源到数据库
- 应用设计层面大量的 Key 在同一时间过期，导致大量的数据回源


# 24 | 业务代码写完，就意味着生产就绪了？
 
生产就绪（Production-ready）需要做哪些工作呢？
- 提供健康检测接口。
- 暴露应用内部信息。
- 建立应用指标 Metrics 监控。

## 1. 准备工作：配置 Spring Boot Actuator

## 2. 健康检测需要触达关键组件

Spring Boot Actuator 帮我们预先实现了诸如数据库、InfluxDB、Elasticsearch、Redis、RabbitMQ 等三方系统的健康检测指示器 HealthIndicator。

如果程序依赖一个很重要的三方服务，我们希望这个服务无法访问的时候，应用本身的健康状态也是 DOWN。比如三方服务有一个 user 接口，要实现这个 user 接口是否正确响应和程序整体的健康状态挂钩的话，很简单，只需定义一个 UserServiceHealthIndicator 实现 HealthIndicator 接口即可。

Spring Boot 2.3.0增强了健康检测的功能，细化了 Liveness 和 Readiness 两个端点，便于 Spring Boot 应用程序和 Kubernetes 整合。

## 3. 对外暴露应用内部重要组件的状态

可以通过 Actuator 的 InfoContributor 功能，对外暴露程序内部重要组件的状态数据。

对于查看和操作 MBean，除了使用 jconsole 之外，你可以使用 jolokia 把 JMX 转换为 HTTP 协议。

## 4. 指标 Metrics 是快速定位问题的“金钥匙”

指标是指一组和时间关联的、衡量某个维度能力的量化数值。通过收集指标并展现为曲线图、饼图等图表，可以帮助我们快速定位、分析问题。

可以使用 Micrometer 框架实现指标的收集，它也是 Spring Boot Actuator 选用的指标框架。

## 5. 总结

完整的应用监控体系一般由三个方面构成，包括日志 Logging、指标 Metrics 和追踪 Tracing。

追踪也叫做全链路追踪，比较有代表性的开源系统是SkyWalking和Pinpoint。一般而言，接入此类系统无需额外开发，使用其提供的 javaagent 来启动 Java 程序，就可以通过动态修改字节码实现各种组件的改写，以加入追踪代码（类似 AOP）。


# 25 | 异步处理好用，但非常容易用错  

大多数业务项目都是由同步处理、异步处理和定时任务处理三种模式相辅相成实现的。

## 1. 异步处理需要消息补偿闭环  

对于异步处理流程，必须考虑补偿或者说建立主备双活流程。

对于 MQ 消费程序，处理逻辑务必考虑**去重（支持幂等）**

生产上的补偿机制，需考虑：  
- 考虑配置补偿的频次、每次处理数量，以及补偿线程池大小等参数为合适的值，以满足补偿的吞吐量。  
- 考虑备线补偿数据进行适当延迟。比如，对注册时间在 30 秒之前的用户再进行补偿，以方便和主线 MQ 实时流程错开，避免冲突。  
- 诸如当前补偿到哪个用户的 offset 数据，需要落地数据库。  
- 补偿 Job 本身需要高可用，可以使用类似 XXLJob 或 ElasticJob 等任务系统。

针对消息的补偿闭环处理的最高标准是，能够达到补偿全量数据的吞吐量。

## 2. 注意消息模式是广播还是工作队列  

注意rabbitmq路由比较复杂。

## 3. 别让死信堵塞了消息队列  

解决死信无限重复进入队列最简单的方式是，在程序处理出错的时候，直接抛出 AmqpRejectAndDontRequeueException 异常，避免消息重新进入队列

要注意始终无法处理的死信消息，可能会引发堵塞 MQ 的问题。


### 一个思考题  
假定2个服务：用户注册服务、会员服务。
在用户注册会发送消息到 MQ，然后会员服务监听消息进行异步处理的场景下，有些时候发现，虽然用户服务先保存数据再发送 MQ，但会员服务收到消息后去查询数据库，却发现数据库中还没有新用户的信息。


原因：
业务代码把保存数据和发MQ消息放在了一个事务中，有概率收到消息的时候事务还没有提交完成。当时开发同学的处理方式是收MQ消息的时候sleep 1秒，或许应该是先提交事务，完成后再发MQ消息，但是这又出来一个问题MQ消息发送失败怎么办？所以后来演化为建立本地消息表来确保MQ消息可补偿，把业务处理和保存MQ消息到本地消息表操作在相同事务内处理，然后异步发送和补偿发送消息表中的消息到MQ









# 26 | 数据存储：NoSQL与RDBMS如何取长补短、相辅相成？

Redis 对单条数据的读取性能远远高于 MySQL，但不适合进行范围搜索。
InfluxDB 对于时间序列数据的聚合效率远远高于 MySQL，但因为没有主键，所以不是一个通用数据库。
ES 对关键字的全文搜索能力远远高于 MySQL，但是字段的更新效率较低，不适合保存频繁更新的数据。


# 27 | 数据源头：任何客户端的东西都不可信任

对于HTTP请求，任何客户端传过来的数据都是不能直接信任的。

## 1. 客户端的计算不可信  

服务端需要明确区分，哪些数据是需要客户端提供的，哪些数据是客户端从服务端获取后在客户端计算的。其中，前者可以信任；而后者不可信任，服务端需要重新计算，如果客户端和服务端计算结果不一致的话，可以给予友好提示。

## 2. 客户端提交的参数需要校验  


## 3. 不能信任请求头里的任何内容  

比如，想获取用户IP时，过于依赖 X-Forwarded-For 请求头来判断用户唯一性的实现方式，是有问题的。

IP 地址或者说请求头里的任何信息，包括 Cookie 中的信息、Referer，只能用作参考，不能用作重要逻辑判断的依据。而对于类似这个案例唯一性的判断需求，更好的做法是，让用户进行登录或三方授权登录（比如微信），拿到用户标识来做唯一性判断。

## 4. 用户标识不能从客户端获取  

如果你的接口直面用户（比如给客户端或 H5 页面调用），那么一定需要用户先登录才能使用。登录后用户标识保存在服务端，接口需要从服务端（比如 Session 中）获取。

Spring Web 的小技巧：如果希望每一个需要登录的方法，都从 Session 中获得当前用户标识，并进行一些后续处理的话，我们没有必要在每一个方法内都复制粘贴相同的获取用户身份的逻辑，可以定义一个自定义注解 @LoginRequired 到 userId 参数上，然后通过 HandlerMethodArgumentResolver 自动实现参数的组装

# 28 | 安全兜底：涉及钱时，必须考虑防刷、限量和防重

## 1. 开放平台资源的使用需要考虑防刷

对于短信验证码这种开放接口，程序逻辑内需要有防刷逻辑。对于短信验证码，有如下 4 种可行的方式来防刷：

- 只有固定的请求头才能发送验证码。
- 只有先到过注册页面才能发送验证码。
- 控制相同手机号的发送次数和发送频次。
- 增加前置图形验证码。

## 2. 虚拟资产并不能凭空产生无限使用  

要产生优惠券必须先向运营申请优惠券批次，批次中包含了固定张数的优惠券、申请原因等信息。 在业务需要发放优惠券的时候，先申请批次，然后再通过批次发放优惠券。

## 3. 钱的进出一定要和订单挂钩并且实现幂等  

- 任何资金操作都需要在平台侧生成业务属性的订单，可以是优惠券发放订单，可以是返现订单，也可以是借款订单，一定是先有订单再去做资金操作。
- 一定要做好防重，也就是实现幂等处理，并且幂等处理必须是全链路的。

比较大的互联网公司一般会把支付独立一个部门。支付部门可能会针对支付做聚合操作，内部会维护一个支付订单号，然后使用支付订单号和三方支付接口交互。最终虽然商品订单是一个，但支付订单是多个，相同的商品订单因为产生多个支付订单导致多次支付。


# 29 | 数据和代码：数据就是数据，代码就是代码

## 1. SQL 注入

有关SQL注入的认知误区：
- 1. 认为 SQL 注入问题只可能发生于 Http Get 请求，也就是通过 URL 传入的参数才可能产生注入点。
- 2. 认为不返回数据的接口，不可能存在注入问题。
- 3. 认为 SQL 注入的影响范围，只是通过短路实现突破登录，只需要登录操作加强防范即可。

### SQL 注入工具`sqlmap`

sqlmap 实现拖库的方式是，让 SQL 执行后的出错信息包含字段内容。

盲注，指的是注入后并不能从服务器得到任何执行结果（甚至是错误信息），只能寄希望服务器对于 SQL 中的真假条件表现出不同的状态。

即使屏蔽错误信息错误码，也不能彻底防止 SQL 注入。真正的解决方式，还是使用参数化查询，让任何外部输入值只可能作为数据来处理。

对于 MyBatis 来说，同样需要使用参数化的方式来写 SQL 语句。在 MyBatis 中，“#{}”是参数化的方式，“${}”只是占位符替换。

有 4 种可行的注入方式，分别是**布尔盲注、报错注入、时间盲注和联合查询注入**


# 30 | 如何正确保存和传输敏感数据？

## 1. 应该怎样保存用户密码？

仅仅使用 MD5 对密码进行摘要，并不安全。
- 不能在代码中写死盐，且盐需要有一定的长度。
  - 最好是每一个密码都有独立的盐，并且盐要长一点，比如超过 20 位。
- 虽然说每个人的盐最好不同，但不建议将一部分用户数据作为盐。
  - 盐最好是随机的值，并且是全球唯一的，意味着全球不可能有现成的彩虹表给你用。
- 更好的做法是，不要使用像 MD5 这样快速的摘要算法，而是使用慢一点的算法。



## 2. 应该怎么保存姓名和身份证？
对称加密与非对称加密。

对称加密常用的加密算法，有 DES、3DES 和 AES。
在业务代码中要避免使用 DES 加密。

- AES 是当前公认的比较安全，兼顾性能的对称加密算法。
  - ECB 模式虽然简单，但是不安全，不推荐使用。除了 ECB 模式和 CBC 模式外，AES 算法还有 CFB、OFB、CTR 模式

## 3. 用一张图说清楚 HTTPS

下图来自专栏文稿：

![](images/HTTPS流程.png)