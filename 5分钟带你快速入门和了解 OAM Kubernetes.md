# 什么是 OAM？
OAM 的全称为开放应用模型（Open Application Model），由阿里巴巴宣布联合微软共同推出。  

## OAM 解决了什么问题？

OAM 本质是为了解耦K8S中现存的形形色色的资源，让每个角色的关注点更为集中和专注。

举个例子，我们在生产环境中部署了Deployment资源，其中容器的image，健康检查，资源请求开发人员一般会了然于胸，但涉及到Pod副本数、PV、PVC、网络带宽、网络策略、对外负载配置等，一般的开发人员根本无从下手。

为什么会无从下手？原因很简单，正常的开发人员根本不会有这些内容的权限，而且绝大部分开发人员没有公司资源的监控数据，无法对应用所使用的资源进行平衡。最简单的情况，如果开放了CPU、内存、共享存储大小、网络带宽的上限配置，我相信很多人一上来就设置为8C，16GB，100GB，100MB这样奢侈配置，实际的集群资源能支持几个这样的应用呢？

所以这类的事情更适合应用运维人员去做，当然应用运维的人员也需要一定的水准。应用运维人员可以根据监控和告警按需的对CPU、内存、共享存储、网络等资源进行弹性伸缩，甚至可以从监控数据中发现一定的规律配置自动的程序来进行自动扩缩容，例如HPA、CronHPA。

## OAM 是如何解决这些问题的？

在 OAM 体系中大致分了三个角色，分别为：
- 基础设施运维人员  
  负责开发、安装和维护各种 Kubernetes 级别的功能。具体工作包括但不限于维护大规模的 K8s 集群、实现控制器/Operator，以及开发各种 K8s 插件。在公司内部，我们通常被称为“平台团队（Platform Builder）
- 业务研发人员  
  以代码形式传递业务价值。大多数业务研发都对基础设施或 K8s 不甚了解，他们与 PaaS 和 CI 流水线交互以管理其应用程序。业务研发人员的生产效率对公司而言价值巨大。
- 应用运维人员  
  为业务研发人员提供有关集群容量、稳定性和性能的专业知识，帮助业务研发人员大规模配置、部署和运行应用程序（例如，更新、扩展、恢复）。请注意，尽管应用运维人员对 K8s 的 API 和功能具有一定了解，但他们并不直接在 K8s 上工作。在大多数情况下，他们利用 PaaS 系统为业务研发人员提供基础 K8s 功能。在这种情况下，许多应用运维人员实际上也是 PaaS 工程人员。

各个角色之间的服务关系为：基础设施运维人员 < 应用运维人员 < 业务研发人员。

与角色对应的也同样的有三个核心内容，分别为：
- Component（组件）  
  由业务开发人员输出，它可能包含微服务集合、数据库和云负载均衡器。
- Trait（运维特征）  
  由应用运维人员管理，例如：弹性伸缩、日志、监控、告警和 Ingress 等功能。
- Application Configuration（应用配置）  
  组合Component和Trait等内容描述应用最终能力的配置文件。

OAM 从资源角度上就隔离了不同角色所关心的内容，而不再将这些内容混杂在一起。  
整体示意图如下：
![](https://img2020.cnblogs.com/blog/384997/202007/384997-20200714175046553-140184622.png)
更详细的内容可以阅读：《[深度解读！阿里统一应用管理架构升级的教训与实践](https://mp.weixin.qq.com/s?__biz=MjM5MjAwODM4MA==&mid=2650740549&idx=1&sn=da4000870f6d3008ae3091366ebf61da&scene=21#wechat_redirect)》

# OAM 社区生态现况
由于OAM是一个新的概念，所以社区的文档目前比较混乱，下面列出还在活动并在维护的几个仓库。

## OAM标准规范  
截至目前的版本
- 1.0.0-alpha1
- 1.0.0-alpha2

https://github.com/oam-dev/spec

## 基于Crossplane的OAM 核心运行时实现  
支持的OAM规范：1.0.0-alpha2  
https://github.com/crossplane/oam-kubernetes-runtime

## OAM工作负载、特征、范围集合  
可以理解为非OAM Runtime必须实现的OAM插件和扩展  
https://github.com/oam-dev/catalog

该库提供了以下内容：
- Workloads
  - Containerized
  - StatefulSet
  - Deployment
  - AdvancedStatefulSet
  - CloneSet
- Traits
  - ManualScaler（手动伸缩器）
  - trait-injector（资源注入器）
  - ServiceExpose（服务导出）
  - IngressTrait（Ingress）
  - HPATrait（自动伸缩）
  - CronHPATrait（基于Cron表达式的定时伸缩）
  - MetricHPATrait（基于指标的自动伸缩）
  - SimpleRolloutTrait（简单的滚动更新）
  - ServiceMonitor（服务监控器，还在PR中）
- Scopes
  - HealthScope

## 不维护或过时的仓库
这边列出一些容易被误导的仓库，大家遇到了可以跳过查阅，避免绕弯路。
### Rudr  
实现了 OAM 规范 1.0.0-alpha1  
https://github.com/oam-dev/rudr

### addon-oam-kubernetes-local  
被 oam-kubernetes-runtime 替代  
https://github.com/crossplane/addon-oam-kubernetes-local

# 快速入门

## 先决条件
- Kubernetes cluster  
  如果没有，可以参考《[比Minikube更快，使用Kind快速创建K8S学习环境](https://mp.weixin.qq.com/s?__biz=MzA5NjgxNTI4NA==&mid=2458145398&idx=1&sn=7a14447ae16922894fe3c9f150387ce1&scene=19)》
- Helm 3

## 安装
```console
helm repo add oam https://github.com/oam-dev/crossplane-oam-sample/tree/master/archives/
kubectl create namespace oam-system
helm install crossplane --namespace oam-system oam/crossplane-oam
```

## 尝试一个应用配置样本

```yaml
# sample_application_config.yaml

apiVersion: core.oam.dev/v1alpha2
kind: Component
metadata:
  name: example-component
spec:
  workload:
    apiVersion: core.oam.dev/v1alpha2
    kind: ContainerizedWorkload
    spec:
      containers:
        - name: my-nginx
          image: nginx:1.16.1
          resources:
            limits:
              memory: "200Mi"
          ports:
            - containerPort: 4848
              protocol: "TCP"
          env:
            - name: WORDPRESS_DB_PASSWORD
              value: ""
  parameters:
    - name: instance-name
      required: true
      fieldPaths:
        - metadata.name
    - name: image
      fieldPaths:
        - spec.containers[0].image
---
apiVersion: core.oam.dev/v1alpha2
kind: ApplicationConfiguration
metadata:
  name: example-appconfig
spec:
  components:
    - componentName: example-component
      parameterValues:
        - name: instance-name
          value: example-appconfig-workload
        - name: image
          value: nginx:latest
      traits:
        - trait:
            apiVersion: core.oam.dev/v1alpha2
            kind: ManualScalerTrait
            metadata:
              name:  example-appconfig-trait
            spec:
              replicaCount: 3
```

```console
$ kubectl apply -f samples/sample_application_config.yaml
component.core.oam.dev/example-component created
applicationconfiguration.core.oam.dev/example-appconfig created
```

```console
$ kubectl get manualscalertraits.core.oam.dev
NAME                      AGE
example-appconfig-trait   4s
```

```console
$ kubectl get containerizedworkloads.core.oam.dev
NAME                         AGE
example-appconfig-workload   58s
```

```console
$ kubectl get deploy
NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
example-appconfig-workload-deployment   3/3     3            3           114s
```

可以看到，通过Component和Trait、ApplicationConfiguration的抽象，开发人员和运维人员所关心的内容不再混合在一起，变得更为清晰。

# 最后
受限于篇幅的原因，本文简单介绍了 OAM 概念并实践了一个简单的基于 OAM 概念的应用程序样本，还没有完全发挥 OAM 的能力。  
下一节将会详解如何一步一步使用kubebuilder为OAM添加一个基于Cron表达式定时伸缩的Train。  

小伙伴们可以前往：https://github.com/oam-dev/catalog/pull/44 查看该 Train 的 PR 情况。  
CronHPATrain 目前已被 oam-dev catalog 接收并合并。