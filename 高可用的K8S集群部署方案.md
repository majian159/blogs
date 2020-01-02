# 涉及到的内容
1. LVS
2. HAProxy
3. Harbor
4. etcd
5. Kubernetes (Master Worker)

# 整体拓补图
![](https://ws1.sinaimg.cn/large/7ecacd23ly1g6nalv8ub6j20z80trq3p.jpg)
以上是最小生产可用的整体拓补图（相关节点根据需要进行增加，但不能减少）

按功能组划分
1. SLB
   - LVS
   - HAProxy
2. etcd
3. K8S Node (Master / Worker)

# SLB
LVS 、HAProxy 被规划为基础层，主要提供了一个高可用的7层负载均衡器。  
由LVS keepalived 提供一个高可用的VIP（虚拟IP）。  
这个VIP DR模式转发到后端的HAProxy服务器。  
HAProxy反代了K8S Master服务器，提供了K8S Master API的高可用和负载均衡能力。

## 可以使用Nginx代替HAProxy吗？
是可以的，这边使用HAproxy是因为k8s文档中出现了HAproxy，且后续可能会有4层反代的要求，从而使用了HAProxy。

## 可以直接从LVS转发到Master吗？
理论上可行，我没有试验。  
如果不缺两台机器推荐还是架设一层具有7层代理能力的服务。  
k8s apiserver、harbor、etcd都是以HTTP的方式提供的api，如果有7层代理能力的服务后续会更容易维护和扩展。

## 推荐配置
| 用途 | 数量 | CPU | 内存 |
| ----------- | :---: | :---: | :---: |
| Keepalive   |   2   |   4   |  4GB  |
| HAProxy     |   2   |   4   |  4GB  |

# etcd
etcd是一个采用了raft算法的分布式键值存储系统。  
这不是k8s专属的是一个独立的分布式系统，具体的介绍大家可以参考官网，这边不多做介绍。  
我们采用了 static pod的方式部署了etcd集群。

## 失败容忍度
**最小可用节点数：(n/2)+1**，下面是一个参考表格，其中加粗的是推荐的节点数量：

| 总数 | 最少存活 | 失败容忍 |
| :--: | :--: | :--: |
| 1 | 1 | 0 |
| 2 | 2 | 0 |
| 3 | 2 | **1** |
| 4 | 3 | 1 |
| 5 | 3 | **2** |
| 6 | 4 | 2 |
| 7 | 4 | **3** |
| 8 | 5 | 3 |
| 9 | 5 | **4** |

## 推荐配置
括号内是官方推荐的配置

| 用途 | 数量 | CPU | 内存 |
| ----------- | :---: | :---: | :---: |
| etcd   |   3   |   4 (8~16)   |  8GB (16GB~64GB)  |

> **官网:**  
> <https://etcd.io/>  
> **官方硬件建议:**  
> <https://etcd.io/docs/v3.3.12/op-guide/hardware/>  
> **Static Pod部署文档:**  
> <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/>  

# Kubernetes集群
kubernetes集群主要有两种类型的节点：Master和Worker。  
Master则是集群领导。  
Worker是工作者节点。  
可以看出这边主要的工作在Master节点，Worker节点根据具体需求随意增减就好了。  
Master节点的高可用拓补官方给出了两种方案。  
1. Stacked etcd topology（堆叠etcd）  
2. External etcd topology（外部etcd）  

**可以看出最主要的区别在于etcd的部署方式。**  
第一种方案是所有k8s Master节点都运行一个etcd在本机组成一个etcd集群。  
第二种方案则是使用外部的etcd集群（额外搭建etcd集群）。  
我们采用的是第二种，外部etcd，拓补图如下：

![](https://ws1.sinaimg.cn/large/7ecacd23ly1g6nck4sziij21yw1cugsm.jpg)

***如果采用堆叠的etcd拓补图则是：***

![](https://ws1.sinaimg.cn/large/7ecacd23ly1g6ncnb681tj21ye1a8458.jpg)

***这边大家可以根据具体的情况选择，推荐使用第二种，外部的etcd。***
> **参考来源:**  
> <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/>

## Master节点的组件
1. apiserver
2. controller-manager
3. scheduler

一个master节点主要含有上面3个组件 ( 像cloud-controller-manager这边就不多做说明了，正常不会用到 )  
**apiserver**: 一个api服务器，所有外部与k8s集群的交互都需要经过它。（可水平扩展）  
**controller-manager**: 执行控制器逻辑（循环通过apiserver监控集群状态做出相应的处理）（一个master集群中只会有一个节点处于激活状态）  
**scheduler**: 将pod调度到具体的节点上（一个master集群中只会有一个节点处于激活状态）  

可以看到除了apiserver外都只允许一个 实例处于激活状态（类HBase）运行于其它节点上的实例属于待命状态，只有当激活状态的实例不可用时才会尝试将自己设为激活状态。
这边牵扯到了领导选举（zookeeper、consul等分布式集群系统也是需要领导选举）

### Master高可用需要几个节点？失败容忍度是多少？
k8s依赖etcd所以不存在数据一致性的问题（把数据一致性压到了etcd上），所以k8s master不需要采取投票的机制来进行选举，而只需节点健康就可以成为leader。  
所以这边master并不要求奇数，偶数也是可以的。  
那么master高可用至少需要2个节点，失败容忍度是(n/0)+1，也就是只要有一个是健康的k8s master集群就属于可用状态。（**这边需要注意的是master依赖etcd，如果etcd不可用那么master也将不可用**）
> **Master组件说明:**  
> <https://kubernetes.io/docs/concepts/overview/components/>  
> **部署文档:**  
> <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/>

## 硬件配置

| 用途 | 数量 | CPU | 内存 |
| :---------: | :---: | :---: | :---: |
| Master   |   3   |   4   |  8GB  |

# 高可用验证
至此生产可用的k8s集群已“搭建完成”。为什么打引号？因为还没有进行测试和验证，下面给出我列出的验证清单

![](https://ws1.sinaimg.cn/large/7ecacd23ly1g6wskfeqz6j22160pujze.jpg)

还有涉及的BGP相关的验证不在此次文章内容中，后续会为大家说明。

# 写在最后
还有一点需要注意的是物理机的可用性，如果这些虚拟机全部在一台物理机上那么还是存在“单点问题”。**这边建议至少3台物理机以上**。

**为什么需要3台物理机以上？**  
主要是考虑到了etcd的问题，如果只有两台物理机部署了5个etcd节点，那么部署了3个etcd的那台物理机故障了，则不满足etcd失败容忍度而导致etcd集群宕机，从而导致k8s集群宕机。
