# Thinking in Events: From Databases to Distributed Collaboration Software

在第 15 届 ACM 分布式和基于事件的系统 (DEBS) 国际会议上的主题演讲
马丁克莱普曼 
剑桥大学 英国剑桥 
mk428@cst.cam.ac.uk
 
## 摘要
在本主题演讲中，我对基于事件的分布式系统的前景进行了主观但系统的概述，重点介绍了我在过去十年中从事的两个领域：使用 Apache Kafka 和相关工具进行大规模流处理，以及基于Google Docs 风格实时协作软件。虽然这些乍一看似乎是非常不同的主题，但也有一些重要的重叠点。本文列出了基于事件的系统的分类法，显示了它们的共性和差异所在。它还强调了在基于事件的系统的实现中出现的一些关键权衡，这些权衡取自分布式系统理论和实际部署的经验。最后，本文概述了该领域的一些开放性研究问题。
### CCS 概念
* 信息系统 → 数据流； 
* 软件及其工程 → 发布-订阅/基于事件的架构；
* 应用计算→ 事件驱动架构； 
* 计算理论→分布式计算模型。

### 关键词
流处理、事件溯源、状态机复制、CRDT、实时协作

### ACM Reference Format:
Martin Kleppmann. 2021. Thinking in Events: From Databases to Distributed Collaboration Software: Keynote at the 15th ACM International Conference on Distributed and Event-Based Systems (DEBS). In The 15th ACM International Conference on Distributed and Event-based Systems (DEBS ’21), June 28-July 2, 2021, Virtual Event, Italy. ACM, New York, NY, USA, 10 pages.
https://doi.org/10.1145/3465480.3467835

Permission to make digital or hard copies of part or all of this work for personal or classroom use is granted without fee provided that copies are not made or distributed for profit or commercial advantage and that copies bear this notice and the full citation on the first page. Copyrights for third-party components of this work must be honored.
For all other uses, contact the owner/author(s).
DEBS ’21, June 28-July 2, 2021, Virtual Event, Italy
© 2021 Copyright held by the owner/author(s).
ACM ISBN 978-1-4503-8555-8/21/06.
https://doi.org/10.1145/3465480.3467835

## 1 介绍
Event一词在计算的不同分支中意味着许多不同的事物。本文的目标是阐明、分类和区分一些主要的基于事件的系统系列，重点是分布式系统。
 
第 2 节提出了一种分类法，该分类法根据简单的技术标准将该领域分为几类。每个类别都通过使用该方法的实际系统示例进行说明，并且包括对权衡的讨论：选择一个类别而不是另一个类别的利弊。我希望这种分类法能为研究人员和使用基于事件的系统的从业者提供有用的结构和词汇，使我们能够更轻松地就此类系统的基本设计选择进行交流和推理。

接下来，在第 3 节我对我在过去十年中从事的两个特定的基于事件的系统系列提出了个人观点：Apache Kafka 及其流处理生态系统（我在 LinkedIn 担任工业软件工程师期间在 2012 年至 2015 年从事的工作） ，以及 Google Docs 风格的多用户实时协作软件（自 2015 年重返学术界以来，这一直是我研究的重点）。在分类学的背景下反思这些系统第 2 节 有助于进一步阐明基于事件的软件架构的特征。

最后，第 4 节概述了一些开放的问题，我认为这些问题值得研究人员进一步关注。

# 2 基于事件的系统分类法
以下分类法根据流程图对事件的不同用途进行分类图1。如此粗略的分类法不一定能捕捉到该领域的所有细微差别，有时类别之间的界限可能会很模糊。尽管如此，我希望它将成为理解基于事件的系统的有用工具。

### 2.1	通知和持久性
概括地说，事件可以是：
* 关于某事发生的事实的通知；
* 对某事发生的事实的持久记录；或者
* 二者都有。

“通知”是指在事件发生后不久，系统调用可能对事件起作用的应用程序代码。通知事件的示例出现在许多应用程序中。在 Web 浏览器中运行的 JavaScript 代码可以注册要在用户执行某些操作时调用的函数，例如在键盘上单击或键入键：这里的单击或按键是事件 [53]。大多数其他用户界面框架都有一个类似的系统，用于将用户输入事件分派给回调函数或事件处理程序。许多操作系统提供异步（非阻塞）I/O 功能，在这种情况下，线程运行一个事件循环，在 I/O 操作完成时处理请求来自操作系统的通知[48]。离散事件模拟采用这种类型的事件来模拟而不是物理的发生[52]。反应式和函数式反应式编程 (FRP) 提供更高级别的 API，例如数据流编程模型，用于处理事件流 [6, 16]。在所有这些示例中，事件是仅存在于单个进程中的内存中的东西。它是短暂的，而不是持久的，因为它通常不会写入磁盘。临时事件仅在进程的生命周期内存在，并且不会在重新启动后保留。

相反，持久化事件不一定要触发什么。例如，时间序列数据库通常用于记录随时间发生的事件，例如传感器的定期读数、金融资产的价格更新以及跟踪人或机器的活动[44]。另一个例子是出现在数据仓库和业务分析系统的星形或雪花模式中的事实表：事实表中的每一行通常是已发生事件的时间戳记录，例如特定产品的购买顾客 [32]。在这样的系统中，事件是记录在数据库中以供将来分析的值，但事件的接收不一定触发任何应用程序代码的立即执行。从用户的角度来看，事件的主要目的是允许对事件历史进行追溯查询和分析，例如检查趋势和生成显示某些指标随时间变化的报告。

一些系统结合了这些特征，使得事件既是事件发生事实的持久记录，也是通知（即调用应用程序代码来处理事件的发生）。我将在术语流处理下对此类系统进行广泛的分组，稍后将它们分解为子类别。

一些数据库提供了诸如物化视图维护和持续查询（其结果随着底层数据变化而自动更新的查询）等设施，这也可以看作是一种通知形式[15, 24]。分布式系统的很多编程模型都是基于消息的发送和接收，而消息的接收也可以看作是触发应用程序代码执行的通知事件。例如，Actor 模型是一种广泛使用的方法，其中可能位于不同机器上的多个 Actor 或轻量级线程通过相互发送消息进行通信[65]。

持久性在实践中是一个模糊的概念，因为一个事件可能会通过多个系统，其中一些可能会将其写入磁盘，而另一些可能会将其视为短暂的（即，如果节点崩溃或网络不可靠，它可能会丢失）。例如，一些参与者框架和消息中间件系统还包括参与者状态和/或消息的持久性机制[7];然而，在许多消息代理中，消息仅在它们成功交付给消费者之前被存储，然后被删除。另一方面，数据库通常会无限期地存储数据，直到它被明确删除。出于此分类的目的，系统的确切持久性特征并不重要，尽管我们稍后讨论的许多系统更倾向于类似数据库的长期存储端。
 
### 2.2	流处理中的窗口化
通过深入提供持久化和通知的系统，我们可以通过检查可以响应事件执行的逻辑类型来进一步区分它们。最简单的流处理器使用无状态运算符，其中每个事件都可以独立于任何其他事件进行处理，仅基于该事件中的信息（例如，选择事件的某个属性属于某个值集的事件）。 然而，无状态处理很少是足够的，大多数有趣的应用程序还需要结合来自多个事件的信息的事件处理逻辑：在数据库术语中，事件的连接或分组[12]。事件可以通过聚合（例如计算一组事件的某些属性的计数、总和或平均值）、通过发出包含来自多个输入事件的属性的复合事件或通过我们在中讨论的更复杂的状态机来组合第 2.4 节。

一些流处理系统主要是为需要组合的事件在时间上非常接近地发生的应用程序而设计的。例如，股票交易系统可能需要计算资产每小时或每天的最低和最高价格，或者欺诈检测系统可能需要检测客户信用卡上近期活动的异常模式。此类操作称为窗口连接或聚合。使用了几种类型的窗口（例如翻滚或滑动窗口 [2])，但它们都有一个共同点，即它们对可以组合的任何两个事件之间的最大时间间隔进行了限制；任何在时间上比这个界限更远的事件都被视为不相关的[3]。窗口化处理通常发生在流的业务分析中，或者复杂的事件处理中[50]。

另一方面，一些应用程序需要组合可能在时间上任意相隔很远的事件，因此窗口化不适用[15]。例如，想象一个类似 Twitter 的社交网络，其中存在三种主要类型的事件：发布推文（消息）、关注用户和取消关注用户。当用户登录时，他们希望看到他们关注的用户发布的推文。在 SQL 数据库中，此查询可能表示如下：

SELECT tweets.* 
FROM tweets 
JOIN follows ON follows.followee_id = tweets.sender_id
WHERE follows.follower_id = logged_in_user_id
ORDER BY tweets.send_timestamp DESC
LIMIT 1000

在基于事件的系统中，一个 tweet-posting 事件对应于在 tweets 表中插入一行，follow 事件对应于在follows 表中插入一行，unfollow 事件对应于从follows 表中删除一行。但是，很可能该用户几年前关注了另一个用户，因此要找到用户关注的当前人员列表需要在关注/取消关注事件流中任意返回。因此，连接需要是非窗口的。

上面的 SQL 查询对于关注很多人的用户来说执行起来很昂贵，因为它需要查找所有关注的人最近的推文并按时间顺序合并它们。出于这个原因，Twitter 为每个用户构建了一个包含上述查询结果的缓存（这被称为主时间线 [42]）。保持这个缓存是最新的需要一个流处理器，它结合了推文、关注和取消关注事件，以便逐步维护上述查询的具体化视图。

当用户发布推文时，流处理器需要找到所有关注发件人的人，并将新推文添加到他们的家庭时间线中。当用户 A 关注用户 B 时，流处理器需要找到 B 最近的推文，并将它们合并到 A 的主时间线中。当用户 A 取消关注用户 B 时，它需要从 A 的主页时间线中删除 B 的所有推文。有效地执行此流连接需要允许查找给定用户的当前关注者集以及给定用户的最近推文的索引。

### 2.3	数据库复制和事件
如果我们将注意力集中在事件是没有窗口的持久通知的系统上，我们可能会注意到与数据库系统中的复制非常相似。复制的目标是在几台不同的机器上拥有相同数据的副本：即任意两个副本收敛到同一个状态，所有提交的事务都反映在该状态中[14]。从基于事件的系统的角度来看，我们可以将每一个修改数据库的事务视为一个更新事件，将一次更新的执行（即在数据库状态中反映更新）视为对该事件的处理，并将复制数据从一台机器到另一台机器作为该事件通过分布式系统的传播。只读事务可能会或可能不会对应于事件，具体取决于系统。

在这样的数据库系统中，复制基础设施确保对应于已提交事务的每个事件最终由每个无故障副本处理。我们可以将这样的复制系统分为两大类：将复制事件安排到日志中的系统，以及使用其他复制机制（例如 gossip 协议 [47]，反熵 [19]，或者客户端独立更新多个副本 [4]）。日志的关键属性是其中的事件是完全有序的（即所有副本都以相同的顺序观察日志条目），并且它是仅追加的（即，如果副本观察日志条目 A，紧随其后的是日志条目 B ，那么就永远不会有日志条目 C 出现在 A 和 B 之间的总顺序）。

这个append-only日志是使用共识协议（如 Raft [55] 或者 Multi-Paxos [46]，或者通过将一个副本指定为领导者（也称为主副本或主副本）并将所有其他副本指定为追随者（也称为从副本或辅助副本）[23]）。事实上，大多数共识协议本质上是基于领导者的复制，结合了一种自动机制，如果当前领导者失败，则选举新领导者[29]。领导者决定将事件附加到日志的顺序，所有其他副本按照领导者决定的顺序处理事件。

在许多数据库中，日志中事件的确切结构和内容传统上是数据库系统的实现细节，不暴露给应用程序。例如，一些数据库系统使用预写日志 (WAL) 进行复制，其中包含指示数据库的磁盘数据结构如何变化的记录；其他人使用单独的复制日志 [28]。最近，已经开发了变更数据捕获技术来从该日志中提取数据变更事件（插入、更新或删除的行），并通过流处理系统将它们提供给应用程序[1, 17].
 
### 2.4	状态机复制 (SMR)
在基于 WAL 的复制和变更数据捕获中，应用程序使用的主要数据模型是应用程序可以改变的数据库状态（例如，通过插入、更新或删除表中的行），并生成数据更改事件自动作为这些突变的副作用。也可以交换这些角色，使数据更改事件成为应用程序的主要数据模型，因此任何状态更改都成为处理这些事件的副作用 [41]。这就是状态机复制 (SMR) 背后的理念 [59]，以及密切相关的事件溯源概念[20, 68]。我 2014 年的演讲 将数据库从里到外 [33] 也帮助普及了这个想法。

在 SMR 和事件溯源中，应用程序不会直接改变数据库的状态。相反，应用程序定义了一组可能发生的事件类型；每当发生某些事情时，适当类型的事件都会附加到事件日志中，并且此后是不可变的。 （在 SMR 中，事件也称为命令。）所有副本都订阅此事件日志，并且每个副本按照事件在日志中出现的顺序处理事件。事件处理函数可以使用任意逻辑来更新数据库状态，只要它是确定性的，它只依赖于当前数据库状态和事件的内容。假设每个副本以相同的顺序处理相同的事件序列，并且每个副本以相同的初始状态开始（例如空数据库），则确定性事件处理逻辑确保所有副本通过相同的状态序列并结束在相同的最终状态[9, 59]。我们可以将每个副本视为一个状态机，其状态是它的数据库副本，其状态转换功能是事件处理功能。换句话说，数据库状态是底层事件日志的物化视图[26]。

这种方法的一个优点是，设计良好的事件通常比仅是状态突变的副作用的事件更好地捕捉操作的意图和意义[68]。例如，“学生 x 由于原因 z 取消了他们在课程 y 中的注册”是一个明确的描述性事件，而“从注册表中删除了一行，增加了课程的 available_places 字段，并在 cancel_reasons 中添加了一行table”不太清晰，并且嵌入了许多与当前数据库模式无关的细节。因此，事件日志使使用系统的人更容易了解它是如何进入特定状态的，这有助于审计和调试[9]。

事件日志的另一个优点是可以重放它以重建生成的数据库状态。如果应用程序开发人员希望更改处理事件的逻辑，例如更改生成的数据库模式或修复错误，他们可以设置新副本，使用新的处理功能重播现有事件日志，将客户端切换到从新副本而不是旧副本读取，然后停用旧副本 [34]。在依赖状态突变作为主要数据模型的数据库中，这种业务逻辑的追溯更改通常是不可能的。此外，如果需要，很容易在同一个底层事件日志上维护几个不同的视图。只要更新率不太高，无限期保留事件日志并根据需要偶尔重播它通常是可行的。在具有高事件率的系统中，这种重放可能不可行。

图 2：使用 Lamport 时间戳（计数器、replicaID 对）获取事件的总顺序。具有相同计数器的两个事件按replicaID排序；这里我们假设 A < B。
 
区块链和分布式账本也使用 SMR，在这种情况下，区块链（以及其中的交易）构成事件日志，账本（例如每个账户的余额）是结果状态，智能合约或网络内置事务处理逻辑是状态转换函数[66].

事件溯源/SMR 方法的缺点是，对于大多数应用程序开发人员来说，它不像可变状态数据库那样熟悉，并且事件日志和生成的数据库状态之间的间接级别增加了某些类型的应用程序的复杂性，这些应用程序更容易用状态突变表示。在具有高事件率的应用程序中，存储和重放日志可能会很昂贵。

如果需要永久删除记录（例如，根据 GDPR 被遗忘权删除个人数据 [62])，不可变的事件日志需要格外小心。建议的解决方案包括定期重写日志以删除任何需要删除的记录，或使用每个用户的密钥加密个人数据，如果用户请求删除其数据，则可以删除该密钥：无法解密的记录被广泛认为是相当于一条已删除的记录 [63]。如果其他系统的状态来自事件日志，则也可以通过首先从日志中删除个人数据然后重播事件来从这些下游系统中删除个人数据。

### 2.5	部分有序事件
基于 WAL 的复制和 SMR 都非常依赖于所有副本以完全相同的顺序处理事件的假设。在单个数据中心内，这是一个合理的假设，因为基于领导者的复制和共识协议在这种情况下运行良好。但是，如果副本分布在多个地理位置，或者如果副本之间的网络不可靠，则构建日志会变得很昂贵，因为将事件附加到日志需要至少等待一次到领导者和/或仲裁的网络往返的副本。在极端情况下，如果我们希望副本能够生成和处理事件，即使它与所有其他副本完全断开连接（CAP定理[21]意义上的“可用”和“分区容忍”系统），完全有序对数的假设变得不可能满足[13, 18]。

在一个允许断开操作的系统中，我们可以保证的最强顺序是因果有序 [5]。相对于日志的全序，这是一个部分有序。在因果有序的系统中，一些事件发生在其他事件之前[45]，并且所有副本以相同的顺序处理这些事件。但是，其他事件可能是同时发生的，这意味着两者都没有发生在另一个之前；在这种情况下，不同的副本可能会以不同的顺序处理这些事件 [10]。部分有序的复制形式也称为乐观复制[58]。

由于不能保证副本以相同的顺序处理事件，确定性事件处理不再足以确保副本最终处于相同的状态。但是，在某些应用程序中，可以确保每当两个事件并发时，处理它们是可交换的（例如，添加数字）；在这种情况下，不确定的处理顺序是没有问题的，并且副本仍然可以收敛。第 2.6 节 扩展了这个想法。

在部分有序的系统中，仍然可以在事后对事件强制执行全序，如图所示图 2。我们通过为每个事件附加一个逻辑时间戳来做到这一点； Lamport 时间戳 [45] 是一个常见的选择。这些时间戳包含一个计数器，该计数器为每个事件递增，以及生成该事件的副本的全局唯一 ID。在图2，两个副本最初具有相同的两个事件 a 和 b，其时间戳分别为 1、A 和 2、A。在两个副本相互断开连接期间，副本A分别生成两个事件c和d，时间戳分别为3，A和4，A，而副本B同时生成事件x和y，时间戳为3，B和4，B .恢复连接后，副本了解彼此的事件，并将它们合并为一个总顺序。此顺序是通过首先按其时间戳的计数器部分对事件进行排序来定义的，然后通过按副本 ID 排序来打破平局（这里我们假设 A < B）。

时间戳排序产生一个完全有序的事件序列，但它不是日志，因为新事件并不总是附加到末尾。在图2，当副本 B 收到来自A的事件 c，它必须在其事件序列中的现有事件 x 和 y 之前插入 c。类似地，副本 A 必须在其现有事件 c 和 d 之间插入 x。在 SMR 中，事件的总顺序在处理完事件后立即固定，但使用时间戳排序，随着从其他副本接收到更多事件，可能需要修改顺序。尽管如此，还是可以在类似 SMR 的方法中使用时间戳排序的事件：也就是说，使用确定性函数以时间戳顺序将事件一一应用到当前副本状态。假设每个副本最终都会接收到每个事件，那么所有副本最终将具有相同的时间戳排序的事件序列，因此所有副本将经历相同的状态序列并最终处于相同的状态。然而，当一个副本不按时间戳顺序处理事件（在时间戳顺序的中间某处插入事件），它必须能够将副本状态回滚到与插入位置对应的时间的状态，应用新事件，并且然后重播时间戳大于新事件的事件[64]. 这种方法被称为时间扭曲[31]。

这种回滚和重放过程的成本取决于事件被乱序接收的程度。如果副本按时间戳升序接收大多数事件，并且它们只是偶尔需要重新排序最近的几个事件，则成本可能会适中 [43]。另一方面，如果副本可能在彼此断开连接的情况下生成大量事件，则将 n 个事件合并为线性序列的成本可能会高达 O n2。

此外，如果处理一个事件除了更新副本状态之外还可能有外部副作用——例如，如果它可能触发发送电子邮件——那么时间扭曲方法需要某种方式来撤消或补偿这些副作用。先前处理的事件受到具有较早时间戳的迟到事件的影响的情况。电子邮件一旦发送就无法取消发送，但如有必要，可以发送带有更正的后续电子邮件。如果这种修正的可能性不可接受，则不能使用乐观复制，而必须使用 SMR 或其他强一致性方法。在许多业务系统中，更正或道歉无论如何都来自正常的业务过程[27]，因此在实践中，可能由于乱序事件而偶尔进行的更正也是可以接受的。

### 2.6 无冲突复制数据类型 (CRDT)
在时间扭曲方法中，就像在 SMR 和事件溯源中一样，事件是应用程序的主要数据模型，副本状态是从事件派生的。类似于第 2.4 节，我们还可以选择交换这些角色，以便可变副本状态是应用程序的主要数据模型，并且作为改变此状态的副作用自动生成事件。如果我们在部分有序系统的上下文中采用可变状态方法，我们将获得一种称为无冲突复制数据类型或 CRDT 的技术[60].
已经为许多常见的抽象数据类型定义了 CRDT：集合、映射、列表、树、图形等 [61]。应用程序可以通过数据类型接口提供的操作来改变这些结构：例如，可以通过插入或删除元素来改变集合或列表；可以通过分配键的值或删除键值对来改变映射。即使副本与系统的其余部分断开连接，这些修改也可能发生。

在基于操作的 CRDT 中，CRDT 算法跟踪数据对象的任何突变并生成描述变化的事件（通常称为操作）。这些事件是部分排序的；例如第 2.5 节，因果顺序是一个常见的选择 [22]。当一个副本接收到另一个副本生成的事件时，它会调用 CRDT 算法来更新其状态。该算法经过精心设计，使得应用并发事件是可交换的：也就是说，不管副本应用一组并发事件的顺序怎样，最终状态是相同的。通过依赖可交换性，CRDT 确保副本收敛到一致的状态，而不要求所有副本以相同的顺序处理事件。许多 CRDT 算法具有可以等效地在时间扭曲模型中表达的行为 [37]。然而，CRDT 的优点是它们通常不需要时间扭曲的回滚和重放过程，因此它们可以提供更高的性能 [57]。 CRDT 的状态突变模型非常适合用户能够或多或少直接操作相关状态的应用程序：例如，在文本编辑器中，用户可以在文档的任何位置插入或删除文本；在图形编辑软件中，用户可以在图片中的任何位置创建、删除、移动或修改图形对象。在这样的应用程序中，不需要事件源提供的间接级别，因为事件只会表达低级别的状态更新（例如“在位置 x 插入字符 A”，或“将对象 A 的坐标更改为 x，是”）。 

CRDT 的一个缺点是它们仅支持数据类型接口提供的预定义操作：例如，虽然列表 CRDT 允许插入或删除元素，但大多数都没有很好地支持重新排序列表项 [35]。相比之下，时间扭曲模型允许使用任何确定性的纯函数来处理事件，使其更加灵活。

## 3 实际例子
在本节中，我通过简要讨论以下模型的实际应用来提供个人观点第 2 节。我从我直接从事的两个领域中抽取示例：使用 Apache Kafka 和相关工具进行流处理，以及允许多人一起处理共享文档的协作软件。

### 3.1 卡夫卡生态系统
Apache Kafka 是一个基于事件日志的发布/订阅消息代理 [41, 67]。它广泛用于一些团队想要发布与其业务运营相关的事件流，而其他团队想要订阅和处理这些事件流的企业中[40]。 Kafka 生态系统包括流处理框架（Kafka Streams、Flink [12]，萨姆扎 [38, 54])，一个基于 SQL 的流查询引擎 (ksqlDB)，以及用于从数据库等外部系统获取事件流的变更数据捕获工具（Kafka Connect、Debezium [1]）。基于 Kafka 的流处理器用于窗口和非窗口处理。
Kafka 中的每个事件都属于一个主题，订阅者可以选择要收听的主题。出于可扩展性的原因，Kafka 不仅提供一个日志：每个主题都包含可配置数量的单独日志（称为分区或分片）。一个给定的分区内的所有事件是完全有序的。这种分区设计的优点是不同的分区可以由不同的节点处理，不需要协调；它的缺点是没有定义跨不同分区的事件顺序。

图 3：用于本地用户输入和与远程用户协作的功能反应式编程模型。图来自[25].

因此，当 Kafka 用于事件溯源/SMR 风格时，我们要么使用单个分区（限制系统的可扩展性），要么将系统状态分解为独立的分区以匹配事件日志的分区。例如，如果状态可以被实体分解（使得每个事件都与一个实体相关，而一个实体的状态仅由与该实体相关的事件决定），那么我们可以确保与该实体相关的所有事件同一个实体被放置在同一个分区中。然后可以独立处理每个事件日志分区，以获得该分区内实体的状态。

如果系统不能被巧妙地分解为单独的分区，情况会更加复杂：例如，如果一个事件可能与多个实体相关（例如，它表示资金从一个实体转移到另一个实体），那么将不再足够独立处理每个分区中的事件。在线事件处理 (OLEP) [36] 是一种通过将此类多分区交互分解为多个流处理阶段来处理此类交互的方法。

在某些应用程序中，需要根据系统的当前状态检查事件以确定它是否被允许。例如，表示预订剧院座位的事件可能仅在该座位尚未被占用时才被允许。在可变状态数据库模型中，这样的验证将由一个事务执行，该事务首先读取数据库的当前状态（座位是否可用），然后仅在检查成功时自动写入其更新。 Kafka 本身并不支持此类验证，但可以使用两阶段流处理管道来实现它们：初始事件仅表示执行某个操作的意图；然后流处理器将该事件与当前状态连接以确定该操作是否被允许，如果允许，则向经过验证的事件流发出一个新事件[36].
 
### 3.2 本地优先的协作软件
在过去的几年里，我和我的合作者一直致力于开发协作软件的新基础，即允许多个用户协作修改共享文件的应用程序。该文件可以是文本文档、绘图、电子表格、待办事项列表或许多其他类型的数据。在移动设备上运行的软件需要离线工作：例如，即使您的手机当前没有蜂窝数据覆盖，您也应该能够将一项添加到您的待办事项列表中。因此很明显，这种类型的应用程序在分类的部分有序、非窗口、持久部分中运行。

更具体地说，我们正在根据本地优先的方法设计这个软件[39]，其中用户访问其数据的每个最终用户设备都被视为使用其本地设备存储的完整副本。当用户修改他们的数据时，他们会立即更新其本地设备上的副本，即使它处于脱机状态，并且任何更新都会在下次网络连接可用时同步到其他设备上的副本。

我们使用 CRDT 来实现这种方法，因此描述数据更新的事件是由 CRDT 算法生成的，作为副本状态突变的副作用。如中所述第 2.6 节，该模型非常适合用户输入采用状态突变形式的协作软件。为此，我们开发了一个名为 Automerge1的 CRDT 库，它提供了 JSON 数据模型。该数据模型足以实现各种应用[25, 39]。

CRDT 生成的事件还用作文档已更改的通知，这使得编辑同一文档的多个用户之间可以进行实时协作。我们使用函数式反应式编程模型处理此类事件，如无花果- 3.用户应用程序的当前状态表示为 Automerge CRDT 数据结构。渲染函数采用此数据结构并将其转换为应显示在屏幕上的相应用户界面元素 [25]。在网络应用程序中，Facebook 的 React2是执行这种渲染的流行库。

呈现的用户界面具有附加的事件处理程序，当用户与相应的用户界面元素交互时调用这些事件处理程序（这些事件是短暂的）。当调用这样的事件处理程序时，它不会直接更新用户界面，而是改变 Automerge CRDT 状态以反映用户输入，然后再次调用渲染函数来更新用户界面。这种函数式反应式编程模型使软件易于推理，因为数据仅以一种方式流动：用户界面状态始终来自 Automerge 状态，而不是相反。

当用户因此改变 Automerge 状态时，CRDT 会生成一个更改事件，将其保存在本地，并使用消息传递中间件或同步协议将其发送到具有该文档副本的所有其他设备。当副本接收到来自远程用户的此类事件时，它会使用 CRDT 逻辑来更新本地 Automerge 状态，然后以与本地用户所做更改完全相同的方式调用渲染函数。使用与本地用户输入相同的代码路径和数据流进行实时协作可以显着简化编程模型。这种方法在我们关于 PushPin 的论文中有更详细的讨论[25], 我们的本地第一软件原型之一。

CRDT 生成的更改事件代表文档的编辑历史；我们可以将这些事件视为类似于 Git 提交历史中的提交。持久化这些事件提供了有用的功能：我们可以重建文档在过去任何时刻的状态，并计算文档版本之间的差异以可视化更改历史。我们还可以支持协作工作流，其中一个用户建议对文档进行更改，而另一个用户可以接受或拒绝更改：在 Git 术语中，一个用户可以创建一个分支（一组尚未包含在主文档版本中的提交)，其他用户可以选择是否合并。可以同时存在任意数量的分支，用户可以尝试将它们组合起来，看看如果分支被合并，文档的状态会是什么。启用此类高级工作流是将应用程序状态表示为一组部分有序的更改事件的直接结果。

## 4 结论和未来工作
事件在各种各样的系统中被广泛使用，本文系统地概述了基于事件的软件领域。的分类学第 2 节基于一些简单的标准对基于事件的系统进行分类：事件是否是持久的，它们是否充当通知（触发应用程序代码的执行），可以加入的事件是否有时间限制，事件是否完全或部分排序，以及应用程序表达变化的主要模型是通过改变状态还是通过生成事件。正如我们在本文中看到的，没有一种真正的方法：分类法中的所有类别都有重要的用例。我已经给出了这些用例的例子，并强调了一些关键的权衡，这些权衡决定了不同类别的优缺点，具体取决于具体情况。

最后，我将概述未来研究的一些开放性问题和挑战。

### 4.1	多版本/分支工作流程
大多数复制系统都基于这样一个假设，即每个副本都应该立即处理它接收到的每个事件，以便它的状态尽可能是最新的。然而，有时这实际上并不是我们想要的：正如在第 3.2 节，我们可能希望暂时对数据集的副本进行一些更改，允许多个用户检查和讨论这些更改，然后再决定是否将它们合并到数据集的主副本中。在源代码存储库中，我们一直使用分支、合并和拉取请求来执行此操作；为什么我们不能对其他形式的数据做同样的事情？

数据库事务通过中止事务允许提交或回滚未提交事务的写入来支持弱形式的多版本并发控制[8]。但是，大多数数据库不允许一个用户与另一个用户共享未提交事务的状态，并且大多数数据库不允许用户找出未提交事务中哪些数据发生了变化（相当于 git diff）。此外，在大多数数据库系统中，未提交的事务可能持有锁，从而阻止其他事务进行。

部分有序的基于事件的系统可以很好地支持这种分支和合并工作流，因为它们已经以事件的形式明确地进行数据更改，并且它们对并发更新的支持允许数据集的多个版本同时并存。 CRDT 为我们提供了一种机制来合并这些不同版本的数据集。然而，关于此类系统应如何处理数据版本化，包括比较和可视化数据集版本之间的差异，以及可以使用哪些内部表示系统来有效地处理此类多版本化数据，存在许多悬而未决的问题。

### 4.2 数据模型更改
许多基于事件的系统中的一个挑战是如何处理模式或数据格式的变化 [56]。在我们无法保证所有副本都运行相同版本的软件的系统中，这个问题尤其明显。例如，在最终用户设备上运行的应用程序中，由用户决定何时安装软件更新；因此，对安装更新犹豫不决的用户可能运行的应用程序版本比总是安装最新版本的用户要旧得多。尽管如此，这些用户应该能够尽可能地互操作，这意味着任何数据格式的更改都必须向前和向后兼容。如果用户能够自定义他们正在运行的软件，挑战会变得更大，比如通过用户的编程[30]。

在基于不可变事件的系统中，一种有前途的方法是使用双向函数（lenses）在数据模型的不同版本之间进行转换，这允许不同的副本使用不同的状态表示，同时仍然能够互操作 [49]。开放的问题包括这种类型的转换可以扩展多远，如何支持一系列不同的数据模型，如何使这种编程模型更容易被应用程序开发人员访问，以及如何使其高效。
 
### 4.3	无处不在的数据更改通知
对在事件流之上执行物化视图维护的系统越来越感兴趣：该领域的初创公司包括 Materialize（基于差分数据流 [51])、RelationalAI和 Event Store。这些努力的一个共同点是希望提供具有强语义的高级编程模型，而 Kafka 生态系统通常优先考虑可伸缩性和故障的低级工作宽容[11]。

尽管取得了这些进展，但我在 2014 年的 Turning the database inside-out 中提出的核心问题 [33]仍未解决。大多数应用程序逻辑仍然在请求-响应模型中执行：当应用程序接收到请求时，它会查询数据库并返回该时间点的结果，但是客户端无法订阅在查询结果更改时收到通知（其他而不是通过重复请求，即轮询，这是缓慢且低效的）。增量维护的物化视图提供了用发布-订阅范式替换请求-响应范式的潜力，在这种范式中，客户端不仅可以获得某些资源的当前状态，还可以订阅提供低延迟通知的事件流每当该状态发生变化时。

我们将 FRP 用于协作软件（第 3.2 节）是在最终用户设备上运行的应用程序软件上下文中这一想法的一个实例。然而，我们还没有看到当今基于云的面向服务/微服务系统的架构发生了类似的变化。改变这种现状需要在编程模型（应用程序开发人员如何表达系统状态必须如何因底层事件而改变？）和执行（系统如何有效地实现这些操作？）方面进行创新。增量维护的 SQL 查询具体化（例如第 2.2 节）是一个好的开始，但要充分实现这一愿景还需要做更多的工作，例如将任意业务逻辑合并到视图物化过程中。

## 致谢
感谢 Jean Bacon、Jamie Brandon、Mariano Guerra、Ian Lewis 和 Eiko Yoneki 对本文草稿的反馈。我感谢 Leverhulme Trust Early Career Fellowship、Isaac Newton Trust、诺基亚贝尔实验室和众筹支持者的支持，包括 Ably、Adrià Arcarons、Chet Corcos、Macrometa、Mintter、David Pollak、RelationalAI、SoftwareMill、Talent Formation Network 和亚当威金斯。

## 参考文献
[1] [n.d.]. Debezium. https://debezium.io/ 
[2] Tyler Akidau, Robert Bradshaw, Craig Chambers, Slava Chernyak, Rafael J Fernández-Moctezuma, Reuven Lax, Sam McVeety, Daniel Mills, Frances Perry, Eric Schmidt, and Sam Whittle. 2015. The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale, Unbounded, Outof-Order Data Processing. Proceedings of the VLDB Endowment 8, 12 (Aug. 2015), 1792–1803. https://doi.org/10.14778/2824032.2824076 
[3] Tyler Akidau, Slava Chernyak, and Reuven Lax. 2018. Streaming Systems: The What, Where, When, and How of Large-Scale Data Processing. O’Reilly Media. 
[4] Hagit Attiya, Amotz Bar-Noy, and Danny Dolev. 1995. Sharing memory robustly in message-passing systems. J. ACM 42, 1 (Jan. 1995), 124–142. https://doi.org/ 10.1145/200836.200869 3https://materialize.com/ 4https://www.relational.ai/ – disclosure: RelationalAI financially supports my work 5https://www.eventstore.com/ 
[5] Hagit Attiya, Faith Ellen, and Adam Morrison. 2015. Limitations of Highly- Available Eventually-Consistent Data Stores. In ACM Symposium on Principles of Distributed Computing (PODC). ACM, 385–394. https://doi.org/10.1145/2767386. 2767419 
[6] Engineer Bainomugisha, Andoni Lombide Carreton, Tom van Cutsem, Stijn Mostinckx, and Wolfgang de Meuter. 2013. A Survey on Reactive Programming. Comput. Surveys 45, 4, Article 52 (Aug. 2013), 34 pages. https://doi.org/10.1145/ 2501654.2501666 
[7] Philip A Bernstein, Sergey Bykov, Alan Geller, Gabriel Kliot, and Jorgen Thelin. 2014. Orleans: Distributed Virtual Actors for Programmability and Scalability. Technical Report MSR-TR-2014-41. Microsoft Research. https://www.microsoft.com/en-us/research/publication/orleansdistributed- virtual-actors-for-programmability-and-scalability/ 
[8] Philip A. Bernstein and Nathan Goodman. 1983. Multiversion Concurrency Control—Theory and Algorithms. ACM Transactions on Database Systems 8, 4 (Dec. 1983), 465–483. https://doi.org/10.1145/319996.319998 
[9] Dominic Betts, Julián Domínguez, Grigori Melnik, Fernando Simonazzi, and Mani Subramanian. 2012. Exploring CQRS and Event Sourcing. Microsoft. http: //aka.ms/cqrs 
[10] Kenneth P Birman, André Schiper, and Pat Stephenson. 1991. Lightweight causal and atomic group multicast. ACM Transactions on Computer Systems 9, 3 (Aug. 1991), 272–314. https://doi.org/10.1145/128738.128742 
[11] Jamie Brandon. 2021. Internal consistency in streaming systems. https:// scattered-thoughts.net/writing/internal-consistency-in-streaming-systems/ 
[12] Paris Carbone, Asterios Katsifodimos, Stephan Ewen, Volker Markl, Seif Haridi, and Kostas Tzoumas. 2015. Apache Flink: Stream and Batch Processing in a Single Engine. IEEE Data Engineering Bulletin 38, 4 (Dec. 2015), 28–38. http: //sites.computer.org/debull/A15dec/p28.pdf 
[13] Tushar Deepak Chandra and Sam Toueg. 1996. Unreliable failure detectors for reliable distributed systems. J. ACM 43, 2 (March 1996), 225–267. https: //doi.org/10.1145/226643.226647 
[14] Bernadette Charron-Bost, Fernando Pedone, and André Schiper (Eds.). 2010. Replication: Theory and Practice. Vol. 5959. Springer LNCS. https://doi.org/10. 1007/978-3-642-11294-2 
[15] Rada Chirkova and Jun Yang. 2012. Materialized Views. Foundations and Trends in Databases 4, 4 (Dec. 2012), 295–405. https://doi.org/10.1561/1900000020 
[16] Evan Czaplicki and Stephen Chong. 2013. Asynchronous Functional Reactive Programming for GUIs. In 34th ACM SIGPLAN Conference on Programming Language Design and Implementation (PLDI). ACM, 411–422. https: //doi.org/10.1145/2491956.2462161 
[17] Shirshanka Das, Chavdar Botev, Kapil Surlaker, Bhaskar Ghosh, Balaji Varadarajan, Sunil Nagaraj, David Zhang, Lei Gao, Jemiah Westerman, Phanindra Ganti, Boris Shkolnik, Sajid Topiwala, Alexander Pachev, Naveen Somasundaram, and Subbu Subramaniam. 2012. All Aboard the Databus! Linkedin’s Scalable Consistent Change Data Capture Platform. In 3rd ACM Symposium on Cloud Computing (SoCC). https://doi.org/10.1145/2391229.2391247 
[18] Susan B Davidson, Hector Garcia-Molina, and Dale Skeen. 1985. Consistency in Partitioned Networks. Comput. Surveys 17, 3 (1985), 341–370. https://doi.org/10. 1145/5505.5508 
[19] Giuseppe DeCandia, Deniz Hastorun, Madan Jampani, Gunavardhan Kakulapati, Avinash Lakshman, Alex Pilchin, Swaminathan Sivasubramanian, Peter Vosshall, and Werner Vogels. 2007. Dynamo: Amazon’s highly available key-value store. In 21st ACM Symposium on Operating Systems Principles (SOSP). ACM, 205–220. https://doi.org/10.1145/1294261.1294281 
[20] Martin Fowler. 2005. Event Sourcing. https://martinfowler.com/eaaDev/ EventSourcing.html 
[21] Seth Gilbert and Nancy A Lynch. 2002. Brewer’s conjecture and the feasibility of consistent, available, partition-tolerant web services. ACM SIGACT News 33, 2 (June 2002), 51–59. https://doi.org/10.1145/564585.564601 
[22] Victor B F Gomes, Martin Kleppmann, Dominic P Mulligan, and Alastair R Beresford. 2017. Verifying strong eventual consistency in distributed systems. Proceedings of the ACM on Programming Languages 1, OOPSLA (2017). https: //doi.org/10.1145/3133933 
[23] Jim N Gray, Pat Helland, Patrick O’Neil, and Dennis Shasha. 1996. The dangers of replication and a solution. In ACM SIGMOD International Conference on Management of Data. ACM, 173–182. https://doi.org/10.1145/233269.233330 
[24] Ashish Gupta and Inderpal Singh Mumick (Eds.). 1999. Materialized Views: Techniques, Implementations, and Applications. MIT Press. 
[25] Peter van Hardenberg and Martin Kleppmann. 2020. PushPin: Towards production-quality peer-to-peer collaboration. In 7th Workshop on Principles and Practice of Consistency for Distributed Data (PaPoC). ACM. https://doi.org/ 10.1145/3380787.3393683 
[26] Pat Helland. 2015. Immutability Changes Everything. In 7th Biennial Conference on Innovative Data Systems Research (CIDR). http://www.cidrdb.org/cidr2015/ Papers/CIDR15_Paper16.pdf 
[27] Pat Helland and Dave Campbell. 2009. Building on Quicksand. In 4th Biennial Conference on Innovative Data Systems Research (CIDR). https://database.cs.wisc. edu/cidr/cidr2009/Paper_133.pdf DEBS ’21, June 28-July 2, 2021, Virtual Event, Italy Martin Kleppmann 
[28] Joseph M Hellerstein, Michael Stonebraker, and James Hamilton. 2007. Architecture of a Database System. Foundations and Trends in Databases 1, 2 (Nov. 2007), 141–259. https://doi.org/10.1561/1900000002 
[29] Heidi Howard and Richard Mortier. 2020. Paxos vs Raft: have we reached consensus on distributed consensus?. In 7thWorkshop on Principles and Practice of Consistency for Distributed Data (PaPoC). ACM. https://doi.org/10.1145/3380787.3393681 
[30] Ink&Switch. 2019. End-user programming. https://www.inkandswitch.com/enduser- programming.html 
[31] David R Jefferson. 1985. Virtual time. ACM Transactions on Programming Languages and Systems 7, 3 (July 1985), 404 – 425. https://doi.org/10.1145/3916.3988 
[32] Ralph Kimball and Margy Ross. 2013. The Data Warehouse Toolkit: The Definitive Guide to Dimensional Modeling (3rd ed.). John Wiley & Sons. 
[33] Martin Kleppmann. 2015. Turning the database inside-out with Apache Samza. https://martin.kleppmann.com/2015/03/04/turning-the-database-insideout. html 
[34] Martin Kleppmann. 2017. Designing Data-Intensive Applications. O’Reilly Media. 
[35] Martin Kleppmann. 2020. Moving Elements in List CRDTs. In 7th Workshop on Principles and Practice of Consistency for Distributed Data (PaPoC). ACM. https://doi.org/10.1145/3380787.3393677 
[36] Martin Kleppmann, Alastair R Beresford, and Boerge Svingen. 2019. Online Event Processing. Commun. ACM 62, 5 (May 2019), 43–49. https://doi.org/10.1145/ 3312527 
[37] Martin Kleppmann, Victor B F Gomes, Dominic P Mulligan, and Alastair R Beresford. 2018. OpSets: Sequential Specifications for Replicated Datatypes (Extended Version). https://arxiv.org/abs/1805.04263 
[38] Martin Kleppmann and Jay Kreps. 2015. Kafka, Samza and the Unix Philosophy of Distributed Data. IEEE Data Engineering Bulletin 38, 4 (Dec. 2015), 4–14. http://sites.computer.org/debull/A15dec/p4.pdf 
[39] Martin Kleppmann, Adam Wiggins, Peter van Hardenberg, and Mark Mc- Granaghan. 2019. Local-First Software: You own your data, in spite of the cloud. In ACM SIGPLAN International Symposium on New Ideas, New Paradigms, and Reflections on Programming and Software (Onward!). ACM, 154–178. https: //doi.org/10.1145/3359591.3359737 
[40] Jay Kreps. 2013. The Log: What every software engineer should know about realtime data’s unifying abstraction. http://engineering.linkedin.com/distributedsystems/ log-what-every-software-engineer-should-know-about-real-timedatas- unifying 
[41] Jay Kreps, Neha Narkhede, and Jun Rao. 2011. Kafka: a Distributed Messaging System for Log Processing. In 6th International Workshop on Networking Meets Databases (NetDB). 
[42] Raffi Krikorian. 2012. Timelines at Scale. In QCon San Francisco. https://www. infoq.com/presentations/Twitter-Timeline-Scalability/ 
[43] Roland Kuhn. 2021. Local-First Cooperation. https://www.infoq.com/articles/ local-first-cooperation/ 
[44] Ajay Kulkarni and Ryan Booz. 2020. What the heck is time-series data (and why do I need a time-series database)? https://blog.timescale.com/blog/whatthe- heck-is-time-series-data-and-why-do-i-need-a-time-series-databasedcf3b1b18563/ 
[45] Leslie Lamport. 1978. Time, clocks, and the ordering of events in a distributed system. Commun. ACM 21, 7 (July 1978), 558–565. https://doi.org/10.1145/359545. 359563 
[46] Leslie Lamport. 2001. Paxos Made Simple. ACM SIGACT News 32, 4 (Dec. 2001), 51–58. 
[47] João Leitão, José Pereira, and Luís Rodrigues. 2007. HyParView: A Membership Protocol for Reliable Gossip-Based Broadcast (37th Annual IEEE/IFIP International Conference on Dependable Systems and Networks). IEEE, 419–429. https://doi.org/ 10.1109/dsn.2007.56 
[48] Linux Programmer’s Manual. 
[n.d.]. select(2) – Linux manual page. https: //www.man7.org/linux/man-pages/man2/select.2.html 
[49] Geoffrey Litt, Peter van Hardenberg, and Orion Henry. 2021. Cambria: Schema Evolution in Distributed Systems with Edit Lenses. In 8th Workshop on Principles and Practice of Consistency for Distributed Data (PaPoC). ACM, Article 8. https: //doi.org/10.1145/3447865.3457963 
[50] David C Luckham. 2002. The Power of Events: An Introduction to Complex Event Processing in Distributed Enterprise Systems. Addison-Wesley. 
[51] Frank McSherry, Derek G Murray, Rebecca Isaacs, and Michael Isard. 2013. Differential dataflow. In 6th Biennial Conference on Innovative Data Systems Research (CIDR). http://cidrdb.org/cidr2013/Papers/CIDR13_Paper111.pdf 
[52] Jayadev Misra. 1986. Distributed Discrete-Event Simulation. Comput. Surveys 18, 1 (March 1986), 39–65. https://doi.org/10.1145/6462.6485 
[53] Mozilla Developer Network. [n.d.]. Event reference. https://developer.mozilla. org/en-US/docs/Web/Events 
[54] Shadi A. Noghabi, Kartik Paramasivam, Yi Pan, Navina Ramesh, Jon Bringhurst, Indranil Gupta, and Roy H. Campbell. 2017. Samza: Stateful Scalable Stream Processing at LinkedIn. Proceedings of the VLDB Endowment 10, 12 (Aug. 2017), 1634–1645. https://doi.org/10.14778/3137765.3137770 
[55] Diego Ongaro and John K Ousterhout. 2014. In Search of an Understandable Consensus Algorithm. In USENIX Annual Technical Conference (ATC). USENIX. 
[56] Michiel Overeem, Marten Spoor, and Slinger Jansen. 2017. The dark side of event sourcing: Managing data conversion. In 24th IEEE International Conference on Software Analysis, Evolution and Reengineering (SANER). IEEE, 193–204. https: //doi.org/10.1109/SANER.2017.7884621 
[57] Kevin De Porre, Florian Myter, Christophe De Troyer, Christophe Scholliers, Wolfgang De Meuter, and Elisa Gonzalez Boix. 2019. Putting Order in Strong Eventual Consistency. In IFIP International Conference on Distributed Applications and Interoperable Systems (DAIS 2019). Springer, 36–56. https://doi.org/10.1007/ 978-3-030-22496-7_3 
[58] Yasushi Saito and Marc Shapiro. 2005. Optimistic Replication. Comput. Surveys 37, 1 (March 2005), 42–81. https://doi.org/10.1145/1057977.1057980 
[59] Fred B Schneider. 1990. Implementing fault-tolerant services using the state machine approach: a tutorial. Comput. Surveys 22, 4 (Dec. 1990), 299–319. https: //doi.org/10.1145/98163.98167 
[60] Marc Shapiro, Nuno Preguiça, Carlos Baquero, and Marek Zawirski. 2011. Conflict-Free Replicated Data Types. In 13th International Symposium on Stabilization, Safety, and Security of Distributed Systems (SSS 2011). Springer, 386–400. https://doi.org/10.1007/978-3-642-24550-3_29 
[61] Marc Shapiro, Nuno Preguiça, Carlos Baquero, and Marek Zawirski. 2011. A comprehensive study of Convergent and Commutative Replicated Data Types. Technical Report 7506. INRIA. http://hal.inria.fr/inria-00555588/ 
[62] Supreeth Shastri, Vinay Banakar, Melissa Wasserman, Arun Kumar, and Vijay Chidambaram. 2020. Understanding and Benchmarking the Impact of GDPR on Database Systems. Proceedings of the VLDB Endowment 13, 7 (March 2020), 1064–1077. https://doi.org/10.14778/3384345.3384354 
[63] Ben Stopford. 2017. Handling GDPR with Apache Kafka: How does a log forget? https://www.confluent.io/blog/handling-gdpr-log-forget/ 
[64] Douglas B Terry, Marvin M Theimer, Karin Petersen, Alan J Demers, Mike J Spreitzer, and Carl H Hauser. 1995. Managing update conflicts in Bayou, a weakly connected replicated storage system. In 15th ACM Symposium on Operating Systems Principles (SOSP). ACM, 172–182. https://doi.org/10.1145/224056.224070 
[65] Ivan Valkov, Natalia Chechina, and Phil Trinder. 2018. Comparing Languages for Engineering Server Software: Erlang, Go, and Scala with Akka. In 33rd Annual ACM Symposium on Applied Computing (SAC). ACM, 218–225. https://doi.org/ 10.1145/3167132.3167144 
[66] Marko Vukolić. 2015. The Quest for Scalable Blockchain Fabric: Proof-of-Work vs. BFT Replication. In IFIP WG 11.4 International Workshop on Open Problems in Network Security (iNetSec). Springer, 112–125. https://doi.org/10.1007/978-3- 319-39028-4_9 
[67] GuozhangWang, Joel Koshy, Sriram Subramanian, Kartik Paramasivam, Mammad Zadeh, Neha Narkhede, Jun Rao, Jay Kreps, and Joe Stein. 2015. Building a replicated logging system with Apache Kafka. Proceedings of the VLDB Endowment 8, 12 (Aug. 2015), 1654–1655. https://doi.org/10.14778/2824032.2824063 
[68] Alexey Zimarev. 2020. What is Event Sourcing? https://www.eventstore.com/ blog/what-is-event-sourcing
