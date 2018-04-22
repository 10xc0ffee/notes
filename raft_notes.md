# RAFT 相关小问题

### Leader election 是如何进行的？

raft的leader election阶段类似使用paxos算法谋求一个共同的值(leader)。大致阶段如下：

1. 每个Proposer初始得到一个election timeout。这个election timeout是在一个范围内随机的。
2. 若max election timeout超时未收到其他proposal，则向quorum发布leader是自己的proposal。
3. 若得到majority acceptor认可，则成为leader。

每个proposal都附带一个sequence number，在raft中称为term。term标志新一轮election的产生。算法允许不同proposer使用相同term propose the leader。

### 为什么所有写操作（包括leader election）都是majority based？

A majority got newest value => any majority group contains the newest value => some minority group don't have the newest value

Then we can choose to stop the service when there are only minority accessible. In the other hand, service can be online only when there are majoirty cab be accessible. That's basically pigenon hole principle.

由于我们没法知道全局中有哪些replica含有最新的数据，是否能做leader都是通过比较判断的。如果我们有全局的信息实时记录谁有最新的数据，我们就能打破majority的限制。一些基于failure detector的协议就是这么做的。

### 如何保证leader的uniqueness

raft算法本身并不维护leader的唯一。raft只维护在相同term下只会产生一个leader。所以算法是会出现同时存在多个replica认为自己是leader的情况。

* 对于写请求，由于只有newest term leader拥有majority的ISR，所以写请求只会在这个leader上成功。
* 对于读请求，可能读到旧值。

整体上基本的raft算法能提供sequential consistency，却达不到linearizability。对于有些需求外部时间或者因果一致的应用，有的会增加了lease保证leader的uniqueness，有的在读请求上做文章，保证读的时候还是leader否则读失败。但是有得必有失，这种extension增加了算法对时间的依赖，即分布式环境中time drift对应用的影响。

### 新上任的Leader什么时候可以对外服务？

在raft中，成为leader并不意味这你确定所有数据已经commit，只能确定committed数据和最新数据（to-commit数据）是你的数据的子集。所以raft中新leader不能直接服务，需要经过数轮AppendEntries commit一个当前term的占位entry才行 （我有个疑问为什么必须提交一个真实的当前term的dummy entry。空的AppendEntries作为心跳也可以把之前lag的entries同步吧？虽然这样有违raft commit的原则，但是我觉得这个时候这些previous term的entries是可以commit的了）。

### 谁可以commit数据？

Leader可以commit数据，不仅仅是leader with newest term。old leader可能会commit数据并告知clients。raft保证old leader的后继leader肯定会commit一遍同样的entries。最新leader也同样包含相同的entries。
