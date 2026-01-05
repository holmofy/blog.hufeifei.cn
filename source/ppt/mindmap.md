---
title: 思维导图
date: 2022-04-15
type: "mindmap"
---

```plantuml
@startmindmap
* 数据结构与算法
** 线性结构
*** 数组
****_ 优点
*****_ 随机访问
****_ 缺点
*****_ 不能动态扩容
*** 链表
****_ 优点
*****_ 支持动态扩容
****_ 缺点
*****_ 不支持随机访问，需要遍历链表
*****_ 存储指针，空间利用率低
*** 扩容数组
**** java.ArrayList/c++ vector
*** 堆栈
****_ FILO的线性表
*** 队列
****_ FIFO的线性表
** hash
*** 计算hash值做数组索引
*** 解决hash冲突的方法
**** 开放定址法(线性探测法)
*****_ ThreadLocalMap
**** 链地址法
*****_ HashMap
**** 再Hash法
*** 常见hash算法
****_ checksum
****_ murmur3/crc32
****_ md5/sha
*** BloomFilter
****_ 通过计算n个不同Hash函数的Hash值，设置相应的n个bit位
****_ 优点
*****_ 时间和空间效率都很高
****_ 缺点
*****_ 不包含判断准确，包含存在误判
*****_ 删除困难，不能直接将bit设置成0，因为存在hash冲突，可能会影响其他元素
****** CountingBloomFilter
****_ 应用
*****_ 缓存问题
******_ 通过BloomFilter预判，减少磁盘IO
*******_ 黑名单过滤/垃圾邮件过滤
*****_ 爬虫防重复爬取
*** 外部hash
****_ 将哈希碰撞写入文件中
** 树
*** 二分查找树
****_ 解决二分查找需要预先排序的问题
****_ 左右子树不平衡会退化成链表
*** 自平衡二叉树
**** AVL树
*****_ 左右子树高度差小于等于1，完全平衡
******_ 删除节点可能导致logn次旋转
**** 红黑树
*****_ 最深路径不超过最浅路径的两倍，大致平衡
******_ 每个红色节点的两个子节点都是黑色/从每个叶子到根的所有路径上不能有两个连续的红色节点
******_ 从任一节点到其每个叶子的所有路径都包含相同数目的黑色节点
******_ 最短的可能路径都是黑色节点，最长的可能路径有交替的红色和黑色节点。因为根据所有最长的路径都有相同数目的黑色节点的性质，这就表明了没有路径能多于任何其他路径的两倍长。
*** 多维数据结构
**** 4叉树(2维)/8叉树(3维)/16叉树(4维)
**** K-D树
*****_ 连续交替按维度划分
*** 面向磁盘的树
**** 磁盘特性
*****_ 速度慢，吞吐量大
*****_ 块式读写——基本读写单位为一个扇区，一个扇区512B或4KB
*****_ 磁盘机械臂的运行方式导致顺序读写快于随机读写
**** B树
*****_ 以块的形式存储节点，一个节点存储n个元素，n+1个子节点指针
*****_ 当节点元素个数x>n,节点分裂，x<n/2，节点合并
*****_ B树扇出大，降低了树的高度，减少了磁盘随机IO次数
**** B+树
*****_ 内部节点只存储Key 叶子节点存储Value
******_ 内部节点能存储更多元素 索引效率更高，对内存缓存也更友好
*****_ 叶子节点以链表形式串联
******_ 方便范围查询，避免了B树范围查询的回溯问题，回溯意味着更多的磁盘IO
**** LSM-Tree (Log-Structured Merge-Tree)
*****_ 将修改缓存在内存数据结构中(MemTable)中 达到阈值后merge到磁盘的数据结构中(SSTable)中
*****_ 磁盘中的SSTable会有多个分段，需要后台进程定时merge
*****_ 整个数据结构就像一个操作日志，后台Merge线程把过期的操作清理掉
*****_ 多个SSTable中读取，牺牲读性能换取写性能
*****_ 使用BloomFilter进行元素预判减少磁盘IO，优化读性能
**** 多维
***** KDB树
******_ KD树与B树的结合
******_ 节点分裂复杂，所以KDB树只适合静态数据
***** BKD树
******_ KDB树+LSM-Tree解决静态数据问题
***** R树(Region)
*** 堆
**** 二叉堆
*****_ 堆是完全二叉树，所以可以用数组实现
******_ 根索引为0，左节点索引为2*0+1，右节点索引为2*0+2 1的左节点索引为2*1+1，右节点索引为2*1+2
*****_ 应用
******_ 优先级队列
******_ topN问题/堆排序/多路归并
**** 多叉堆
*****_ 根索引为0，第一个子节点索引为n*0+1，第二个子节点索引为n*0+2...第n个子节点n*0+n
*** 哈夫曼树
**** 数据压缩
*** 遍历
**** 先序遍历(先根)/后序遍历(后根)/中序遍历(中根)
** 图
*** 有向图(DirectedGraph)
****_ 出度
****_ 入度
*** 有向无环图(DAG)
*** 无向图(UnDirectedGraph)
*** 有权图(WeightedGraph)
****_ 边上有权重
*** 遍历
****_ 深度优先遍历
***** 栈
****** 连通性问题
****_ 广度优先遍历
***** 队列
****** 最短路径问题
** 排序
*** 内排序
**** 选择排序
***** 直接选择排序
******_ 从无序区选择最小元素跟在有序区尾部(比较多，交换少，区域后移)
******_ 不稳定，平均时间复杂度O(n^2)
***** 堆排序
******_ 将选择过程中的比较结果记录在堆中，每次卸除堆顶最小元素放入有序区，重新恢复堆
******_ 不稳定，平均时间复杂度O(nlogn)
**** 插入排序
***** 直接插入排序
******_ 把无序区的第一个元素插入到有序区的合适位置(比较少，交换多，插入后移)
******_ 稳定，平均时间复杂度O(n^2)
***** 希尔排序
******_ 为了加大插入排序的移动步伐，预设一个间隔 每轮根据预先设置的间隔进行插入排序，依次缩短间隔直至为1
******_ 不稳定，平均时间复杂度O(nlogn)
**** 冒泡排序
*****_ 从无序区通过交换选出最大元素放到有序区头部
*****_ 稳定，平均时间复杂度O(n^2)
**** 快速排序
*****_ 从区间中随机选择一个元素作为基准值，将小于基准值的元素放在基准值前，大于基准值的放后面，再递归对小数区和大数区排序
*****_ 不稳定，平均时间复杂度O(nlogn)
**** 归并排序
*****_ 将数据分成两段，从两段中选择最小元素移入新数据段末尾
*****_ 稳定，平均时间复杂度O(nlogn)
**** 计数排序
***** 直接计数排序
******_ 统计每个元素的出现频次以下标形式存入计数器，然后从小到大将元素回写
***** 基数排序
*** 外排序
**** 内排序+磁盘读写+多路归并
** 字符串匹配
*** KMP/Rabin-Karp/Boyer-Moore
** 算法思想
*** 分而治之/动态规划/贪心/回溯
** 密码学
*** 对称加密
**** 块加密
*****_ DES/3DES/AES
**** 流加密
*****_ 伪随机数生成器
******_ 线性同余
*****_ RC4
*** 非对称加密
****_ RSA
*** 混合密码系统
****_ 用非对称加密保护会话密钥，用对称加密提升速度
*** 单向散列/摘要函数
****_ MD5/SHA-1/SHA-256/SHA-384/SHA-512
*****_ 消息认证码(MAC)
******_ 保证数据完整性
******_ HMAC标准
*** 数字签名
****_ 用私钥签名，公钥校验
*****_ 保证了不可否认性
*** 公钥证书(PKC)
****_ 用认证机构(CA)的私钥对个人或企业的公钥进行签名
****_ 上层认证机构(CA)对认证机构(CA)的公钥进行签名，从而产生证书链
@endmindmap
```

```plantuml
@startmindmap
* 操作系统
** 什么是操作系统
***_ 一种特殊的软件，对下管理硬件，对上管理用户软件
** 管理硬件
*** CPU
****_ 进程管理
*** 内存
****_ 内存管理
*** IO设备
**** 硬盘
*****_ 文件管理
**** 网络
*****_ TCP/IP
**** 其他IO设备
*****_ 设备驱动
** 用户软件
*** 通过操作系统提供的接口使用硬件设备
@endmindmap
```

![TCP报文格式](https://github.com/user-attachments/assets/8a6bf8ec-de9c-48d3-89a1-804f092f62b5)

![TCP状态图](https://github.com/user-attachments/assets/b3d2d6e7-73cb-4e9c-b99a-43b3b4e97946)

![DNS工作原理](https://github.com/user-attachments/assets/9447e275-7ef7-4370-a038-ea73d72ce5cd)

![Java8GC算法总结](https://github.com/user-attachments/assets/0ec369df-8cec-4f64-83d0-f84f52127b7d)


```plantuml
@startmindmap
* 计算机网络
** 网络协议分层
*** OSI
****_ 物理层 / 数据链路层 / 网络层 / 传输层 / 会话层 / 表示层 / 应用层
****_ 理论模型，强调分层思想
*** TCP/IP
****_ 链路层 / 网络层 / 传输层 / 应用层
****_ 工程实现模型，实际互联网使用
** 五层建议模型
*** 物理层
****_ 单工 / 半双工 / 全双工
****_ 多路复用
***** 时分复用(TDM)
***** 频分复用(FDM)
***** 码分复用(CDM)
****_ 传输介质
***** 双绞线 / 光纤 / 无线
*** 链路层
**** 以太网
*****_ MAC地址(48bit)
****** FF-FF-FF-FF-FF-FF为局域网广播地址
*****_ MTU(1500字节)
**** 协议
*****_ ARP(IP=>MAC)
******_ ARP缓存表
*****_ ARP(IP=>MAC) / BOOTP / DHCP
******_ DHCP四次交互
*******_ Discover / Offer / Request / ACK
*** 网络层
**** IP
*****_ IPv4(32bit) / IPv6(128bit)
*****_ CIDR无类地址
******_ 子网掩码
**** IP数据包分片
**** 路由
***** 静态路由
******_ route命令
***** 动态路由(路由器间相互通信)
****** 内部网关协议(自治系统内部)
*******_ 链路状态路由协议
******** OSPF
*******_ 距离矢量路由协议
******** RIP/RIPv2/IGRP/EIGRP
****** 外部网关协议(自治系统间)
******* BGP(边界网关协议)
**** ICMP
*****_ 差错报告 / 网络诊断
*****_ ping/traceroute
**** 广播
***** 受限广播
******_ 255.255.255.255
*******_ 未指定网络，只在本地网络传输
***** 定向广播
******_ netid.255.255.255
*******_ 对netidA类网络进行广播
*** 传输层
**** UDP
*****_ 无连接 / 不可靠 / 面向报文
*****_ 适用场景：DNS / 视频 / 语音
**** TCP
***** 报文格式
******_ 源端口和目标端口用于确定发送端与接收端的应用进程
******* 16bit端口 < 65535
******_ 32bit序列号标识数字字节流，可以循环使用
******_ 确认序列号是确认端期望接受的下一个序列号，所以确认序列号=上次成功接收的序列号+1
******* 只有ACK标识位为1时才生效 发送ACK无序额外代价，因为和确认序号一样总是TCP首部的一部分 因此操作系统在建立连接后，ACK标识位总被设置为1
******_ 4bit首部长度(数据偏移)，因为有任选字段，所以TCP最多有(2^4-1)*4=60字节的首部，没有任选字段，正常为20字节(该字段为5)
******_ 6个保留位和6个标志位
******* URG | ACK | PSH | RST | SYN | FIN
******_ 16bit窗口大小用于流量控制
******_ 16bit校验和用于校验整个TCP报文：TCP首部+TCP数据
******_ 16bit紧急指针用于发送紧急数据
******* 只有当URG标志位为1时有效
******_ 可选项和数据体
***** 三次握手
****** SYN => SYN + ACK => ACK
******* TCP连接是双向的，所以建立连接需要双方确认 TCP是全双工的，所以相应端发出同步报文的同时可以连带发送上一个报文的确认
***** 四次挥手
****** FIN=>ACK...FIN=>ACK
******* TCP是全双工的，所以两个方向必须单独关闭，当只有一方关闭时，会出现半关闭状态
***** 流量控制
****** 滑动窗口 / 满启动 / 拥塞避免 / 快速重传 / 快速恢复
***** 超时重传
*** 应用层
**** DNS
***** 根DNS服务器
***** 权威域名服务器
**** FTP
**** HTTP
***** 报文格式
****** http method
******* GET | POST | PUT | DELETE | HEAD | OPTIONS | TRACE | CONNECT
****** http状态码
******* 1xx
******** 信息提示。表示该请求已被服务器接收，但需要继续处理
********* 101 切换协议
******* 2xx
******** 请求成功。服务器成功处理了请求
********* 200 成功
******* 3xx
******** 客户端重定向。告诉客户端浏览器，他们访问的资源已被移动，并告诉客户端新的资源位置
********* 301 永久重定向｜302 临时重定向｜304 资源未修改
******* 4xx
******** 客户端信息错误。客户端可能发送了服务器无法处理的内容 比如请求的格式错误，或者请求了一个不存在的资源
********* 400 请求参数有误｜401未授权｜403禁止访问｜404资源不存在｜405Method有误
******* 5xx
******** 服务器出错。客户端发送了有效的请求，但是服务器自身出现错误，比如Web程序运行出错
********* 500服务内部异常｜502网关异常
***** 持久连接
****** http1.0一次http通信结束就要断开一次TCP连接
****** http1.1持久连接
******* keep-alive
******** 复用TCP连接
***** 缺点
****** 无状态
******* Cookie和Session
******** Session是服务端保存的状态，Cookie是实现Session的通用手段
********* Header：Cookie ｜ Set-Cookie
****** 安全性
******* HTTPS
****** 服务器无法主动发数据
******* WebSocket & WebPush
**** HTTPS
***** SSL/TLS握手过程
** 网络安全
*** 网络
**** 链路层
***** ARP欺骗
****** 修改IP映射的MAC地址，使数据包发送到被修改后的MAC地址上
***** MAC泛洪攻击
****** 伪造ARP包攻击交换机，MAC表满了，交换机会进入集线器模式广播MAC表里没有的数据包
**** 网络层
***** IPsec
****** VPN
**** 传输层
***** SYN泛洪攻击
****** 发送伪造IP地址建立连接，消耗服务器连接资源
***** SSL/TLS
****** 明文=裸奔 对称加密：key唯一=明文 非对称：server->client不安全 对称+非对称：中间人攻击 CA签发证书 + 混合加密
**** 应用层
***** DNS欺骗
****** 冒充域名服务器
***** DNS劫持
****** 拦截DNS报文，篡改IP地址
***** HTTPS=HTTP + SSL/TLS
*** Web
**** XSS
**** SQL注入
@endmindmap
```

```plantuml
@startmindmap
* 软件工程
** 面向对象原则
*** 单一职责
****_ 每个类实现的功能应该尽可能单一，不要啥都做。笼统地把功能放在一个类中，就退化成面向过程编程
*** 开闭原则
****_ 对扩展开放，对修改关闭
*** 里氏替换
****_ 基类是子类的抽象化，基类出现的地方可以用子类的实现替换
*** 依赖倒置
****_ 应该依赖于抽象，而不应该依赖实现细节，也就是面向接口编程
*** 接口隔离
****_ 接口设计粒度应该细，这样系统才能更灵活
*** 迪米特法则
****_ 一个类应该尽可能少的了解另一个类的实现，降低类与类之间的耦合
** 设计模式
*** 创建型
**** 单例模式 / 原型模式 / 工厂方法 / 抽象工厂 / 建造者模式
*** 结构型
**** 代理模式
***** JDK的Proxy
**** 适配器模式
***** SpringMVC的HandlerAdapter
**** 桥接模式
**** 装饰模式
***** FilterInputStream/FilterOutputStream
**** 外观模式
**** 享元模式
***** StandardCharsets
*** 行为型
**** 模版方法
***** ThreadPoolExecutor.beforeExecute/afterExecute   LInkedHashMap.removeEldestEntry
**** 策略模式
**** 命令模式
**** 责任链模式
***** SpringMVC的HandlerInterceptor Servlet的Filter
**** 状态模式
**** 观察者模式
***** 各种Listener响应事件
**** 访问者模式
***** Files.walkFileTree
**** 中介者模式
**** 迭代器模式
**** 备忘录模式
**** 解释器模式
** 系统工程
*** 系统设计与架构
**** https://github.com/donnemartin/system-design-primer
**** SOA与微服务
***** https://github.com/aphyr/distsys-class
***** https://github.com/mfornos/awesome-microservices
***** https://github.com/binhnguyennus/awesome-scalability
*** 数据库选型
**** https://github.com/igorbarinov/awesome-data-engineering
*** https://github.com/AlaaAttya/software-architect-roadmap
** 测试
*** 测试用例管理
*** 集成测试
*** 单元测试
*** 测试自动化
**** https://github.com/atinfo/awesome-test-automation/blob/master/java-test-automation.md
*** https://github.com/fityanos/Quality-Assurance-Road-Map
** 运维
*** docker
**** docker compose
***** 单机容器编排
**** docker swarm
***** 多机容器编排
**** k8s
***** 集群容器编排
** 持续集成(Continuous integration)
*** Jenkins
**** https://github.com/ligurio/awesome-ci
** 团队管理
*** 敏捷开发
**** Scrum
***** https://github.com/lorabv/awesome-agile
** 职业生涯发展规划
*** https://github.com/orsanawwad/awesome-roadmaps
*** https://github.com/sindresorhus/awesome
@endmindmap
```

<img width="700" height="537" alt="InnoDB内存和磁盘布局" src="https://github.com/user-attachments/assets/cc7a53e3-dd94-4943-a0d6-b53bcad81914" />

```plantuml
@startmindmap
* 数据库
** RDB
*** 索引结构
**** B+树
***** 范围查询性能比较好
**** hash索引
***** 能精确查询，但不支持范围查询，而且冲突较多时性能不是很好
**** 位图索引
***** 只适合性别，是否活跃状态等只有两种状态的字段
*** Join原理
**** NestedLoopJoin
***** 暴力两层循环
***** block-based NestedLoopJoin
****** 小表放在外层循环，优化IO
**** SortMergeJoin
***** 先把两个表排序，再merge 2 sort list
**** HashJoin
*** MySQL
**** 存储引擎
***** InnoDB/MyISAM/Memory/CSV/...
**** 数据类型
***** 数值型
****** bit(1b)/tinyint(1B)/smallint(2B)/mediumint(3B)/int(4B)/bigint(8B)
***** 字符型
****** char/varchar/tinytext/text/mediumtext/longtext binary/varbinary/tinyblob/blob/mediumblob/longblob
***** 日期型
****** date/time/datetime/timestamp
**** InnoDB
***** 表空间文件名以.idb为后缀
***** 聚簇索引
****** 主键为聚簇索引，二级索引的叶子节点会存主键值，这样减少了数据行移动或者叶分裂时的二级索引维护工作
****** 使用二级索引查找会进行两次B+树查找，先从二级索引找到主键，再从主键索引找到数据行
***** 优点
****** 聚簇索引将索引和数据行保存在一个B+树中，查询聚簇索引可以直接获取数据，而非聚簇索引需要多次IO，所以聚簇索引比非聚簇索引查找更快
****** 因为聚簇索引是按主键排列的，索引对于主键的范围查找效率很高
****** 二级索引使用覆盖索引可以直接使用叶子节点中的主键值
***** 缺点
****** 插入速度严重依赖插入顺序。按主键自增顺序往InnoDB插入是最快的
****** 聚簇索引插入时，可能导致页分裂。页分裂会导致表占用更多的表空间(不要用UUID这样的随机值做主键，而应该用单调递增的值做主键)
****** 二级索引访问数据需要两次B+树查找，而且二级索引保存了主键列，二级索引可能会占用更多空间(索引选择一个短的主键列是有利的)
***** ACID
****** Atomicity
******* undo log
******** 实现原子性的关键，是当事务回滚时能够撤销所有已经成功执行的sql语句
******** InnoDB实现回滚，靠的是undo log：当事务对数据库进行修改时，InnoDB会生成对应的undo log；如果事务执行失败或调用了rollback，导致事务需要回滚，便可以利用undo log中的信息将数据回滚到修改之前的样子。
****** Durability
******* redo log
******** InnoDB作为MySQL的存储引擎，数据是存放在磁盘中的，但如果每次读写数据都需要磁盘IO，效率会很低。为此，InnoDB提供了缓存(Buffer Pool)，Buffer Pool中包含了磁盘中部分数据页的映射，作为访问数据库的缓冲： 当从数据库读取数据时，会首先从Buffer Pool中读取，如果Buffer Pool中没有，则从磁盘读取后放入Buffer Pool；当向数据库写入数据时，会首先写入Buffer Pool，Buffer Pool中修改的数据会定期刷新到磁盘中（这一过程称为刷脏）。
******** Buffer Pool的使用大大提高了读写数据的效率，但是也带了新的问题：如果MySQL宕机，而此时Buffer Pool中修改的数据还没有刷新到磁盘，就会导致数据的丢失，事务的持久性无法保证。
******** redo log采用的是WAL（Write-ahead logging，预写式日志），所有修改先写入日志，再更新到Buffer Pool，保证了数据不会因MySQL宕机而丢失，从而满足了持久性要求。
******** redo log与bin log区别
********* 作用不同
********** redo log(crash recovery) | bin log(point-in-time recovery/replication)
********* 层次不同
********** redo log(InnoDB引擎) ｜ bin log(MySQL server)
********* 内容格式不同
********** redo log(物理数据) | bin log(binlog_format指定是sql还是数据本身或者两者混合)
********* 写入时机不同
********** redo log(innodb_flush_log_at_trx_commit指定是否在事务提交时刷盘，master线程也会定时刷盘) ｜ bin log(事务提交时写入)
****** Isolation
******* 保证事务执行尽可能不受其他事务影响；InnoDB默认的隔离级别是RR，RR的实现主要基于锁机制（包含next-key lock）、MVCC（包括数据的隐藏列、基于undo log的版本链、ReadView）
******* Read Uncommitted
******* Read Committed
******** 解决了脏读
********* 读取了其他事务还未提交的内容
******* Repeatable Read
******** 解决了不可重复度
********* 事务重复读取前后不一致，读取到另一个事务提交的修改
******** InnoDB的RR隔离级别通过Next-Key Lock解决了幻读问题
******* Serializable
******** 完全串行，并发性差
******* InnoDB默认RR，MS SqlServer和Oracle默认都是RC
****** Consistency
******* 数据库的完整性约束没有被破坏，事务执行的前后都是合法的数据状态
***** 覆盖索引
****** 查询的所有列都在索引中，不需要回表查询
***** MySQL5.6索引条件下推(ICP)优化
***** MVCC
****** DATA_TRX_ID、DATA_ROLL_PTR
******* DATA_ROLL_PTR来串起undo log中的版本链表
******* 在RC级别下，MVCC的快照读解决了不可重复读的问题
***** 锁
****** https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html
****** 共享锁(S)： 读锁 普通的select  排他锁(X)： 写锁 update/delete/insert
****** 意向共享锁(IS)： SELECT ... FOR SHARE 意向独占锁(IX)： SELECT ... FOR UPDATE
******* 意向锁是表锁，用于指明事务稍后要对表中的行进行哪些(S | X)锁定
******* 在事务申请S锁之前,它必须先在表上申请IS锁或更强的锁
******* 在事务申请X锁之前, 它必须在表上申请IX锁
******* InnoDB支持行级锁，所以意向锁不会阻塞除了全表扫描以外的任何请求
****** 表锁
******* https://dev.mysql.com/doc/refman/8.0/en/lock-tables.html
****** 行锁
******* Record Lock
******* Gap Lock
******* Next-Key Lock
******** 解决幻读的问题
****** 解决死锁的方法
******* timeout
******** FIFO，会导致权重大的事务被回滚了，更新了很多行
******* wait-for graph
****** 乐观锁
******* 通过版本号比较并交换
**** MyISAM
***** 索引文件为.myi，数据文件为.myd
***** MyISAM的主键索引和二级索引一样，叶子节点存储的是数据行在.myd文件中的偏移量
*** PostgreSQL
** NoSQL
*** Redis
**** 特点
***** 单线程，使用IO多路复用处理请求，因为数据都在内存中，所以速度比较快
**** 5种数据类型
***** string
****** <=512M
****** 原子性自增自减(incr | incrby)
******* Spring Data Redis种的RedisAtomicXxx
****** 支持bit操作(getbit | setbit)
******* 实现BloomFilter
****** 分布式锁
*******  SET resource_name my_random_value NX PX 30000
******** https://redis.io/topics/distlock
***** list
****** quicklist = linkedlist(双向链表)+ziplist(数组)
******* 结合了链表和数组的优点
****** 应用场景
******* 分布式阻塞队列
******** lpush | rpush | blpop | brpop
******* 双端队列
******** lpush | lpop | rpush | rpop | lset | lrem | ltrim | lrange | lindex | llen
***** hash
***** set
***** sort-set
****** 跳跃链表
******* 根据key对应的score排序
****** 可以实现阻塞的优先级队列
******* bzpopmax ｜ bzpopmin
**** 持久化
***** RDB(默认)
****** fork子进程定时将数据快照保存到dump.rdb文件中
****** save/bgsave命令可以保存快照，前者同步，后者fork子进程
****** 优点
******* 数据紧凑，适合数据备份
***** AOF
****** 记录服务接收的每一个写操作追加到日志文件中，启动时再次执行一遍恢复原始数据
**** 过期淘汰策略
***** 惰性删除
****** 问题是有些可能永远不会访问
***** 定期采样删除
****** 每100ms随机测试20个key，删除已过期的。如果采样中超过1/4已过期，再执行一次
**** 坑
***** keys查找key，在大数据量下会阻塞，要该用scan
*** MongoDB
**** 基于B树存储
**** WiredTiger存储引擎基于LSM-Tree结构
*** HBase
**** 基于LSM-Tree结构
*** ElasticSearch
**** Lucene
***** 基于LSM-Tree结构，分段存储，后台线程按照一定的策略Merge分段
****** Merge策略
******* LogMergePolicy(指数分级策略)
******** LogDocMergePolicy(以分段文档数为指标)
******** LogByteSizeMergePolicy(以分段字节数为指标)
***** 全文索引
****** 词典(.tim | .tip)
******* FST
******** 前后缀都压缩的有穷状态级
****** 倒排表(.doc|.pos|.pay)
******* SkipList
******** 使用跳跃链表优化“与”查询
***** PointValue(.dim|.dii)
****** BKD-Tree
******* 索引数值类型字段，且支持多维数据类型
******** Lucene6.0之前，数值类型都转换成字符串类型进行全文索引
***** StoredFields(.fdt|.fdx)
****** 以压缩格式存储实际的	文档字段值
***** DocValues(.dvm|.dvd)
****** 字段的列式存储
******* 便于文档排序和聚合统计
**** ES严重依赖于操作系统的文件缓存技术，所以给ES的Java进程分配的堆大小不能太大，给操作系统预留内存来缓存文件内容，1:1
**** bulk大数据量到ES时，将refresh间隔设为-1关闭
** NewSQL
*** TiDB
* 分布式
** 理论
*** CAP理论
**** https://zhuanlan.zhihu.com/p/33999708
**** 舍弃分区容错性，保证一致性和高可用
***** 传统单机SQL数据库，或单机房同步
**** 舍弃一致性，保证高可用和分区容错性
***** NoSQL
****** 最终一致性
**** 舍弃可用性，保证强一致性和分区容错
***** 传统分布式事务，数据同步时间无限延迟
****** 银行等交易型业务必须保证强一致性
*** BASE理论
**** Basically Available
*** 一致性hash
*** 2PC
*** 3PC
*** Paxos
*** Raft
** MQ
*** 作用
**** 削峰填谷，异步解耦
*** ActiveMQ
**** 基于数据库的队列表实现
***** 堆积能力一般
*** Kafka
**** 基于磁盘文件高性能顺序读写的特性来设计的存储结构
**** 利用操作系统的 PageCache 来缓存数据，减少 IO 并提升读性能
**** 使用批量处理的方式来提升系统吞吐能力
**** pull
***** 在 push-based 的系统中，当消费速率低于生产速率时，consumer 往往会不堪重负
***** 它可以大批量生产要发送给 consumer 的数据
****** push-based 系统必须选择立即发送请求或者积累更多的数据，然后在不知道下游的 consumer 能否立即处理它的情况下发送这些数据
*** RocketMQ
**** 优化Kafka多个topic的性能问题，topic多意味着随机写越多
**** 所有的topic共用同一个Log，topic队列只存储每个消息在Log文件的偏移量
**** 事务性消息
***** 实现TransactionListener接口
***** 接口中executeLocalTransaction执行本地事务
***** 接口中checkLocalTransaction提供RocketMQ回查状态
*** 重发
**** 异常或超时，返回重发状态
*** 重复消费问题
**** 保证消费端的幂等性
** 网关
*** Zuul
** RPC
*** Dubbo
*** 一个服务可能有多个实例，你在调用时，要如何获取这些实例的地址呢
**** 注册中心
*** 选哪个调用好呢？
**** 负载均衡
***** 客户端负载均衡
*** 避免每次调用都访问注册中心，注册中心宕机也能保证服务通信
**** 缓存容灾
*** 接口修改，不可能让多个调用方一次性改掉，要保证老接口还能用
**** 版本控制
** 注册中心/服务发现
*** Zookeeper ｜ Eureka｜ Consul ｜Etcd
** 配置中心
*** Nacos ｜ Spring Cloud Config
*** 作用
**** 配置区分环境(开发｜测试｜生产)
**** 静态配置动态化
**** 避免配置过于分散
** 高并发
*** 缓存
**** BloomFilter避免缓存击穿
*** 限流
**** 保证自己的服务不超过一定的负载，超过则拒绝
*** 熔断
*** 降级
** 事务
*** Seata
** 链路追踪
*** Jaeger | Zipkin
** 调度
*** SchedulerX
** 监控
@endmindmap
```

![JVM偏向锁](https://github.com/user-attachments/assets/dac61922-5e68-4a99-8aab-d6e8bb04a65e)

![RocketMQ的文件存储原理](https://github.com/user-attachments/assets/ea371242-4e66-44cc-9661-c53ecfb18be2)

![RocketMQ事务性消息流程图](https://github.com/user-attachments/assets/9039212f-845e-4110-a0e0-119d6342a56b)

```plantuml
@startmindmap
* Java
** 基础
*** 集合
**** List
***** ArrayList
****** 1.5倍扩容，初始容量为10(Java7有延迟初始化)
***** Vector
****** 默认两倍扩容，初始容量为10
******* 可用Collections.synchronizedList(new ArrayList())替代
***** LinkedList
****** 双向链表
***** CopyOnWriteArrayList
****** 写时拷贝，保证并发时读写分离，不适合写频繁的场景
**** Map
***** HashMap
****** 容量为2的幂，初始容量为16 负载因子(元素数量与容量比)大于0.75时扩容
******* 用位运算代替取余：index=hash&(length-1)
******* rehash时元素计算的newIndex=hash&(length*2-1)                                             =hash&(length-1)+hash&length                                             =oldIndex+hash&length 也就是看元素hash值高一位是0还是1，决定是否移动到新桶
****** 拉链法解决hash冲突
******* java8进行了优化，当元素数量超过64个会有红黑树优化：当拉链增长到8会进化成红黑树，当链表缩短到6退化成链表
***** LinkedHashMap
****** 在HashMap的基础上加了一条链表存储元素的插入顺序
****** 可以通过重载removeEldestEntry方法实现LRU缓存
***** WeakHashMap
****** Entry持有key的弱引用，当外部的key被回收时，Entry会加入到回收队列，下次读写会把这个Entry从表中移除
***** IdentityHashMap
****** 语义上相等(equals)的不同对象被认为是不同的key
******* 不看对象重载的hashCode()和equals()方法
***** Hashtable
****** 可用Collections.synchronizedMap(new HashMap())替代
***** TreeMap
****** 基于红黑树实现的有序Map，红黑树查找类似于二分查找，所以红黑树的增删改查时间复杂度是O(logn)
***** ConcurrentHashMap
****** 解决HashMap并发问题，比如多个线程同时resize会导致环路
****** 并发版本的HashMap，Java7用分段锁减小锁的粒度提高并发度，Java8通过CAS+synchronized控制并发
***** ConcurrentSkipListMap
****** 使用跳跃链表实现，跳跃链表是与二叉搜索树相当的数据结构，基于多级并联的链表实现 相较二叉查找树消耗更多的内存资源，但是实现比二叉查找树简单，而且支持并发
**** Set
***** HashSet/LinkedHashSet/TreeSet/CopyOnWriteArraySet/ConcurrentSkipListSet
**** Queue
***** PriorityQueue
****** 基于二叉堆实现
***** ConcurrentLinkedQueue
****** 使用CAS实现并发操作
**** BlockingQueue
***** ArrayBlockingQueue/LinkedBlockingQueue/SynchronousQueue/DelayQueue/PriorityBlockingQueue
**** Deque
***** ArrayDeque/LinkedList/ConcurrentLinkedDeque
**** BlockingDeque
***** LinkedBlockingDeque
**** TransferQueue
***** LinkedTransferQueue
**** Collections
***** unmodifiable/synchronized/checked
***** fill/reverse/frequency/disjoint/max/min/sort/binarySearch/shuffle/rotate
***** newSetFromMap/asLifoQueue
****** Collections.newSetFromMap(new ConcurrentHashMap())  => ConcurrentHashSet
*** 并发
**** Java内存模型
***** 对称多处理机(SMP)的内存可见性
****** volatile与synchronized
**** CAS
***** 底层调用CPU同步元语实现
****** Intel的cmpxchg指令
**** 进程
***** Runtime.exec()/ProcessBuilder
**** 线程
***** 线程中段机制
****** interrupted()与isInterrupted()的区别
***** 线程局部变量
****** ThreadLocal
******* ThreadLocalMap开放地址法解决hash冲突
**** 线程池
***** ThreadPoolExecutor
****** 1、当有任务提交时，会创建核心线程去执行任务(即使有核心线程空闲也会创建)； 2、当核心线程数达到corePoolSize时，后续提交的任务都会进入BlockingQueue中排队； 3、当BlockingQueue满了(offer()失败)，就会创建临时线程处理任务(临时线程空闲一定时间后，会被销毁)； 4、当线程总数达到maximumPoolSize时，后续提交的任务会被RejectedExecutionHandler拒绝。
****** allowsCoreThreadTimeOut()可以让核心线程空闲后销毁 prestartAllCoreThreads()可以预先创建所有的核心线程
****** 如何选择线程数
******* FixedThreadPool和CacheThreadPool的区别
******** 计算密集型还是IO密集型任务
****** 如何选择队列
******* ArrayBlockingQueue、LinkedBlockingQueue、SynchronousQueue的区别
***** ScheduledThreadPoolExecutor
****** 内部是一个基于时间排序的优先级队列
****** fixRate和fixDelay区别
******* fixRate是以任务开始时刻计算间隔，fixDelay是以任务结束时刻计算间隔
***** ForkJoinPool
****** 适合比较大的可拆分的计算型任务
******* java8 Stream api
******** Spliterator
******** CountedCompleter extends ForkJoinTask
**** 并发工具
***** 锁
****** ReentrantLock
******* 重入锁
******** 公平锁
********* 线程需要排队，先来先服务
********** 排队意味着大概率会挂起线程
******** 非公平锁
********* 新来的线程可以抢占已经排队的锁
********** 性能更好，避免无谓的线程挂起
****** ReentrantReadWriteLock
******* 读写锁
******** 读共享，写互斥
****** StampedLock
******* 读取支持乐观锁
******* 不支持重入
***** 同步工具
****** CyclicBarrier
******* 多个线程await()等待，达到一定数量后一起执行
******** 赛马
****** CountDownLatch
******* 若干线程countDown()直到latch为0后await线程启动
******** 点火箭
********* countDown()
********* await()
****** Exchanger
******* 可理解为双向的SynchronousQueue
******** 一手交钱一手交货
****** Phaser
**** synchronized实现
***** JVM规范
****** monitorenter ｜ monitorexit
***** Hotspot实现
****** 重量级锁
******* 会放弃CPU，会导致线程切换，比较耗时
****** 轻量级锁
******* 使用CAS操作尝试将头字段存入栈中的lock_record （可以用洗手间等待大小便形象比喻）
****** 偏向锁
******* 优化同一个线程多次申请同一个锁的竞争 一旦出现其他线程竞争资源，偏向锁就会被撤销
******** -XX:-UseBiasedLocking 禁用偏向锁
*** IO
**** 传统IO
***** InputStream&OutputStream(字节流) ｜ Reader&Writer(字符流)
**** NIO
***** 文件读写
****** Files&Paths
***** Buffer
****** 0<=mark<=position<=limit<=capacity
****** flip() / rewind()
****** heap(Java堆内存) / direct(操作系统内存映射)
***** Selector
****** I/O多路复用
******* select()/pool() [POSIX]
******** 基于轮询实现，复杂度O(n)
******* epoll(Linux) | kqueue(BSD)
******** 类似于阻塞队列，复杂度O(1)
***** AIO
****** AsynchronousChannel
******* aio_read() / aio_write() [POSIX]
******* IOCP [Windows]
**** 序列化
***** 二进制
****** Java序列化
******* ObjectInputStream/ObjectOutputStream
***** 字符
****** xml
******* SAX
******** push
********* SAXParser
******** pull
********* XMLEventFactory | XMLInputFactory | XMLOutputFactory
******* DOM
******** DocumentBuilder
******* Bind
******** JAXB
****** json
******* Jackson | Gson | FastJson
**** 网络
***** 链路层
****** NetworkInterface ｜ InterfaceAddress
***** 传输层
****** InetAddress
******* Inet4Address(IPv4) | Inet6Address(IPv6)
***** 传输层
****** 套接字
******* SocketAddress
******** InetSocketAddress
****** UDP
******* DatagramSocket ｜ DatagramPacket DatagramChannel(nio)
****** TCP
******* ServerSocket | Socket SSLServerSocket | SSLSocket ServerSocketChannel | SocketChannel
***** 应用层
****** 万维网三要素
******* URL ｜ HTML ｜ HTTP
****** HTTP
******* HttpURLConnection
**** RPC
***** RMI ｜ JWS
*** 安全
**** 加密(Cipher)
***** 对称加密
****** KeyGenerator ｜ SecretKey
****** SecretKeyFactory
***** 非对称加密
****** KeyFactory ｜ KeyPairGenerator ｜ KeyPair ｜ PublicKey ｜ PrivateKey
**** 单向散列
***** MessageDigest(消息摘要)
***** Mac(消息认证码)
**** 伪随机数
***** Random ｜ SecureRandom ｜ ThreadLocalRandom
**** 签名与证书
***** Signature ｜ Certificate ｜ CertificateFactory
*** 反射
**** 动态代理
***** Proxy ｜ InvocationHandler
**** 内省机制
***** Introspector ｜ PropertyDescriptor ｜ PropertyEditor
**** 字节码操作
***** Cglib
****** Enhancer
***** Javasist ｜ ByteBuddy ｜ ASM
** JVM
*** 工作原理
**** 解释执行
***** 由javac编译器翻译成与平台无关的字节码，执行时再由java解释器将字节码翻译成对应平台的机器码
**** 编译执行
***** Hotspot的JIT即时编译器支持三种模式
****** interpreted-only
******* 完全解释执行
****** compilation
******* 每次调用方法都会强制编译成机器码，并将机器码缓存起来，但代码缓存大小有限
****** mixed(默认)
******* 热点方法编译成机器码，其他方法有解释器临时解释执行
**** 逃逸分析
***** 将函数范围内的小对象分配在栈上，减少GC运行频率
*** JVM规范定义的内存结构
**** pc寄存器
***** 当前线程执行JVM指令的位置
**** 虚拟机堆栈
***** 当前线程的函数执行栈
****** -Xss=-XX:ThreadStackSize
**** Java堆
***** Java对象存放地
**** 方法区
***** 类结构和代码块的字节码存放地
**** 运行时常量池
***** 编译时已知的字面量，运行时解析的方法引用和字段引用
**** 本地方法栈
***** C/C++语言实现的native代码的执行堆栈
*** GC
**** 如何判断垃圾对象
***** 引用计数
****** 循环引用问题
***** 可达性分析
****** 引用树遍历
******* GC Root
******** 栈里的参数和局部变量 类的静态字段
**** 如何回收对象
***** 基础GC算法
****** 标记清除算法(Mark-Sweep)
******* 缺点
******** 大量内存碎片，导致后面创建的大对象无法申请连续内存
****** 拷贝算法(Copying)
******* 缺点
******** 移动对象后需要更新对象引用
******** 1、内存使用率高的对象 2、内存使用率不高，会有一半的内存空闲 3、对于存活率高的对象，会产生大量的拷贝操作
****** 标记整理(Mark-Compact)
******* 缺点
******** 前面有一块内存是垃圾对象，后面的对象都需要往前移动 存活对象较多时，移动耗时基本与内存大小成正比。
***** 分代GC
****** 新生代(Minor GC)
******* Eden
******** 新建对象的地方，经过一轮新生代GC后会被拷贝到幸存者区
******* Survivor
******** 两个幸存者区来回拷贝，若干次GC后被移入老年代
****** 老年代(Major GC)
****** 永久代
*** Hotspot GC
**** 串行收集器
***** Serial ｜ Serial Old(Mark-Compact)
****** Stop The World
**** 并行收集器
***** ParNew | Parallel Scavenge | ParOld(Mark-Compact)
****** Stop The World
**** 并发收集器
***** CMS
****** 阶段
******* 初始标记
******** 标记GC Roots能直接关联到的对象，速度很快
******* 并发标记
******** 和用户线程并发进行标记，时间长
******* 重新标记
******** 修正并发标记过程中变动的对象
******* 并发清理
******** 和用户线程并发进行清理，可能会产生浮动垃圾
****** 优点
******* 只有初始标记和重新标记阶段会StopTheWorld， 停顿时间比较短，适合响应速度有要求的应用
****** 缺点
******* 使用标记清除算法，会产生大量内存碎片， 当无法找到连续的内存时会提前出发一次FullGC
******* 无法处理浮动垃圾，可能出现浮动垃圾在未清除前又把老年代塞满了， 导致“Concurrent Mode Failure”从而触发另一次FullGC
***** G1
****** 分区
******* 
*** 分析工具
**** jdk自带
***** CLI
****** jps
******* 查看Java进程
****** jstack
******* 线程栈
****** jmap
******* 内存使用情况，内存dump信息
****** jstat
******* JVM的统计信息，内存使用情况
****** jhat
******* 内存分析工具，分析dump文件
***** GUI
****** jconsole
****** visualVM
**** 第三方
***** Alibaba arthas
***** Eclipse memory analyzer
*** 常见参数
**** -XX:+HeapDumpOnOutOfMemory
***** 内存溢出时导出堆内存信息
**** -XX:HeapDumpPath=path
***** 堆dump文件路径
**** -XX:ParallelGCThreads=threads
***** 年轻代和老年代并行GC线程数
**** -XX:ConcGCThreads=threads
***** 并发GC线程数
**** -XX:+DisableExplicitGC
***** 禁止显示调用System.gc()
**** -Xmn == -XX:NewSize
***** 年轻代堆区初始容量
**** -XX:MaxNewSize=size
***** 年轻代堆区最大容量
**** -Xms == -XX:InitialHeapSize
***** 堆区初始容量
**** -Xmx == -XX:MaxHeapSize
***** 堆区最大容量
**** -XX:MaxTenuringThreshold=threshold
***** 最大老化年龄
**** -XX:SurvivorRatio=ratio
***** eden/survivor区的比例
** 社区框架
*** Spring
**** ioc
***** 容器
****** BeanFactory
******* XmlBeanFactory / DefaultListableBeanFactory
****** ApplicationContext
******* GenericApplicationContext
***** 扩展接口
****** BeanPostProcessor (Bean创建后，对Bean进行额外处理)
******* AsyncAnnocationBeanPostProcessor
******* CommonAnnotationBeanPostProcessor
****** BeanFactoryPostProcessor (BeanFactory创建后，此时Bean还未创建)
******* ConfigurationClassPostProcessor
***** Bean生命周期
****** 实例化
******* IABPP.beforeInstantiation() -> 实例化 -> IABPP.afterInstantiation()
****** 属性注入
******* IABPP.properties() -> 注入
****** 初始化
******* Invoke XxxAware(BeanFactoryAware...)  -> BPP.beforeInitialization() -> 初始化 -> InitializingBean.afterPropertiesSet() -> init-method -> BPP.afterInitialization()
****** running
****** 销毁
******* DABPP.beforeDestruction() -> DisposiableBean.destroy() -> destroy-method
**** aop
***** 动态代理
****** JDK基于接口的动态代理
******* Proxy | InvokeHandler
****** Cglib基于类的动态代理
******* Enhancer ｜ MethodInterceptor
**** mvc
**** tx
***** 传递性
**** SpringBoot
***** Spring4.0中的@Conditional(ConditionEvaluator)
***** SpringBoot中的@EnableAutoConfiguration会从所有类路径下的jar包里的 META-INF/spring.factories中读取所有的EnableAutoConfiuration
*** data access
**** Hibernate ORM
***** 懒加载问题
****** N+1问题
******* for循环里懒加载关联对象
**** MyBatis
**** Spring Data
@endmindmap
```

| 特性         | G1 GC                            | ZGC                         | Shenandoah GC                 |
| ---------- | -------------------------------- | --------------------------- | ----------------------------- |
| **引入版本**   | JDK 7（实验），JDK 9+ 默认              | JDK 11（实验），JDK 17+ 稳定       | JDK 12+                       |
| **目标**     | 平衡吞吐与延迟                          | 极低停顿，几乎不随堆增大                | 极低停顿，减少压缩停顿                   |
| **暂停时间**   | 10ms~200ms（取决于堆和负载）              | <10ms（通常）                   | 10ms~50ms（取决于堆大小）             |
| **堆大小支持**  | 中大型（几十 GB）                       | 多 TB                        | 中大型（几十 GB）                    |
| **内存回收策略** | 分代 + 区域划分（Region）                | 并发 + 分段压缩（Colored Pointers） | 并发标记压缩（Concurrent Relocation） |
| **停顿控制**   | 可设目标停顿时间（Pause Time Goal）        | 自动控制，几乎不受堆大小影响              | 可配置停顿时间，实时压缩                  |
| **并发特性**   | 并发标记阶段                           | 大部分并发，标记/重定位几乎无停顿           | 大部分并发，压缩并发执行                  |
| **调优难度**   | 中等，常用调优参数：`-XX:MaxGCPauseMillis` | 低，需要的调优参数少                  | 中等，需要关注压缩策略                   |
| **吞吐影响**   | 中等，适合大多数应用                       | 极小，低延迟优先                    | 中等，稍微影响吞吐                     |
| **适用场景**   | 延迟敏感不极端，大多数企业应用                  | 高并发、低延迟、超大堆（TB级）            | 延迟敏感、堆大、企业服务或中大型 JVM 服务       |
| **默认启用**   | JDK 9+ 默认                        | 不默认，需要 `-XX:+UseZGC`        | 不默认，需要 `-XX:+UseShenandoahGC` |

JDK 9 起：官方标记 CMS 已废弃（deprecated） JDK 14+：正式移除 CMS

官方推荐：对大多数场景 → G1 GC； 对低延迟 / 大堆 → ZGC 或 Shenandoah
