# CoreDNS Deployment
We will use an ArogCD Cluster generator to deploy the CoreDNS components on each region marked with **routing** tag.

## Required variables
For each region define the following variables:

* edgecdnx.com/namepsace: Defines the working namespace for reading CRDs
* edgecdnx.com/public-ip: Defines the Public IP of the DNS endpoint
* edgecdnx.com/ns: Identifies the NS id. e.g. "1", turns to ns1, "2" to ns2
* edgecdnx.com/basedomain: Basedomain to serve. e.g. "cdn.edgecdnx.com." - note the dot at the end
* edgecdnx.com/domainemail: Email address listed in SOA. e.g. "noc.edgecdnx.com"

Example:
```yaml
kind: Secret
metadata:
  annotations:
    edgecdnx.com/basedomain: cdn.edgecdnx.com.
    edgecdnx.com/domainemail: noc.edgecdnx.com
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
                chartVersion: 1.43.0
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
      name: edgecdnx-coredns-{{ index .metadata.labels "edgecdnx.com/location" }}
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
                tag: latest
                pullPolicy: Always
              serviceType: LoadBalancer
              service:
                externalTrafficPolicy: Local
              isClusterService: false
              replicaCount: 1
              servers:
                - port: 53
                  plugins:
                    - name: ready
                    - name: debug
                    - name: metadata
                    - name: health
                      configBlock: |-
                        lameduck 5s
                    - name: edgecdnxprefixlist
                      parameters: '{{ index .metadata.annotations "edgecdnx.com/namespace" }}'
                    - name: geoip
                      parameters: /etc/edgecdnx/geolookup/GeoLite2-City.mmdb
                      configBlock: |
                        edns-subnet
                    - name: edgecdnxgeolookup
                      configBlock: |
                        namespace {{ index .metadata.annotations "edgecdnx.com/namespace" }}
                        consulEndpoint http://edgecdnx-consul-consul-server:8500
                        consulcachettl 5s
                        recordttl 30
                    - name: edgecdnxservices
                      configBlock: |
                        namespace {{ index .metadata.annotations "edgecdnx.com/namespace" }}
                        soa {{- range .clusters -}}{{ if eq .name $.name }} ns{{ index .metadata.annotations "edgecdnx.com/ns" }}{{ end -}}{{- end }}
                        email {{ index .metadata.annotations "edgecdnx.com/domainemail" }}
                        {{- range .clusters }}
                        ns ns{{ index .metadata.annotations "edgecdnx.com/ns" }} {{ index .metadata.annotations "edgecdnx.com/public-ip" }}
                        {{- end }}
                  zones:  
                    - zone: '{{ index .metadata.annotations "edgecdnx.com/basedomain" }}'
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
          targetRevision: 0.1.1
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
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true # Big CRDs.
      ignoreDifferences: []

```
