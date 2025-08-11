---
tags:
  - edgecdnx-controller
  - caching
---
# EdgeCDN-X Controller - Cache
The controller is the same one as [this](edgecdnx-controller.md) but running in a `cache-controller` role. In such case the controller is responsible for managing the objects related to Caching.

## Responsibilities
In `cache-controller` mode the reconcile loop serves the following tasks:

* Manages origin servers
* Manages S3 Origins
* Manages ingress configuration

The main caching logic is happening here. Detailed use cases is visible in the User Guide section.

## CRDs
There are 3 CRDs in the system:

* Location
* Service
* PrefixList

Locations define the infrastructure. Services define the services served across the plaftom, and PrefixLists can alter the routing in the ecosystem. For details click [here](crds.md)

# Deployment
Deploy the EdgeCDN-X Controller in `cache-controller` mode on clusters with label **edgecdnx.com/caching** set to **true/yes**. For this purpose you can use the following Applicationset on the [control](argocd.md) plane.

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: edgecdnx-controller-caching
  namespace: argocd
spec:
  goTemplate: true
  syncPolicy:
    preserveResourcesOnDeletion: false
  generators:
    - clusters:
        values:
          chart: edgecdnx-controller
          chartVersion: 0.5.4
          chartRepository: https://edgecdn-x.github.io/helm-charts/
          namespace: edgecdnx-controller-system
        selector:
          matchExpressions:
            - key: edgecdnx.com/caching
              operator: In
              values:
                - "true"
                - "yes"
  template:
    metadata:
      name: edgecdnx-controller-caching-{{ index .metadata.labels "edgecdnx.com/location" }}
    spec:
      project: default
      sources:
        - chart: "{{ .values.chart }}"
          repoURL: "{{ .values.chartRepository }}"
          targetRevision: "{{ .values.chartVersion }}"
          helm:
            releaseName: edgecdnx-controller-caching
            ignoreMissingValueFiles: true
            valuesObject:
              role: cache-controller
      destination:
        namespace: "{{ .values.namespace }}"
        server: "{{ .server }}"
      syncPolicy:
        automated:
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true # Big controller.
      ignoreDifferences: []

```

[CRDs](crds.md) are deployed alongside this helm deployment.
In the values you have to specify the **role**.