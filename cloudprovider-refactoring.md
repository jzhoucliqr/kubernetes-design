##cloudprovider 重构

随着kubernetes的快速发展，

### 当前cloudprovider代码现状
目前kube-controller-manager, kubelet, api-server三个部分都有对cloudprovider的依赖。

#### kube-controller-manager
kube-controller-manager由一系列的控制器组成，其中依赖cloudprovider的控制器有：

 - nodeController
 - volumeController
 - routeController
 - serviceController

nodeController 通过cloudprovider接口来检测节点是否已经被云平台删除。如果已经被删除，那么nodeController会立即将其相关信息从kubernetes系统中清除。这样做的主要考量是，当发现某个节点没有响应时，可以立即确定该节点已经不存在，从而避免等待一定的时间之后再做出反应。

volumeController 主要通过cloudprovider管理系统的存储卷功能，包括创建、删除、attach / detach 等。

routeController 主要配置主机在云平台里面的路由信息。

serviceController 主要负责管理负载均衡器。

#### kubelet
在kubelet 中，cloudprovider 接口主要用于以下功能:

 - 获得主机名，主要用于:
    1. 查询主机的config map
    2. 为了nodeInformer 确定本主机的唯一标识
    3. 获取系统中对本节点的信息

 - 在初始化节点的时候，获取instanceID, providerID, ExternalID, Zone 等相关信息
 - 定期检测节点IP 是否有变化
 - 设置check 让节点不能接受系统调度，直到路由信息设置完成。
 - 可以让云平台处理DNS相关信息

#### apiserver
apiserver 中cloudprovider 相关代码主要用于分发SSH KEY 到各个节点, 同时在admissionController中为存储卷设置标签信息.

### controller-manager 重构策略
计划将当前的kube-controller-manager按照是否依赖cloudprovider拆分为两个部分：cloud-controller-manager以及kube-controller-manager。
cloud-controller-manager将作为系统的一个附加服务，而kube-controller-manager将仍然作为kubernetes的核心组成部分，其将不依赖于任何云平台。

我们计划cloud-controller-manager将仍然包含Kubernetes相关的业务逻辑，而不是仅仅是一个cloud接口。这将使重构更简单有效，避免拆开其共享的数据结构和公共函数，保证没有太多的重复代码。

cloud-controller-manager的控制逻辑循环仍然存在于 kubernetes 核心代码库，通过cloudprovider的接口实现云平台的控制。不同的云厂商可以将cloud-controller-manager的控制逻辑和其自己的cloudprovider链接一起，来发布自己的cloud-controller-manager程序。

routeController和serviceController是完全的cloud specific, 因此这两个controller很容易拆分开。

nodeController除了和cloud接口外，还有以下功能：
1. 管理CIDR
2. 监控节点状态
3. Node pod eviction

对于节点监控，kubelet报告的节点状态为ConditionUnknown或者ConditionFalse，然后控制器检测节点是否已经被云平台删除。如果已经被删除，那么控制器将立即删除节点对象数据，避免等待monitorGracePeriod之后再删除。只有这一部分逻辑需要拆分到kube-controller-manager。

volumeController比较难搞，这一部分将和Flex Volume一起解决。

### kubelet重构策略
kubelet中主要的云平台接口调用是在节点初始化构建节点对象数据时进行的，以及配置节点路由，scurbbing DNS， 并且周期性检查节点IP 是否有更新。

这些调用，除了节点初始化，其他功能都可以由cloud-controller-manager完成。具体来说，周期性检测节点IP，以及配置节点路由可以通过新的控制器来完成。

scrubbing DNS 可以删除，是冗余的功能。

节点初始化部分比较tricky。pods可以被调度到未完成初始化的节点上。这样可能导致pods调度到不兼容的区域等问题。因此需要kubelet可以创建节点对象数据，并且标注为“NotReady”，然后某个异步进程在合适的时机修改其为NodeReady。这可以通过Taints来解决。

这样要求kubelet在启动时就标注为某种Taint，系统根据这个Taint将不会把该节点纳入调度系统中，直到cloud-controller-manager在完成节点初始化之后，移除这个Taint。

### apiserver 重构策略

apiserver分发ssh key的功能可以迁移到cloud-controller-manager。
admission controller for 存储卷可以通过kubelet类似的taint功能来实现。

这样apiserver可以做到完全不依赖于cloudprovider代码。

### 存储卷重构策略
存储卷功能依赖于cloudprovider。主要的存储卷控制逻辑代码在controller-manager中。这部分控制器需要迁移到cloud-controller-manager。cloud-controller-manager需要在初始化时输入云平台相关配置，可以通过config map来实现。

另外一个正在进行的方法，通过把所有的存储卷逻辑拆分为plugin，Flex Volume来解决这个问题。通过Flex Volume, 所有vendor相关的存储卷逻辑代码将各自编译为单独的二进制代码，通过plugin的方式接入到kubernetes。讨论认为这是最合适的存储卷重构方法。

### 系统upgrade/downgrade

本次重构将引入新的二进制文件，可以通过kubectl - apply *.xml的方式安装。

#### 升级kubelet和proxy
kubelet和proxy的升级方法没有改变。

#### 升级插件
本次重构不影响 cni 和 flex等插件升级。

#### kubernetes 核心服务升级
主节点的服务包括 kube-controller-manager/scheduler/apiserver等升级方式不变。

#### 安装 cloud-controller-manager
这是唯一需要的安装步骤。我们将提供cloud-controller-manager.yaml文件，可以通过运行 kubectl apply -f cloud-controller-manager.yaml 来安装相关服务. 

### roadmap
#### transition 计划
v1.6：完成第一版本的cloud-controller-manager。目的是可以让用户同时运行两个controller-manager, 这样任何我们尚未发现的问题可以得到解决。同时也作为以后即将进行的外部external-controller-manager的参考设计。因为cloud-controller-manager将运行cloud相关的逻辑，需要确保kube-controller-manager不要运行这些控制器。这可以通过不设置kube-controller-manager的 cloud-provider 标识来实现。在这个阶段，cloud-controller-manager是出于alpha版本，可以选择不安装。

v1.7：这个版本所有的相关逻辑将默认使用cloud-controller-manager。用户仍然可以选择使用之前的方式。用户将运行一个monolithic的cloud-controller-manager的二进制文件。cloud-controller-manager将仍然使用现在的cloudprovider代码库。但是会进行部分重构以避免代码冗余。同时将会有发布提示用户迁移至新的cloud controller manager.

v1.8：这个版本将把cloudprovider拆分为单个的二进制文件。用户仍然可以选择继续使用之前的方式。同时会有新的提示 kube-controller-manager的cloud-provider选项将会deprecate。

v1.9：本版本将完全迁移至新的cloud-provider-controller.




