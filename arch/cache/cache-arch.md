

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
Traffic Server将一个`Cache带`代表的存储区域看做一个一致性的字节集合，而在内部会对每一个`Cache带`进行独立地对待。这一节描述的数据结构对于每一个`Cache带`都是一样的。在代码中会用Vol类来代表`Cache带`, 用CacheVol来代表`Cache分卷`，一个`Cache分卷`由位于所有不同设备上的`Cache带`组成。

    在对一个对象进行操作之前必须先明确这个对象所在的`Cache带`, 因为每个`Cache带`分别拥有着独立的索引空间。
    如果缓存对象所在的`Cache带`发生变化的话那么这个缓存对象也将失效, 因为在新的`Cache带`中并没有这个缓存对象的索引。


#### Cache索引
`Cache带`中存储的内容是通过索引来进行定位的，索引中的每一个元素我们称之为目录项，代码中使用`Dir`来表示。每一个目录项代表了cache中一段连续的存储空间。这里会涉及到各种概念包括分片、分段、文档等。本文会使用"分片"这个术语，这也是代码中最常用到的概念。"文档"这个术语用来表示一个分片的头部数据。目录项被视为通过Cache ID做为key而计算得到的哈希值。查看索引探测这一节可以看到如何通过Cache ID来定位一个目录项。默认情况下Cache ID是通过对象的URL来计算得到。

索引数据会被持久化在内存中，这也意味着目录项必须足够的小(目前只有10个字节)，这也将导致可存储的信息不够多。从另外一个方面来考虑，绝大多数的cache miss情况是不需要任何的磁盘I/O操作的，这是一个很大的性能收益。

此外，当一个`Cache带`初始化之后，那么它对应的索引空间大小也就明确下来，而且不会再改变。索引空间的大小和`Cache带`的大小相关(线性)，因此Traffic Server的内存占用也会和磁盘大小相关。由于索引空间的总大小是不变的，也就意味着占用的内存大小也是固定的，因此当Traffic Server在cache中存储更多的对象的时候并不会消耗额外的内存。如果有足够的内存保证Traffic Server在空白存储的情况下正常运行，那么在cache存满的情况下仍然可以正常运行。

![Dir](https://docs.trafficserver.apache.org/en/latest/_images/cache-directory-structure.png)

每一个目录项中都保存了在`Cache带`中的偏移量和大小，目录项中存储的大小是一个能够包含分片中实际数据大小的粗略值，实际的大小保存在分片的头部区域中(在磁盘上)。

    只有通过读取磁盘才能得到HTTP头部保存的数据，这部分数据中保存着对象的原始URL。

索引空间是通过哈希表来组织的，以链表的方式来解决冲突。由于每个目录项很小，因此目录项会被直接作为哈希桶的链表头。

通过对索引中的所有项进行分组的方式来实现链表，第一层的分组就是索引桶，包含了固定数目(目前是4)的目录项。每个索引桶中的第一个目录项将作为这个哈希桶的根。

    The term “bucket” is used in the code to mean both the conceptual bucket for hashing and for a 
    structural grouping mechanism in the directory and so these will be qualified as needed to 
    distinguish them. The unqualified term “bucket” is almost always used to mean the structural 
    grouping in the directory.


多个索引桶会集成为段，一个`Cache带`中的所有段拥有相同数目的索引桶。在计算一个`Cache带`有多少个段时，要保证每个段要拥有尽可能多的索引桶数目，同时要保证一个段拥有的目录项个数不能超过65535。

![segment](https://docs.trafficserver.apache.org/en/latest/_images/dir-segment-bucket.png)

同一个段中的每个目录项以链表的形式组织起来，目录项会体现出向前和向后的索引。由于一个段中的目录项不会超过65535，因此16位足以表示出索引值。在`Cache带`的头部会保存一个目录项数组，数组的每一项是对应段中的空闲目录项的链表头。Active entries are stored via the bucket structure. 当一个`Cache带`初始化的时候, 每个目录桶中的第一个目录项会被清零(标示为未使用)，而所有的其他项会被放入对应的段空闲链表中。这就意味着每个目录桶中的第一项会被当做哈希桶的第一项，它们不会被放入空闲列表中，而是会被清零。目录桶中的其他项会被优先选择进入对应的哈希桶中，但不是强制的。每个段中的空闲目录项链表在初始化的时候会让每个目录桶中的其他项顺序的添加进来，先是每个目录桶中的第二项，然后是第三项、第四项。由于空闲链表采用的是先进先出的策略，所以在选择的时候会先选择所有目录痛的第四项，然后才是第三项，以此类推。当需要从一个目录桶中分配出一个新的目录项时，会从第一项到最后一项顺序查找，这样可以让目录桶中的目录项尽可能的本地化(通过Cache ID计算得到的哈希桶会尽量选择同一个目录桶中的目录项)。

    目录桶是存储格式上的划分，每个桶中有4项；而哈希桶则是查找时使用的数据结构，每个哈希桶中的所有冲突项以链表的形式组织。
    计算时哈希桶和目录桶是一一对应的，但是哈希桶中的冲突项可能会多于4，因此这个时候会将其他目录桶中的空现项拿来用连到本哈希桶的链表中。
    哈希桶在组织链表时会优先选择本哈希桶对应的目录桶中的这4个目录项，然后才会去使用其他目录桶中的空闲项。

![hash](https://docs.trafficserver.apache.org/en/latest/_images/dir-bucket-assign.png)

一个在使用中的目录项会从空闲链表中移除，当这个目录项不在使用时会重新回到空闲链表。当需要把一个分片设置对应的目录项时，会通过cache ID来定位所在的哈希桶(也会拿来定位所在的段和目录桶)，如果对应的目录桶中的第一项并未使用，那么会直接拿来给这个分片使用，否则会查看一下这个目录桶中的其他项，如果有空闲则会拿来使用。如果还是找不到空闲项，将会会使用空闲链表中的第一项，这一项会被链到哈希桶的冲突链表中，以确保可以通过cache ID来查找到。


#### 存储布局
存储布局指的是`Cache带`的元信息，由三部分组成 - 头部、索引数据、尾部。`Cache带`的元信息存储了两份，头部和尾部在代码中使用的是相同的数据结构VolHeaderFooter，这个数据结构的尾部包含一个变化长度的数组，这个数组用来保存每个段的空闲目录链表的表头，每一项包含对应段中空闲链表的第一项的索引，尾部其实是头部的拷贝，但不包含每个段的空闲链表数组。因此头部的大小会受目录项大小影响，但是尾部不会。

![layout](https://docs.trafficserver.apache.org/en/latest/_images/cache-stripe-layout.png)

每一个`Cache带`包含以下几个能够描述基本布局的变量：

**skip**

`Cache带`数据的开始位置，物理磁盘上最开始的一段数据会被保留下来，这样可以避免对操作系统造成一些干扰，这个值也代表了其他的`Cache带`在`Cache设备`上的偏移量。

**start**

`Cache带`元信息之后的数据区在磁盘上的偏移量。

**length**

`Cache`带的字节大小，Vol::len。

**data length**

`Cache带`上内容区的总块数(512字节为一块)，Vol::data_blocks。

    这里必须要留意代码中提到的长度和大小这些词，因为在不同的地方会分别用刀三种不同的单位(字节，cache块，存储块)。

索引区的总大小(目录项的个数)的计算方法是用`Cache带`的大小除以平均对象大小来得到，索引区会消耗等量大小的内存，如果cache存储变大那么Traffic Server消耗的内存也就越多，平均对象大小默认是8000字节，可以通过`proxy.config.cache.min_average_object_size`来配置。增加平均对象大小会减少索引区对内存的占用，同时也意味着cache中能够存储的不同对象的数量也会减少。


磁盘上的内容区域保存真正的对象数据，内容区域会被当做一个环形的缓冲区，新对象会覆盖掉最早cache下来的对象。新的缓存对象在`Cache带`中写到的位置被称为写光标，这意味着写光标到达的区域所保存的原来的对象将会被淘汰，即便这个对象还没有过期。当一个在磁盘上的对象被覆盖时，并不会立即的检测到因为索引并没有做更新，而是在后面读取对象分片的时候才会检测到失败。

![cursor](https://docs.trafficserver.apache.org/en/latest/_images/ats-cache-write-cursor.png)

    磁盘上的cache数据从来不会更新

这是一个需要特别注意的事情，更新操作(对于收到304响应对过期对象进行刷新)实际上就是将对象的新拷贝写到写光标的位置。The originals are left as “dead” space which will be consumed when the write cursor arrives at that disk location.当`Cache带`中的索引发生更新操作时(内存中)，cache上的原有分片数据将失效，这也是比较常见的存储管理手段。当需要从cache中删除一个对象时，只需要更新一下索引即可，不需要其他的操作，特别是不需要任何的I/O操作。

#### 对象数据结构
每个对象会存储两种类型的数据，元数据和内容数据。元数据包括对象的HTTP header和描述信息，而内容数据包含对象的真正内容，发送给客户端的字节流。

cache中的对象用Doc这个数据结构来表示，Doc可以认为是分片的头部数据，而且会存储在每个分片的开始位置(对象的每个分片都是一个Doc)。对象的第一个分片被称为`first Doc`并且会保存有对象的元数据，**任何对一个对象的操作都需要先读取这第一个分片**。分片的定位方法是将对象的cache key转换为cacheID然后通过这个cacheID来查找对象的目录项，目录项中保存了对象第一个分片在磁盘上的偏移量和大小，然后就会从磁盘上读取出来。对象的地一个分片会包含对象的请求头和响应头以及对象的所有描述属性(比如content length)。

Traffic Server支持对象内容多样化，也称之为多副本。所有副本的全部元信息都会保存在对象的第一个分片中，包括每个副本的HTTP header信息。因此当从磁盘上读取出对象的`first Doc`之后就可以做副本选择。**如果一个对象拥有多个副本，那么每个副本会独立地分别存在其他分片中。如果对象只有一个副本，那么对象的内容有可能和元信息同时存在第一个分片中。每个副本的内容都会对应一个目录项，而每个副本目录项的查找key都会保存在第一片中的元信息中**

在4.0.1版本之前，header数据会保存在CacheHTTPInfoVector这个结构中，这个结构会被序列化之后存在磁盘上，在这个结构的后面会保存和对象其他片相关的一些附加信息。

![CacheHTTPInfoVector](https://docs.trafficserver.apache.org/en/latest/_images/cache-doc-layout-3-2-0.png)

这样存在一个问题，如果一个对象有多个副本，那么只有一个分片table是不够的。因此在元数据中不在单独保存分片信息，而是将分片信息合并到CacheHTTPInfoVector结构中，这样就产生了下面的格式：

![CacheHTTPInfoVector-4.0.1](https://docs.trafficserver.apache.org/en/latest/_images/cache-doc-layout-4-0-1.png)

向量中的每一个元素代表了一个副本，包含的信息有HTTP header、分片表和一个cache key，这个cache key对应的目录项用来定位对象的earliest doc`，这也是该副本的开始分片的索引。

当一个对象最开始被缓存的时候，它只会有一个副本，因此内容也会同时保存在`first Doc`中(如果内容不大的话)，在代码中称之为常驻副本，这只会在对象最初被保存的时候出现。如果元数据发生改变(比如发送If-Modified-Since请求之后接收到了304响应)，那么对象的内容数据会被保留在原始分片中，但是会用新的分片来保存对象的`first Doc`(对象内容过小的情况除外)，这样对象不会再有常驻副本，这里提到的过小是要小于配置中指定的`proxy.config.cache.alt_rewrite_max_size`的值。

    CacheHTTPInfoVector只会保存在`first Doc`中，包括`earliest Doc`在内的其他Doc中的hlen的值应该是0，否则会被忽略。

大对象会被切分成多个分片在cache中存储下来，
