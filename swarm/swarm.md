# docker swarm加快开发、部署
## docker理解
　　传统项目一般只会搭建到一台物理机，但是随着各种中间件加入、项目模块化、项目数量扩大，一台物理机已经满足不了我们的需求。这时虚拟机技术如Vagrant、virtl box被应用进来。
　　虚拟机很好解决了项目之间资源隔离的问题，但是更多的问题问题又出现了，比如虚拟机本身运行占用的资源多，部署慢，可控性差。
　　随着更轻量的lxc虚拟技术成熟，容器化技术快速上位，取代了vm在开发、运维中的角色。
　　传统开发中，开发、部署单位是项目。而基于docker的开发，项目被进一步封装，开发、部署单位变成了容器。这样做的好处在于，开发设计的时候就能把部署时的工作最小化，使得部署人员面向的是外观一致的容器，而非各式各样的项目、中间件。
## swarm mode理解
　　类似于计算机以前单核发展到一定程度会变成多核。docker也从以前单一节点变成了多个节点。
* swarm mode的重点在于overlay网络，它使得跨接点docker机可以做到服务发现和通信传输。
* 它可以维护服务的状态实现高可用。
* 兼容compose文件规则实现服务编排
* 自动判断负载情况实现服务的节点分配。
## swarm 集群搭建
　　一台docker机通过`docker swarm init`即可创建一个集群，一个集群包含多台docker机，但是一个docker机只能加入一个集群。其他docker机可以作为manager或worker加入集群，manager节点包含worker节点功能的同时，拥有控制服务的能力（分配任务），集群中一般建立多个manager节点，实现高可用，防止单点错误。加入命令在`docker swarm init`执行后会显示，其后可以在manager节点获取。命令：`docker swarm join-token worker`或`docker swarm join-token manager`
## swarm 服务部署
　　docker swarm有两种启动服务的方式
1. 单个启动 ``docker service create [OPTIONS] IMAGE [COMMAND] [ARG...]``
2. 批量启动 `docker stack deploy -c compose-file.yml STACK`

## 项目启动依赖问题
　　当中间件、项目比较多的时候，很容易出现项目之间的启动依赖问题，compose文件中的depends_on属性是没有效果的，docker给出的[解决方案](https://docs.docker.com/compose/startup-order/)是在容器的启动脚本上添加自定义启动条件判断。比如tomcat项目容器里面自定义脚本等待db可用后再执行默认的tomcat启动脚本。
　　如果各式各样的项目数量增多，要实现这样的全自动部署需要熟悉shell脚本知识，维护各类容器镜像改造，还要经过多次的测试。
　　针对这个问题，我的解决方案是做半自动部署，分批次进行服务启动。一般分为三批(三个compose文件)：base.yml、service.yml、apps.yml。使用stack deploy -c compose_file stack_name命令先进行base批次的部署，服务全部启动后，再进行下一个批次的部署（**注意**stack_name要一致）。新添加的服务根据启动时期配置到对应的批次中，这样就能解决服务依赖的问题。

## docker swarm的容错机制：
1. docker service检测服务状态，当出现服务的容器异常退出时，能自动创建一个新的容器继续提供服务。
2. docker swarm自动管理复制集群，指定每个服务运行多个相同的实例，提高冗余实现容错，同时提高处理能力。

## docker的负载均衡：
　　传统应用里面，一般使用nginx在七层网络上做负载均衡。它需要手动部署集群后，配置多个upstream，通过反向代理功能实现负载均衡。
　　在docker swarm中，引入了服务发现的功能，一个服务启动多个实例后，同一个docker stack里其他应用通过服务名即可访问到该服务下的某个实例，服务发现里面已经包含负载均衡的实现，是通过vip（docker默认，基于第四层网络，性能好）或者dns的方式。

　　
