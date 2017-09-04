## apiserver design and implementation

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
