## How pod is implemented

A pod (as in a pod of whales or pea pod) is a group of one or more containers (such as Docker containers), the shared storage for those containers, and options about how to run the containers. A pod’s contents are always co-located and co-scheduled, and run in a shared context. A pod models an application-specific “logical host” - it contains one or more application containers which are relatively tightly coupled — in a pre-container world, they would have executed on the same physical or virtual machine.
While Kubernetes supports more container runtimes than just Docker, Docker is the most commonly known runtime, and it helps to describe pods in Docker terms.
The shared context of a pod is a set of Linux namespaces, cgroups, and potentially other facets of isolation - the same things that isolate a Docker container. Within a pod’s context, the individual applications may have further sub-isolations applied.
Containers within a pod share an IP address and port space, and can find each other via localhost. They can also communicate with each other using standard inter-process communications like SystemV semaphores or POSIX shared memory. Containers in different pods have distinct IP addresses and can not communicate by IPC.
Applications within a pod also have access to shared volumes, which are defined as part of a pod and are made available to be mounted into each application’s filesystem.
In terms of Docker constructs, a pod is modelled as a group of Docker containers with shared namespaces and shared volumes. PID namespace sharing is not yet implemented in Docker.
Like individual application containers, pods are considered to be relatively ephemeral (rather than durable) entities. As discussed in life of a pod, pods are created, assigned a unique ID (UID), and scheduled to nodes where they remain until termination (according to restart policy) or deletion. If a node dies, the pods scheduled to that node are scheduled for deletion, after a timeout period. A given pod (as defined by a UID) is not “rescheduled” to a new node; instead, it can be replaced by an identical pod, with even the same name if desired, but with a new UID (see replication controller for more details). (In the future, a higher-level API may support pod migration.)
When something is said to have the same lifetime as a pod, such as a volume, that means that it exists as long as that pod (with that UID) exists. If that pod is deleted for any reason, even if an identical replacement is created, the related thing (e.g. volume) is also destroyed and created anew.

kubelet/dockershim/docker_sandbox.go

// RunPodSandbox creates and starts a pod-level sandbox. Runtimes should ensure
// the sandbox is in ready state.
// For docker, PodSandbox is implemented by a container holding the network
// namespace for the pod.
// Note: docker doesn't use LogDirectory (yet).
func (ds *dockerService) RunPodSandbox(config *runtimeapi.PodSandboxConfig) (id string, err error) {



kubelet/dockershim/docker_container.go
CreateContainer()
	securityOpts, err := ds.getSecurityOpts(config.Metadata.Name, sandboxConfig, securityOptSep)

func applySandboxSecurityContext(lc *runtimeapi.LinuxPodSandboxConfig, config *dockercontainer.Config, hc *dockercontainer.HostConfig, network *knetwork.PluginManager, separator rune) error {
	modifySandboxNamespaceOptions(sc.GetNamespaceOptions(), hc, network)
	modifyHostNetworkOptionForSandbox(hostNetwork, network, hostConfig)

func modifyHostNetworkOptionForContainer(hostNetwork bool, sandboxID string, hc *dockercontainer.HostConfig) {
	sandboxNSMode := fmt.Sprintf("container:%v", sandboxID)
	hc.NetworkMode = dockercontainer.NetworkMode(sandboxNSMode)
	hc.IpcMode = dockercontainer.IpcMode(sandboxNSMode)
	hc.UTSMode = ""

    NetworkMode - Sets the networking mode for the container. Supported standard values are: bridge, host, none, and container:<name|id>. Any other value is taken as a custom network’s name to which this container should connect to.
