zookeeper程序员引导文档 使用zookeepr开发分布式应用
===========================
由于对zookeeper比较感兴趣，所以想出了翻译官网引导文档

官方原文地址：http://zookeeper.apache.org/doc/trunk/zookeeperProgrammers.html#ch_zkDataModel
****

###　　　　　　　　　　　　译:菜鸟楠
###　　　　　　　　　 E-mail:502028249@qq.com

### 介绍

这篇文档是给希望使用zookeeper协调服务创建分布式应用的引导。这篇文档包括一些概念和实用的信息。

这里有四个章节高度概括的了zookeeper的各种概念。知道zookeeper的工作原理和如何使用zookeeper工作是非常有必要的。这里不包含源码的介绍，但是假设你熟悉分布式计算
相关的问题。第一组内容包含这些部分内容：

    * zookeeper数据模型。
    * ZooKeeper Sessions。
    * ZooKeeper Watches。
    * 一致性保证。

接下来4部分包含实用编程信息。他们是：

    * zookeeper操作引导。
    * Bindings。
    * 程序结构和一些例子。
    * 常见的一些问题和故障排除。

本书的附录包含一些其他有用的链接(zookeeper相关的信息)。这篇文档的大部分内容可作为独立的参考资料。然而在开始你第一个zookeeper的应用之前你至少需要阅读ZooKeeper数据模型
和ZooKeeper基本操作这两个章节。同样的，简单的程序示例这章节对你理解zookeeper client应用基本结构也是非常有帮助的。

### ZooKeeper的数据模型

zookeeper有一个命名空间体系，非常像文件系统。唯一不同的就是每个在命名空间中的node可以包含数据和children。像文件系统一个文件也可是一个目录。到达node的path经常是规范的，绝对的，
分离的paths，没有相对路径的参考。任何unicode字符可以使用在path，不过有以下限制：

    * null字符不能作为path名字(因为在C语言绑定的时候有问题)。
    * 接下来的这些字符也不能使用，因为他们不能很好的展示或者展示混乱(\u0001 - \u001F and \u007F - \u009F)。
    * \ud800 - uF8FF, \uFFF0 - uFFFF字符也是不允许使用的。
    * "."只能作为名字的一部分，但是"."和".."不能独立使用作为node的path，因为zookeeper不是使用相对路径。
    像"/a/b/./c"或者"/a/b/../c"这样的路径是无效的。

### ZNodes

在zookeeper目录树中的每个node节点叫做znode。znodes维护一个统计结构里面包含：数据更改或者acl更改的版本号。这个统计结构也包含时间戳。版本号和时间戳能验证cache并且能协调更新。每次
znode的数据发生改变版本号就增加。例如：client无论何时获取数据，他们也获取数据的版本号。并且当一个client执行update或者delete操作的时候，改变数据的znode的节点必须提供修改数据的版本号。
如果提供的版本号不匹配数据的真实版本号操作将失败(这个行为是可以被覆盖的，详情见补充)。

注意：

    * 在分布式应用中，node数据通用主机，一个server，客户端进程等等。在zookeeper文档中，znodes就是指数据节点。
    servers就是指启动ZooKeeper service的机器。quorum peers就是指构成整体的servers。

znodes就是程序员访问的主要实例。这里有几个值得一提的特征。

##### Watches

client可以在znodes上放置watches。znode上的改变将触发watch然后清除watch。当一个watch被触发了，zookeeper向client发送一个通知。ZooKeeper Watches将介绍更多关于watches的信息。

##### Data Access(数据访问)

在命名空间中的每个znode节点的数据read或者write操作都是原子的。read操作获取一个node上所有关联的bytes数据，write替换所有的数据。每个node都有acl约束谁可以做什么。
zookeeper不是被设计成为一个一般的数据库或者大对象存储的东西。相反的它是管理和协调数据的。数据可以来源于配置，状态信息，rendezvous等等。各式各样的相对较小的协调数据：
在千字节之内的。zookeeper client和server实现由智能检测确保znode只能有小于1M的数据，但数据应该远远小于平均值。操作相对较大的数据将会导致一些操作时间较其他的操作长
很多，并且会导致操作延时。因为需要更多额外的时间花费在网络数据传输和媒体存储。如果真的有大数据需要存储，最常用的手段就是将这些数据放在分布式存储上比如：NFS,HDFS，并且
在zookeeper上存储数据的存储引用。

##### Ephemeral Nodes(临时节点)

zookeeper也有临时节点的概念。znode存在的时间和创建znode的session的活跃时间一样长。当session结束了znode会被删除。因为临时节点是不允许有children的。

##### Sequence Nodes -- Unique Naming(有序节点，唯一命名)

当你创建一个znode的时候也可以请求zookeeper追加一个单调递增的值到path的末尾。这个值在父path下是唯一的。这个值格式是%010d样的--由10个0填充(这个值得格式是简化排序的一种方式)。就是这样的
"<path>0000000001"。这里有使用这一特性的例子[Queue Recipe ](http://zookeeper.apache.org/doc/trunk/recipes.html#sc_recipes_Queues "Queue Recipe ")。注意：当
增加的数值超过2147483647数字的时候，这个counter将会溢出的(得到的结果就是"<path>-2147483647")。

### Time in ZooKeeper

zookeeper跟踪时间的多种方式：

    * zxid - zookeeper状态每次改变都从zxid(zookeeper事物ID)中获取时间戳。zxid展示了zookeeper所有改变是有序的。
    每次改变都会有一个唯一的zxid并且如果zxid1小于zxid2那就表示zxid1发生在zxid2的前面。
    * Version numbers - 一个node上的每次改变都是产生一个自增的版本号。这三种版本号是：version(这个znode上的数据change number)，
    cversion(这个znode上的children的change number)，aversion(这个znode上acl的change number)。
    * Ticks - 当使用多服务器zookeeper的时候，servers使用ticks定义事件的时间比如：状态上传，session超时，链接超时等等。tick时间
    只是间接暴露了最小的session超时时间(两次tick时间)。如果client请求session超时间小于最小的session超时时间，server将告诉
    client session 超时时间是最小的session超时时间。
    * Real time - zookeeper不使用真实时间或者时钟。除非在znode创建和修改znode的时候，设置时间戳到stat structure(统计结构)。

### ZooKeeper Stat Structure(zookeeper统计结构)
