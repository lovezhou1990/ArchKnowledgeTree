# redis

## 缘起

常识：

- 硬盘
    - 寻址时间：ms级别
    - 带宽、吞吐：百兆、G
- 内存
    - ns

1. 文件，量大起来之后？
- 全量扫描，会变慢

2. 数据库
快一些
数据分治，避免全量扫描
可以加索引，进一步加速


数据量进一步变大，CRUD操作：写一定会变慢，
读看场景：
- 只有一个连接、带where条件，索引命中，基本仍是ms级别
- 并发很大时，内存、吞吐会有限制，读的速度会变慢


瓶颈一直是硬盘！！！


另一种极端：纯内存数据库  SAP HANA  成本过高


为何redis没做成 sql数据库，而是做成了no-sql数据库？
数据库有约束（约束/范式），如果一张表A依赖于另一张表B，那redis中缓存了A就也得缓存B？显然这种约束会带来问题
redis：一般只是缓存一部分数据。


特点:
key-value
基于内存
value是有类型的：string, list, set, hash, zset, 且每种类型有自己的本地方法
    - 数据向计算移动
        memcache 都是string类型
    - 计算向数据移动
        redis 类型有自己的本地方法，省去很多IO


单线程：
- 6.x 以下：
    - worker单线程
- 6.x 支持多线程
    - IO threads: IO多线程，读到的内容仍是单线程去处理


memcache 都是string类型
    数据向计算移动


## 工作线程是单线程
比如，秒杀场景：

商品库存：redis 保存，调用redis decr 减库存

单线程：串行化
——再高并发


连接池：List<Socket>，可能由一个或是多个线程处理
线程池：多个线程
——nio/多路复用


redis 5.x
centos 6.x


企业级部署
每个redis实例不要存太多数据，在G级别，几G，十几G即可，方便恢复。数据多可以采用多个redis实例分别存


NIO   epoll
多路复用

