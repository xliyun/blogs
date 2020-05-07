# Redis Sentinel

## Redis Sentinel故障转移 

多个sentinel发现并确认master有问题

选举出一个sentinel作为领导

选出一个slave作为master

通知其余slave成为新的master的slave

通知客户端主从变化 

等待老的master复活成为新的master的slave

## 主从复制问题

手动故障转移

写能力和存储能力受限（其他节点只是副本）

![1584771558380](F:\GITHUB-BLOG\hexo\source\_posts\RedisSentinel\1584771558380.png)

主节点slaveof no one

从节点slaveof new master成为主节点 

