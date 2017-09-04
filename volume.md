[readmore](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/volumes.md)

# 存储卷设计
## 动机
不管容器运行的UID为多少，kubernetes的存储卷都应该可用。这个要求适用于所有的存储卷类型，所以需要有一个统一适用于所有存储卷类型的解决方案。

设计目标：
1. 列举所有的存储卷使用案例
2. 定义kubernetes中所有权和权限控制的目标状态

设计说明：
1. 本文中涉及到权限时，`D`代表任意值。比如，`07D0`权限代表该文件所有者具有`7`权限，其他人权限为`0`，同组的权限为任意值。
2. 容器存储卷读写权限定义为：
	- 存储卷由容器的UID所拥有，权限为`07D0`
	- 存储卷由容器的GID，或者某个supplemental group所拥有，权限为`0D70`
3. 存储卷插件不应该设置存储卷的权限
4. 目前不支持限制同一个pod中不同容器对同一个存储卷的读写，（通过使用不同的容器UID）
5. 不支持同一个容器中不同的进程使用不同的UID。如果有需求使用不同的UID，应该将不同的UID定义为不同的pods。

## 当前情况
### kubernetes
kubernetes的存储卷可分为两大类：
1. 非共享存储：
	- 这类存储卷存在于主机的本地文件系统：空目录，git repo, secret, downard api, 等。所有的类型都使用`EmptyDir`作为其底层的实现。这些存储卷的权限设置为`root:root`
	- 基于网络块存储设备：比如 AWS EBS, iSCSI, RBD, 等。这些只能供一个pod使用
2. 共享存储：
	- `hostPath` 属于共享存储，因为其被容器和主机同时使用
	- 网络文件系统，如NFS GlusterFS, Cephfs等。这些存储卷权限由共享存储系统本身来确定
	- 基于可共享的块设备的存储卷，可能为`ReadOnlyMany`或者`ReadWriteMany`模式，这些存储卷可能同时被多个pod使用。

`EmptyDir`在最近被从`0750`修改为`0777`权限，以便非root UID可以使用。

### Docker
docker 近期添加了supplemental group支持。这个使得可以为容器指定额外的组。