---
title: Manage API Server load with the Flow Control API
content_template: templates/concept
min-kubernetes-server-version: v1.18
---

{{% capture overview %}}

Controlling the behavior of the Kubernetes API server in an overload situation
is a key task for cluster administrators. The `kube-apiserver` has some controls
available (i.e. the `--max-requests-inflight` and
`--max-mutating-requests-inflight` command-line flags) to limit the amount of
outstanding work that will be accepted, preventing a flood of inbound requests
from overloading and potentially crashing the API server, but beyond that, the
Priority and Fairness (or flow-control) API provides a configurable mechanism to
ensure that in a high-load situation, forward progress will be made on the right
requests. It also allows for a limited amount of queuing, so that in the event
of _bursty_ traffic that remains below acceptable levels on average, no requests
need be dropped.

{{% /capture %}}

{{% capture body %}}

## Enabling the Flow Control API

{{< feature-state state="alpha" >}}

The Flow Control API is part of the API Priority and Fairness
[feature](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/20190228-priority-and-fairness.md),
which is presently in alpha. To enable it, add the following command-line flags
to your `kube-apiserver` invocation:

```shell
kube-apiserver \
--feature-gates=APIPriorityAndFairness=true \
--runtime-config=flowcontrol.apiserver.k8s.io/v1alpha1=true \
(other flags as usual)
```

## Concepts
There are several distinct features involved in the flow control API. Incoming
requests are classified by attributes of the request using FlowSchemas, and
assigned to priority levels. Priority levels add a degree of isolation by
maintaining separate concurrency limits, so that requests assigned to different
priority levels cannot starve each other. Within a priority level, a
fair-queuing algorithm prevents requests from different sources from starving
each other, and allows for requests to be queued to prevent bursty traffic from
causing failed requests when the average load is acceptably low.

### Priority Levels
Without the Priority and Fairness feature enabled, overall concurrency in the
API server is limited by the `kube-apiserver` flags `--max-requests-inflight`
and `--max-mutating-requests-inflight`. With the feature enbaled, the
concurrency limit defined by these flags is divided up among a configurable set
of _priority levels_. Incoming requests are assigned to a single priority level,
and each priority level will only dispatch as many concurrent requests as its
configuration allows.

The default configuration, for example, includes separate priority levels for
leader-election requests, requests from built-in controllers, and requests from
Pods. This means that an ill-behaved Pod that floods the API server with
requests cannot prevent leader election or actions by the built-in controllers
from succeeding.

### Fair Queuing
Even within a priority level there may be a large number of distinct sources of
traffic. In an overload situation, it is valuable to prevent one consumer from
starving the others. This is handled by use of a fair-queuing algorithm to
process requests that are assigned the same priority level. Each request is
given a _flow distinguisher_ — in the current design, this is either the
requesting user or the resource's namespace — and the system attempts to give
approximately equal weight to requests with different flow distinguishers.

The details of the queuing algorithm are tunable for each priority level, and
allow administrators to trade off memory use against fairness and tolerance for
bursty traffic.

### Exempt requests
Some requests are considered sufficiently important that they are not subject to
any of the limitations imposed by this feature. This prevents an
improperly-configured flow control configuration from totally disabling an API
server.

## Defaults
The Priority and Fairness feature ships with a default configuration that the
authors hope will work well for most clusters. It groups requests into five
priority classes:

* The `system` priority level is for requests from the `system:nodes` group,
  i.e. Kubelets, which must be able to contact the API server in order for
  workloads to be able to schedule on them.

* The `leader-election` priority level is for leader election requests from
  built-in controllers (in particular, requests for `endpoints`, `configmaps`,
  or `leases` coming from the `system:kube-controller-manager` or
  `system:kube-scheduler` users and service accounts in the `kube-system`
  namespace). These are important to isolate from other traffic because failures
  in leader election cause their controllers to fail and restart, which in turn
  causes more expensive traffic as the new controllers sync their informers.
  
* The `workload-high` priority level is for other requests from built-in
  controllers.
  
* The `workload-low` priority level is for requests from any other service
  account, which will typically include all requests from controllers runing in
  Pods.
  
* The `global-default` priority level handles all other traffic, e.g.
  interactive `kubectl` commands run by nonprivileged users.

Additionally, there are two PriorityLevelConfigurations and two FlowSchemas that
are built in and may not be overwritten:

* The special `exempt` priority level is used for requests that are not subject
  to flow control at all: they will always be dispatched immediately. The
  special `exempt` FlowSchema classifies all requests from the `system:master`
  group into this priority level. You may define other FlowSchemas that direct
  other requests to this priority level, if appropriate.
  
* The special `catch-all` priority level is used in combination with the special
  `catch-all` FlowSchema to make sure that every request gets some kind of
  classification. Typically you should not rely on this catch-all configuration,
  and should create your own catch-all FlowSchema and PriorityLevelConfiguration
  (or use the `global-default` configuration that is installed by default) as
  appropriate. To help catch configuration errors that miss classifying some
  requests, the mandatory `catch-all` priority level only allows one concurrency
  share and does not queue requests, making it relatively likely that traffic
  that only matches the `catch-all` FlowSchema will be rejected with an HTTP 429
  error.

## Resources
The flow control API involves two kinds of resources.
[PriorityLevelConfigurations](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#prioritylevelconfiguration-v1alpha1-flowcontrol) 
define the available isolation classes, the share of the available concurrency
budget that each can handle, and allow for fine-tuning queuing behavior.
[FlowSchemas](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#flowschema-v1alpha1-flowcontrol)
are used to classify individual inbound requests, matching each to a single
PriorityLevelConfiguration.

### PriorityLevelConfiguration
A PriorityLevelConfiguration represents a single isolation class. Each
PriorityLevelConfiguration has an independent limit on the number of outstanding
requests, and limitations on the number of queued requests. An example
PriorityLevelConfiguration looks like this:

Concurrency limits for PriorityLevelConfigurations are not specified in absolute
number of requests, but rather in "concurrency shares." The total concurrency
limit for the API Server is distributed among the existing
PriorityLevelConfigurations in proportion with these shares. This allows a
cluster administrator to scale up or down the total amount of traffic to a
server by restarting `kube-apiserver` with a different value for
`--max-requests-inflight` (or `--max-mutating-requests-inflight`), and all
PriorityLevelConfigurations will see their maximum allowed concurrency go up (or
down) by the same fraction.
{{< caution >}}
With the Priority and Fairness feature enabled, the total concurrency limit for
the server is set to the sum of `--max-requests-inflight` and
`--max-mutating-requests-inflight`. There is no longer any distinction made
between mutating and non-mutating requests; if you want to treat them
separately for a given resource, make separate FlowSchemas that match the
mutating and non-mutating verbs respectively.
{{< /caution >}}
{{< caution >}}
In the present implementation, requests classified as "long-running" — primarily
watches — are not subject to the API Priority and Fairness filter at all. This
is expected to change.
{{< /caution >}}

When the volume of inbound requests assigned to a single
PriorityLevelConfiguration is more than its permitted concurrency level, the
`type` field of its specification determines what will happen to extra requests.
A type of `Reject` means that excess traffic will immediately be rejected with
an HTTP 429 (Too Many Requests) error. A type of `Queue` means that requests
above the threshhold will be queued, and the fair queuing algorithm will be used
to balance progress between requests based on their assigned flow distinguisher.

The queuing configuration allows tuning the fair queuing algorithm for a
priority level. Details of the algorithm can be read in the
[KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/20190228-priority-and-fairness.md),
but in short:

* Increasing `queues` reduces the rate of collisions between different flows, at
  the cost of increased memory usage. A value of 1 here effectively disables the
  fair-queuing logic, but still allows requests to be queued.

* Increasing `queueLengthLimit` allows larger bursts of traffic to be sustained
  without dropping any requests, at the cost of increased memory usage.

* Changing `handSize` allows you to adjust the probability of collisions between
  different flows and the overall concurrency available to a single flow in an
  overload situation, traded off against resource consumption. 
    {{< note >}}
    The probability that two flows with different flow distinguishers
    can starve each other of resources is `1 / choose(queues, handSize)`. The choice
    of `handSize` that minimizes this probability is therefore `Queues / 2`. However,
    much smaller values of `handSize` will in practice make such collisions very
    unlikely while consuming less resources. A larger `handSize` also increases the
    effective concurrency share available to a single flow in an overload situation.
    {{< /note >}}

### FlowSchema
A FlowSchema matches some inbound requests and assigns them to a
PriorityLevelConfiguration. Every inbound request is matched against every
FlowSchema in turn, starting with those with highest `MatchingPrecedence` and
working down. 

{{< caution >}}
Only the first matching FlowSchema for a given request matters.
If multiple FlowSchemas match a single inbound request, it will be assigned
based on the one with the highest `MatchingPrecedence`.
{{< /caution >}}

A FlowSchema matches a given request if at least one of its `rules`
matches. A rule matches if at least one of its `subjects` *and* at least
one of its `resourceRules` or `nonResourceRules` (depending on whether the
incoming request is for a resource or non-resource URL) matches. 

For the `name` field in subjects, and the `verbs`, `apiGroups`, `resources`,
`namespaces`, and `nonResourceURLs` fields of resource and non-resource rules,
the wildcard `*` may be specified to match all of the appropriate object,
effectively removing it from consideration.

A FlowSchema's `flowDistinguisherMethod` determines how requests matching that
schema will be separated into flows for the fair queuing algorithm. It may be
either `ByUser`, in which case one requesting user will not be able to starve
other users of capacity, or `ByNamespace`, in which case requests for resources
in one namespace will not be able to starve requests for resources in other
namespaces of capacity. The correct choice for a given FlowSchema depends on the
resource and your particular environment.

## Diagnostics
Every HTTP response from an API server with the priority and fairness feature
enabled will have two new headers, `X-Kubernetes-PF-FlowSchemaUID` and
`X-Kubernetes-PF-PriorityLevelUID`, noting the flow schema that matched the request
and the priority level to which it was assigned, respectively. The API objects'
names are not included in these headers in case the requesting user does not
have permission to view them, so when debugging you can use a command like 

```shell
kubectl get flowschemas -o custom-columns="uid:{metadata.uid},name:{metadata.name}"
kubectl get prioritylevelconfigurations -o custom-columns="uid:{metadata.uid},name:{metadata.name}"
```

to get a mapping of UIDs to names for both FlowSchemas and
PriorityLevelConfigurations.

## Observability
The API Priority and Fairness feature, when enabled, causes several new metrics
to be exported. Monitoring these can help you determine whether your
configuration is inappropriately throttling important traffic, or find
poorly-behaved workloads that may be harming system health.

* `apiserver_flowcontrol_rejected_requests` counts requests that were rejected,
  grouped by the name of the assigned priority level and the reason for
  rejection.
    * Reason may be `queue-full`, indicating that too many requests were already
      queued (this includes the case where the PriorityLevelConfiguration is
      configured to reject rather than queue excess requests), or `time-out`,
      indicating that the request was still in the queue when its deadline
      expired.

* `apiserver_flowcontrol_current_inqueue_requests` gives the instantaneous total
  number of queued (not executing) requests for each priority level.

* `apiserver_flowcontrol_current_executing_requests` gives the instantaneous
  total number of executing requests for each priority level.

* `apiserver_flowcontrol_request_queue_length` gives a histogram of queue
  lengths for the queues in each priority level.
    {{< note >}}
    An outlier value in a histogram here means it is likely that a single flow
    (i.e., requests by one user or for one namespace, depending on
    configuration) is flooding the API server, and being throttled. By contrast,
    if one priority level's histogram shows that all queues for that priority
    level are longer than those for other priority levels, it may be appropriate
    to increase that PriorityLevelConfiguration's concurrency shares.
    {{< /note >}}

* `apiserver_flowcontrol_request_concurrency_limit` gives the computed
  concurrency limit (based on the API server's total concurrency limit and PriorityLevelConfigurations'
  concurrency shares) for each PriorityLevelConfiguration.

* `apiserver_flowcontrol_request_wait_duration_seconds` gives a histogram of how
  long requests spent queued, grouped by the FlowSchema that matched the
  request, the PriorityLevel to which it was assigned, and whether or not the
  request successfully executed.
    {{< note >}}
    Since each FlowSchema always assigns requests to a single
    PriorityLevelConfiguration, you can add the histograms for all the
    FlowSchemas for one priority level to get the effective histogram for
    requests assigned to that priority level.
    {{< /note }}

* `apiserver_flowcontrol_request_execution_seconds` gives a histogram of how
  long requests took to actually execute, grouped by the FlowSchema that matched the
  request and the PriorityLevel to which it was assigned.
    

{{% /capture %}}

{{% capture whatsnext %}}

This feature is still in alpha. Its design and future plans can be found in the [relevant
KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/20190228-priority-and-fairness.md).
Feature requests and suggestions for improvements may be made to the [API
Machinery SIG](https://github.com/kubernetes/community/tree/master/sig-api-machinery).

Current plans include flow control for long-running requests (watches) and
improved observability features (particularly logging and tracing).

{{% /capture %}}
