---
title: Zero downtime deployments with externalTrafficPolicy Local
date: 2025-08-31
tags: ["kubernetes"]
---

[externalTrafficPolicy
Local](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip)
is a configuration option for Kubernetes Services that configures nodes to only forward external
traffic to local Service endpoints, reducing latency and preserving the client source IP. [The ins
and outs of networking in GKE](https://www.youtube.com/watch?v=y2bhV81MfKQ) is my favourite video
for understanding this configuration option.

This post describes how a workload exposed using a Service of type LoadBalancer with
externalTrafficPolicy Local can be configured to avoid requests being dropped when performing a
rolling update.

As a specific example we are going to use Nginx Ingress Controller running behind a GKE Service of
type LoadBalancer with [GKE
subsetting](https://cloud.google.com/kubernetes-engine/docs/how-to/internal-load-balancing#gke-subsetting)
but the concepts should apply to other Kubernetes environments; when GKE subsetting is enabled
Service of type LoadBalancer are implemented using
[ingress-gce](https://github.com/kubernetes/ingress-gce) instead of
[cloud-provider-gcp](https://github.com/kubernetes/cloud-provider-gcp).

# PreStop Lifecycle Hook

When using externalTrafficPolicy Local, kube-proxy serves a health endpoint which only passes when
there is at least one running and ready Pod selected by the Service on the local node. This health
endpoint is probed by the load balancer to ensure that packets are only forwarded to nodes that are
running Pods to receive them.

However, when the last Pod selected by the Service on the local node is terminated, there is a race
condition between when the Pod is stopped and when the load balancer stops forwarding packets.

To ensure that the Pod continues to serve until the load balancer stops sending packets, a
[PreStop](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#container-hooks)
lifecycle hook can be configured to delay the kubelet sending a TERM signal to the Nginx Ingress
Controller container process until the load balancer health check has failed. In our case this is [6
seconds](https://github.com/kubernetes/ingress-gce/blob/203252bfcbe898dac338acd0790751b772097cd3/pkg/healthchecksl4/healthchecksl4.go#L50)
and we add a small amount of additional time to allow kube-proxy to observe the terminating Pod and
start failing the health endpoint:

```yaml
containers:
  - name: controller
    lifecycle:
      preStop:
        sleep:
          seconds: 10
```

Note that typically we expect the load balancer to remove nodes from load balancing more quickly
than the health check because the load balancer controller [watches
EndpointSlices](https://github.com/kubernetes/ingress-gce/blob/203252bfcbe898dac338acd0790751b772097cd3/pkg/neg/syncers/endpoints_calculator.go#L36-L51).

Note also that Nginx Ingress Controller ships with a
[`wait-shutdown`](https://github.com/kubernetes/ingress-nginx/blob/106e633655e7e5799ccf28d747b07d78833cd860/deploy/static/provider/baremetal/deploy.yaml#L446-L450)
binary that is meant to be used instead of a sleep lifecycle hook together with the
`--shutdown-grace-period` flag, however using this binary causes the readiness probe to start
failing straight away which can cause traffic to be blackholed; see GitHub issue
[#13689](https://github.com/kubernetes/ingress-nginx/issues/13689) for details.

# Readiness Probe

Once the load balancer health check has failed new connections will no longer be sent to the node,
however packets corresponding to existing connections will continue to be routed. In our case load
balancer connection draining is configured to be [30
seconds](https://github.com/kubernetes/ingress-gce/blob/203252bfcbe898dac338acd0790751b772097cd3/pkg/backends/backends.go#L37),
so we want to continue serving until these connections have been closed.

One option is to simply extend our PreStop lifecycle hook for 30 seconds so that the load balancer
resets the connections before the Nginx process is shut down, however this does not give Nginx a
chance to drain HTTP requests gracefully (e.g. by sending GOAWAY frames on HTTP/2 connections).

Instead, by ensuring that Nginx Ingress Controller Pods remain ready after receiving a TERM signal
from the kubelet, we can take advantage of [KEP-1669: Proxy Terminating
Endpoints](https://github.com/kubernetes/enhancements/tree/787f515ac4ddb93d0d1c381a17bcd330b8caf9b0/keps/sig-network/1669-proxy-terminating-endpoints)
to ensure that existing connections continue to be forwarded until they are closed by Nginx.

As described in [#13689](https://github.com/kubernetes/ingress-nginx/issues/13689), the Nginx
Ingress Controller readiness probe currently starts failing as soon as the TERM signal is received
so we need to make sure that `terminationGracePeriodSeconds` is set such that the Pod is killed
before the Pod can been marked as not ready to avoid packets being blackholed. In other words we
need:

```text
lifecycleHookPreStopSleepSeconds + (readinessProbePeriodSeconds * (readinessProbeFailureThreshold - 1)) >= terminationGracePeriodSeconds
```

The following configuration satisfies this requirement:

```yaml
containers:
  - name: controller
    lifecycle:
      preStop:
        sleep:
          seconds: 10
    readinessProbe:
      failureThreshold: 3
      httpGet:
        path: /healthz
        port: 10254
        scheme: HTTP
      initialDelaySeconds: 10
      periodSeconds: 10
      successThreshold: 1
      timeoutSeconds: 1
terminationGracePeriodSeconds: 30
```

# Readiness Gate

So far we have discussed how to ensure that connections are closed gracefully when terminating a
Pod, but we also need to ensure that new Pods are successfully registered with the load balancer on
start up before marking the Pod as ready. Without this, capacity could be reduced or the load
balancer could even temporarily have no nodes to forward packets to!

[Pod readiness
gates](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-readiness-gate) can
help here by checking that the Pod is registered with the load balancer before marking the Pod as
ready. When using GKE container-native load balancing, we can use the
[cloud.google.com/load-balancer-neg-ready](https://cloud.google.com/kubernetes-engine/docs/concepts/container-native-load-balancing#pod_readiness)
readiness gate out of the box. This works because the load balancer health check is made directly to
the Pod and so is decoupled from the kube-proxy health check. However, when using a GKE Service of
type LoadBalancer, there is a cyclic dependency on the readiness of the node; the readiness gate
would not pass until the load balancer has marked the local node as ready which would not happen
until the readiness gate passes.

Since the GKE Service of type LoadBalancer implementation considers [both ready and not ready Pods
](https://github.com/kubernetes/ingress-gce/blob/203252bfcbe898dac338acd0790751b772097cd3/pkg/neg/types/types.go#L337)
we could implement a readiness gate that waits for the local node to be added into the load balancer
but just does not wait for it to be healthy, taking advantage of the fact that GCP internal
passthrough Network Load Balancers [distribute new connections to all backend
VMs](https://cloud.google.com/load-balancing/docs/internal/int-netlb-traffic-distribution#failover)
if they are all unhealthy. This does not provide the same guarantees as the GKE container-native
load balancing readiness gate, but it does protect against the load balancer controller being down
(e.g. during a upgrade of the GKE control plane).

As a simpler work around, the readiness probe can be delayed using `initialDelaySeconds` for an
amount of time that gives a high chance that the local node has been registered before marking the
Pod as ready (e.g. 20 seconds, slightly longer than the [Lease
duration](https://github.com/kubernetes/ingress-gce/blob/203252bfcbe898dac338acd0790751b772097cd3/pkg/flags/flags.go#L43-L44)).

Note that GKE recommends using
[`minReadySeconds`](https://cloud.google.com/kubernetes-engine/docs/how-to/container-native-load-balancing#align_rollouts)
which works great when performing a rolling update but is not considered in other circumstances that
cause Pods to be terminated and recreated (e.g. evictions caused by nodes being drained and vertical
Pod autoscaling).

# Conclusion

As described above, there are a number of configuration options to improve the chance of zero
downtime deployments when running behind a Service of type LoadBalancer with externalTrafficPolicy
Local, but they do not guarantee this even when all cluster components are working correctly.

However, the remaining race conditions become increasingly rare as we increase the number of
replicas and reducing Pod unavailability by using a PodDisruptionBudget or [surge
updates](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment).

In the future there may be additional options to mitigate these issues, for example [Pod termination
gates](https://github.com/kubernetes/kubernetes/issues/106476#issuecomment-2749150204).
