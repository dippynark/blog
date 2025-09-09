---
title: Extending cert-manager with issuer-lib
date: 2025-09-09
tags: ["kubernetes"]
---

[cert-manager](https://github.com/cert-manager/cert-manager) is the standard tool in the Kubernetes
ecosystem for automatically provisioning and managing TLS certificates. cert-manager has a pluggable
architecture allowing users to write their own [external
issuers](https://cert-manager.io/docs/contributing/external-issuers/) (controllers that reconcile
[CertificateRequests](https://cert-manager.io/docs/usage/certificaterequest/)) while integrating
with the rest of the cert-manager certificate management lifecycle (e.g. renewals):

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example
  namespace: example
spec:
  # Provision certificate from my external issuer
  issuerRef:
    group: example.com
    kind: ExampleIssuer
    name: example
  secretName: example-tls
  dnsNames:
    - example.com
```

This capability is incredibly powerful in an enterprise environment where many complicated (and
often perplexing) requirements may exist that cannot be handled by cert-manager's built-in issuers.

# issuer-lib

[issuer-lib](https://github.com/cert-manager/issuer-lib) is a Go library for building cert-manager
issuers, built and maintained by the cert-manager maintainers.

By referencing the [simple
example](https://github.com/cert-manager/issuer-lib/tree/main/examples/simple) provided in the
library repository and implementing the
[Check](https://github.com/cert-manager/issuer-lib/blob/d99ad6fb1498659b04babc6635bb6f65b74a6455/examples/simple/controller/signer.go#L66)
and
[Sign](https://github.com/cert-manager/issuer-lib/blob/d99ad6fb1498659b04babc6635bb6f65b74a6455/examples/simple/controller/signer.go#L70)
functions, users can extend the functionality of cert-manager with their own certificate
issuing logic.

# Example

One application of issuer-lib that I have found particularly powerful is using it to watch
CertificateRequests in workload clusters and then fulfilling these CertificateRequests using a
built-in cert-manager issuer running in a management cluster (perhaps after performing
CertificateRequest validation).

This allows highly privileged credentials for the built-in issuer to be stored in the management
cluster without exposing them directly to the workload clusters, by effectively proxying
CertificateRequest resources from workload clusters to the management cluster.

In the following diagram, the workload clusters cannot authenticate directly to the enterprise TLS
certificate management solution. Instead, any CertificateRequests created in the workload clusters
referencing the external issuer will be picked up and fulfilled by the management cluster. Note that
cert-manager still needs to run in the workload clusters to manage the certificate lifecycle:

![issuer-lib](/img/issuer-lib.png)
