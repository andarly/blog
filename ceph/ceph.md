# <center>rexray使用ceph做docker共享存储</center>
　　Docker Swarm使得分布式、集群的搭建部署速度提升了指数级别，原本的部署方式可能要用一天的时间，改用docker部署后可以减少到十分钟以内。  
　　Docker swarm一般用来部署无状态应用，应用可以在任何节点上运行，以便达到横向扩展。当然，我们也可以使用docker swarm部署有状态应用，但是有个限制问题就是：有状态应用节点转移后，数据部分不能跟着转移。  
　　Docker提供的解决方案是使用volume plugin，把数据存储到统一的地方，使得不同节点之间可以共享数据。Docker 默认存档到本地文件系统

## 官方解决

　　在查看完官方[所有的可用方案](https://docs.docker.com/engine/extend/legacy_plugins/#finding-a-plugin)后，并没有马上得到一个可行易用的方案：

 

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


## 技术选型
　　经过大量搜索，国内来说，rexray、flocker、glusterfs是volume plugin推荐方案top3。通过对比[rexray+ceph](https://rexray.readthedocs.io/en/stable/user-guide/storage-providers/ceph/)的方案比较符合我们的需求。
![rexray官网ceph模块图片](https://andarly.github.io/blog/ceph/pic1.png)

## Ceph
　　包含文件存储、块存储、对象存储三个存储方式，volume plugin使用的是块存储，桥接本地文件系统的接口。其他两个可当作附加服务，在业务应用内直接使用。另外，Ceph搭建比较简单，支持docker搭建，支持分布式，支持横向拓展。
### Ceph搭建
　　Ceph官方只有k8s上的搭建教程，但是docker store上有ceph的镜像，上面有比较详细的搭建说明。经测试，可以直接使用。[该镜像](https://store.docker.com/community/images/ceph/daemon)集成多个组件的镜像，通过启动命令参数，指定组件的类型。另外有针对不同组件的独立镜像。
### 部署失败恢复状态
`rm -rf /etc/ceph/* && rm -rf /var/lib/ceph/ && \
docker rm -f mon osd mgr`

### 部署集群监控服务
`docker run -d --net=host  --name=mon \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph \
-e MON_IP=10.32.3.147 \
-e CEPH_PUBLIC_NETWORK=10.32.3.0/24 \
ceph/daemon mon`  
运行这个服务后，会在/etc/ceph、/var/lib/ceph下生成配置文件：


### 复制配置文件到每个节点
`scp -r  root@10.32.3.147:/etc/ceph/* /etc/ceph/ && \
scp -r root@10.32.3.147:/var/lib/ceph/bootstrap*   /var/lib/ceph/`

在集群的情况下，其他节点需要知道mon初始化的集群，需要把mon生成的相关文件复制到每个节点上。这个步骤可以借助ceph-deploy工具实现。

### 部署管理节点
`docker run -d --net=host --name=mgr \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
ceph/daemon mgr`

### 部署存储节点
`sudo docker run -d --net=host --name=osd \
--privileged=true \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph:/var/lib/ceph \
ceph/daemon osd`
### Ceph服务测试
进入mon节点，执行ceph -s查看服务状态

## Plugin插件使用
插件使用的要求前提是rbd、ceph命令可用，按照官方文档，可以通过官方安装教程使用ceph-deploy install client-ip  在所有节点进行安装。经过偶然发现，安装ceph-common包，即可满足要求。
### 具体步骤
借助一台辅助机器，copy mon节点下/etc/ceph/文件夹中的配置文件到辅助机，




