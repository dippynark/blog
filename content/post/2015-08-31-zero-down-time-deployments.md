---
title: Zero down-time deployments with externalTrafficPolicy Local
date: 2025-08-31
tags: ["kubernetes"]
---

[externalTrafficPolicy
Local](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip)
is a configuration option for Kubernetes Services that preserves the client source IP and reduces
latency by only forwarding external packets to Pods on the local node. This post describes how a
workload running behind a Service of type LoadBalancer can be configured to reduce the chance of
requests being dropped when performing a rolling update.

As a specific example we are going to be using Nginx Ingress Controller running behind a GKE Service
of type LoadBalancer with [GKE
subsetting](https://cloud.google.com/kubernetes-engine/docs/how-to/internal-load-balancing#gke-subsetting)
enabled, but the concepts should extend to other Kubernetes environments.

# PreStop Lifecycle Hook

kube-proxy serves a health check which only succeeds when there is at least one running and ready
Pod selected by the Service on the local node. This health check is probed by the load balancer to
ensure that packets are only forwarded to nodes that are ready to receive them.

However, when the last Pod selected by the Service on the local node is terminated, there is a race
condition between when the Pod is stopped and when the load balancer stops forwarding packets.

To ensure that the Pod continues to serve until the load balancer stops sending packets, a
[PreStop](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#container-hooks)
lifecycle hook can be configured to wait until the load balancer health check has failed. In our
case this is [6
seconds](https://github.com/kubernetes/ingress-gce/blob/203252bfcbe898dac338acd0790751b772097cd3/pkg/healthchecksl4/healthchecksl4.go#L50)
plus a small amount of additional time for kube-proxy to observe the terminating Pod and start
failing the health endpoint:

```yaml
lifecycle:
  preStop:
    sleep:
      seconds: 10
```

TODO: Speak about /wait-shutdown binary

# Readiness Probe

Once the load balancer health check has failed new connections will no longer be sent to the node,
however existing connections will still be drained. In our case connection draining is configured to
be [30
seconds](https://github.com/kubernetes/ingress-gce/blob/203252bfcbe898dac338acd0790751b772097cd3/pkg/backends/backends.go#L37),
so we want to continue serving until these connections have been terminated.

One option is to simply extend our PreStop lifecycle hook for 30 seconds so that the load balancer
resets the connections before the Pod is stopped, however this does not give Nginx a chance to drain
HTTP requests gracefully (e.g. by sending GOAWAY frames on HTTP/2 connections).

Instead, by ensuring the Nginx Ingress Controller Pods remain ready after receiving a TERM signal
from the kubelet, we can take advantage of [KEP-1669: Proxy Terminating
Endpoints](https://github.com/kubernetes/enhancements/tree/787f515ac4ddb93d0d1c381a17bcd330b8caf9b0/keps/sig-network/1669-proxy-terminating-endpoints)
to ensure that existing connections continue to be forwarded until they are closed by Nginx.

Note that currently the Nginx Ingress Controller readiness probe [starts failing as soon as the TERM
signal is received](https://github.com/kubernetes/ingress-nginx/issues/13689) so we need to make
sure that the terminationGracePeriod is configure such that the Pod is killed if Nginx is still
draining connections after the Pod as been marked not ready. In other words we need:

```sh
lifecycleHookPreStopSleepSeconds + (readinessProbePeriodSeconds * (readinessProbeFailureThreshold - 1)) >= terminationGracePeriodSeconds
```

# Readiness Gate

So far we have discussed how to ensure that connections are drained when terminating a Pod, but we
also need to ensure that a new Pod is successfully registered with the load balancer on start up.
Without this, if the load balancer is controller is down, we could end up terminating all the old
Pods before the new ones have been registered with the load balancer.

[Pod readiness
gates](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-readiness-gate) can
help here by checking that the Pod is registered with the load balancer before marking the Pod as
ready. When using GKE container-native load balancing, you can use the
[cloud.google.com/load-balancer-neg-ready](https://cloud.google.com/kubernetes-engine/docs/concepts/container-native-load-balancing#pod_readiness)
readiness gate out of the box which works because the load balancer health check is made directly to
the Pod and so is decoupled from the kube-proxy health check. However, when using a GKE Service of
type LoadBalancer, no such readiness gate exists because there is a cyclic dependency on the
readiness of the node; the readiness gate would not pass until the load balancer has marked the
local node as ready which would not happen until the readiness gate passes.

Since the GKE Service of type LoadBalancer implementation considers [both ready and not ready Pods
](https://github.com/kubernetes/ingress-gce/blob/203252bfcbe898dac338acd0790751b772097cd3/pkg/neg/types/types.go#L337)
we could implement a readiness gate that waits for the local node to be added into the load balancer
but just does not wait for it to be healthy, taking advantage of the fact that internal passthrough
Network Load Balancers [distribute new connections to all backend
VMs](https://cloud.google.com/load-balancing/docs/internal/int-netlb-traffic-distribution#failover)
if they are all unhealthy. However, even in this case there is a race condition between kube-proxy
configuring routes to the new Pods.

As a simpler work around, the readiness probe can be delayed using `initialDelaySeconds` for an
amount of time that gives a high chance that the local node has been registered before marking the
Pod as ready (e.g. 20 seconds, slightly longer than the [lease
duration](https://github.com/kubernetes/ingress-gce/blob/203252bfcbe898dac338acd0790751b772097cd3/pkg/flags/flags.go#L43-L44))
but this does not provide any guarantees.

Note that GKE recommends using
[`minReadySeconds`](https://cloud.google.com/kubernetes-engine/docs/how-to/container-native-load-balancing#align_rollouts)
which works great when performing a rolling update but is not considered in other circumstances that
cause Pods to be terminated and recreated (e.g. evictions caused by nodes being drained and vertical
Pod autoscaling).

# Conclusion

As we can see, the mechanisms for ensuring zero down-time when running behind a Service of type
LoadBalancer with externalTrafficPolicy Local cannot provide guarantees in a number of edge cases,
however we can significantly reduce the chance of experiencing these edge cases by using more
replicas and reducing eviction rates using a PodDisruptionBudget and surge upgrades.

In the future there may be additional options to mitigate these issues, for example [Pod Termination
Gates](https://github.com/kubernetes/kubernetes/issues/106476#issuecomment-2749150204).
