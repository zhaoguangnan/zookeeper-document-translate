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

