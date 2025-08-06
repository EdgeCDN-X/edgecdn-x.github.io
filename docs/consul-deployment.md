---
tags:
  - consul
  - healthchecks
  - routing
  - coredns
---
# Consul
[Consul](https://developer.hashicorp.com/consul) is used for healchchecks in the system. This is likely just a temporary solution and Consul will eventully be replaced to something else. So far it helps us healthchecking the Nginx endpoints.

## Consul ESM
Consul itself can't make healthchecks without an Agent, for this [Consul-esm](https://github.com/hashicorp/consul-esm) is in conjuction with consul.


# Deployment
Consul needs to be deployed on the clusters where CoreDNS is running. It requires a PVC to persist data. This document will not cover how to set up a PVC, since it differs from cloud provider to cloud provider. Install `consul` and `consul-esm` using the following `Applicationset` manifest:


```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: edgecdnx-consul
  namespace: argocd
spec:
  goTemplate: true
  syncPolicy:
    preserveResourcesOnDeletion: false
  generators:
    - clusters:
        values:
          chart: consul
          chartVersion: 1.7.2
          chartRepository: https://helm.releases.hashicorp.com
          namespace: edgecdnx-routing
        selector:
          matchExpressions:
            - key: edgecdnx.com/routing
              operator: In
              values:
                - "true"
                - "yes"
  template:
    metadata:
      name: edgecdnx-consul-{{ index .metadata.labels "edgecdnx.com/location" }}
    spec:
      project: default
      sources:
        - chart: "{{ .values.chart }}"
          repoURL: "{{ .values.chartRepository }}"
          targetRevision: "{{ .values.chartVersion }}"
          helm:
            releaseName: edgecdnx-consul
            ignoreMissingValueFiles: true
            valuesObject:
              global:
                tls:
                  enabled: false
                federation:
                  enabled: false
                acls:
                  manageSystemACLs: false
              connectInject:
                enabled: false
              server:
                storageClass: local-path
                replicas: 1
                tolerations: |
                  - key: "node-role.kubernetes.io/master"
                    operator: "Exists"
                    effect: "NoSchedule"
                  - key: "node-role.kubernetes.io/control-plane"
                    operator: "Exists"
                    effect: "NoSchedule"
              client:
                enabled: false
              dns:
                enabled: true
              syncCatalog:
                enabled: false
        - chart: consul-esm
          repoURL: https://edgecdn-x.github.io/helm-charts
          targetRevision: 0.1.1
          helm:
            releaseName: edgecdnx-consul-esm
            ignoreMissingValueFiles: true
            valuesObject:
              config:
                consulServer: "http://edgecdnx-consul-consul-server:8500"
      destination:
        namespace: "{{ .values.namespace }}"
        server: "{{ .server }}"
      syncPolicy:
        automated:
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true # Big CRDs.
      ignoreDifferences: []
```

This manifest will deploy `consul` and `consul-esm` on clusters, where **edgedcnx.com/routing** is present in the labels. It is deployed in namespace **edgecdnx-routing** using the storageClass local-path. Feel free to adjust the settings based on your needs.

-------
Once deployed configure your CoreDNS instance to use Consul as a [source of truth](coredns.md#geolookup-routing) for healthchecks. E.g.:

```
        edgecdnxgeolookup {
            namespace edgecdnx-routing
            consulEndpoint http://edgecdnx-consul-consul-server:8500
            consulcachettl 5s
            recordttl 30
        }
```
