# The LRU-K Page Replacement Algorithm For Database Disk Buffering

> Elizabeth J. O'Neil, Patrick E. O'Neil, Gerhard WWeikum

# 摘要 (ABSTRACT)

本文介绍了一种新的数据库磁盘缓冲方法，称为 LRU-K 方法。LRU-K 的基本思想是跟踪最近 K 次对热门数据库页面的引用时间，并利用此信息逐页统计估计引用的到达间隔时间。尽管 LRU-K 方法在相对标准的假设下执行最佳统计推断，但它相当简单，并且几乎不产生簿记开销。正如我们通过模拟实验所证明的那样，LRU-K 算法在区分频繁和不频繁引用的页面方面优于传统的缓冲算法。事实上，LRU-K 是一种缓冲算法的行为方法，其中具有已知访问频率的页面集被手动分配到具有特定调整大小的不同缓冲池。然而，与此类定制缓冲算法不同，LRU-K 方法是自调整的，不依赖于有关工作负载特征的外部提示。此外，LRU-K 算法可以实时适应不断变化的访问模式。

> This paper introduces a new approach to database disk buffering, called the LRU-K method. The basic idea of LRU-K is to keep track of the times of the last K references to popular database pages, using this information to statistically estimate the interarrival times of references on a page by page basis. Although the LRU-K approach performs optimal statistical inference under relatively standard assumptions, it is fairly simple and incurs little bookkeeping overhead. As we demonstrate with simulation experiments, the LRU-K algorithm surpasses conventional buffering algorithms in discriminating between frequently and infrequently referenced pages. In fact, LRU-K an approach the behavior of buffering algorithms in which page sets with known access frequencies are manually assigned to different buffer pools of specifically tuned sizes. Unlike such customized buffering algorithms however, the LRU-K method is self-tuning, and does not rely on external hints about workload characteristics. Furthermore, the LRU-K algorithm adapts in real time to changing patterns of access.

# 1. 介绍 (Introduction)

## 1.1 问题陈述 (Problem Statement)

所有数据库系统在磁盘页面从磁盘读入并被特定应用程序访问后，都会在内存缓冲区中保留一段时间。目的是让常用页面驻留在内存中并减少磁盘 I/O。在 Gray 和 Putzolu 的“五分钟规则”中，他们提出了以下权衡：我们愿意在一定范围内为内存缓冲区支付更多费用，以降低系统的磁盘臂成本（[GRAYPUT]，另请参阅 [CKS]）。当需要为即将从磁盘读入的页面添加新的缓冲区槽，并且所有当前缓冲区都在使用中时，就会出现关键的缓冲决策：应该从缓冲区中删除哪个当前页面？这称为页面替换策略，不同的缓冲算法根据它们实施的替换策略类型而得名（例如，请参阅 [COFFDENN]、[EFFEHAER]）。

> All database systems retain disk pages in memory buffers for a period of time after they have been read in from disk and accessed by a particular application. The purpose is to keep popular pages memory resident and reduce disk I/O. In their "Five Minute Rule", Gray and Putzolu pose the following tradeoff: We are willing to pay more for memory buffers up to a certain point, in order to reduce the cost of disk arms for a system ([GRAYPUT], see also [CKS]). The critical buffering decision arises when a new buffer slot is needed for a page about to be read in from disk, and all current buffers are in use: What current page should be dropped from buffer? This is known as the page replacement policy, and the different buffering algorithms take their names from the type of replacement policy they impose (see, for example, [COFFDENN], [EFFEHAER]).

几乎所有商业系统都采用的算法称为 LRU，即最近最少使用算法。当需要新的缓冲区时，LRU 策略会从缓冲区中删除最长时间未访问的页面。LRU 缓冲最初是为指令逻辑中的使用模式而开发的（例如，[DENNING]、[COFFDENN]），并不总能很好地适应数据库环境，正如 [FEITER]、[STON]、[SACSCH] 和 [CHOUDEW] 中所述。事实上，LRU 缓冲算法存在一个问题，本文将对此进行解决：它根据太少的信息来决定从缓冲区中删除哪些页面，将自己限制在最后引用的时间。具体而言，LRU 无法区分引用相对频繁的页面和引用非常不频繁的页面，除非系统浪费了大量资源将不频繁引用的页面长时间保留在缓冲区中。

> The algorithm utilized by almost all commercial systems is known as LRU, for Least Recently Used. When a new buffer is needed, the LRU policy drops the page from buffer that has not been accessed for the longest time. LRU buffering was developed originally for patterns of use in instruction logic (for example, [DENNING], [COFFDENN]), and does not always fit well into the database environment, as was noted also in [FEITER], [STON], [SACSCH], and [CHOUDEW]. In fact, the LRU buffering algorithm has a problem which is addressed by the current paper: that it decides what page to drop from buffer based on too little information, limiting itself to only the time of last reference. Specifically, LRU is unable to differentiate between pages that have relatively frequent references and pages that have very infrequent references until the system has wasted a lot of resourees keeping infrequently referenced pages in buffer for an extended period.

**例 1.1.**  考虑一个多用户数据库应用程序，它通过聚类 B 树索引键 CUST-ID 引用随机选择的客户记录以检索所需信息（参见 [TPC-A]）。简单假设存在 20,000 个客户，客户记录长度为 2000 字节，并且叶级 B 树索引所需的空间（包括可用空间）为每个键条目 20 字节。然后，如果磁盘页面包含 4000 字节的可用空间并且可以装满，我们需要 100 个页面来保存 B 树索引的叶级节点（有一个 B 树根节点），以及 10,000 个页面来保存记录。引用这些页面的模式（忽略 B 树根节点）显然是：I1、R1、I2、R2、I3、R3、...，交替引用随机索引叶页面和记录页面。如果我们只能为此应用程序在内存中缓冲 101 个页面，则 B 树根节点是自动的；我们应该缓冲所有 B 树叶页，因为每个页面被引用的概率为 0.005（每 200 次一般页面引用一次），而用数据页取代其中一个树叶页显然是浪费，因为数据页被引用的概率只有 0.00005（每 20,000 次一般页面引用一次）。但是，使用 LRU 算法，内存缓冲区中保存的页面将是最近引用的 100 个页面。初步估计，这意味着 50 个 B 树叶页和 50 个记录页。考虑到页面在最近被引用两次不会获得额外积分，并且这种情况更有可能发生在 B 树叶页上，内存中存在的数据页甚至会比树叶页略多。对于非常常见的磁盘访问范例，这显然是不合适的行为。

> **Example 1.1.**  Consider a multi-user database application, which references randomly chosen customer records through a clustered B-tree indexed key, CUST-ID, to retrieve desired information (cf. [TPC-A]). Assume simplistically that 20,000 customers exist, that a customer record is 2000 bytes in length, and that space needed for the B-tree index at the leaf level, free space included, is 20 bytes for each key entry. Then if disk pages contain 4000 bytes of usable space and can be packed full, we require 100 pages to hold the leaf level nodes of the B-tree index (there is a single B-tree root node), and 10,000 pages to hold the records. The pattern of reference to these pages (ignoring the B-tree root node) is clearly: I1, Rl, I2, R2, I3, R3, ..., alternate references to random index leaf pages and record pages. If we can only afford to buffer 101 pages in memory for this application, the B-tree root node is automatic; we should buffer all the B-tree leaf pages, since each of them is referenced with a probability of .005 (once in each 200 general page references), while it is clearly wasteful to displace one of these leaf pages with a data page, since data pages have only .00005 probability of reference (once in each 20,000 general page references). Using the LRU algorithm, however, the pages held in memory buffers will be the hundred most recently referenced ones. To a first approximation, this means 50 B-tree leaf pages and 50 record pages. Given that a page gets no extra credit for being referenced twice in the recent past and that this is more likely to happen with B-tree leaf pages, there will even be slightly more data pages present in memory than leaf pages. This is clearly inappropriate behavior for a very common paradigm of disk access.

**例 1.2.**  作为 LRU 将不适当的页面保留在缓存中的第二种情况，请考虑具有良好共享页面引用“局部性”的多进程数据库应用程序，这样 100 万个磁盘页面中的 5000 个缓冲页面将获得并发进程的 95% 的引用。现在，如果一些批处理进程开始对数据库的所有页面进行“顺序扫描”，则顺序扫描读入的页面将用不太可能再次引用的页面替换缓冲区中经常引用的页面。这是许多商业情况下的常见抱怨：顺序扫描导致的缓存淹没导致交互响应时间明显恶化。响应时间恶化是因为顺序扫描读入的页面使用磁盘臂替换通常保存在缓冲区中的页面，导致通常驻留的页面的 I/O 增加，从而积累了较长的 I/O 队列。

> **Example 1.2.**  As a second scenario where LRU retains inappropriate pages in cache, consider a multi-process database application with good "locality" of shared page reference, so that 5000 buffered pages out of 1 million disk pages get 95% of the references by concurrent processes. Now if a few batch processes begin "sequential scans" through all pages of the database, the pages read in by the sequential scans will replace commonly referenced pages in buffer with pages unlikely to be referenced again. This is a common complaint in many commercial situations: that cache swamping by sequential scans causes interactive response time to deteriorate noticeably. Response time dete riorates because the pages read in by sequential scans use disk arms to replace pages usuaily held in buffer, leading to increased I/O for pages that usually remain resident, so that long I/O queues build up.

重申一下我们在这两个例子中看到的问题，LRU 无法区分被引用频率相对较高的页面和被引用频率很低的页面。一旦页面从磁盘读入，LRU 算法就会保证其缓冲区寿命较长，即使该页面之前从未被引用过。文献中已经提出了解决此问题的方法。先前的方法主要分为以下两类。

> To reiterate the problem we see in these two examples, LRU is unable to differentiate between pages that have relatively frequent reference and pages that have very infrequent reference. Once a page has been read in from disk, the LRU algorithm guarantees it a long buffer life, even if the page has never been referenced before. Solutions to this problem have been suggested in the literature. The previous approaches fail into the following two major categories.

- *页池调优：*

  Reiter 在他的域分离算法 [RETIER] 中提出，DBA 应该更好地提示正在访问的页面池，将它们从本质上划分为不同的缓冲池。因此，B 树节点页面将只与其他节点页面竞争缓冲区，数据页面将只与其他数据页面竞争，并且如果重新引用的可能性不大，DBA 可以限制可用于数据页面的缓冲区空间量。一些商业数据库系统支持此类“池调整”功能，并用于具有高性能要求的应用程序（例如，参见 [TENGGUM、DANTOWS、SHASHA]）。这种方法的问题在于，它需要大量的人力，并且不能正确处理热点访问模式的演变问题（数据页面池内的局部性，随时间而变化）。

> - *Page Pool Tuning:*
>
>   Reiter, in his Domain Separation algorithm [RETIER], proposed that the DBA give better hints about page pools being accessed, separating them essentially into different buffer pools. Thus B-tree node pages would compete only against other node pages for buffers, data pages would compete only against other data pages, and the DBA could limit the amount of buffer space available for data pages if re-reference seemed unlikely. Such "pool tuning" capabilities are supported by some commercial database systems and are used in applications with high performance requirements (see, e.g., [TENGGUM, DANTOWS, SHASHA]). The problem with this approach is that it requires a great deal of human effort, and does not properly handle the problem of evolving patterns of hot-spot access (locality within the data page pool, changing over time).

- *查询执行计划分析：*

  另一个建议是，查询优化器应提供更多关于查询执行计划所设想的使用类型的信息，以便系统知道是否可能被计划重新引用并采取相应行动（参见 [SACSCH] 的热集模型、[CHOUDEW] 的 DBMIN 算法）及其扩展 [FNS、NFS、YUCORN]、[CHAKA、HAAS、ABG、JCL、COL] 的提示传递方法和 [PAZDO] 的预测方法）。这种方法在由同一计划重新引用是缓冲的主要因素的情况下可以很好地发挥作用。

  在上面的示例 1.2 中，我们大概知道足够多的信息来删除通过顺序扫描读入的页面。如果整个引用字符串由单个查询生成，DBMIN 算法也可以很好地处理示例 1.1 中的引用。但是，示例 1.1 中简单多用户事务的查询计划并不优先考虑在缓冲区中保留 B 树页面或数据页面，因为每个页面在计划期间只引用一次。在多用户情况下，查询优化器计划可能会以复杂的方式重叠，而查询优化器咨询算法不会告诉我们如何考虑这种重叠。必须存在更全局的页面替换策略才能做出这样的决定。

> - *Query Execution Plan Analysis:*
>
>   Another suggestion was that the query optimizer should provide more information about the type of use envisioned by a query execution plan, so that the system will know if re-reference by the pla n is likely and can act accordingly (see the Hot Set Model of [SACSCH], the DBMIN algorithm of [CHOUDEW]) and its extensions [FNS, NFS, YUCORN], the hint-passing approaches of [CHAKA, HAAS, ABG, JCL, COL] and the predictive approach of [PAZDO]). This approach can work well in circumstances where re-reference by the same plan is the main factor in buffering.
>
>   In Example 1.2 above, we would presumably know enough to drop pages read in by sequential scans. The DBMIN algorithm would also deal well with the references of Example 1.1 if the entire reference string were produced by a single query. However, the query plans for the simple multi-user transactions of Example 1.1 give no preference to retaining B-tree pages or data pages in buffer, since each page is referenced exactly once during the plan. In multi-user situations, query optimizer plans can overlap in complicated ways, and the query optimizer advisory algorithms do not tell us how to take such overlap into account. A more global page replacement policy must exist to make such a decision.

## 1.2 本文的作用 (Contribution of this Paper)

上述两类解决方案都认为，由于 LRU 不能很好地区分频繁和不频繁引用的页面，因此有必要让其他代理提供某种提示。本文的贡献是推导出一种新的自力更生的页面替换算法，该算法更多地考虑每个页面的访问历史，以更好地区分应保留在缓冲区中的页面。这似乎是一种明智的方法，因为 LRU 算法使用的页面历史记录非常有限：仅仅是上次引用的时间。

> Both of the above categories of solutions take the view point that, since LRU does not discriminate well between frequently and infrequently referenced pages, it is necessary to have some other agent provide hints of one kind or an other. The contribution of the current paper is to derive a new self-reliant page-replacement algorithm that takes into account more of the access history for each page, to better discriminate pages that should be kept in buffer. This seems a sensible approach since the page history used by the LRU algorithm is quite limited: simply the time of last reference.

在本文中，我们仔细研究了考虑最后两次引用（或更一般地，最后 K 次引用，K >= 2）的历史记录的想法。本文开发的特定算法考虑到了对页面的最后两次引用的大量知识，称为 LRU-2，自然泛化为 LRU-K 算法；我们在此分类法中将经典的 LRU 算法称为 LRU-1。事实证明，对于 K > 2，LRU-K 算法在稳定的访问模式下比 LRU-2 的性能略有提高，但对访问模式的变化的响应较慢，这对于某些应用程序来说是一个重要的考虑因素。

> In this paper we carefully examine the idea of taking into account the history of the last two references, or more generally the last K references, K >= 2. The specific algorithm developed in this paper that takes into amount knowledge of the last two references to a page is named LRU-2, and the natural generalization is the LRU-K algorithm; we refer to the classical LRU algoritbm within this taxonomy as LRU-1. It turns out that, for K > 2, the LRU-K algorithm provides somewhat improved performance over LRU-2 for stable patterns of access, but is less responsive to changes in access patterns, an important consideration for some applications.

尽管 LRU-K 算法的优势在于页面访问频率的附加信息，但 LRU-K 与最不常用 (LFU) 替换算法有着根本区别。关键区别在于 LRU-K 具有内置的“老化”概念，仅考虑对页面的最后 K 次引用，而 LFU 算法无法区分页面的最近和过去引用频率，因此无法应对不断变化的访问模式。LRU-K 也与采用基于引用计数器的老化方案的更复杂的基于 LFU 的缓冲算法截然不同。此类算法（例如 GCLOCK 和 LRD [EFFEHAER] 的变体）主要依赖于对指导老化过程的各种工作负载相关参数的仔细选择。另一方面，LRU-K 算法不需要任何此类手动调整。

> Despite the fact that the LRU-K algorithm derives its benefits from additional information about page access frequency, LRU-K is fundamentaly different from the Least Frequently Used (LFU) replacement algorithm. The crucial difference is that LRU-K has a built-in notion of "aging", considering only the last K references to a page, whereas the LFU algorithm has no means to discriminate recent versus past reference frequency of a page, and is therefore unable to cope with evolving access patterns. LRU-K is also quite different from more sophisticated LFU-based buffering algorithms that employ aging schemes based on reference counters. This category of algorithms, which includes, for example, GCLOCK and variants of LRD [EFFEHAER], depends critically on a careful choice of various workload-dependent parameters that guide the aging process. The LRU-K algorithm, on the other hand, does not require any manual tuning of this kind.

LRU-K 算法具有以下显著特性：

- 它能够很好地区分具有不同引用频率级别的页面集（例如，索引页面与数据页面）。因此，它接近将页面集分配给特定大小的不同缓冲池的效果。此外，它还能适应不断变化的访问模式。
- 它能够检测查询执行中的引用局部性、同一事务中的多个查询以及多用户环境中多个事务中的引用局部性。
- 它是自力更生的，因为它不需要任何外部提示。
- 它相当简单，并且几乎不会产生簿记开销。

> The LRU-K algorithm has the following salient properties:
>
> - It discriminates well between page sets with different levels of reference frequency (e.g., index pages vs. data pages). Thus, it approaches the effect of assigning page sets to different buffer pools of specifically tuned sizes. In addition, it adapts itself to evolving access patterns.
> - It detects locality of reference within query executions, across multiple queries in the same transaction, and also locality across multiple transactions in a multi-user environment.
> - It is self-reliant in that it does not need any external hints.
> - It is fairly simple and incurs little bookkeeping overhead.

本文的其余部分概述如下。在第 2 节中，我们介绍了 LRU-K 磁盘页面缓冲方法的基本概念。在第 3 节中，我们给出了非正式论证，即在了解每个页面最近的 K 次引用的情况下，LRU-K 算法在某种明确定义的意义上是最优的。在第 4 节中，我们展示了 LRU-2 和 LRU-K 与 LRU-1 的模拟性能结果对比。第 5 节是总结。

> The remainder of this paper has the following outline. In Section 2, we present the basic concepts of the LRU-K approach to disk page buffering. In Section 3 we give informal arguments that the LRU-K algorithm is optimal in a certain well defined sense, given knowledge of the most recent K references to each page. In Section 4, we present simulation performance results for LRU-2 and LRU-K in comparison with LRU-1. Section 5 has concluding remarks.

# 2. LRU-K缓存的概念 (Concept of LRU-K Buffering)

在本文中，我们基于 [COFFDENN] 第 6.6 节中*分页独立参考模型*中的一些假设，从统计学角度看待页面引用行为。我们从直观的公式开始；[OOW] 中介绍了更完整的数学展开。假设给定一个磁盘页面集合 $N = {1,2,...,n}$，用正整数表示，并且所研究的数据库系统连续引用这些页面，这些页面由引用字符串指定：$r_1,r_2,...,r_t,...$，其中 $r_t = p(p G  N)$ 表示引用字符串中编号为 t 的项引用磁盘页面 p。请注意，在 [COFFDENN] 的原始模型中，引用字符串表示单个用户进程对页面的引用，因此假设字符串反映系统的所有引用是错误的。在以下讨论中，除非另有说明，否则我们将用引用字符串中连续页面访问的计数来衡量所有时间间隔，这就是通用术语下标用“t”表示的原因。在任何给定时刻 t，我们假设每个磁盘页面 p 都有一个明确定义的概率 $b_p$，成为系统引用的下一个页面：对于所有 p E N，Pr(rt+1=p)=bp。这意味着引用字符串是概率性的，是一系列随机变量。改变访问模式可能会改变这些页面引用概率，但我们假设概率 bp 具有相对较长的稳定值周期，并从概率对于引用字符串的长度不变的假设开始；因此我们假设 bp 与 t 无关。

> In the current paper we take a statistical view of page reference behavior, based on a number of the assumptions from the *Independent Reference Model* for paging, in Section 6.6 of [COFFDENN]. We start with an intuitive formulation; the more complete mathematical development is covered in [OOW]. Assume we are given a set N = {1,2,...,n} of disk pages, denoted by positive integers, and that the database system under study makes a succession of references to these pages specified by the reference string: rl,r2,...,rt,..., where rt = p (p G N) means that term numbered t in the references string refers to disk page p. Note that in the original model of [COFFDENN], the reference string represented the page references by a single user process, so the assumption that the string reflects all references by the system is a departure. In the following discussion, unless otherwise noted, we will measure all time intervals in terms of counts of successive page accesses in the reference string, which is why the generic term subscript is denoted by 't'. At any given instant t, we assume that each disk page p has a well defined probability, bp, to be the next page referenced by the system: Pr( rt+ 1 = p ) = bp, for all p E N. This implies that the reference string is probabilistic, a sequence of random variables. Changing access patterns may alter these page reference probabilities, but we assume that the probabilities bp have relatively long periods of stable values, and start with the assumption that the probabilities are unchanging for the length of the reference string; thus we assume that bp is independent of t.
