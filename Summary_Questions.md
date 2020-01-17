# 面经题解答  

## [阿里1](<https://www.nowcoder.com/discuss/186196>)  

### 简历面  

1. volatile的底层如何实现，怎么就能保住可见性了？  
  
2. 三个线程如何实现交替打印ABC
3. 线程池有哪些创建方式和安全性问题
4. 有哪些线程池的类型
5. 线程池中LinkedBlockingQueue满了的话，线程会怎么样
6. 线程池的底层原理和实现方法
7. 线程之间的交互方式有哪些？有没有线程交互的封装类 （join）
8. 算法：堆排序、栈实现队列、反转链表
9. Java锁机制，都说一下
10. 除了@ResponseBody，controller层如何标准返回给前端所要的数据类型？你会怎么实现？
11. 异常捕获处理
12. Spring MVC的原理和流程
13. HashMap和ConcurrentHashMap哪个效率更高？为什么？
14. Redis的缓存淘汰策略有哪些？
15. Java内存模型说一下
16. mybatis如何进行类型转换
17. mybatis的xml有什么标签
18. MySQL锁机制
19. 如何修改linux的文件权限
20. jvm的回收算法
21. 你会怎么设计数据库表结构
22. 数据库有哪些索引？
23. 如何防止sql注入
24. 抽象类和接口有什么不同
25. mysql间歇锁的实现原理
26. future的底层实现异步原理
27. SpringBoot Starter原理
28. rpc原理
29. 多个服务端上下线怎么感知
30. 缓存和数据一致性，怎么处理。流式计算
31. 多线程讲一下，FutureTask
32. Java和mysql的锁介绍，乐观锁和悲观锁
33. 分布式一致性讲一讲
34. 分布式锁的实现方式，zk实现和redis实现哪个比较好
35. 多点登陆怎么实现
36. 把乐观锁加在数据库上面，怎么实现
37. 项目介绍
38. 降级处理hystrix了解过么
39. 两次点击，怎么防止重复下订单
40. ioc原理详细讲讲，源码看过么
41. 静态代理和动态代理的区别
42. JUC说说你知道的东西  
  ConcurrentHashMap等  
  locks目录下的AQS和Lock  
  atomic目录下的原子变量  
  BlockingQueue和ThreadPool

43. B+树的叶子节点  

### 菜鸟一面  

1. Java内存模型
2. full gc怎么触发
3. gc算法
4. 高吞吐量的话用哪种gc算法
5. ConcurrentHashMap和HashMap
6. JDK8的stream的操作
7. volatile原理
8. 有参与过开源的项目
9. 项目介绍
10. 线程池原理，拒绝策略，核心线程数
11. 1亿个手机号码，判断重复
12. 是否有写过小工具
13. 单元测试介绍一下，多模块依赖怎么单元测试。Mockito  

### 菜鸟二面  

1. 项目介绍
2. dubbo、netty介绍原理
3. 限流算法
4. zk挂了怎么办
5. 秒杀场景设计，应付突然的爆发流量
6. redis的热点key问题
7. redis的更新策略（先操作数据库还是先操作缓存）
8. 分布式数据一致性
9. 一致性哈希
10. 消息队列原理介绍（不太会）
11. full gc问题，怎么排查
  -XX:+PrintGC 输出GC日志
  -XX:+PrintGCDetails 输出GC的详细日志
  -Xloggc:../logs/gc.log 日志文件的输出路径
  从日志中看，不同的问题会有标注，按照不同问题来解决。  

12. jvm的回收策略
13. ClassLoader原理和应用
14. 注解的原理
15. 数据库原理，数据库中间件，索引优化
16. aop原理和应用
17. 大数据相关，MapReduce
18. 机器学习有了解么？
19. Java的新技术，以及技术最新进展
20. Docker的原理  

### 菜鸟三面&四面  

1、项目介绍
2、分布式事务
3、Java三大特性
4、数据库表设计
5、RPC原理
6、netty原理
7、降级策略和降级框架  

补充： ping原理  
