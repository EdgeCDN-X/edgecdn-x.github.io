---
tags:
  - nginx
  - cache
---

# Ingress Nginx
Ingress Nginx is used as it is and customized with configuration options. Deploy the NGINX with a helm chart. The upstream helm chart just works perfect. Ingress Nginx is extensible with custom snippets. If you wan't to use a cluster with mixed caches (e.g. SSD, HDD, TMPFS etc) you will have to deploy multiple instances of the ingres-nginx helm chart. In this example we will assume we only have a single cache type called `ssd`.

**Configuration**:

* deploy the ingress as a Daemonset
* mount the cache storage as a hostPath
* specify your `ingressClass`
* set the `nodeSelector`
* specify `cache` configuration with an extra `http-snippet`


Use the following manifest to deploy the Ingress controller:

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: edgecdnx-cache
  namespace: argocd
spec:
  goTemplate: true
  syncPolicy:
    preserveResourcesOnDeletion: false
  generators:
    - clusters:
        values:
          chart: ingress-nginx
          chartVersion: 4.12.1
          chartRepository: https://kubernetes.github.io/ingress-nginx
          namespace: edgecdnx-cache
          cache: ssd
          path: /var/cache/ssd
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
      name: edgecdnx-cache-{{ .values.cache }}-{{ index .metadata.labels "edgecdnx.com/location" }}-{{ index .metadata.labels "edgecdnx.com/tier" }}
    spec:
      project: default
      sources:
        - chart: "{{ .values.chart }}"
          repoURL: "{{ .values.chartRepository }}"
          targetRevision: "{{ .values.chartVersion }}"
          helm:
            releaseName: ingress-nginx-{{ .values.cache }}
            ignoreMissingValueFiles: true
            valuesObject:
              controller:
                metrics:
                  enabled: false
                  serviceMonitor:
                    enabled: false
                    additionalLabels:
                      release: prometheus-stack
                allowSnippetAnnotations: true
                ingressClass: "{{ .values.cache }}"
                ingressClassResource:
                  enabled: true
                  default: false
                  name: "{{ .values.cache }}"
                  controllerValue: "edgecdnx.com/cache-{{ .values.cache }}"
                service:
                  enabled: false
                hostPort:
                  enabled: true
                kind: DaemonSet
                nodeSelector:
                  edgecdnx.com/cache: "{{ .values.cache }}"
                extraVolumes:
                  - name: nginx-cache
                    hostPath:
                      path: "{{ .values.path }}"
                      type: Directory
                extraVolumeMounts:
                  - name: nginx-cache
                    mountPath: "{{ .values.path }}"
                config:
                  proxy-buffering: "on"
                  annotations-risk-level: "Critical"
                  http-snippet: |
                    proxy_cache_path {{ .values.path }} levels=1:2 keys_zone={{ .values.cache }}:100m inactive=10080m use_temp_path=off;
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