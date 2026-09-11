---
tags:
  - coredns
  - routing
  - deployment
  - argocd
---
# CoreDNS Deployment

EdgeCDN-X uses ArgoCD to deploy the DNS controller (CoreDNS with the `edgecdnx` plugin) to each edge location. Cluster selection is based on the **`edgecdnx.com/routing`** label in the cluster metadata.

## Required Cluster Annotations

For each routing location, define the following annotations on the cluster Secret in ArgoCD:

| Annotation | Description | Example |
| --- | --- | --- |
| `edgecdnx.com/namespace` | Kubernetes namespace containing EdgeCDN-X CRDs (Location, DNSEndpoint, PrefixList, Zone) | `edgecdnx` |
| `edgecdnx.com/public-ip` | Public IP address for this DNS endpoint (used in NS records) | `188.167.203.182` |
| `edgecdnx.com/ns` | NS record identifier suffix; `"1"` → `ns1`, `"2"` → `ns2` | `"1"` |

**Example Cluster Secret**:
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
    edgecdnx.com/routing: "true"  # Cluster selector key
  name: cluster-us-east-1.k8s.edgecdnx.com
```

## Deployment Prerequisites

* EdgeCDN-X CRDs installed in the cluster
* Kubernetes RBAC configured to allow CoreDNS to read Location, DNSEndpoint, PrefixList, and Zone resources
* GeoIP database (MaxMind GeoLite2-City) accessible during pod initialization

## Deployment Components

### CoreDNS Required Modules

* `ready` - CoreDNS readiness probe integration for liveness checks
* `debug` - Debug logging for CoreDNS operations
* `metadata` - Metadata propagation for plugin chaining
* `log` - Request/response logging
* `errors` - Error logging with stack traces
* `health` - Health check endpoint with lameduck period
* `geoip` - MaxMind GeoIP database lookup with EDNS client subnet support
* `edgecdnx` - EdgeCDN-X unified DNS controller plugin (replaces earlier separate modules):
  - Prefix-based routing via `PrefixList` CRDs
  - Geolocation-based routing via `Location` CRDs
  - DNSEndpoint and Zone CRD support
  - Service discovery and health-aware node selection

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
                tag: 1.14.2-edgecdnx-v1.5.0
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
                    - name: geoip
                      parameters: /etc/edgecdnx/geolookup/GeoLite2-City.mmdb
                      configBlock: |-
                        edns-subnet
                    - name: edgecdnx
                      configBlock: |-
                        namespace {{ index .metadata.annotations "edgecdnx.com/namespace" }}
                        recordttl 30
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
          targetRevision: 0.1.5
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

## Deployment Configuration Details

### Image and Version
- **Repository**: `fr6nco/coredns`
- **Tag**: `1.14.2-edgecdnx-v1.5.0` (CoreDNS 1.14.2 with EdgeCDN-X plugin v1.5.0)
- **PullPolicy**: Always (ensures latest image on every deployment)

### Service Configuration
- **Type**: `LoadBalancer` — exposes DNS on a public network load balancer
- **External Traffic Policy**: `Local` — maintains source IP and reduces load balancer overhead
- **Proxy Protocol**: Enabled via `kubernetes.civo.com/loadbalancer-enable-proxy-protocol: send-proxy-v2` annotation (Civo-specific)
- **DNS Port**: 53 (standard DNS port)
- **Node Port**: 30053 (for direct node access if needed)

### Plugin Chain
The CoreDNS plugin chain executes in order:

1. **ready** — CoreDNS readiness check
2. **debug** — Debug logging
3. **metadata** — Context propagation for chained plugins
4. **log** — Log all DNS queries and responses in combined format
5. **errors** — Log errors with stack traces
6. **health** — Health check endpoint (lameduck period: 5s)
7. **geoip** — MaxMind GeoIP database lookup with EDNS client subnet support
8. **edgecdnx** — EdgeCDN-X DNS controller (routes based on Location, PrefixList, DNSEndpoint, Zone CRDs)
9. **prometheus** — Metrics export on port 9153
10. **forward** — Upstream DNS resolution fallback (1.1.1.1, 8.8.8.8)
11. **cache** — 30-second TTL cache

### edgecdnx Plugin Configuration
```txt
edgecdnx {
  namespace {{ index .metadata.annotations "edgecdnx.com/namespace" }}
  recordttl 30
  soa ns{{ index .metadata.annotations "edgecdnx.com/ns" }}
  ns ns{{ index .metadata.annotations "edgecdnx.com/ns" }} {{ index .metadata.annotations "edgecdnx.com/public-ip" }}
  # Additional NS records for multi-primary DNS setup
}
```

- **namespace**: Where to watch for CRDs (DNSEndpoint, Location, PrefixList, Zone)
- **recordttl**: Default TTL for generated A/AAAA records (30 seconds)
- **soa**: SOA MNAME label (e.g., `ns1` → SOA record `ns1.zone.`)
- **ns**: NS record entries with hostname and IPv4 address (repeatable for multiple primaries)

### Geolocation Database
- **Init Container**: `curlimages/curl:8.14.1` downloads MaxMind GeoLite2-City MMDB during pod startup
- **Source**: `https://share.tbotech.sk/api/shares/...` (configured organization storage)
- **Mount Path**: `/etc/edgecdnx/geolookup/GeoLite2-City.mmdb`
- **Storage**: `emptyDir` volume (ephemeral, redownloaded on pod restart)

### Scaling and Availability
- **Replicas**: 2 (high availability for DNS)
- **Service RBAC**: Deployed via `coredns-rbac` Helm chart (v0.1.5)
- **Service Account**: Uses `default` service account with RBAC permissions
- **Sync Strategy**: Manual selfHeal disabled (ArgoCD won't auto-correct drift)

## Deployment Flow

1. **Cluster Discovery**: ArgoCD discovers clusters with label `edgecdnx.com/routing: "true"`
2. **ApplicationSet Generation**: For each cluster, an Application is generated with:
   - CoreDNS Helm chart (official, v1.43.3) with custom image and plugin configuration
   - RBAC chart for service account permissions
3. **Namespace Creation**: `edgecdnx-routing` namespace is created automatically
4. **GeoIP Database Initialization**: Init container downloads MMDB before CoreDNS starts
5. **Plugin Startup**: CoreDNS loads the `edgecdnx` plugin, which connects to Kubernetes API and starts watching CRDs
6. **Readiness Check**: CoreDNS waits for informers to sync before reporting ready
7. **DNS Service**: LoadBalancer exposes CoreDNS on public IP for clients to query

## Updating Deployment

### Update CoreDNS Image Version
1. Modify the `image.tag` field in the ApplicationSet
2. Commit to GitOps repository
3. ArgoCD automatically syncs to all routing clusters

### Add a New Routing Location
1. Create a new cluster Secret in ArgoCD with:
   - Label: `edgecdnx.com/routing: "true"`
   - Annotations: `edgecdnx.com/namespace`, `edgecdnx.com/public-ip`, `edgecdnx.com/ns`
2. Commit to GitOps repository
3. ArgoCD detects the new cluster and deploys CoreDNS automatically

### Update GeoIP Database
1. Update the curl URL in the init container command
2. Commit and apply
3. ArgoCD re-creates pods, which download the new MMDB
