# docker使用rexray基于ceph做共享存储
## 背景
　　Docker Swarm使得分布式、集群的搭建部署速度提升了指数级别，原本的部署方式可能要用一天的时间，改用docker部署后可以减少到十分钟以内。  
　　Docker swarm一般用来部署无状态应用，应用可以在任何节点上运行，以便达到横向扩展。当然，我们也可以使用docker swarm部署有状态应用，但是有个限制问题就是：Docker 默认存档到本地文件系统，有状态应用节点转移后，数据部分不能跟着转移。  
　　Docker提供的解决方案是使用volume plugin，把数据存储到统一的地方，使得不同节点上的不同容器之间可以共享数据。  
 　　下面搭建为了避免swarm相关知识和操作，仅仅基于docker，如果有swarm相关知识，很容易把它应用到swarm中。

## 技术选型

*官方插件列表*  

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
REX-Ray	|	|

　　在查看、对比完官方[所有的可用方案](https://docs.docker.com/engine/extend/legacy_plugins/#volume-plugins)后，并没有马上得到一个可行易用的方案：
　　经过大量搜索，国内来说，rexray、flocker、glusterfs是volume plugin推荐方案top3。通过进一步对比,并且发现rexray支持ceph后,认为[rexray+ceph](https://rexray.readthedocs.io/en/stable/user-guide/storage-providers/ceph/)的方案比较符合我们的需求。

## Ceph简介
　　[ceph](http://docs.ceph.org.cn/)包含文件存储、块存储、对象存储三种存储服务，rexray/rbd使用的是块存储，使用该插件之后，原本存储到本地文件的方式变成存储到ceph服务中。另外两个可当作附加服务，在业务应用内直接使用。另外，Ceph搭建比较简单，支持docker搭建，支持分布式，支持横向拓展（添加硬件后启动osd服务加入ceph集群即可）。  
　　Ceph官方只有k8s上的搭建教程，但是docker store上有[ceph的镜像](https://store.docker.com/community/images/ceph/daemon)，上面有比较详细的搭建说明。虽然玩不转k8s，但是有了容器镜像的使用说明，把它部署成swarm mode是轻而易举的事情。该镜像集成多个组件的镜像，通过启动命令参数，指定组件的类型。另外有针对不同组件的独立镜像。
## 搭建环境
三个docker节点。标记为
- B1(10.32.3.147)
- B2(10.32.3.148)
- B3(10.32.3.149)

## Ceph搭建
- 启动mon
- 启动mgr
- 启动osd
- 查看ceph状态
- ceph demo快速启动
- 恢复清理

### 启动mon
* B1上初始化集群，mon是monitor的缩写，用于监控集群。  
`docker run -d --net=host  --name=mon \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph \
-e MON_IP=10.32.3.147 \
-e CEPH_PUBLIC_NETWORK=10.32.3.0/24 \
ceph/daemon mon`  
运行这个服务后，会初始化一个ceph集群，并在/etc/ceph、/var/lib/ceph下生成配置文件。
* 运行`docker exec mon ceph -s` 查看集群状态,显示不存在mgr、osd服务的警告

　　在集群的情况下，其他服务要加入mon初始化的集群，需要把mon生成的相关文件复制到相应节点上再启动服务。 
***注意***mon服务类似于zookeeper，可在B2、B3运行多个mon防止单点故障,
### 启动mgr
* mgr是管理集群的服务，可以运行在跟mon同一个主机上，但为了体现ceph的分布式特性，我们选择在B2启动mgr服务
* 进入B2，从B1 copy文件到B2  
`scp -r  root@10.32.3.147:/etc/ceph/* /etc/ceph/` 
`scp -r root@10.32.3.147:/var/lib/ceph/bootstrap*   /var/lib/ceph/`
* B2上启动mgr  
`docker run -d --net=host --name=mgr \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph/ \
ceph/daemon mgr`  
* B1上运行`docker exec mon ceph -s` 查看集群状态。


### 启动osd
osd是ceph中接受，存储数据的服务。
* 进入B3，从A copy文件到B3
`scp -r  root@10.32.3.147:/etc/ceph/* /etc/ceph/` 
`scp -r root@10.32.3.147:/var/lib/ceph/bootstrap*   /var/lib/ceph/`
* 启动osd  
`sudo docker run -d --net=host --name=osd \
--privileged=true \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph:/var/lib/ceph \
ceph/daemon osd`
* osd可以通常部署多个，但是一个docker节点只能有一个osd服务。在swarm环境下，为了最大化硬盘的使用，可以在所有节点上都启动osd服务。

### 查看ceph状态
进入mon容器，执行ceph -s查看服务状态。快速命令
```
docker exec ${mon_container_id} ceph -s
```  
当集群状态为  
health HEALTH_OK  
pgmap  active+clean  
表示集群可用

### 快速启动demo
基于ceph的搭建比较繁杂，如果想测试rexray/ceph插件的可用性，可以使用以下命令启动一个包含所有服务的容器。   
``
docker run -d --net=host --name=ceph \
-v /etc/ceph:/etc/ceph \
-v /var/lib/ceph/:/var/lib/ceph \
-e MON_IP=10.32.3.147 \
-e CEPH_PUBLIC_NETWORK=10.32.3.0/24 \
ceph/demo
``

### 恢复清理
一门技术的入门难免会遇到很多问题，以下命令用于重新搭建前做清理。  
`rm -rf /etc/ceph/* && rm -rf /var/lib/ceph/ && \
docker rm -f mon osd mgr`

### 可能出现的问题
* ceph需要开放的端口  
`firewall-cmd --zone=public --add-port=6789/tcp --permanent && \
firewall-cmd --zone=public --add-port=6800-7300/tcp --permanent && \
firewall-cmd --reload`



## ceph插件使用 
- 安装rbd、ceph客户端
- 安装rexray插件
- 测试统一存储

### 安装rbd、ceph客户端
　　插件使用的要求前提是安装rbd、ceph相关包，使命令可用，按照官方文档，可以通过官方安装教程使用ceph-deploy install client-ip  在指定节点进行安装，但是安装的包比较多而且容易报错。经过偶然发现，安装ceph-common包，即可满足要求，而且安装依赖少，时间快。如果能解决报错问题，推荐使用官方方法，避免以后出现意想不到的事情。  
 　　安装前，首先确保docker宿主机器上/etc/ceph/存在集群配置文件。因为之前ceph集群搭建后，B1-3三台机器都已经存在了配置文件，所以可用忽略这一步。对新加入swarm的机器，需要注意这个步骤。  
B1-3安装rbd、ceph包：执行`yum install ceph-common -y`  
* 测试命令：  
rbd:`rbd ls`  
ceph:`ceph -s`
### 安装rexray/rbd插件
* B1-3上执行  
`docker plugin install rexray/rbd`  
此插件下载比较慢，当swarm集群节点比较多时，建议转存到私有库后再下载。插件没有镜像一样使用build -t的操作。插件转到私库执行以下步骤：  
1.使用别名参数拉取插件`docker plugin install --alias  10.32.3.112:5011/rexray/rbd rexray/rbd`  
2.存储到私库`docker push 10.32.3.112:5011/rexray/rbd`
### 测试统一存储
有两个方式创建数据卷、启动容器：  
1.自动  
`docker run --volume-driver 10.32.3.112:5011/rexray/rbd -v test:/test -it --rm busybox sh`  
2.手动  
`docker volume create -d  10.32.3.112:5011/rexray/rbd test`
`docker run -v test:/test -it --rm busybox sh`
* 启动报错unable to map rbd时手动执行：  
`rbd map test --name client.admin`  
这可能是插件的一个bug。
* 在B1启动成功后，进入test目录创建文件"can_you_see_me"。
* 退出容器进入B2，`rbd ls`查看到B1上创建的目录。
![rbd ls](https://andarly.github.io/blog/ceph/pic3.png)
* 使用相同命令在B2启动容器，检验是否能在B2上访问B1中创建的文件。
![rbd ls](https://andarly.github.io/blog/ceph/pic4.png)
至此，docker使用rexray基于ceph做统一共享存储搭建完成。

## 技巧/工具
### 你需要一台辅助机器
* 辅助机上，你可以搭建maven私库、docker私库、jenkins等不同环境共用的工具、项目。
* 按照ceph官方搭建教程，需要一台辅助机安装ceph-deploy,再控制其他节点安装部署ceph集群。
![ceph install](https://andarly.github.io/blog/ceph/pic5.png)   
虽然我没有使用官方搭建方案（原生系统搭建），但是作为一种思想，我们可以在这台机器上做免密配置，从而快速的控制不同环境下的swarm集群节点
### 你需要学ansible
* 当你的集群包含5台以上的机器时，为了把ceph的配置文件copy到五台docker机上，你可能需要花1分钟。当你的集群包含20台以上时你觉得只需要花4分钟。但是，你确定你不会遗漏？
* ceph-common、rexray/rbd等包都需要手动安装，安装的时候还非常依赖网速，当你知道ansible可以并行控制n台机器，你会毫不犹豫的点开[这里](http://www.ansible.com.cn/)。
## 遗留问题
　　一个数据卷被多个节点使用过，只要在一个节点删除，ceph上也就同时删除，但是其他节点还保留着对这个数据卷的映射，导致无法重新创建删除的数据卷，除非手动删除映射关系或者修改数据集名称。