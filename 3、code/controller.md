
那么问题来了，k8s 是如何 "监视" 资源对象，以确保其始终保持我们声明的状态的呢？ 
答案就是 -- Controller。除了组件中的 kube-controller-manager，我们可以编写自己的 Controller，也叫自定义控制器(为了方便下文统称为自定义 Controller)。




controller中监听的资源列表来自etcd?


1）**_大步骤1_**: Reflector 将资源对象的事件添加进 Delta FIFO queue 中

将资源对象的事件添加到delta FIFO queue中

即 **_资源对象的变化都会被添加到 Delta FIFO queue 中！_**


2）**_大步骤2_**: Informer 将 Delta FIFO queue 中的对象数据 添加到本地 cache 中。

补充一下这个本地 cache 缓存的就是监听资源对象的最新版。就是缓存的当前集群里面的资源信息。

为了提高效率和减少对API Server的频繁访问，k8s的controller组件通常会使用informer机制来实现资源对象的本地缓存和监听，Informer 是一种高效的机制，它会将 API Server 中的资源对象的状态信息缓存到本地内存中，并定期与 API Server 同步，以便及时获取资源对象的更新信息。

具体来说，Informer 通过在 API server 上访问资源对象的 REST API 接口来获取对象的状态，并使用 HTTP 长轮询或 WebSocket 等机制来监听资源对象的变化。API server 通过 Watch API 实现了对资源对象的监听功能，Informer 则通过 Watch 接口建立到 API server 的长连接，并通过该连接接收对象的变化通知。



**_大步骤3_**: 使用 workqueue 处理业务逻辑。

然后咱们结合社区给的编写的[自定义Controller用例](https://link.zhihu.com/?target=https%3A//github.com/kubernetes/client-go/blob/master/examples/workqueue/main.go)来做源码分析。


使用workqueue处理业务逻辑（业务逻辑？对资源对象的一些操作）


第三大步骤主要就是对 workqueue 的调用。而 workqueue 有三大类：

1.  普通队列
2.  延迟队列
3.  限速队列


我们一般使用限速队列，方便我们在处理错误的时候重试。

 workerQueue是一种用于存储待处理事件的队列，可以帮助controller进行事件的异步处理。
在controller监听到某个资源发生改变的时候，他会将该事件添加到workQueue中，并理解返回，继续监听下一个事件。

控制事件处理的速度：比如限速队列。
避免重复处理事件，当处理完一个事件之后，立马将该事件从队列中移除。




所以这就要引申一个问题 **_为什么要用 workqueue ？_** 原因如下：

1、在不依赖 Delta FIFO queue 的情况下，将资源事件变得有序。

2、workqueue 也可以当作缓存看。将要处理的事件以 key 的方式先缓存在 workqueue 中。

缓存的作用相信很多人都清楚：解决两个组件处理速度不匹配的问题，如 cpu 和 硬盘之间经常是用 内存做缓存。

我们的业务处理逻辑大概率肯定是慢于事件的生成的，而且还延迟队列类型做选择

3、方便失败后重试

处理组件速度不匹配的问题


因此，Informer 在 Kubernetes 中扮演着一个重要的角色，它可以帮助 Controller 组件高效地监控 Kubernetes 中的资源对象状态，并及时做出相应的调整和维护，从而保证 Kubernetes 系统的稳定性和可靠性。


-   Reflector：通过调用 `LISTWATCH` 接口与 ApiServer 通信，监听特定资源（此处监听的资源需要我们指定），并把资源的更新动态存入 Delta FIFO 队列
-   Informer：从Delta FIFO队列拿出对象，完成此操作的函数是processLoop。
-   Indexer: 提供线程级别安全来存储对象和key。