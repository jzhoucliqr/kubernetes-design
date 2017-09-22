# kubernetes apiserver 设计与实现深入解析(v1.7)

如果你想要了解apiserver的设计，并且一上来就从看代码开始，那么很有可能会像我一样感到一头雾水，无从下手。
本文试着从另一个角度来解析apiserver的设计与实现。读完本文，希望你能在头脑中有一个基本的认识，如果你自己需要设计类似的api系统，该从何下手。

我们首先从apiserver的最基本功能入手，然后思考如果我们自己来设计一个系统，为了实现这些最基本功能所需要的设计方法，以及
系统实现所需要的基本模块。然后和现有apiserver的设计实现做对比来了解其设计思路。最后逐步添加
高级功能并思考如何在现有的基本设计上来实现。
通过这样的方式你会发现，即使复杂如apiserver的系统，虽然代码有重重抽象，但也是有迹可循，并非无从下手。

## apiserver的基本功能
apiserver的最基本功能是为用户提供REST API 实现对系统资源的CURD。所以从REST API SERVER出发，apiserver需要的功能模块有:

. RESTful resource 定义与存储
. URI to resource 匹配

而apiserver也确实是从这两条基本线展开。但是由于其增加了很多抽象和更多功能，所以代码看起来比较不那么smooth。

### resource definition
每一个resource有其scheme和type的定义。

### API group
API group 将相关的resource组合在一起，统一管理其升级等。

### multi-version
apiserver通过apigroup来管理不同group之间的版本独立。
对于每个资源，有internal-type，和对每个版本的object。internal-type在db中存储。对于API，将会有相应的conversion，进行转换。这样来支持多版本。

### API 动态扩展
API可以动态扩展。第三方apiserver可以在运行时注册新的API资源到核心apiserver. 核心apiserver将对应的请求proxy到相应的apiserver。这部分功能由kube-aggregator实现。

### URI to resource 匹配


 
 .



## api handler install
cmd/kube-apiserver/app/server.go
    CreateServerChain->CreateKubeAPIServer->kubeAPIServerConfig.New()
pkg/master/master.go
    New()->installAPIs(RESTStorageProvider...)->GenericAPIServer.InstallAPIGroup(apiGroup)
staging/apiserver/pkg/server/genericapiserver.go
    InstallAPIGroup()->installAPIResources()->apiVersion.InstallREST(goRestfulContainer)
staging/apiserver/pkg/endpoints/groupversion.go
    InstallREST(container)->installer.Install(ws)
staging/apiserver/pkg/endpoints/installer.go
    APIInstaller.Install(ws)->registerResourceHandlers()->ws.Route(route)

## api scheme install
server.go -> import pkg/master
pkg/master.init() -> import all /install for api groups
Install: Add scheme registry to global factory registry

apimachinery.api.meta.RESTMapper: resource (url) to groupVersionKind
apimechinery.runtime.Scheme: GroupVersionKind to the actual golang struct

## api object serialization/deserialize

## sharedInformer
https://github.com/kubernetes/kubernetes/issues/27984
@hongchaodeng have you gotten a handle on the current semantics? I think I'd summarize it like this.

On the producer side:

A reflector watches an API endpoint.
For each watch event, it queues the event in the delta fifo.
The event's object is inspected to determine a key.
The delta fifo uses the key to add a delta to the list of deltas associated with the key.
It also adds the key to a list of "these keys were changed". If the key is already in "these keys were changed"
On the consumer side:

call pop and wait until a key is available in the "these keys were changed"
find all the deltas for that key
call the processFunc with all the deltas
I think we're pretty happy with those semantics. It's like a first come, first served, but don't waste time with outdated information.

There are details around compression I didn't get into, but you can choose to compress the list of deltas to eliminate unnecessary ones based on a function of your choosing.

## multiversion
// 0. Your API objects have a common metadata struct member, TypeMeta.
// 1. Your code refers to an internal set of API objects.
// 2. In a separate package, you have an external set of API objects.
// 3. The external set is considered to be versioned, and no breaking
//    changes are ever made to it (fields may be added but not changed
//    or removed).
// 4. As your api evolves, you'll make an additional versioned package
//    with every major change.
// 5. Versioned packages have conversion functions which convert to
//    and from the internal version.
// 6. You'll continue to support older versions according to your
//    deprecation policy, and you can easily provide a program/library
//    to update old versions into new versions because of 5.
// 7. All of your serializations and deserializations are handled in a
//    centralized place.
/
