# sentinel

## 如果在 failover 的过程中 sentinel leader 挂了怎么办  

### Demo1: 
假设现在有 redisA,redisB,redisC 三台 redis, 其中 redisA 为 master, reidsB 和 redisC 为 slave.  

### 背景知识:
#### <u>sentinel leader</u>(以下简称 leader) 执行 failover 的过程:  
1. 从 slave 中选出新 master
2. 提升新 master (向选出来 slave 发送 `SLAVE no one`)  
3. 向剩余的 slave 发送 `SLAVEOF` 命令，让它们成为新 master 的 slave

#### sentinel 周期 pub
sentinel 会周期执行  
```shell
PUBLISH __sentinel__:hello "{{s_ip}},{{s_port}},{{s_runid}},{{s_epoch}},{{m_name}},{{m_ip}},{{m_port}},{{m_epoch}} 
# s_ 表示 sentinel 本身的信息, m_ 表示主 redis 的信息
```

#### sentinel 监管 redis 状态
Sentinel 本地缓存着所有 redis 的配置（缓存的配置不一定与当前所有 redis 实例的状态一致)    
  
即使没有自动故障迁移操作在进行， Sentinel 总会尝试将当前的配置设置到被监视的 redis 实例上面。

以 Demo1 举例：
1. sentinel **认为** redisB 是 slave, 如果因为某种原因 redisB 变成 master, sentinel 会向 redisB 发送 `slaveof {{m_ip}} {{m_port}}`, 让 redisB 变回 slave.  
2. sentinel **认为** redisB 是 redisA 的 slave, 如果因为某种原因 redisB 不是 redisA 的 slave, sentinel 会让 redisB 变回 redisA 的 slave (向 redisB 发送 `slaveof {{m_ip}} {{m_port}}`).  
3. sentinel **认为** redisA 是 master, 如果因为某种原因 redisA 变成 slave, sentinel 会认为 redisA 发生故障了，以 failover 的方式重新选择 master.

不过，Sentinel 在对实例进行重新配置之前仍然会等待一段足够长的时间，确保可以接收到其他 Sentinel 发来的配置更新，从而避免自身因为保存了过期的配置而对实例进行了不必要的重新配置。

### 解答 
> 只考虑 leader 挂了后仍有三台以上 sentinel 在正常工作的情况，且不考虑其他意外事故

从 leader 挂的时间来分三种情况讨论:
1. leader 如果在 PUBLISH 新 master 之前挂了(可以简单理解为 leader 还未选出新 master)。其他 sentinel 会等待 failover 超时（默认是3分钟）, 然后开始下一次 failover  

2. leader 如果在 PUBLISH 新 master 之后且在发送 slave no one 之前挂了。
其他 sentinel 会等待 failover 超时, 然后开始下一次 failover。
与上一种情况不同的是，这里 `下一次 failover` 其他 sentinel 会认为新 master 出故障了  
```
以 Demo1 举例，假设 redisA 挂了:
1. leader 从 slaves 中选出了新主节点 redisB,

2. leader 之后 PUBLISH 的 master 信息都是 redisB 的相关的信息.  

3. 其他 sentinel 接收到 leader PUBLISH 的消息，会更新本地缓存的 redis 信息,
   把 redisB 设置为 master, 同时把 redisA 和 redisC 设置为 slave

4. 因为 leader 没有向 redisB 发送 `SLAVE no one`, 所以 redisB 一直还是以 slave 方式运行.

5. 在 sentinels 的视角, redisB 是 master, 但 redisB 以 slave 身份运行，  
   等到本次的 failover 超时， sentinels 会认为主节点 redisB 出了故障，所以会开始新一次 failover
```  

3. leader 在发送 slave no one 之后挂了。这种情况就依赖于 sentinel 管理 redis 状态的功能。如果 slave 不是新 master 的 slave, 其他 sentinel 会让这些 slave 成为新 master 的 slave
