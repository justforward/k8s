k8s作用：用于管理分布式节点集群中的微服务或者容器化应用程序。


组件：

master服务端（主控节点）主要负责管理和控制整个k8s集群。所有的控制命令都是由Master服务端接受并处理。组件包含：
1）kube-api 
 集群的HTTP REST API接口，是集群控制的入口。
2）kube-controller-manager
集群中所有资源对象的自动化控制中心。
3）kube-schedule
集群中pod资源对象的调度服务

Kubernetes scheduler独立运作与其他主要组件之外(例如API Server)，它连接API Server，watch观察，如果有PodSpec.NodeName为空的Pod出现，则开始工作，通过一定得筛选算法，筛选出合适的Node之后，向API Server发起一个绑定指示，申请将Pod与筛选出的Node进行绑定。


Node客户端（工作节点），node节点上的工作由Master服务端进行分配，组件包含：
1）kubectl
负责管理节点上容器的创建、删除、启停等任务，与master节点进行通信

2）kube-proxy 组件
负责k8s服务的通信以及负载均衡的服务

3）container组件
负责容器的基础管理服务，接受kubelet组件的指令。



组件的详细介绍
1）kubectl
命令行工具，kbectl会将用户的命令行转化为http的请求，发送给kebe-apiserver，等待kube-apiserver 请求完成之后，会讲结果反馈给kubectl
2）client-go
通过编程的方式与kube-api server 进行打交道。

3）kube-apiserver
- 以RestFul风格的形式对外暴露服务，提供操作k8s资源组/资源版本/资源的功能。
- k8s的所有组件都是通过kube-apiserver组件操作资源对象
- kube-apiserver 组件也是唯一与ETCD集群进行交互的核心组件
- 提供了集群各个组件的通信和交互功能


4）kube-controller-manager
- 提供一些控制器，比如deployment控制器、statefulset控制器、Namespace控制器等来负责保证k8s系统的实际状态变动到所需状态，
- 每个控制器通过kube-apiserver 提供的接口实时监控整个集群中每个资源对象的当前状态，当因为发生各种故障而导致系统状态出现变化的时候，会尝试将系统状态修复到 期望状态。
- 具备高可用性，采用Raft 来保证

5）kube-scheduler
- 负责在k8s集群中为一个pod资源对象找到合适的节点并在该节点上运行，
- 调度器每次只调度一个pod资源对象，为每一个pod资源对象寻找合适节点的过程是一个调度周期
- 该组件会监控整个集群的pod资源对象和node资源对象，会通过调度策略：预选调度策略和优先调度策略，或者优先级调度、抢占机制以及亲和性调度等来选择最优的节点。

6）kubectl
- 管理节点
- 接受、处理、上报kube-apiserver组件下发的任务
- 定期监控所在节点上的资源使用状态上报给api-server组件
- 负责同容器的运行时（docker）进行交互。交互接口为CRI
- 定义的三个组件：CRI（容器运行时）、CNI（容器网络接口）、CSI(容器存储接口)

7）kube-proxy
- 节点上的网络代理
- 通过iptables/ipvs 等配置负载均衡器，为一组pod提供同一个TCP/UDP 流量转发和负载均衡



k8s scheluer

`Pod.spec.nodeSelector`是通过kubernetes的label-selector机制进行节点选择，由scheduler调度策略MatchNodeSelector进行label匹配，调度pod到目标节点，该匹配规则是强制约束。启用节点选择器的步骤为：

调度的亲和度：指定NodeName和使用NodeSelector调度是最简单的，可以将Pod调度到期望的节点上。

节点亲和（Node Affinity）是指在 Kubernetes 集群中，通过指定 Pod 需要调度到的节点的标签（Label），来限制 Pod 能够被调度到哪些节点的一种方式。

其中，用来指定节点亲和的标签是 `nodeSelector`。可以通过在 Pod 的 YAML 文件中使用 `nodeSelector` 字段来指定需要匹配的标签，

nodeselector字段来制定需要匹配的标签