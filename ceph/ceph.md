###使用ceph volume plugin做docker共享存储
Docker Swarm使得分布式、集群的搭建部署速度提升了指数级别，原本的部署方式可能要用一天的时间，改用docker部署后可以减少到十分钟以内。
Docker swarm一般用来部署无状态应用，应用可以在任何节点上运行，以便达到横向扩展。当然，我们也可以使用docker swarm部署有状态应用，但是有个限制问题就是：有状态应用节点转移后，数据部分不能跟着转移。Docker提供的解决方案是使用volume plugin，把数据存储到统一的地方，使得不同节点之间可以共享数据。Docker 默认存档到本地文件系统

官方的列表
https://docs.docker.com/engine/extend/legacy_plugins/#finding-a-plugin
在查看完所有的可用方案后，并没有马上得到一个方案：

 

插件|	缺点	|优点
- | :-: | -: 
DigitalOcean、Virtuozzo、blockbridge	|收费	|
Beegfs、convoy、drbd、Infinit、ipfs、OpenStorage	|小众、难以学习|	
contiv	|无法使用，断更两年	|支持ceph
fuxi	|基于openstack，研究成本高	
flocker	|没有合适的backends	|大众化
gce-docker	|基于google云，收费	|
Quobyte|	不支持分布式	|Docker安装
GlusterFS	|无法docker部署	|官方文档好
Horcrux	|无法成功搭建，文档支持差	|Minio还是比较好搭建的
REX-Ray	|	|支持ceph

https://rexray.readthedocs.io/en/stable/user-guide/storage-providers/ceph/

###Rexray
国内来说，rexray、flocker、glusterfs是volume plugin最优方案top3。

![Aaron Swartz](图片1.png)