## paxos算法是什么？

阶段一：Proposer生成提案(基于p2c的不变性)

1. Proposer选择一个新的提案编号M[n]，然后向Acceptor某个超过半数的子集成员发送Prepare请求。
1. 如果已经批准过小于M[n]的任何提案，Acceptor就返回已经批准的最大编号的提案值。并承诺不再批准编号小于M[n]的提案。

 如果响应过更大编号的请求，或者不小于当前编号的请求已批准，这个请求就可以忽略。如果响应/批准过的最大编号被预写日志持久化，就可以在节点故障或者重启时也能保证p2c的不变性。

阶段二：Acceptor批准提案

1. 如果Proposer收到了来自半数以上的Acceptor的响应结果，就可以发送[M[n],V[n]]提案的Accept请求。这里的V[n]是所有响应中编号最大的提案的值，如果半数以上的Acceptor都没有批准过任何提案，V[n]就可以任意选择。
1. Acceptor只要尚未响应过任何编号大于M[n]的Prepare请求，那么就可以接受这个M[n]的提案。

 批准的提案可以发送给特定的Learn集合，再扩散。可以通过选取主Proposer保证算法的灵活性，避免竞争Prepare请求的“死循环”。

## paxos算法的推导和证明

假设当前已经存在外部组件可以生成全局唯一的编号，用于标识提案(包括编号和值)。

paxos算法中的三种角色包括：Proposer, Acceptor和Learner。

## 推导过程

单一的Acceptor选定第一个提案，实现简单却有单点问题，所以paxos算法要求(p1):半数以上的节点是Acceptor，使集合每次增减都会至少有一个公共成员(作为革命的火种)保存上次选举结果。另外规定一个Acceptor最多只能批准一个提案，就可以保证最终只有一个提案被选定了。

在没有失败和消息丢失的情况下，只提出一个提案也可以被选出，就需要一个Acceptor必须批准第一个收到的提案。然后Acceptor可以相互多播通信，按照某规则迭代筛选，最终选定唯一提案。

不同的Proposer分别提出多个提案，是无法选定一个提案的。那么就要结合提案编号约定提案的约束条件(p2)：如果[编号M[0],值V[0]]的提案被选定了，那么编号大于M[0]的选定提案的值必须也是V[0]。满足这个条件的多个提案可以选定唯一提案。

对该约束条件前推2个充分条件：
(p2a)如果[编号M[0],值V[0]]的提案被选定了，那么编号大于M[0]，且被Acceptor批准的提案的值必须也是V[0]。

(p2b)如果[编号M[0],值V[0]]的提案被选定了，那么编号大于M[0]的提案的值都是V[0]。

## 数学归纳法证明

假设未知的更充分的约束条件可以使这个充分条件自然成立，那么这个充分条件就可以被第二数学归纳法的证明(从证明过程可以找到“未知的更充分的约束条件”的线索)，具体表述是：假设M[0]到M[n-1]的提案的值都是V[0]，证明M[n]提案的值也是V[0]。

M[0]的提案被选定是由半数以上的Acceptor批准的。提案范围扩展到M[n-1]，批准的提案也是V[0]。要扩展到M[n]，就需要保证M[n]提案的值也是V[0]。

原理上(p2c)，对于任意的M[n]和V[n]，如果[M[n],V[n]]提案被提出，那么就一定存在一个由半数以上的Acceptor组成的集合S，满足条件：S的成员没有批准过小于M[n]的提案，或者批准过的小于M[n]的最大编号提案的值也是V[n]。

在此条件下，由于批准M[n-1]的集合和批准M[n]的集合有公共成员，该公共成员批准的编号最大的提案是V[0]，那么，就可以保证M[n]提案的值也是V[0]。

这样，未知的更充分的约束条件就找到了，paxo算法对提案的生成方法就找到了。

这个原理(p2c)的不变性推导充分条件(p2b)的第二数学归纳法表述是：提案[M[0],V[0]]被选定了，对于按照上述生成方法产生的所有更大编号的提案[M[n],V[n]]，都存在V[n]=V[0]。

1. M[n]=M[0]+1时，由于提案[M[0],V[0]]已被选定，提出M[n]提案的必然前提是存在一个Acceptor的半数以上的子集S批准了小于M[n]的提案。理论上的最大编号为M[0]，而批准M[0]的Acceptor集合与S有交集，那么Proposer对V[n]的取值就会选择V[0]。
1. 第二数学归纳法假设编号从M[0]+1到M[n]-1的所有提案的值为V[0]，来证明编号M[n]提案的值也是V[0]。根据原理(p2c)，首先一定存在一个Acceptor的子集S，批准了小于M[n]的提案。那么编号M[n]提案的值只能是这个多数集S中编号小于M[n]的最大编号提案的值。如果对应的最大编号在M[0]到M[n]-1范围内，那么这个值就是V[0]了。否则如果对应的最大编号小于M[0]，由于会与批准提案[M[0],V[0]]的Acceptor有交集，那么这个值就已经是V[0]。

这个证明过程要小心。