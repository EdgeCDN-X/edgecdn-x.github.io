---
tags:
  - coredns
  - routing
---
# CoreDNS Deployment
We will use an ArogCD Cluster generator to deploy the CoreDNS components on each region marked with **edgecdnx.com/routing** tag.

## Required variables
For each region define the following variables:

* edgecdnx.com/namespace: Defines the working namespace for reading CRDs
* edgecdnx.com/public-ip: Defines the Public IP of the DNS endpoint
* edgecdnx.com/ns: Identifies the NS id. e.g. "1", turns to ns1, "2" to ns2

Example:
```yaml
kind: Secret
metadata:
  annotations:
    edgecdnx.com/namespace: edgecdnx
    edgecdnx.com/ns: "1"
    edgecdnx.com/public-ip: 188.167.203.182
  labels:
    argocd.argoproj.io/secret-type: cluster
    edgecdnx.com/location: us-east-1
    edgecdnx.com/routing: "true"
  name: cluster-us-east-1.k8s.edgecdnx.com
```

## Applicationset Manifest

### Prerequisites

 * CRDs Installed

### Components

* CoreDNS
* Module Configuration
* Geolookup MMDB-Lite DB
* CoreDNS RBAC - to be able to read CRDs.

### CoreDNS Required modules

* Metadata
* GeoIP
* Ready
* EdgeCDN-X Specific Modules:
    * edgecdnxprefixlist
    * edgecdnxgeolookup
    * edgecdnxservices

Use the following applicationset on the control plane to roll out CoreDNS to each region.

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: edgecdnx-routing
  namespace: argocd
spec:
  goTemplate: true
  syncPolicy:
    preserveResourcesOnDeletion: false
  generators:
    - matrix:
        generators:
          - clusters:
              flatList: true
              selector:
                matchExpressions:
                  - key: edgecdnx.com/routing
                    operator: In
                    values:
                      - "true"
                      - "yes"
          - clusters:
              values:
                chart: coredns
                chartVersion: 1.43.3
                chartRepository: https://coredns.github.io/helm
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
      name: edgecdnx-coredns-{{ .name }}
    spec:
      project: default
      sources:
        - chart: "{{ .values.chart }}"
          repoURL: "{{ .values.chartRepository }}"
          targetRevision: "{{ .values.chartVersion }}"
          helm:
            releaseName: edgecdnx-coredns
            ignoreMissingValueFiles: true
            valuesObject:
              image:
                repository: fr6nco/coredns
                tag: 1.12.1-ex-8
                pullPolicy: Always
              serviceType: LoadBalancer
              service:
                externalTrafficPolicy: Local
                annotations:
                  kubernetes.civo.com/loadbalancer-enable-proxy-protocol: send-proxy-v2
              isClusterService: false
              replicaCount: 2
              servers:
                - port: 53
                  nodePort: 30053
                  plugins:
                    - name: ready
                    - name: debug
                    - name: metadata
                    - name: log
                      parameters: . "{combined}"
                    - name: errors
                      configBlock: |-
                        stacktrace
                    - name: health
                      configBlock: |-
                        lameduck 5s
                    - name: edgecdnxprefixlist
                      parameters: '{{ index .metadata.annotations "edgecdnx.com/namespace" }}'
                    - name: geoip
                      parameters: /etc/edgecdnx/geolookup/GeoLite2-City.mmdb
                      configBlock: |-
                        edns-subnet
                    - name: edgecdnxgeolookup
                      configBlock: |-
                        namespace {{ index .metadata.annotations "edgecdnx.com/namespace" }}
                        recordttl 30
                    - name: edgecdnxservices
                      configBlock: |-
                        namespace {{ index .metadata.annotations "edgecdnx.com/namespace" }}
                        soa {{- range .clusters -}}{{ if eq .name $.name }} ns{{ index .metadata.annotations "edgecdnx.com/ns" }}{{ end -}}{{- end }}
                        {{- range .clusters }}
                        ns ns{{ index .metadata.annotations "edgecdnx.com/ns" }} {{ index .metadata.annotations "edgecdnx.com/public-ip" }}
                        {{- end }}
                  zones:
                    - zone: '.'
              initContainers:
                - name: edgecdnx-mmdb-init
                  image: curlimages/curl:8.14.1
                  volumeMounts:
                    - name: geolookup-mmdb
                      mountPath: /etc/edgecdnx/geolookup
                  command:
                    - sh
                    - -c
                    - |
                      curl -L https://share.tbotech.sk/api/shares/7aAqdIUO/files/4e046472-f00d-4275-be7b-b5228ff200ce -o /etc/edgecdnx/geolookup/GeoLite2-City.mmdb
              extraVolumes:
                - name: geolookup-mmdb
                  emptyDir: {}
              extraVolumeMounts:
                - name: geolookup-mmdb
                  mountPath: /etc/edgecdnx/geolookup
        - chart: coredns-rbac
          repoURL: https://edgecdn-x.github.io/helm-charts
          targetRevision: 0.1.4
          helm:
            releaseName: edgecdnx-coredns-rbac
            ignoreMissingValueFiles: true
            valuesObject:
              serviceAccount: default
      destination:
        namespace: "{{ .values.namespace }}"
        server: "{{ .server }}"
      syncPolicy:
        automated:
          selfHeal: false
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true # Big CRDs.
      ignoreDifferences: []
```
