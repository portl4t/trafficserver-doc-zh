

## Cache 架构

### 简介
Apache Traffic Server不仅是HTTP代理服务器, 还是HTTP缓存服务器。Traffic Server可以缓存任何字节流数据, 不过目前仅支持通过HTTP协议来传输这些字节流数据。当这样的一个字节流被缓存住时(还有相关的HTTP协议头)我们称之为缓存中的一个对象。每个对象可以通过全局唯一的缓存key的来定位。

这篇文档旨在对Traffic Server缓存系统的基本框架和实现细节进行一下描述。为了帮助理解cache系统的内部机制, 我们也会对cache系统的配置做一些简单的讨论。这篇文当对从事TrafficServer核心开发以及插件开发的人员都会很有帮助。这里假定读者已经熟悉了管理员指南这部分内容，特别是Http反向代理缓存、缓存配置以及相关的配置文件和具体的取值。

遗憾的是内部的一些术语并不是特别一致, 因此为了试图创造一些一致性，这篇文档会经常以不同的形式来使用这些术语。

### Cache的布局
接下来的章节会介绍持久化下来的cache数据是如何组织的。Traffic Server将其持久化存储设备看成常规字节的集合, 假定存储设备上没有其他结构。而且Traffic Server不会使用操作系统上面的文件系统功能, 一个文件仅仅是用来标识出一个字节集合。

#### Cache存储
Traffic Server使用的裸存储设备定义在配置文件storage.config中。文件中的每一行定义了一个具有一致性存储特征的`cache设备`。

![Two cache spans](https://docs.trafficserver.apache.org/en/latest/_images/cache-spans.png)

Traffic Server的管理员可以根据实际情况将存储空间组织成一系列分卷，这些分卷定义在volume.config配置文件中, `cache分卷`是管理存储配置的基本单位。

`Cache分卷`的容量可以通过存储百分比来定义, 也可以定义为一个绝对的数值。默认情况下，每一个`cache分卷`定义的存储空间都会分散到所有的`cache设备`中，这是出于健壮性的考虑(一个盘有问题不会影响这个`cache分卷`在其他盘上的存储空间)。`cache分卷`和`cache设备`的交集是`cache带`，每个`cache设备`会被切分成若干个`cache带`, 而每个`cache分卷`是由一系列来自不同的`cache设备`中的`cache带`组成。

如果`Cache分卷`按下面这样定义:

![volumes](https://docs.trafficserver.apache.org/en/latest/_images/ats-cache-volume-definition.png)

那么对于前面定义的`Cache设备`的实际布局将会如下所示:

![layout](https://docs.trafficserver.apache.org/en/latest/_images/cache-span-layout.png)

`Cache带`是cache设计实现过程中的最基本的单位。一个缓存对象会被完整的存储在单一的`Cache带`中，因此也就存储在单一的`Cache设备中`，对象从不跨`Cache设备`或跨`Cache分卷`存储。每一个对象会通过回源的URL来计算一个hash值，然后得到对应的`Cache分卷`。可以通过配置hosting.config文件来指定那些域名的数据存储在哪些`Cache分卷`中。此外，从4.0.1版本开始可以指定一个`Cache分卷`包含哪些`Cache设备`(也就指定了`Cache带`)。

traffic_server进程启动的时候会根据storage.config和cache.config配置文件来计算`Cache设备`, `Cache分卷`, `Cache带`的布局和结构，因此对这些文件的修改会导致对原有cache数据的全部重新校验。

#### Cache带数据结构
Traffic Server将一个`Cache带`代表的存储区域看做一个一致性的字节集合，而在内部会对每一个`Cache带`进行独立地对待。这一节描述的数据结构对于每一个`Cache带`都是一样的。在代码中会用Vol类来代表`Cache带`, 用CacheVol来代表`Cache分卷`。

    在对一个对象进行操作之前必须先明确这个对象所在的`Cache带`, 因为每个`Cache带`分别拥有着独立的索引空间。
    如果缓存对象所在的`Cache带`发生变化的话那么这个缓存对象也将失效, 因为在新的`Cache带`中并没有这个缓存对象的索引。


#### Cache索引
`Cache带`中存储的内容是通过索引来进行定位的，索引中的每一个元素我们称之为索引项，代码中使用`Dir`来表示。每一个索引项代表了cache中一段连续的存储空间。这里会涉及到各种概念包括分片、分段、文档等。本文会使用"分片"这个术语，这也是代码中最常用到的概念。"文档"这个术语用来表示一个分片的头部数据。索引项被视为通过Cache ID做为key而计算得到的哈希值。查看索引探测这一节可以看到如何通过Cache ID来定位一个索引项。默认情况下Cache ID是通过对象的URL来计算得到。

索引数据会被持久化在内存中，这也意味着索引项必须足够的小(目前只有10个字节)，这也将导致可存储的信息不够多。从另外一个方面来考虑，绝大多数的cache miss情况是不需要任何的磁盘I/O操作的，这是一个很大的性能收益。

此外，当一个`Cache带`初始化之后，那么它对应的索引空间大小也就明确下来，而且不会再改变。索引空间的大小和`Cache带`的大小相关(线性)，因此Traffic Server的内存占用也会和磁盘大小相关。由于索引空间的总大小是不变的，也就意味着占用的内存大小也是固定的，因此当Traffic Server在cache中存储更多的对象的时候并不会消耗额外的内存。如果有足够的内存保证Traffic Server在空白存储的情况下正常运行，那么在cache存满的情况下仍然可以正常运行。

![Dir](https://docs.trafficserver.apache.org/en/latest/_images/cache-directory-structure.png)

每一个索引项中都保存了在`Cache带`中的偏移量和大小，索引项中存储的大小是一个能够包含分片中实际数据大小的粗略值，实际的大小保存在分片的头部区域中(在磁盘上)。

    只有通过读取磁盘才能得到HTTP头部保存的数据，这部分数据中保存着对象的原始URL。

索引空间是通过哈希表来组织的，以链表的方式来解决冲突。由于每个索引项很小，因此索引项会被直接作为哈希桶的链表头。

