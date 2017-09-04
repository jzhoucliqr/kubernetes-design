# 节点可分配资源

[Read more](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node-allocatable.md)

kubernetes节点内除了运行有kubernetes系统程序(kubelet, runtime)，用户创建的Pods，还有OS系统程序。当前kubernetes默认节点的全部资源(Capacity)都可以用于创建用户Pods。但事实上，OS系统程序也占用了相当可观的系统资源，并且这些资源对于系统稳定而言非常重要。为了解决这个问题，我们引入"Allocatable"的概念（可分配资源），用来代表事实上可以分配给用户pods的资源量。具体而言，kubelet将提供具体的机制来给OS后台程序以及kubernetes后台程序预留系统资源。

显示的预留系统资源的目的在于避免节点过度承诺(overcommiting)以导致系统后台程序和用户pods竞争系统资源。用于系统后台程序以及用户pods的资源将可以由用户设定的资源数量来控制。

如果节点提供了Allocatable数据，那么调度器(scheduler)将根据该Allocatable数据来调度，而不是根据系统容量capacity。

## 设计
### 资源名称定义
1. Node Capacity: 节点资源容量。由NodeStatus.Capacity提供。代表系统总的容量，通过节点数据获得，默认为恒定值。
2. System reserved (proposed)：预留给kubernetes系统以外程序的资源。目前这包括了所有的OS程序。
3. Kubelet Allocatable: 这是节点真正可以用于kubernetes调度的资源，是本设计关注的重点。
4. kube reserved (proposed)：预留给kubernetes系统程序的资源，包括docker/kubelet/proxy等

### API 
在NodeStatus中加入Allocatable:
```
type NodeStatus struct {
	...
	// Allocatable represents schedulable resources of a node
	Allocatable ResourceList `json:"allocatable,omitempty"`
	...
}
```
Allocatable 由kubelet计算获得，然后汇报给apiserver。其计算方式为：
```
[Allocatable] = [Node Capacity] - [Kube-Reserved] - [System-Reserved] - [Hard-Eviction-Threshold]
```
调度器将使用Allocatable而不是Capacity来进行调度。同时kubelet也将用其进行admission检查。

需要注意的是，kernel使用的系统资源会一直变动，并且是在kubernetes的控制之外，将会通过其他途径report，比如metrics API，这部分不在本次设计范围之内。

#### Kube-Reserved
Kube-Reserved是预留给kubernetes各系统模块的资源。将由kubelet启动时通过配置或命令行提供，因此在kubelet运行期不可改变。

该项数据通过序列化的ResourceList来配置，比如
```
--kube-reserved=cpu=500m,memory=5Mi
```
在第一版本将支持cpu和memory，后续可扩展到比如系统磁盘，系统io等

#### System-Reserved
System-Reserved的机制类似于kube-Reserved，区别在于其代表了非kubernetes模块的资源。

#### kubelet Eviction Thresholds
为了提高节点的可靠性，kubelet将会在耗尽内存或磁盘的时候清理一些pods。

在v1.5版本，通过计算相对于节点总容量的资源使用情况来，以及QoS和用户设定的阈值来清理pods。

从v1.6版本开始，如果节点具有Allocatable数据，那么pods所使用的资源将不能超过该值。内存和CPU通过cgroup进行控制，但是磁盘使用并无有效途径来控制。使用linux quota控制磁盘使用并不可行，因为不是hierarchical。在Allocatable支持磁盘资源之前，kubelet将必须同时根据capacity和allocatable进行pods清理。

需要注意的是，清理阈值只适用于pods，系统进程将不受影响，除非有system-reserved限制。

举个例子：
如果节点资源有32Gi 内存 (Capacity)，kube-reserved is 2Gi, system-reserved is 1Gi, eviction-hard is `<100Mi`
那么对于该节点，Node Allocatable is 28.9Gi。如果我们通过top level cgroup来限制28.9Gi的Allocatable，那么pods所使用资源将不会超过28.9Gi，在kernel消耗资源大于100Mi时，将会进行pods清理。

## 基本指南
系统后台进程应该享有‘Guaranteed’pods的优先级，他们可以在自己的cgroup范围内burst，这需要作为kubernetes部署的一部分来管理。比如说，kubelet可以拥有其自己的cgroup，并且和容器运行时共享kube-reserved资源。但是，一旦KubeReserved限制有效，那么kubelet则不可能占用节点所有的系统资源。

当用户决定启用SystemReserved资源限制时需要格外当心，因为这可能导致关键服务无法调度CPU，或者耗尽内存而被系统kill。我们的建议是，只有在用户做了非常详细的系统性能测试并且得出了精确的结果再启用SystemReserved资源限制。

一开始，可以只是启用Allocatable限制on pods only。在具有详细的系统监控、报警服务来跟踪kubernetes系统进程时，再根据heuristics来启用KubeReserved。
更多请见 [Phase 2](#phase-2-enforce-allocatable-on-pods)

随着kubernetes的发展，kubernetes系统进程所占用资源将逐渐增加。开发团队将尝试优化，但目前这优先级较低。所以，可以预见的是，在系统总资源量中，Allocatable将会有所降低。

systemd-logind 为用户ssh在/user.slice中创建session数据，这部分资源使用将不被计算在节点容量中。所以在配置systemReserved资源时，请将/user.slice考虑其中。理想情况下，/user.slice应该存在于systemReserved的顶层cgroup中。

## Cgroup推荐设置
以下为kubernetes节点推荐的cgroup配置。所有的OS进程应该位于一个顶层的SystemReserved cgroup。Kubelet和container runtime 应该位于KubeReserved cgroup。Container runtime这么配置的原因如下：

 - kubernetes节点中的container runtime应该只是由kubelet使用，不应该被移作他用
 - container runtime的资源使用和节点中运行的pods息息相关

 以下实例推荐使用专门的cgroup分别跟踪kubelet和runtime。
```
 / (Cgroup Root)
.
+..systemreserved or system.slice (Specified via `--system-reserved-cgroup`; `SystemReserved` enforced here *optionally* by kubelet)
.        .    .tasks(sshd,udev,etc)
.
.
+..podruntime or podruntime.slice (Specified via `--kube-reserved-cgroup`; `KubeReserved` enforced here *optionally* by kubelet)
.	 .
.	 +..kubelet
.	 .   .tasks(kubelet)
.        .
.	 +..runtime
.	     .tasks(docker-engine, containerd)
.	 
.
+..kubepods or kubepods.slice (Node Allocatable enforced here by Kubelet)
.	 .
.	 +..PodGuaranteed
.	 .	  .
.	 .	  +..Container1
.	 .	  .        .tasks(container processes)
.	 .	  .
.	 .        +..PodOverhead
.	 .        .        .tasks(per-pod processes)
.	 .        ...
.	 .
.	 +..Burstable
.	 .	  .
.	 .	  +..PodBurstable
.	 .	  .	    .
.	 .	  .	    +..Container1
.	 .	  .	    .         .tasks(container processes)
.	 .	  .	    +..Container2
.	 .	  .	    .         .tasks(container processes)
.	 .	  .	    .
.      	 .     	  .    	    ...
.	 .	  .
.	 .	  ...
.	 .
. 	 .
.	 +..Besteffort
.	 .	  .
.	 .	  +..PodBesteffort
.	 .	  .    	    .
.	 .	  .	    +..Container1
.	 .	  .	    .         .tasks(container processes)
.	 .	  .	    +..Container2
.	 .	  .	    .         .tasks(container processes)
.	 .	  .	    .
.      	 .     	  .    	    ...
.	 .	  .
. 	 .	  ...

 ```


SystemReserved 和 KubeReserved cgroups应该由用户创建。如果由kubelet创建为其自己以及docker daemon创建cgroup，那么他也将自动创建KubeReserved cgroup。

kubePods cgroup将由kubelet自动创建，其创建将和QoS Cgroup支持相关联，通过--cgroups-per-qos来设置。如果cgroup驱动设置为systemd，那么kubelet将通过systemd来创建kubepods.slice。默认情况下，kubelet将会通过 cgroupfs自动执行`mkdir /kubepods` 来创建。

#### kubelet容器化
如果kubelet本身也通过container runtime来管理，那么runtime将为kubelet在KubeReserved cgroup下创建。

### Metrics
Kubelet监控自身的cgroup并将其资源使用情况通过Summary metrics API (/stats/summary)对外发布。对于docker runtime，kubelet将会同时监控docker runtime的cgroup。为了提供节点全面的资源使用情况，kubelet将会expose metrics from cgroups enforcing `SystemReserved`, `KubeReserved` & `Allocatable` too.

## 实现阶段

### Phase 1 - 引入Allocatable但不启用
**Status**: 已经在v1.2中实现
在这个阶段中，kubelet将会支持命令行选项设置KubeReserved SystemReserved 资源配置。选项的默认设置为`""`，代表不对cpu和mem设限。kubelet将会计算Allocatable，并更新至Node.Status。调度器将使用Allocatable进行调度。

### Phase 2 - 启动Allocatable限制
***Status**: 目标在v1.6
在本阶段，kubelet将会自动创建顶层cgroup来enforce Node Allocatable，across all user pods。可以根据命令行选项`--cgroups-per-qos`来控制创建。

kubelet将支持为KubeReserved何SystemReserved指定顶层cgroup，并且支持在该cgroup中设置资源限额 (可选)

用户可根据其部署需求来设置KubeReserved以及SystemReserved cgroups。

kubelet和runtime所使用的系统资源通常情况下和pods的数量成正比。在用户确定最大的pod密度以后，则可以通过[系统性能面板](http://node-perf-dash.k8s.io/#/builds)来计算KubeReserved数值。[这篇blog](http://blog.kubernetes.io/2016/11/visualize-kubelet-performance-with-node-dashboard.html)具体解释了怎么使用该性能面板。需要注意的是，截至目前，面板仅仅提供了docker runtime的使用详情。

在该阶段也将引入pod清理。

新引入的flag如下：

1. `--enforce-node-allocatable=[pods][,][kube-reserved][,][system-reserved]`
	* 在v1.6中，该标识将默认为'pods'
	* 该标识只在设置了`--kube-reserved`或`--system-reserved`标识之后才有效
	* 如果`--cgroups-per-qos=false`，那么该标识将设置为`""`，否则kubelet将会出错退出
	* 在升级至v1.6之前，建议将节点设置drain并重启。这也是在v1.6中默认启动的`--cgroups-per-qos`选项。
	* 用户可以将该标识设置为`""`来关闭该功能
	* 如果`--kube-reserved-cgoup`未设置，那么该标识设置为`kube-reserved`是无效的
	* 如果`--system-reserved-cgoup`未设置，那么该标识设置为`system-reserved`是无效的
	* 如果该标识设置包含`kube-reserved`或者`system-reserved`，并且设置了以下两个标识，那么kubelet将会启动资源控制。
2. `--kube-reserved-cgroup=<absolute path to a cgroup>`
	* 该标识使得kubelet识别管理所有出于kubeReserved之下的kubernetes系统进程的cgroup，比如`/kube.slice`。需要使用绝对路径，并且不支持systemd命名规则。
3. `--system-reserved-cgroup-<absolute path to a cgroup>`
	* 该标识使得kubelet识别管理所有处于SystemReserved之下的OS系统进程cgroup，比如`/system.slice`。需要使用绝对路径，并且不支持systemd命名规则。
4. `--experimental-node-allocatable-ignore-eviction-threshold`
	* 这个标识用于`opt-out`选项，避免Node Allocatable中包含Hard eviction，从而避免影响已经部署的集群。默认值为false

#### Rollout 细节
该阶段将会提升节点稳定性，但是需要用户设置--kube-reserved和--system-reserved选项。

由于`KubeReserved`和`SystemReserved`默认值为`""`，所以节点的`Allocatable`不会自动变化。由于该阶段需要Node drains (pod restarts/terminations) 因此对于用户而言是比较具有破坏性的(disruptive)

如果要回滚，那么设置`--enforce-node-allocatable=""`来取消Node Allocatable，以及`--experimental-node-allocatable-ignore-eviction-threshold=true`来避免hard eviction。

系统升级可能会导致以下问题：
1. 如果`--kube-reserved`和`--system-reserved`都有设置，那么OOM将会杀死容器，或者清理pods。这将主要针对`Burstable`和`BestEffort`pods，因为这些pods无法使用节点的全部资源。
2. 节点全部可分配资源将减少，导致pods处于`Pending`状态，因为hard eviction thresholds已经考虑到node allocatable中。


### Phase 3 - Metrics & support for storage
*Status*: 目标v1.7
在本阶段，kubelet将expose 资源使用metrics，包括`kuberserved, systemresered, Allocatable` 顶层的cgroups。同时也将引入Storage作为可预留的资源。

## Know Issues
### Kubernetes 预留资源小于 kubernetes 系统资源使用
**Solution**: 一开始，do nothing （best effort)。让kubernetes系统进程overflow，and hope for the best。如果节点使用资源小于Allocatable，那么kubernetes进程仍然有空间可以overflow。如果节点使用资源已经到达Allocatable，那么节点可能处于不稳定状态。

另外一个方法是，在kubelet支持后（phase 2），就开启KubeReserved。以后我们可能根据KubeReserved设置一个父cgroup for kubernetes 进程。

### 第三方调度器
社区应该会收到通知，调度器应该更新，但是如果调度器没有更新，那么将默认不考了allocatable。













	

















