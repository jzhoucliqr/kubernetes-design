https://github.com/kubernetes/kubernetes/pull/18261

https://github.com/kubernetes/community/blob/master/contributors/design-proposals/nodeaffinity.md

# Node affinity and NodeSelector

## Introduction

本文档介绍了一种新的标签选择器: "NodeSelector"。NodeSelector在很多方面和LabelSelector很类似，但是更灵活，并且专门用于节点选择。

节点选择举例：Node from a shared pool with greater than a certain kernel version that doesn't have gpus but does the FUSE kernel module.

当前调度器使用`PodSpec`中的`map[string]string`字段来确定是否可将pod调度到某节点。本文档介绍一个新的字段`Affinity`，该字段包含一个或多个affinity信息。我们将讨论`NodeAffinity`，详情如下：

* 字段`RequiredDuringSchedulingRequiredDuringExecution `，该字段为一个`NodeSelector`，将当前调度器

```go
type Affinity struct {
	NodeAffinity *NodeAffinity `json:"nodeAffinity,omitempty"`
}

type NodeAffinity struct {
	// pod调度时只会调度到满足该要求的节点
	// pod调度之后，在运行阶段，如果运行该pod的节点不再满足要求，比如节点标签更新
	// 那么系统也会最终将该pod清除出该节点
	RequiredDuringSchedulingRequiredDuringExecution *NodeSelector  `json:"requiredDuringSchedulingRequiredDuringExecution,omitempty"`
	// pod调度时只会调度到满足该要求的节点
	// 运行时如果节点不再满足，系统不会追求将pod清除出该节点
	RequiredDuringSchedulingIgnoredDuringExecution  *NodeSelector  `json:"requiredDuringSchedulingIgnoredDuringExecution,omitempty"`
	// 调度时会倾向于将pod调度到满足该要求的节点
	// 但是也可能会选择到并非完全满足该要求的节点
	// 可根据该要求计算节点的优先级，比如对于所有满足调度要求的节点
	// (比如RequiredDuringScheduling, 资源需求)
	// 将该字段的所有要求按照一定的权值加权，加权值最高的节点优先级最高
	PreferredDuringSchedulingIgnoredDuringExecution []PreferredSchedulingTerm  `json:"preferredDuringSchedulingIgnoredDuringExecution,omitempty"`
}
```
这样的设计可以让pod指定节点特定的需求，比如：Pod1需要Intel CPU，或者Pod2需要运行在zone 2。


### NodeSelector

```go
// A node selector represents the union of the results of one or more label queries
// over a set of nodes; that is, it represents the OR of the selectors represented
// by the nodeSelectorTerms.
type NodeSelector struct {
	// nodeSelectorTerms is a list of node selector terms. The terms are ORed.
	NodeSelectorTerms []NodeSelectorTerm `json:"nodeSelectorTerms,omitempty"`
}

// An empty node selector term matches all objects. A null node selector term
// matches no objects.
type NodeSelectorTerm struct {
	// matchExpressions is a list of node selector requirements. The requirements are ANDed.
	MatchExpressions []NodeSelectorRequirement `json:"matchExpressions,omitempty"`
}

// A node selector requirement is a selector that contains values, a key, and an operator
// that relates the key and values.
type NodeSelectorRequirement struct {
	// key is the label key that the selector applies to.
	Key string `json:"key" patchStrategy:"merge" patchMergeKey:"key"`
	// operator represents a key's relationship to a set of values.
	// Valid operators are In, NotIn, Exists, DoesNotExist. Gt, and Lt.
	Operator NodeSelectorOperator `json:"operator"`
	// values is an array of string values. If the operator is In or NotIn,
	// the values array must be non-empty. If the operator is Exists or DoesNotExist,
	// the values array must be empty. If the operator is Gt or Lt, the values
	// array must have a single element, which will be interpreted as an integer.
    // This array is replaced during a strategic merge patch.
	Values []string `json:"values,omitempty"`
}

// A node selector operator is the set of operators that can be used in
// a node selector requirement.
type NodeSelectorOperator string

const (
	NodeSelectorOpIn           NodeSelectorOperator = "In"
	NodeSelectorOpNotIn        NodeSelectorOperator = "NotIn"
	NodeSelectorOpExists       NodeSelectorOperator = "Exists"
	NodeSelectorOpDoesNotExist NodeSelectorOperator = "DoesNotExist"
	NodeSelectorOpGt           NodeSelectorOperator = "Gt"
	NodeSelectorOpLt           NodeSelectorOperator = "Lt"
)
```

### LabelSelector
```go
/ A label selector is a label query over a set of resources. The result of matchLabels and
// matchExpressions are ANDed. An empty label selector matches all objects. A null
// label selector matches no objects.
type LabelSelector struct {
	// matchLabels is a map of {key,value} pairs. A single {key,value} in the matchLabels
	// map is equivalent to an element of matchExpressions, whose key field is "key", the
	// operator is "In", and the values array contains only "value". The requirements are ANDed.
	// +optional
	MatchLabels map[string]string `json:"matchLabels,omitempty" protobuf:"bytes,1,rep,name=matchLabels"`
	// matchExpressions is a list of label selector requirements. The requirements are ANDed.
	// +optional
	MatchExpressions []LabelSelectorRequirement `json:"matchExpressions,omitempty" protobuf:"bytes,2,rep,name=matchExpressions"`
}

// A label selector requirement is a selector that contains values, a key, and an operator that
// relates the key and values.
type LabelSelectorRequirement struct {
	// key is the label key that the selector applies to.
	// +patchMergeKey=key
	// +patchStrategy=merge
	Key string `json:"key" patchStrategy:"merge" patchMergeKey:"key" protobuf:"bytes,1,opt,name=key"`
	// operator represents a key's relationship to a set of values.
	// Valid operators ard In, NotIn, Exists and DoesNotExist.
	Operator LabelSelectorOperator `json:"operator" protobuf:"bytes,2,opt,name=operator,casttype=LabelSelectorOperator"`
	// values is an array of string values. If the operator is In or NotIn,
	// the values array must be non-empty. If the operator is Exists or DoesNotExist,
	// the values array must be empty. This array is replaced during a strategic
	// merge patch.
	// +optional
	Values []string `json:"values,omitempty" protobuf:"bytes,3,rep,name=values"`
}

// A label selector operator is the set of operators that can be used in a selector requirement.
type LabelSelectorOperator string

const (
	LabelSelectorOpIn           LabelSelectorOperator = "In"
	LabelSelectorOpNotIn        LabelSelectorOperator = "NotIn"
	LabelSelectorOpExists       LabelSelectorOperator = "Exists"
	LabelSelectorOpDoesNotExist LabelSelectorOperator = "DoesNotExist"
)
```

为什么需要重新定义一个NodeSelector而不是重用LabelSelector呢？通过其定义可以看出，这两者主要的区别有两点：

1. LabelSelector在多个MatchExpression取And。而NodeSelector在多个MatchExpression取And之外又定义了顶层的OR操作。目前为止这个OR并没有具体的使用案例，只是预留以后扩展，所以这一点并非其本质的区别
2. NodeSelector比LabelSelector的操作运算多了两个：OpGt 以及OpLt。那么为什么不能将这两个操作符添加到LabelSelector中呢？主要原因是LabelSelector是系统中使用非常频繁的操作，因此对性能要求非常高。其他的操作符可以创建Forward和Reverse Index以达到最高的查询性能，添加了这个操作符会导致需要更复杂的查询算法，最终导致较低的系统性能。这也是为什么没有使用Regex的原因。另外一个原因是，LabelSelector目的是为了解决一些通用的问题，但是需要避免过于通用而导致太难于理解。比如通过LabelSelector来选择某个service或者replicaSet的pod，我们应该使得LabelSelector可以一目了然，而不需要用户进行复杂的计算才能搞清楚哪些pod属于哪些service，多个service的pod之间是否有交集等。
3. NodeSelector用于选择节点，节点所具有的属性可能差别较大，因此需要更灵活的选择方式。而对于Pod而言，其属性及标签都由用户定义，我们需要一定程度上来限制可能出现的过于复杂过于灵活的选择方式。



其他方法：

1. Constraints/Preference
2. Plugin to extend
3. Regular Expression
4. Resources? 
5. Sliding Scale of Affinity: STRONGLY_PREFERRED/PREERRED/WEAKLY_PREFERRED


One other comment: I think the idea of allowing a "sliding scale" of affinity is an interesting idea, but I have two comments:
(1) It seems that REQUIRED is always going to have to be treated differently from everything else, for the same reason scheduling separates the concepts of "predicates" and "priority functions" (or correspondingly in Borg, "feasibility" and "scoring").
(2) So then the question is, is there usefulness to being able to specify different degrees of non-requiredness? My initial thought here is that it gives the user the illusion of more control than we can realistically (or at least easily) give them. What would the semantics of STRONGLY_PREFERRED/PREERRED/WEAKLY_PREFERRED be in terms of a scheduling algorithm? Would we guarantee that we would never satisfy a WEAKLY_PREFERRED affinity if there is a different node available where we could satisfy a STRONGLY_PREFERRED affinity? How about if there is a node that satisfies three WEAKLY_PREFERRED affinities vs. a node that satisfies one STRONGLY_PREFERRED affinity? Moreover, given the way we do priority functions in the Kubernetes scheduler today (adding up the output of a set of functions) I am not sure it is even possible to give consistent semantics. And if we just say "these are hints, we make no guarantees, the scheduler will do what it wants" then I'm not sure the user gets any benefit compared to just having one dial setting for everything that isn't a strict requirement.















why need a new NodeSelector, rather than use a general one: LabelSelector

We have two choices:
(1) represent node selection using LabelSelector (used to be called PodSelector)
(2) represent node selection using something new

I feel strongly that for node selection, we need Gt and Lt, and somewhat less strongly that we need top-level OR. This means that for (1) we'd need need to add Gt/Lt and top-level OR to LabelSelector. @bgrant0607 was very opposed to this -- I believe his reason was that you want to build a standard set of forward and reverse indexes for LabelSelector because lookup speed is very important for selection operations related to pods, and having more complex expressions (like Gt/Lt) makes such implementation more complicated. The only option that satisfies his and my requirements is (2), hence NodeSelector as a separate type.

(Example use case for Gt is "OS version is greater than ...")
  @bgrant0607
  bgrant0607 on Jan 6, 2016 Owner
  Performance is part of the reason (e.g., that's one reason why regexp isn't supported), but also semantics and user experience.

  As @mikedanese pointed out, we may need to select nodes based on arbitrary attributes, such as fields and annotations. I don't want to allow that flexibility for pods, where users fully control both the specifications of the pods and the labels attached to them.

  Also, the selection power is deliberately restricted to address common use cases but not be so general and complex that it's hard to reason about. We want to be able to visualize overlapping sets of pods in a sane way, such as to represent a service and replication controllers.

  Users and a number of components also need to reason about set overlap. The controllers themselves are one example, but load balancers and monitoring systems do as well, so that pods aren't double counted. The restricted label selection scheme makes overlap determination pretty straightforward, especially when employing simple conventions (pods having label keys in common), which could be enforced with the mechanism discussed in #17097.

  
