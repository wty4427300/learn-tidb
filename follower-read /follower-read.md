# 一.follower read
这个特性的出现出要是因为在一个raft集群中主要的有leader副本对外提供服务,
follower主要负责时刻同步数据或者说当failover的时候投票切换leader.所以
当操作热点数据的时候就会出现leader被打满但是follower却冷眼旁观的情况.

第一个优化点:
如果follower也处理客户端的读请求,这样就可以分担leader的压力.

## read index
1.首先考虑第一个问题,如何保证在follower读到的一定是最新的数据呢?直接读取follower上最近的committed index上的数据肯定是不行的.因为raft是quorum-based的算法(也就是说最小基数)
,一条log的写入成功,并不需要所有的peers都写入成功,只需要多数节点同意就可以了,所以此时有些
follower上的本地数据还是老数据,这样就破坏线性一致性了.

2.当出现网络隔离的时候,原leader不是最新的leader,并且被隔离在了少数派这边,多数派选出了新的leader,但是老的leader没有感知,在任期内还是会给客户端返回老数据.

3.每次请求都走一次去quorum read(应该是走一边最小基数的节点,这样保证肯定能读到最新的数据),但是有点太重了.这里的根本问题是老的leader不确定自己是不是新的leader.优化很简单,在leader处理请求的时候确定自己是leader就好了.这个就是readindex.

在处理请求的时候记录当前leader最新commit index,然后通过一次quorum的心跳确保自己仍然是leader,确定之后返回这条记录就好.保证了线性一致性.尽管readindex仍然需要进行一次多数派的网络通信,但是通信只需要传输元信息,能极大的减少网络io,进而提升吞吐.

在 TiKV 这边比标准的 ReadIndex 更进一步，实现了 LeaseRead。其实 LeaseRead 的思想也很好理解，只需要保证 Leader 的租约比重选新的 Leader 的 Election Timeout 短就行，这里就不展开了。(这里的我的理解是我的租期比新的选举的超时时间短,这样当新的leader选出的时候的,老的leader肯定已经过期了)

## Follower Read
最土的办法就是将请求转发给leader,然后leader返回最新的committed的数据就好,但这样只是把follower作为leader的代理而已压力还是在leader上,并没有解决问题.一个解决方案就是leader告诉
follower最新的commit index就够了,因为无论如何,即使这个follower本地没有这条日志,最终这条日志迟早都会在本地apply.

TiDB 目前的 Follower Read 正是如此实现的，当客户端对一个 Follower 发起读请求的时候，这个 Follower 会请求此时 Leader 的 Commit Index，拿到 Leader 的最新的 Commit Index 后，等本地 Apply 到 Leader 最新的 Commit Index 后，然后将这条数据返回给客户端，非常简洁。所以我的理解这里异步处理就非常棒.

最后文章里面补充了一点,就是tikv的异步apply机制导致的一些问题.当leader告诉follower最新的commit index,但是leader对这条log的apply是异步进行的,在follower那边可能在leader apply前已经读到这条记录apply了,,这样在follower上就能读到这条记录了,但是leader上可能过一会才能读到.

这种 Follower Read 的实现方式仍然会有一次到 Leader 请求 Commit Index 的 RPC，所以目前的 Follower read 实现在降低延迟上不会有太多的效果。

## 最后
文章提到了写对于未来的畅想,就是当某个表是热点数据的时候,通过pd给tikv发送请求动态创建多副本,用作follower read,当热点过去之后就将副本删除.类似与k8s,对容器的动态管理一样.

https://pingcap.com/blog-cn/follower-read-the-new-features-of-tidb/