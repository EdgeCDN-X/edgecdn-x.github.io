---
tags:
  - crds
  - coredns
  - routing
  - gslb
  - dns
---
# CoreDNS

EdgeCDN-X uses [CoreDNS](https://coredns.io/) as a Global Server Load Balancer (GSLB). The DNS controller routes requests based on client location, IP prefix, and service health, directing traffic to the most appropriate edge location.

## DNS Routing Engine

The EdgeCDN-X DNS controller (`edgecdnx` plugin) routes DNS queries through the following decision logic:

1. **Direct Node Resolution** (optional): For queries matching the pattern `nodename.location.node.service.`, return the IP address of the specified node directly.
2. **DNSEndpoint Lookup**: Match the query name and type to a `DNSEndpoint` CRD resource.
   - For `Simple` endpoints, return configured target addresses directly.
   - For `Geolocation` endpoints, determine the best location using prefix routing and geolookup.
3. **Location Selection**:
   - **Prefix Routing**: Match the client's source IP (or EDNS client subnet) to a `PrefixList` CRD for direct location assignment.
   - **Geolocation Routing**: If prefix routing doesn't apply, use geolocation data to select a location based on configured geo attributes and weights.
4. **Candidate Node Pool**: Within the selected location, build a pool of healthy nodes:
   - Include nodes from node groups whose labels (merged with location labels) match the endpoint's `routeSelector`.
   - Exclude nodes and node groups in maintenance mode.
   - Include nodes from child locations (locations with `spec.parent` pointing to the chosen location) if the child location is healthy (not in maintenance mode, no active alerts).
   - Use deterministic hashing on the query name to select a specific node from the pool (maximizing cache affinity and minimizing cache misses).
   - Enforce health checks: only include nodes with successful IPv4/IPv6 health status matching the query type (`A` for IPv4, `AAAA` for IPv6).
   - Filter out nodes with active Prometheus alerts.
5. **Health-Aware Fallback**:
   - If no healthy node exists in the chosen location, try the parent location (if configured via `spec.parent`).
   - If the parent location also has no healthy nodes, try each location in the parent's `spec.fallbackLocations` in order.
   - If there is no parent, try locations in the chosen location's `spec.fallbackLocations` directly.
   - Skip any location that is in maintenance mode (`spec.maintenanceMode: true`) or has active alerts (`status.alerts` is non-empty).
   - Continue until a location with a healthy node is found, or exhausts all fallback options.
6. **Response Generation**:
   - **A/AAAA Response**: Return the IP address of the selected node.
   - **CNAME Response**: Return a CNAME pointing to the node using the format `node_name.location.node.original-request.`
   - **Response Type Selection**: Use the configured `dnsresponsetype` for normal DNS queries, or `grpcresponsetype` if the request originated from gRPC.
7. **Zone Authority** (if no DNSEndpoint matched): Fall back to zone-authoritative behavior using `Zone` CRDs to return SOA, NS, or NXDOMAIN responses.

Deploy this engine to each location where **edgecdnx.com/routing** label is set in [metadata](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Cluster/).

## Prefix-Based Routing

The `PrefixList` CRD defines IP address ranges (CIDR blocks) and their destination locations. The DNS controller matches incoming client source IPs (or EDNS client subnet values) against these prefixes to determine the target location.

**Capabilities**:
- IPv4 and IPv6 CIDR routing
- EDNS0 client subnet extension support for fine-grained client location detection
- Non-overlapping prefix consolidation (managed by the EdgeCDN-X controller)

**Example PrefixList**:
```yaml
apiVersion: infrastructure.edgecdnx.com/v1alpha1
kind: PrefixList
metadata:
  name: prefixlist-us-west-1
spec:
  destination: us-west-1  # Target location
  prefix:
    v4:
      - address: 192.168.100.128
        size: 27
      - address: 192.168.100.0
        size: 27
    v6: []
```

## Geolocation-Based Routing

When prefix routing does not match (or is not applicable), the DNS controller falls back to geolocation routing. Each `Location` CRD defines geolocation attributes and weights for scoring requests by geographic proximity.

**Example Location**:
```yaml
apiVersion: infrastructure.edgecdnx.com/v1alpha1
kind: Location
metadata:
  name: nyc1-c1
  labels:
    tier: primary
    region: us-east
spec:
  # (Optional) Parent location for hierarchical fallback and shared configuration
  parent: us-east-1
  # Fallback targets when this location has no healthy nodes
  fallbackLocations:
    - fra1-c1
    - lax1-c1
  # Maintenance mode: when true, this location is skipped in routing decisions
  maintenanceMode: false
  # Geolocation-based routing configuration
  geoLookup:
    weight: 100  # Weight relative to other locations
    attributes:
      geoip/continent/code:
        weight: 1000
        values:
          - value: NA  # North America
          - value: SA  # South America
  # Node groups organize nodes by cache profile and configuration
  nodeGroups:
    - name: ssd
      flavor: ""  # Optional flavor to distinguish variants of the same cache type
      # Kubernetes label selector for discovering nodes via DaemonSet
      nodeSelector:
        kubernetes.civo.com/civo-node-pool: nyc1-c1
      # Labels merged with Location labels for route selector matching
      labels:
        cache-tier: primary
      # Generic metadata for cache configuration (replaces deprecated cacheConfig)
      metadata:
        path: /var/cache/ssd
        maxSize: 4096m
        keysZone: 100m
        inactive: 10080m  # Inactivity timeout
      # Individual nodes in this group
      nodes:
        - name: n1
          ipv4: 74.220.30.216
          ipv6: "2001:db8::1"
          # Maintenance mode: when true, this node is excluded from routing
          maintenanceMode: false
```

**Route Selector Matching**: When a DNSEndpoint specifies a `routeSelector`, the DNS controller matches it against the combined labels of each location and its node groups. For example:
```yaml
# In DNSEndpoint
routeSelector:
  tier: primary  # Must match a label from Location or NodeGroup

# Matches this Location because it has tier: primary at the metadata level
# OR if a NodeGroup has labels: {tier: primary}
```

Node groups can be configured with different cache profiles for services requiring varied caching performance. Multiple node groups with the same name but different flavors can coexist within a location for redundancy or specialized caching needs.


## DNSEndpoint Configuration

`DNSEndpoint` CRDs define which domains are served and how they should be routed (Simple or Geolocation-based).

**Simple Endpoint** (fixed target addresses):
```yaml
apiVersion: infrastructure.edgecdnx.com/v1alpha1
kind: DNSEndpoint
metadata:
  name: static-origin
spec:
  fqdn: static.example.com
  recordType: A
  targets: ["203.0.113.10"]  # Direct answers
  recordTTL: 300
```

**Geolocation Endpoint** (location-aware routing):
```yaml
apiVersion: infrastructure.edgecdnx.com/v1alpha1
kind: DNSEndpoint
metadata:
  name: cdn-service
spec:
  fqdn: cdn.example.com
  recordType: A
  routeSelector:  # Labels to match against location definitions
    tier: primary
  recordTTL: 60
  dnsResponseType: A_AAAA  # or CNAME
```

## Zone Configuration

`Zone` CRDs enable you to serve DNS zones with authoritative SOA and NS records, useful for hosting your own domain apex or delegating subdomains.

**Example Zone**:
```yaml
apiVersion: infrastructure.edgecdnx.com/v1alpha1
kind: Zone
metadata:
  name: example.com
spec:
  zone: example.com
  email: noc@example.com
```


## DNS Controller Configuration

The DNS controller is configured in the CoreDNS Corefile via the `edgecdnx` plugin directive:

```txt
.:53 {
  errors
  health
  ready
  
  edgecdnx . {
    namespace edgecdnx              # K8S namespace containing EdgeCDN-X CRDs
    soa ns1                         # SOA MNAME label prefix
    ns ns1.edge.example.com. 203.0.113.10   # NS records (repeatable)
    ns ns2.edge.example.com. 203.0.113.11
    recordttl 60                    # Default TTL for generated answers
    dnsresponsetype A_AAAA          # Response type for DNS queries (A_AAAA or CNAME)
    grpcresponsetype CNAME          # Response type for gRPC-originated requests
  }
  
  prometheus :9153
  forward . 1.1.1.1 8.8.8.8        # Upstream resolvers for fallthrough
  cache 30
  reload
}
```

**Configuration Directives**:
| Directive | Required | Default | Description |
| --- | --- | --- | --- |
| `namespace` | Yes | none | Kubernetes namespace to watch for CRDs |
| `soa` | Yes | none | SOA MNAME label prefix (e.g., `ns1` → `ns1.zone.`) |
| `ns` | Recommended | empty | NS record entries (repeatable); format: `ns <hostname> <ipv4>` |
| `recordttl` | No | 60 | Default TTL for generated answers (in seconds) |
| `dnsresponsetype` | No | A_AAAA | Response type for normal DNS queries: `A_AAAA` or `CNAME` |
| `grpcresponsetype` | No | CNAME | Response type for gRPC requests: `A_AAAA` or `CNAME` |

## Operational Considerations

### Maintenance and Health Checks
- **Maintenance Mode**: Set `spec.maintenanceMode: true` on a Location or Node to temporarily exclude it from routing without deleting the resource.
- **Health Conditions**: Nodes have health conditions tracked for IPv4 and IPv6 separately (`IPV4HealthCheckSuccessful`, `IPV6HealthCheckSuccessful`). The DNS controller respects these when selecting nodes for `A` or `AAAA` queries.
- **Prometheus Alerts**: Active alerts on locations or nodes (via `status.alerts` or node `status.nodeStatus[name].alerts`) cause them to be skipped in routing. Configure `spec.alerts` with `AlertName` and optional label matchers to watch external Prometheus alerts.

### Plugin Readiness
- The DNS controller plugin readiness is tied to Kubernetes informer synchronization for `Zone`, `DNSEndpoint`, and `PrefixList` watchers.
- If informers have not synced yet, the CoreDNS `ready` probe integration will report the plugin as not ready.
- Location informers are watched independently and do not block plugin readiness.

### Fallthrough Behavior
- If the DNS controller cannot find a matching DNSEndpoint, location, or healthy node, the query is passed to the next plugin in the CoreDNS chain (typically `forward` to upstream resolvers).
- Direct node requests that reference a non-existent service, location, or node also fall through.

## Related Resources

- [EdgeCDN-X CRD Reference](crds.md)
- [EdgeCDN-X Controller - Routing](edgecdnx-controller-router.md)
- [Location CRD Configuration](locations.md)
- [CoreDNS Documentation](https://coredns.io/)