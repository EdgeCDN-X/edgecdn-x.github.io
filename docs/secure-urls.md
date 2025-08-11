---
tags:
  - url-signatures
  - secure-urls
---
# Secure URLs

The [Secure URLs](https://github.com/EdgeCDN-X/secure-urls) repository is responsible for validating URL signatures to serve private content via the CDN. It is also possible to serve Private content.

## Introduction
This application is responsible for validating URLsignatures and Secure Cookies. Sometimes we want to hide the content distributed via the CDN, this is possible with distributing secure URLs to the clients.

# Recommendations
it is recommended that the user distributes the Secure URLs via HTTPS Protocol. It is possible to maintain multiple keys, but it is recommended to rotate them periodically and revoke old keys onec they're not used.

# Deployment
Secure URLs is deployed alongside the Routing Nodes. Use the following Applicationset do deploy the Secure URLs component:

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: secure-urls
  namespace: argocd
spec:
  goTemplate: true
  syncPolicy:
    preserveResourcesOnDeletion: false
  generators:
    - clusters:
        values:
          chart: secure-urls
          chartVersion: 0.1.1
          chartRepository: https://edgecdn-x.github.io/helm-charts/
          namespace: edgecdnx-cache
        selector:
          matchExpressions:
            - key: edgecdnx.com/caching
              operator: In
              values:
                - "true"
                - "yes"
            - key: edgecdnx.com/tier
              operator: In
              values:
                - edge
  template:
    metadata:
      name: secure-urls-{{ index .metadata.labels "edgecdnx.com/location" }}
    spec:
      project: default
      sources:
        - chart: "{{ .values.chart }}"
          repoURL: "{{ .values.chartRepository }}"
          targetRevision: "{{ .values.chartVersion }}"
          helm:
            releaseName: secure-urls
            ignoreMissingValueFiles: true
            valuesObject:
              command:
                - /secure-urls
                - --log-level
                - debug
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

This will deploy the helm chart `secure-urls` from the repository `https://edgecdn-x.github.io/helm-charts` to each cluster marked with **edgecdnx.com/caching** label set to **true/yes**