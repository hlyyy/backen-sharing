# kubernetes 集群调度

## k8s架构简介
 Kubernetes是一项**容器编排技术**。它是容器集群管理系统，是一个开源的平台，可以实现容器集群的自动化部署、自动扩缩容、维护等功能。

架构图：
![](https://upload-images.jianshu.io/upload_images/21203385-35e37d05a5052596.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Kubernetes属于主从分布式架构，主要由Master节点和Node节点组成。Master节点作为控制节点，对集群进行调度管理；Node节点作为真正的工作节点，运行容器。

##### Master节点
```Kube-Apiserver``` : 所有服务访问的统一入口，各组件的协调者

```Kube-controller-manager``` : 控制器，主要维护k8s资源对象

```Kube-scheduler```: 调度器，为 Pod 选择一个 Node 节点

```Etcd```: 分布式数据库来储存集群中的重要信息，比如 pod、service 等对象信息

##### Node节点
```pod```:pod是一个非常重要的概念。它是k8s的最小的服务单位，可以理解为把一组关系紧密的容器放在一起，它们共享了同一个 Network Namespace，并且可以声明共享一个 Volume。**所以，Pod其实是一组共享了某些资源的容器**。

```Kubelet```: 主要来执行关于资源操作的指令，负责pod的维护。比如创建容器、Pod挂载数据卷、获取节点信息和状态等工作。


```Kube-proxy```: 负责代理服务，在多个pod之间做负载均衡。

## 调度器
Scheduler 是 kubernetes 的调度器，它的主要任务是**为新创建出来的pod寻找一个最佳node 节点**，听起来非常简单，但有很多要考虑的问题：
- 公平：如何保证每个节点都能被分配资源
- 资源高效利用：集群资源能被最大化的使用
- 效率：调度的性能要好，能够尽快地对大批量的pod完成调度工作
- 灵活：允许用户根据自己的需求控制调度的逻辑

Scheduler 是作为单独的程序运行的，它启动之后会一直监听 API server ，获取```Spec.NodeName```为空的 pod。调度执行完之后，调度器会将 Pod 对象的 NodeName 字段的值，修改为 Node 的名字，表明 pod 应该放在哪个节点上，这个步骤在 k8s中称为```Bind```。

## 调度过程

调度分为几个部分：首先是过滤掉不满足条件的节点，这个过程称为```predicate```；然后对通过的节点按照优先级排序，这个是```priority```；最后从中选出优先级最高的节点作为最终的结果。

Predicate 有一系列算法可以使用，比如：
- ```PodFitResources```：节点上剩余的资源是否大于 pod 请求的资源
- ```PodFitHostPorts```：节点上已经使用的 port 是否和 pod 申请的 port 冲突
- ```PodSelectorMatches```：过滤掉和 pod 指定的 label 不匹配的节点

如果在 predicate 过程中没有合适的节点，pod 会一直在 ```pending```状态，不断重试调度，直到有节点满足条件；经过这个步骤，如果有多个节点满足条件，就继续 priority过程。

priority 由一系列键值对组成，键是优先级项的名称，值是它的权重。这些优先级包括：
- ```LeastRequestedPriority```：通过计算 CPU 和 Memory 的使用率来决定它的权重，换句话说就是选择空闲资源最多的节点
- ```BalancedResourceAllocation```：节点上的 CPU 和 Memory 使用率越接近，权重越高
- ```ImageLocalityPriority```：倾向于已经有要使用镜像的节点，镜像总大小值越大，权重越高


调度的具体过程，如下图
![](https://upload-images.jianshu.io/upload_images/21203385-846e75520953673d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，Kubernetes 调度器的核心，**实际上就是两个相互独立的控制循环**。

**第一个控制循环称为```Imformer Path```**,它主要目的是启动一系列imformer，来监听 Etcd 中 Pod、Node、Service 等与调度相关的对象的变化。比如当一个待调度 Pod（即：它的 nodeName 字段是空的）被创建出来之后，调度器就会通过 Pod Informer 的 Handler，将这个待调度 Pod 添加进调度队列。

同时，调度器的 informer 还要负责对调度器缓存( scheduler cache）进行更新。Kubernetes 调度部分进行性能优化的一个最根本原则，**就是尽最大可能将集群信息 Cache 化，以便从根本上提高 Predicate 和 Priority 调度算法的执行效率**。

**第二个控制循环称为```Scheduling Path```**，是调度器负责 pod 调度的主循环。这部分的主要逻辑，就是不断地从调度队列里出队一个 Pod。然后，调用 Predicates 算法进行“过滤”。这一步“过滤”得到的一组 Node，就是所有可以运行这个 Pod 的宿主机列表。

接下来，调度器就会再调用 Priorities 算法为上述列表里的 Node 打分，分数从 0 到 10。得分最高的 Node，就会作为这次调度的结果。

当然，上面两个调度算法所需的 node 数据，都是从 Scheduler Cache 里直接拿到的，这是调度器保证算法执行效率的主要手段。

算法执行完后，调度器会执行 Bind 操作，将 Pod 对象的 nodeName 字段的值，修改为上述 Node 的名字。但是，**为了不在这个关键调度步骤中远程访问 API server，在 Bind 阶段，调度器只会更新 Scheduler Cache 里的 Pod 和 Node 信息**。这种基于“乐观”假设的 API 对象更新方式，在 Kubernetes 里被称作** Assume**。

等 Assume 之后，调度器才会创建一个 Goroutine 来异步向 API server 发起更新 Pod 的请求，来完成真正的 Bind 操作。对应节点的 kubelet 会进行一个 Admit 的操作，再次确认该 pod 能否运行在该节点上。

除了上面所说的，k8s 调度器还有一个重要设计，那就是**无锁化**。

在 Scheduling Path 上，调度器会启动多个 Goroutine 并发执行 Predicates 算法，从而提高这一阶段的执行效率。而与之类似的，Priorities 算法也会以 MapReduce 的方式并行计算然后再进行汇总。而在这些所有需要并发的路径上，调度器会避免设置任何全局的竞争资源，从而免去了使用锁进行同步带来的巨大的性能损耗。

总结一下，上面介绍了调度的具体过程，以及提升调度效率的三个方法：Cache化、乐观假设、无锁化。

## 亲和性调度

### 节点亲和性
```pod.spec.nodeAffinity```分为软策略和硬策略
- ```preferredDuringSchedulingIgnoredDuringExecution```：软策略
- ```requredDuringSchedulingIgnoredDuringExecution```：硬策略

当硬亲和性规则不满足时，Pod会置于Pending状态，软亲和性规则不满足时，会选择一个不匹配的节点。

当节点标签改变而不再符合此节点亲和性规则时，不会将Pod从该节点移出，仅对新建的Pod对象生效。

对于软策略，会使用权重 weight 定义优先级，1~100 值越大优先级越高。
```
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity
  namespace: test
spec:
  containers:
  - name: nginx-pod
    image: nginx:latest
  affinity:
    # 节点亲和
    nodeAffinity:
      # 硬策略，条件必须成立
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        # 要调度到node的标签是kubernetes.io/hostname=node01的节点上
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: NotIn
            values:
            - node01
      # 软策略，条件尽量要成立
      preferredDuringSchedulingIgnoredDuringExecution:
      # 多个软策略的权重，范围在1-100内，越大计算的得分越高
      - weight: 1
        preference:
          matchExpressions:
          # 要调度到node的标签是area=beijing的节点上
          - key: area
            operator: NotIn
            values:
            - beijing
```
键值运算关系：
- In：label 的值在某个列表中
- NotIn：label 的值不在某个列表中
- Gt：label 的值大于某个值
- Lt：label 的值小于某个值
- Exists：某个 label 存在
- DoesNotExist：某个 label 不存在

### pod亲和性
pod 亲和性主要解决 pod 可以和哪些 pod 部署在同一个拓扑域中的问题（其中拓扑域用主机标签实现，可以是单个主机，也可以是多个主机组成的 cluster、zone 等等），而 pod 反亲和性主要是解决 pod 不能和哪些 pod 部署在同一个拓扑域中的问题，它们都是处理的 pod 与 pod 之间的关系。

同理，```pod.spec.affinity.podAffinity/podAntiAffinity```也分为软策略和硬策略。

举例：
```
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity
  namespace: test
spec:
  containers:
  - name: nginx-pod
    image: nginx:latest
  affinity:
    # pod亲和 要求与指定的pod在同一个拓扑域
    podAffinity:
      # 硬策略
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - nginx01
        # 拓扑域（node上的标签key）
        topologyKey: kubernetes.io/hostname
    # pod亲和 要求与指定的pod不在同一个拓扑域
    podAntiAffinity:
      # 软策略
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - nginx02
          # 拓扑域（node上的标签key）
          topologyKey: kubernetes.io/hostname
```
## 污点和容忍度
上面说的节点亲和性，可以理解为 pod 的一种偏好(或者是硬性的)属性，它使得 pod 被吸引到一类特定的节点。Taint(污点)则相反，它使得节点能排斥一类特定的 pod。

**Taint 和 Toleration 相互配合，可以用来避免 pod 被分配到不合适的节点上。每个节点都可以有一个或多个污点，而对于不能容忍这些污点的pod，是不会被该节点接受的。如果将 Toleration 应用于 pod 上，则表示这些 pod 可以(但不要求)被调度到具有匹配 taint 的节点上。**

### 污点(Taint)
通过```kubectl taint```命令可以给某一个node节点设置污点，node上设置了污点之后，pod可以拒绝 node 的调度，甚至可以将node上已经存在的pod驱逐出去。

污点可以用于集群节点迁移准备工作，通过打污点来使当前节点上的pod迁移出去。k8s 的master节点自带污点。

污点的组成为：
```
key = value : effect
```
每个污点有一个 key 和 value 作为污点的标签，其中 value 可以为空，effect 描述污点的作用。当前 taint 的 effect 支持如下三个选项：
- ```NoSchedule```：k8s不会把pod调度到该节点上
- ```PreferNoSchedule```：k8s尽量不会把pod调度到该节点上
- ```NoExecute```：k8s不会把pod调度到该节点上，同时会把已有节点驱逐出去
```
# 设置污点
kubectl taint nodes node1 key1=value:NoSchedule
# 去除污点
Kubectl taint nodes node1 key1:NoSchedule-
```
### 容忍(Toleration)
pod 可以设置容忍，即使 node 有污点，也可以分配。

```pod.spec.toleration```：
```
tolerations:
- key: "key1"
  operator: " Equal"
  value: "value1"
  effect: "NoSchedule"
  tolerationSeconds: 3600
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
- key: "key2"
  operator: "Exists"
  effect: "NoSchedule"
```
- 其中key，vaule，effect要与Node中设置的taint保持一致
- tolerationSeconds 用于描述当 Pod 需要被驱逐时可以在 Pod 上继续保留运行的时间

k8s的namespace可以提供资源的逻辑隔离，但是无法提供物理隔离。物理隔离可以通过污点与容忍来实现。

比如想隔离不同产品线的服务，可以提前给 node 打上不同的污点，不同的产品线的pod容忍对应的污点即可。

## 调度器的可扩展设计
默认调度器的可扩展机制，在 Kubernetes 里面叫作 Scheduler Framework。这个设计的主要目的，就是在调度器生命周期的各个关键点上，为用户暴露出可以进行扩展和实现的接口，从而实现由用户自定义调度器的能力。
![](https://upload-images.jianshu.io/upload_images/21203385-6f97868f6d376816.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

每一个绿色的箭头都是一个可以插入自定义逻辑的接口.比如,上面的 Queue 部分,就意味着你可以在这一部分提供一个自己的调度队列的实现,从而控制每个Pod 开始被调度(出队)的时机。

Predicates部分,则意味着你可以提供自己的过滤算法实现,根据自己的需求,来决定选择哪些机器

上述这些可插拔式逻辑,都是标准的Go语言插件机制(Go plugin 机制),也就是说,你需要在编译的时候选择把哪些插件编译进去。

### 总结
以上主要介绍了 k8s 集群调度的主要流程、影响调度器调度的几个因素，以及自定义调度器的可拓展设计。
