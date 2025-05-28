面试官您好，我叫贾小博，目前有3年的数据库内核研发经验。我平时工作主要负责的是SQL引擎模块的研发，对SQL解析器、优化器和执行器比较熟悉。非常感激海量数据给予的这次面试机会。

问题：

- 咱们的产品的发展方向是怎么样的？
- 工作的主要职责是什么？研发的方向是怎样的？
- 部门/团队在公司中扮演的角色是怎样的？
- 团队基本情况？

架构：

从单机介绍：多进场架构 master server listener dispatcher

集群和单机一致：多了cserver cdispatcher

# DBLink 项目

- 项目背景：国防大学数据库适配迁移项目，用户之前有在使用 Oracle 并用到了 DBLink 功能，因此希望我们的数据库也能提供类似的功能。所以开发了这样一个功能，方便用户在数据库迁移适配期间能够方便地查询原库的数据，并做一些轻量的数据同步工作。
- 项目周期：2024年12月27日～2025年2月28日
- 项目人员：研发组（5人，包含我）需求/测试组（4人）
- 项目细节：这个项目简单说就是要求我们的数据库提供访问异构数据库（主要是 Oracle ）对象的功能，并且也能够执行查询。因为访问异构库的方式只有通过官方API，如 JDBC/ODBC 的方式，再加上我们的数据库是 C 语言开发，大家对 ODBC 接口相对熟悉，所以我们访问异构库的功能选择基于 ODBC 来实现。此项目主要分为这几部分：Catalog 管理、 DBLink 管理、 SQL 处理（全DBLink，半DBLink两部分）。Catalog 管理的内容主要是：DBLink 作为一个新引入的数据库对象，它在系统中会新增很多的内容，比如系统表、视图、相关权限管理等，这些信息是 DBLink 的静态形式。然后 DBLink 作为一个用在 SQL 语句中的元素，它以一个 DBLink 实例的形式存在，这算是 DBLink 的动态形式。DBLink 管理就是管理 DBLink 实例的生命周期。包括了 ODBC 驱动管理、和句柄的管理以及日志管理。 SQL 处理主要是新的 DBLink 相关语句的 DDL 语句、DML 语句分为全 DBLink 语句、混合 DBLink 语句。混合 DBLink 语句中的子查询如果是连表查询，那么需要把远程的数据拉取到本地进行处理，如果子查询是全远程表，则组织成远程查询直接由远程库进行处理。事务相关的内容是关闭远程库的自动提交，在提交的时候，把当前会话的DBLink连接相关的事务全部提交，回滚也是一样的逻辑。
- 参考文档：《Oracle SQL Ref.》，《SQL Connect SQL Server》
- 我的分工：三个部分都有参与，我在这个项目中负责 Catalog 管理的 DBLink 相关字典表、 DBLink 实例状态表（Fixed 表）、DBLink 实例管理的 DBLink Context 设计和开发以及 Driver 管理。SQL 处理主要是负责了 DDL（CREATE）、DML（INSERT SELECT）语句的处理。
- 性能要求：增加代码后的性能下降率不得超过原始版本的01.%。
- 项目难点：远程SQL生成（扫描解析树）、DBLink 管理。
- **JOIN优化办法**：由于数据源不是本地的对象，因此优化的手段比较有限
  - 根据本地的SQL，生成远程的JOIN表的查询语句时，尽量使用本地语句中已经具有的过滤条件（如ON子句、WHERE子句等）来降低远程表的数据量。
  - 如果JOIN中存在多个同数据源的表，那么就把这几个表的查询做到一个JOIN语句里进行一次性的查询。
    - 多个同源表的Join操作不一定能够直接发送，需要根据情况而定。


---

# 多表插入项目

- 项目背景：POC适配项目，客户方要求支持 Oracle 数据库的 Multiple Insert 语法。
- 项目周期：2023/02 ～ 2023/07
- 项目人员：研发组（3人，包含我）、需求/测试组（1人）
- 项目细节：这个项目是实现 Oracle 的 Multiple Insert 语法的功能。能够通过一个语句向多个目标表有选择地插入数据。这个项目的语法非常复杂（主要的点在于多个Into Clause分支与Insert Source的属性之间的匹配），并且没有类似的语句可供参考，这个项目的主要思想是把目标表和它们的插入条件作为一个Pair放入一个环（执行环）中，在入环之前先进行合并（目标表合并、插入条件合并），在插入的时候，遍历这个环中的每个Pair，当符合Pair中的条件时则向Pair中的目标表插入数据。
- 特点：Write Query Node，Read Query Node
- 我的分工：我在这个项目中负责语法支持、验证以及执行器的设计和实现。
- 性能要求：不影响现有功能的情况。
- 项目难点：语法复杂、执行逻辑复杂。

---

# Aggregation Filter项目

- 项目背景：该项目是作为 Pivot/Unpivot 项目的子项目进行实施的。在开发 Pivot 功能的过程中，我们注意到了当执行 Pivot 语法的替代语句（即分组聚合）时，若能够利用 FILTER 子句来明确指定聚合条件，查询语句的可读性和功能性都将得到显著提升，并且也符合SQL标准，因此决定支持该功能，以便用户能够更加精确地控制聚合过程中的数据过滤逻辑。
- 项目周期：2023/5/30 ~ 2023/6/16
- 项目人员：研发2人（刘军，我），测试2人
- 项目细节：这个功能的实现除了语法支持的部分以外，其他的改动主要是在执行器的部分，Group By的主要过程是从Table Access/Index Access执行器那里获取数据，然后插入到Hash Instant当中（内部实现叫做Merge Record），Aggregation Filter就是在Merge Record中做的（提取Hash Key以及插入Hash Instant）。
- 我的分工：
- 性能要求：
- 项目难点：工作量大

```sql
-- ITEM  REGION PRICE AMOUNT
-- ----- ------ ----- ------
-- apple daegu  25000     30

-- pivot
SELECT *
  FROM sales
 PIVOT (
 	SUM( price * amount ) FOR item IN ( 'apple' APPLE )
 );
-- aggr-where
SELECT region
     , SUM( price * amount ) AS apple
  FROM sales
 WHERE item = 'apple'
 GROUP BY region;
-- aggr-filter
SELECT region
     , SUM( price * amount ) FILTER( WHERE item = 'apple' ) AS apple
  FROM sales
 GROUP BY region;
```



---

# OCI API 封装

- 项目背景：这是一个 POC 适配项目，客户方面要求的是要支持兼容 Oracle 数据库的 OCI 接口，为了方便客户数据库的迁移工作，决定开发此项目。
- 项目周期：2021/12 ～ 2022/02
- 项目人员：测试/开发均是我自己
- 项目细节：此项目给我的需求是要我实现 30+ 常用的 Oracle OCI 接口，实现方式是基于 ODBC 来进行封装。这个项目的工作量主要是集中在前期调研 OCI/ODBC 接口的使用方式和区别，确定用户对 OCI 接口的使用情况（用户使用这个库的情况是，用的都是一些常用的接口）。两个库的架构很类似，都是操作用户不可见的句柄对象，接口的使用流程稍有区别。因为用户。主要考虑的点是 ：句柄的封装；数据类型的封装；接口的封装。
- 我的分工：设计、开发和测试都是我一人。
- 性能要求：通过官方Demo。
- 项目难点：工作量大。然后很多不支持的属性就只能标识出来不支持。还有就是不支持的数据类型：例如，BLOB/CLOB 类型，只能通过封装 LONG VARCHAR / LONG VARBINARY 来实现，但是用法不太一致，就是人家是通过一次调用就能查出来，而ODBC只能通过多次查询出结果。因此这个封装比较麻烦点。其他的就没有什么难度了。

---

# 解决的问题

## Limit问题

这个问题是在客户现场进行测试的时候发生的错误：

---

# 数据迁移

基于CDC的数据复制工具

分为master端

---

#  SUNDB分布式概念

- 集群组数量无上限
- 集群组内集群成员最多为32个
- 所有成员之间不区分应用程序服务器、数据服务器及Meta服务器，即各服务器是对等的，均可独立提供服务。所有的节点均拥有相同的数据字典。
- 集群表分为克隆表和（水平）分片表
- 当集群新增组时，分片表进行数据重分布时会从已有的每个组中都分出适量的分片到新组中，最终达到各组中分片数量的均匀。
- 全局二级索引（GSI）：如果没有 GSI 就无法在集群中执行Undetermined DML。GSI 把同一条数据在集群中全局唯一的GRID和每个节点中不同的ROWID进行绑定（B树）。这样用同一个GRID就能找到不同节点中的（ROWID不同）同一行。
- 集群中的DDL：分为Lock阶段和Execute两个阶段。Lock阶段对所有集群成员按顺序获取DDL所需的锁，Execute阶段同时在所有节点执行DDL操作。
- 集群中的SELECT：远程数据收集（collection）以及本地的数据操作（manipulation）由 Cluster Puller Plan Node 负责。然后根据本地数据的收集方法和是否区分收集远程数据时的远程服务器，Cluster Puller Plan Node 可以分为3中：Plan Based Cluster、Single Cluster和Multiple Cluster。
- 集群中的DML：在集群DML执行时，会选定各Group的Master节点（一个，选择成员号最小的那个）和Slave节点（除Master节点外的其他节点）。DML语句执行整体上分Master变更阶段和Slave变更阶段。根据所执行的DML不同，分为基于Query（仅限于各服务器使用Generated SQL执行可以获得一致结果时使用，及确定性语句）和基于GRID（非确定性语句）的两种。
- 基于Query的DML：当语句中不包含限定变更集群组条件时，在所有集群组上执行，否则只在限定的集群组上执行。基于Query的DML由DML  Cluster来执行（Fetch/Non-Fetch/ShardKeyUpdate）
- 基于GRID的DML：添加新记录或无法使用基于Query的DML时使用基于GRID的DML。两个阶段：收集GRID信息和操作GRID对应记录（先Master、后Slave）。
- SCN由GCN，DCN和LCN组成，及Global Change Number、Domain Change Number和Local Change Number。
- GCN：全局事务完成时，集群中的所有节点的GCN都自增（并且是一致的）。
- DCN：全局事务涉及的Group完成时，在Group范围内自增，并组内成员保持一致。
- LCN：全局事务涉及的成员完成时，自增并在Group范围内保持一致。

# SUNDB语句优化

## Rewrite

### Filter Push Down

- 把条件从Join下推至Table Access
- 把条件从Join下推至View内部的Table Access

### DISTINCT Elimination

- 聚合函数的结果本来就是一行，所以可以消除 DISTINCT。
- 当使用 Group By 且 Group By 的所有 Key Column 存在于查询列表时，结果本身就是 unique 的，所以可以消除 DISTINCT 。
- 主键的所有 Key Column 存在于查询列表时，结果本身时 unique 的，所以可以消除 DISTINCT 。

### ORDER BY Elimination



# Raft

Raft把共识算法分为了三个子问题：领导者选举、日志复制和安全性。

> **复制状态机（Replicated State Machine）**
>
> 相同的初始状态 + 相同的输入 -> 相同的最终状态：复制状态机就是由 Leader 把一系列 Command 封装到 Log 中，然后分发给所有的 Follower ，Follower 按照 Log 的 Command 执行操作，最终所有的节点就能达到一致状态。
>
> **状态简化**
>
> 任意时刻，每一个节点都处于 Leader 、 Follower 和 Candidate 三个状态之一（相比于 Paxos 的状态共存和互相影响， Raft 只用考虑状态转换，简化了许多）。

## 领导者选举（Leader Election）

- 一开始系统中的节点都是 follower 节点，并开启周期性的心跳超时倒计时，直到首个 follower 意识到没有 leader （即心跳超时了还没有收到 leader 发来的信号），它自己就切换到 candidate 并增加自己的任期号并首先给自己投一票，然后向所有节点发送投票请求（在这个过程中其他节点可能也发现系统者没有 leader 并发起请求）。
- candidate 发送投票请求之后，如果能够接收到超过半数的选票，那么该节点就转为 leader ，并开始周期性地向其他所有节点发送心跳，告知自己是新的 leader 。
- candidate 发送投票请求之后，如果收到了其他 leader 发送的心跳信息，且新 leader 的任期号不小于自己当前的任期号，那么就表明自己选举失败了，这时 candidate 就会重新转为 follower 。
- candidate 发送投票请求之后，没有获得过半数的投票（这表示系统中的 candidate 太多，投票结果过于分散），那么 candidate 就会在随机的选举超时（论文中是 150～300ms）之后，重新发起下一轮投票请求（任期号加一、投自己一票、发送投票请求等动作）。

## 日志复制（Log Replication）

- leader 选出后，开始为客户端请求提供服务。
- leader 接收到客户端指令后，将质量作为新的日志条目追加到日志中。
- 日志会包含 3 个信息：
  - 日志索引号
  - leader 任期号
  - 操作指令
- leader 并行发送追加日志的 RPC 给 follower ，让它们复制该日志条目。当超过半数的 follower （包括 leader 在内）复制完成后， leader 就可以在本地执行该指令并把结果返回给客户端。
- 把本地执行指令，也就是 leader 应用日志与状态机这一步，称作提交。 

---

排序算法总结

| 排序算法 | 平均时间复杂度 | 空间复杂度 | 稳定性                   |
| -------- | -------------- | ---------- | ------------------------ |
| 选择排序 | $O(n^2)$       | $O(1)$     | NO                       |
| 冒泡排序 | $O(n^2)$       | $O(1)$     | YES                      |
| 插入排序 | $O(n^2)$       | $O(1)$     | YES                      |
| 快速排序 | $O(n\log n)$   | $O(n)$     | NO                       |
| 归并排序 | $O(n\log n)$   | $O(n)$     | YES                      |
| 堆排序   | $O(n\log n)$   | $O(1)$     | NO                       |
| 桶排序   | $O(n+k)$       | O(n+k)     | 取决于桶内排序算法       |
| 计数排序 | $O(n+m)$       | $O(n+m)$   | 取决于输出的方式         |
| 基数排序 | $O(nk)$        | $O(n+d)$   | 每位用稳定计数排序时稳定 |

---

### 堆

1. 建堆的时间复杂度（输入数组建堆）：$O(n)$
2. 入堆（push）和出堆（pop）的时间复杂度都是 $O(\log n)$

