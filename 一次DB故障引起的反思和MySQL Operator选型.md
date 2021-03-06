# 前言

在一次数据库故障后，我们发现业务库会根据业务的等级会划分多个 MySQL 实例，许多业务库会同时属于一个 MySQL 实例，当一个库引发问题后整个实例的状态是不可控的。从而导致这个实例上的所有业务不稳定甚至造成中断。

# 故障反思

## 微服务架构

![](https://i.loli.net/2020/01/21/eO4fIJC87bpQNFS.jpg)

微服务架构在公司已经采用并坚持了近十年，我们也从传统的 VM 部署架构转身至当下的“容器+K8S”，无论是软件架构还是部署架构，我们都朝着高可用，可伸缩，快速迭代的目标演进。

通过上述技术，我们将应用的可用性提升到了一个新的高度，不幸的是我们还是遇见了故障。

## 木桶效应

![](https://i.loli.net/2020/01/21/WPckpm5yEo4KdsC.png)

木桶效应是讲一只水桶能装多少水取决于它最短的那块木板。一只木桶想盛满水，必须每块木板都一样平齐且无破损，如果这只桶的木板中有一块不齐或者某块木板下面有破洞，这只桶就无法盛满水。

## 墨菲定律

**墨菲定律不是一种心理学效应，是一种数学推理。**

如果有两种或两种以上的方式去做某件事情，而其中一种选择方式将导致灾难，则必定有人会做出这种选择。根本内容是：如果事情有变坏的可能，不管这种可能性有多小，它总会发生。

## 总结

任何事情都有发生的概率，当短板缺陷暴露后会造成毁灭性打击。

# 找到短板

可以看到，我们这次故障是因为一个业务库发生了问题，从而影响了这个业务库所在数据库实例上的所有业务库。  
为什么不是那个业务库的具体问题呢？我们永远无法避免一个库永远不会出现问题，但我们应该要尽量保障在单个库故障的时候其它的库和业务尽量不被影响。

现代软件架构中可以发现，我们的应用架构一般是朝微服务架构的演讲，但我们的基础设施、中间件层（数据库、MQ 等）确迟迟没有进行“微”化。

## 我的方案

核心数据库按现状进行管理，由 DBA 进行悉心照顾。  
边缘数据库则进行拆库、隔离。

# “微数据库”？

通过上诉的问题，我们可以做的一件事情是拆分数据库，将各个业务库拆分出不同的实例。通过资源限制策略（VM、容器？）将数据库所使用的资源进行隔离。  
这样当一个库引起故障（节点故障，CPU 跑满、IO 跑满、内存溢出）时只要整体计算资源足够，其它业务库就不会有影响。

# 如何“微数据库”？

经过这次事件后，我得出的结论是拆库，降低故障影响面，但如何拆库呢？  
从最低保障来说，我们至少要保障 1 节点故障的可用度，也就是说一个数据库实例至少有一个从库，也就说 2 个数据库实例(2 个计算节点)。  
如果按传统的方式来管理，一个业务一个库（2 个实例、2 个节点）按照公司上百个数据库来带来的硬件成本和维护成本是巨大的。

## 业务现状

公司虽然业务繁杂，有着非常多业务线和其衍生出的软件工程，但核心业务其实非常少，核心库就更少了。

# 容器 + K8S

好在我们在 2018 年后全部步入了“容器+K8S 模式”，有着丰富的容器和 K8S 使用和运维经验，我们开启了数据库容器化的探索。

## 方案选型

如果使用容器 + K8S 进行数据库运维，目前有两种方案可以选：

1. 直接使用 StatefulSet 运行 MySQL 镜像
2. 使用 MySQL Operator 进行管理（Operator 是 K8S 中的一种对外提供扩展的能力）

我们选择了方案 2。  
原因是 Operator 维护成本更低，更为简单。缺点是不如直接使用 StatefulSet 灵活，但对于非数据库专业人员来说，绝大多数情况 MySQL Operator 会比我们考虑的更周到。

## MySQL Operator 选型

关于 MySQL Operator，选择性其实很少，目前看来还能进行选择的只有两个，一个是 oracle 工作人员开源的，还有一个是社区的，两个项目整体活跃度都不高。

先说结论：我们选择了 **presslabs/mysql-operator**。
为什么我们没用选用以 oracle 官方明义发布的 operator 呢？  
主要是因为官方的库除了数据上活跃一些，其实它并不活跃，而且比较老旧，也只支持 MySQL 8.0+，现状大多数 MySQL 应该还是在 5.7+。  
下面可以看下具体的分析。

### oracle/mysql-operator

> https://github.com/oracle/mysql-operator

项目情况：  
652 star  
303 issues (64 open)  
151 commits

#### 特点

1. 使用 oracle 官方发布的 MGR 高可用架构
2. 自动、按需备份
3. 自动故障检测和恢复
4. 从备份还原

#### 不足

1. 版本老旧
   不支持 k8s 1.16+的版本，因为使用了过时的 k8s api，在较新的 k8s 集群上现状需要修改源码。我们在调研的时候也修改了源码以支持新版本的 k8s。
2. 只支持 MySQL8.0+版本
3. 不够活跃，最后有用的提交在一年前
4. 是 oracle 工作人员利用空闲时间维护的
5. 备份不支持备份到 pv，只能备份到谷歌云、S3 兼容的环境上

### presslabs/mysql-operator

> https://github.com/presslabs/mysql-operator

项目情况：  
289 star  
245 issues (74 open)  
568 commits

#### 特点

1. 使用 Orchestrator 完成高可用（github 在用开源的项目）
2. 自动、按需备份
3. 自动故障检测和恢复
4. 从备份还原
5. 集成 mysql prometheus 指标导出
6. Orchestrator 有提供 WebUI，可以方便的进行集群管理
7. 支持异步、半同步、同步复制

#### 不足

1. 非官方支持
2. 目前不支持 MySQL8+
3. 不支持 Helm3 方式安装（这个我们有找到解决方案）
4. 备份不支持备份到 pv，只能备份到谷歌云、Azure 或 S3 兼容的环境上
5. 最多支持双主

# 思考

## 为什么数据库等基础设施很少运行在容器中？

这个问题我有在很多线上社群和线下跟很多人讨论过，中间也产生过非常激烈的探讨。

### 支持运行在容器中

1. 容器是未来的趋势
2. 容器带来的性能损耗是有限的，绝大多数数据库不需要那么极端的性能

### 反对运行在容器中

1. 容器不稳定
2. 容器性能低
3. 容器的未知性可能会造成不可逆的故障（数据丢失等）
4. 数据持久化有问题

### 我的看法

数据库容器化是趋势，是必然的，只是当下并不能全部容器化。  
这边涉及到的内容会比较多，如果有不太了解的可以搜索引擎一下。  
首先这个问题有几个解读：

1. 数据库就单只 MySQL 吗？
2. 数据库能不能放进 K8S？
3. K8S 跟容器的关系？
4. 容器就是 Docker 吗？
5. 数据库能不能用 Docker 运行？
6. MySQL 能不能用容器运行？
7. MySQL 能不能用 Docker 运行？

#### 数据库就单只 MySQL 吗？

这个答案当然是否定的，Redis、MongoDB、Etcd、MySQL、Oracle、SQLServer 等都算作数据库。

#### 数据库能不能放进 K8S？

需要讨论的本质是容器与数据库，跟 K8S 并没有关系，如果抱着这个问题大家可以直接排除，不用带入 K8S。

#### K8S 跟容器的关系？

K8S 是调度平台，容器的运行跟他没任何关系，当 K8S 不可用容器不会发生任何变化，除非容器内的程序自己改变自己的状态（异常退出等）。

#### 容器就是 Docker 吗？

否，容器当下最热的实现是 Docker（在细分里面还有 containerd 等不细分了，这边就指通过 cgroups 技术实现的容器解决方案），还有像 Kata、gVisor 这样非 cgroups 技术的实现。简单来说一个容器可以是一个**极度精简后的虚拟机**。

#### 数据库能不能用 Docker 运行？

有些可以有些不行，Redis 等通过 Docker 运行业界已经有很多大厂案例了，Oracle 肯定是不能容器化的了。

#### MySQL 能不能用容器运行？

上面说了，容器不单只 cgroups 技术的实现，这个问题大家可以理解成 MySQL 能不能运行在虚拟机中？如果你们现有的 MySQL 运行在裸金属物理机上才需要考虑，否则那么一定是可以的。

#### MySQL 能不能用 Docker 运行？

其实这个才是大家最常见的理解方式，MySQL 能否运行在以 cgroups 技术实现的容器中。  
阿里很早就开始将 cgroups 运用在 MySQL 中了。  
Docker 封装了 cgroups 的复杂度，提供了实际使用所需的许多内容，我觉得是可以使用 Docker 运行的。

#### 常见误区

1. 在 K8S 中数据持久化非常麻烦，Pod 重启数据会丢失，除非使用共享存储，但共享存储性能又很低。  
   K8S 中可以将 Pod 中的路径直接映射到本地磁盘，同时还可以使用 PV、PVC 的概念，Local Persistent Volume。相当于直接读写 Node 所在的磁盘。
2. 容器中会丢失数据
   以 cgroups 实现的容器技术，其实还是 Node 上的一个进程，权限和待遇跟其他进程并没有太多区别，如果丢失数据那么普通进程也会存在这样的问题，大多数数据丢失并不是容器的锅，而是架构或使用不当引起的。

#### 确实存在的问题

1. 性能下降  
   容器性能下降主要来源于网络 IO。  
   磁盘、内存、CPU 等损失几乎没有。  
   网络 IO 造成的性能损失幅度还会跟 CNI 选型有关，现在基于 3 层网络转发的 CNI 插件（Calico、Flannel host-gw 等）来说损失度非常有限。  
   任何包装和封装都会对性能造成影响，这边大家需要评估的是这些损失的性能有没有造成很大影响？对比于带来的优势来说值不值得？
2. 不稳定  
   任何新的尝试都没有人能保证 100%没有问题，现阶段确实没有人敢将数据库放入容器中在核心业务中使用（在边缘业务中也有许多公司尝试）

### 数据库容器化总结

我觉得可以尝试边缘业务的数据库进行容器化，这也是未来的趋势。  
核心数据库（公司命脉）除非你有很大的把握，否则不要去动。

其实可以发现除传统数据库外，新起的分布式数据库都将 K8S、容器当成一等公民（TiDB 等）

# 写在最后

由于篇幅原因，将 “presslabs/mysql-operator” 部署和使用的内容放到后续的文章中，将会涉及 Local Persistent Volume 等内容。
