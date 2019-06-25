据说:  
redis cluster 伸缩容麻烦  

## 引言

在 redis cluster 出现之前,通常通过以下几种方式搭建redis集群:

### 客户端分片(Smart Client)

这种方案将分片工作放在业务程序端，程序代码根据预先设置的路由规则，直接对多个Redis实例进行分布式访问。  
这样的好处是，不依赖于第三方分布式中间件，实现方法和代码都自己掌控，可随时调整，不用担心踩到坑。  
Smart Client 性能比 Proxy 更好（少了一个中间分发环节）, 但升级麻烦, 修改分片逻辑需要重启客户端。

### 代理分片(Porxy)

这种方案将分片工作交给专门的代理程序来做。
代理程序接收到来自业务程序的数据请求，根据路由规则，将这些请求分发给正确的Redis实例并返回给业务程序。  
这种机制下，一般会选用第三方代理程序，因为后端有多个Redis实例，所以这类程序又称为分布式中间件。
这种方式业务程序不用关心后端 Redis 实例, 但是会带来些性能损耗。  

## Redis Cluster

### node & slot

redis cluster 有多个 node, 每个 node 包含若干个 slot。  
redis cluster 有且仅有 10384 个 slot.  

![slot](https://raw.githubusercontent.com/ForeverEnjoy/markdown-images/master/b653d7fcdc8e4c5db46628265e1e173d-redis-cluster-node.png)

### client 和 redis cluster 建立连接过程：

￼client 和 redis cluster node 建立连接的过程与 client 和 redis standalone(单例模式) 的连接过程没有区别。

在 ￼client 只需要配置集群部分的 node 信息。启动 client 时，client 与其中一个 node 建立连接后，可以通过 CLUSTER SLOTS 命令获取集群的所有状态信息，然后缓存 node 的连接信息和 slots 与 nodes 的关系表到本地

￼当 client 执行 redis command 时，首先计算 key 所在的 slot ( redis cluster 约定了 hash 的算法,  client可以通过 hash 算法计算 key 的 slot 值)， 然后通过 slot 获取 node, 最后把请求发送给 node。

ps : 理论上客户端拿到集群所有 node 的信息，就可以向任何一个 node 发送请求，但是如果请求的命令带有 key, 且 key 的 hash slot 不属于当前 node, node 会返回一个需要重定向的错误信息( MOVED {{slot}} {{ip:port}} ) ，告诉 client 应该请求哪个 node（类似 http 的重定向)。

当服务器进行扩容，迁移数据的时候，client 的路由表不会立即更新，而是在被迁移的Slot上操作时，因为Slot已经不在原先的节点上了，Redis Cluster 返回 Moved 指令，告诉客户端该 slot 现在所在的节点。此时，客户端应该更新自己的路由表信息.

## redis cluster 不支持的命令：

￼redis cluster 要求 multi-key commands, transactions and Lua scripts  的 key 必须在同一个 slot。

￼有些命令不能在集群的层面执行，比如 scan 命令只能针对 node。

Ref:  
https://mp.weixin.qq.com/s/-AhrpD2JfPxAtIhyfSdYOA
