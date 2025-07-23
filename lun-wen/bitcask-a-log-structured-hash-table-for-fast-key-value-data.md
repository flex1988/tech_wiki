# Bitcask: A Log-Structured Hash Table for Fast Key/Value Data

acBitcask 的起源和 Riak 分布式数据库的历史紧密的联系在一起。在一个 Riak的 kv 集群，每个节点都使用了可插拔的本地存储，几乎任何 kv形式的引擎都可以用做每个服务器上的存储引擎。这种可插拔的特性可以使存储引擎在不影响极限代码的情况下，可以单独的测试和优化。

许多此类本地键值存储已经存在，包括但不限于Berkeley DB、Tokyo Cabinet和Innostore。我们在评估这些存储引擎时设定了多个目标，其中包括：

* 读写的低延时
* 高吞吐，特别是写入随机的数据流时
* 能够处理远超 RAM 的数据集
* crash优化，快速恢复和不丢数据
* 易于备份和恢复
* 相对简单，可理解的代码结构和数据格式
* 在大负载和大容量时有可预期的行为
* license 允许用到 riak

满足部分条件的存储有很多，但是全部满足这些条件的存储基本没有。

现有的存储系统在满足上述目标方案均未达到理想状态，我们在与Eric Brewer讨论这个问题时，他提出了一个关键见解：哈希表日志合并，这样可以做到甚至超过 LSM 的水平。

这使我们探索了log-structured 文件系统中用的一些技术，这些探索引领了 Bitcask的开发，能够满足上述所有特点的存储系统。尽管最开始 Bitcask是用于 Riak的，但它是一个通用性的 kv存储，也可以用在其他存储上。

我们最终采用的模型在概念上非常简单。一个 Bitcask的实例就是一个目录，同一时间只允许一个进程打开一个实例写入。可以把这个进程当做数据库进程。在任意时刻，只有一个文件是 active 的，可以被数据库写。当文件大小达到一个阈值后，该文件就会被关闭，然后创建一个新active文件。一旦一个文件被 close，有可能是到了大小限制，也有可能是服务退出，但不允许再次被打开写入。

active file只支持追加的写入，这也意味着顺序写不需要磁盘寻址（指 HDD）。

<figure><img src="../.gitbook/assets/image (4).png" alt=""><figcaption><p>kv entry 格式</p></figcaption></figure>

每次写入，一个新的entry就会被追加到 active file中。可以注意到，删除也是一个特殊的标记写入，下次的 merge就可以把数据删除。所以，一个 bitcask数据文件就是一个序列的的这种 entry：

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption><p>active file 格式</p></figcaption></figure>

append完成后就会更新内存的结构 keydir。keydir就是一个哈希表，映射了key到一个定长的数据结构「文件，offset，value\_size」。

<figure><img src="../.gitbook/assets/image (1) (1) (1).png" alt=""><figcaption><p>keydir</p></figcaption></figure>

当发生写的时候，keydir就会自动更新数据的最新位置。老的数据仍然存在盘上，但新的读会读到keydir的最新版本。接下来可以看到，merge流程最终会删掉旧的数据。

读一个值是很简单的，最多只是一个 disk seek。我们在我们的keydir中查找key，然后用 keydir中的位置信息去读 key的值。在很多场景下，文件系统的 read-ahead cache会让这个操作更快。

<figure><img src="../.gitbook/assets/image (2) (1).png" alt=""><figcaption><p>读流程</p></figcaption></figure>

这个简单的模型随着时间的推移会占用大量空间，因为我们只写新值不处理旧值。我们通过一个称为merging的压缩流程来解决这个问题。merging流程遍历所有的 non-active文件，输出一组只有有效 kv 的数据文件。

merging完成后，我们还会为每个数据文件创建一个 hint file，和 data file相比，他们只有 key 的位置信息没有真正的数据。

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption><p>merge process</p></figcaption></figure>

当 Bitcask实例被一个Erlang进程打开后，它会检查是否已经有另外一个 ErLang实例打开了这个Bitcask实例。如果发现了，它会共享给进程 keydir，否则它会遍历所有的 data file构建一个新的 keydir。如果 datafile有 hintfile，那么就会加速整个启动时间。

这些基础操作都是bitcask系统的基础元素，显然，在本文中我们没有试图去穷尽所有细节，我们的目标是帮助你理解 bitcask的基本运行机制。对于前文简略提到的几个领域，或许需要补充说明：

* 提到我们依赖文件系统的 cache来提高读性能。我们也讨论过添加一个bitcask内部的读cache，但考虑到已经从现有的方案中获得了免费收益，这一优化能提升多少并不清楚
* 我们很快会发布多种类似 api 存储系统的 benchmark。然而我们最开始 bitcask的目标并不是最快的存储引擎，而是足够快，而且有高质量而且简洁的代码，设计，和文件格式。不过在我们的基准测试中，bitcask在多数场景下的表现都显著优于其他高速存储系统
* 一些最复杂的实现细节或许也最无趣，所以我们也没有在这个文档里包括这些。
* bitcask并没有做任何的数据压缩，因为这个做法的损益比明显是要看具体的应用的

然后让我们看一下最初设定的目标：

* 读写低延时\
  bitcask很快，我们还计划开发一些吞吐型的benchmark，之前的测试延时中位数基本是在几毫秒，因此我们对于性能非常有信息
* 高吞吐，特别是随机的写入数据流\
  在早期测试中，我们使用笔记本和一些很慢的磁盘，能达到 5000-6000写/s
* 能够处理远大于 RAM的数据集\
  前面提到的测试用了 10 倍 RAM的数据集，并没有发现性能有变化
* crash友好，快速恢复和不丢数据\
  数据文件和 commit log在 bitcask里是同一件事，恢复流程不需要 replay。hintfile还能加速启动进程。
* 容易备份和恢复\
  由于文件在翻转后只读，运维人员可轻松使用任何偏好的系统级机制进行备份。恢复操作仅需将数据文件置于目标目录即可完成。
* 相对简单，容易理解的代码和数据格式\
  bitcask在概念上非常简单，代码干净，数据文件非常容易理解和管理。我们感到支持 bitcask上的系统非常舒适
* 在大负载和大容量下有可预期的行为\
  在高并发访问场景下，Bitcask已展现出优异的性能表现。目前测试仅涉及数十GB量级的数据规模，但我们即将进行更大规模的验证。从架构设计来看，Bitcask在更大数据量下的性能表现预计不会出现显著差异——唯一可预见的例外是键目录(keydir)结构会随键数量增加而小幅增长，且必须完全驻留内存。这一限制在实际应用中影响甚微，因为即便处理数百万键时，当前实现的内存占用量也远低于1GB。

总结一下，考虑到预先设置的几个目标，和其他我们有的系统对比，bitcask 非常优秀的满足了我们的目标。

API也非常简单：

<table><thead><tr><th width="318.796875">API</th><th>说明</th></tr></thead><tbody><tr><td>bitcask:open(DirectoryName, Opts)→ BitCaskHandle | {error, any()}</td><td>打开一个新的，或者已经存在的bitcask 存储</td></tr><tr><td>bitcask:open(DirectoryName)→ BitCaskHandle | {error, any()}</td><td>只读打开一个新的，或者已经存在的bitcask 存储</td></tr><tr><td><p>bitcask:get(BitCaskHandle, Key)→ </p><p>not found | {ok, Value}</p></td><td>从bitcask 存储中读取一个key 的 value</td></tr><tr><td>bitcask:put(BitCaskHandle, Key, Value)→ ok | {error, any()}</td><td>在 bitcask 存储中保存key 和 value</td></tr><tr><td>bitcask:delete(BitCaskHandle, Key)→ ok | {error, any()}</td><td>在 bitcask 存储中删除一个 key</td></tr><tr><td>bitcask:list keys(BitCaskHandle)→ [Key] | {error, any()}</td><td>列出bitcask 存储中所有的key</td></tr><tr><td>bitcask:fold(BitCaskHandle,Fun,Acc0)→ Acc</td><td>遍历bitcask 存储中所有的键值对，Fun 的形式是: F(K,V,Acc0) → Acc.</td></tr><tr><td>bitcask:merge(DirectoryName)→ ok | {error, any()}</td><td>合并bitcask 存储中的几个数据文件到一个更压缩的形式，而且为每个 datafile 生成一个hintfile 用于快速启动。</td></tr><tr><td>bitcask:sync(BitCaskHandle)→ ok</td><td>把写刷到盘上去</td></tr><tr><td>bitcask:close(BitCaskHandle)→ ok</td><td>关闭bitcask 实例并把pending 的写入刷到盘上</td></tr></tbody></table>
