##  代码结构
```
/filter
|	|- filter.go
```
---

## filter.go

- `type RoutingFilter func(discovery.NetworkMember) bool` RoutingFilter 定义了一个针对 NetworkMember 的 predicate，用来断言对于一个 message 一个给定的 NetworkMember 是否可以被选中（其实就是一种过滤规则）##PS：discovery.NetworkMember 就是一个 peer 的代表##

- `func CombineRoutingFilters(filters ...RoutingFilter) RoutingFilter`　给定多个过滤规则（也就是所说的 predicate），返回多个过滤规则的逻辑与（logical AND）

- `func SelectPeers(k int, peerPool []discovery.NetworkMember, filter RoutingFilter) []*comm.RemotePeer` 给定一个 peer 池，过滤，从过滤后的 peer 中选择最多 k 个（如果超过 k 个peer 满足过滤条件，则随机选择 k 个） peer，将 peer 的相关信息组合成 `*comm.RemotePeer` 结构体，返回 `[]*comm.RemotePeer`（##PS：`&comm.RemotePeer{PKIID: p.PKIid, Endpoint: p.PreferredEndpoint()}`##）

- `func First(peerPool []discovery.NetworkMember, filter RoutingFilter) *comm.RemotePeer` 从给定的 peer 切片中，选择第一个满足过滤条件的 peer，拼接相关信息，返回 *comm.RemotePeer，如果没有则返回 nil

- `func AnyMatch(peerPool []discovery.NetworkMember, filters ...RoutingFilter) []discovery.NetworkMember`　给定 peer　切片与多个 filter，返回不满足所有 filter　的 peer　的切片

（ps：记住代表 peer 的 discovery.NetworkMember　类型与 comm.RemotePeer 之间的关系）
（ps： discovery.NetworkMember　结构体下有 Endpoint 与 InternalEndpoint　两个字段，这两个字段之间的区别和联系是什么？）



