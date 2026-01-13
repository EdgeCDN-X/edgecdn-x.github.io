---
tags:
  - crds
  - coredns
  - routing
---
# CoreDNS
CoreDNS is used as a routing engine of the EdgeCDN-X Platform. The DNS resolves to IP address based on the user's location. CoreDNS is extended with several custom plugins for enhanced functionality.

## Routing
Routing component routes the individual requests via the following steps:

* Prefix static routing to individual location (sourced from static prefix list)
* GeoLookup to locations if static routing returns no destination
* Consistent hashing in location to maximize cache-hit ratio
* Active healthchecks to make sure destinations are healthy and available
* Fallback routing to different location if location has no active nodes

Roll out this engine to each location where **edgecdnx.com/routing** label is set in [metadata](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Cluster/)


### Static Prefix routing
[edgecdnx-prefixlist](https://github.com/EdgeCDN-X/edgecdnx-prefixlist) CoreDNS module gathers all the prefixes for the individual locations. These prefixes must be normalized and must be non overlapping. There's a helper [operator](https://github.com/EdgeCDN-X/edgecdnx-controller) which helps to achieve Prefix Consolidation and Supernet subnetting, to make sure there are no overlaps in the Prefixes. The Module is using CoreDNS's Metadata interface to find the desired destination for a given prefix. 

**Features**:

* Routing to location based on Client IP address 
* Prefixes are stored in a fast balanced AVL Tree to ensure speedy lookups
* EDNS0 Subnet extension support
* IPv4 and IPv6 Supported

The prefixes are defined with a CRD and an example definition is seen here:

```yaml
---
apiVersion: infrastructure.edgecdnx.com/v1alpha1
kind: PrefixList
metadata:
  name: prefixlist-us-west-1
spec:
  source: Static
  destination: us-west-1
  prefix:
    v4:
      - address: 192.168.100.128
        size: 27
      - address: 192.168.100.0
        size: 27
      - address: 192.168.113.225
        size: 24
    v6: []
```

These CRDs are consumed by the module in CoreDNS. The module has to be configured accordingly:
```
edgecdnxprefixlist {namespace}
```

### GeoLookup routing
[edgecdnx-geolookup](https://github.com/EdgeCDN-X/edgecdnx-geolookup) CoreDNS module finds the most suitable locatio based on MMDB2 DB. This module uses [geoip](https://coredns.io/plugins/geoip/) metadata to enrich the necessary fields.
Geolookup module assigns weights and score for each request and the location is based on this score. If multiple locations are found with the same score, the requests are balanced based on associated weight. (e.g. eu-west and eu-east routing to Germany in ratio of 40:60)

Geolookup configurations are coming from CRDs with an example configuration shown here:
```yaml
---
apiVersion: infrastructure.edgecdnx.com/v1alpha1
kind: Location
metadata:
  annotations:
  name: nyc1-c1
spec:
  fallbackLocations:
  - fra1-c1
  geoLookup:
    attributes:
      geoip/continent/code:
        values:
        - value: AN
        - value: NA
        - value: OC
        - value: SA
        - value: AS
        weight: 1000
    weight: 50
  nodeGroups:
  - cacheConfig:
      inactive: 10080m
      keysZone: 100m
      maxSize: 4096m
      name: ssd
      path: /var/cache/ssd
    name: ssd
    nodeSelector:
      kubernetes.civo.com/civo-node-pool: nyc1-c1
    nodes:
    - ipv4: 74.220.30.216
      name: n1
```

The nodes for the specific location have to be defined here. These nodes are automatically healthchecked and evaluated when routing decisions are made. Nodes can be groupped into nodeGroups. Each Node Group can have a different cache configuration. This is used if we have services with varying profiles, where different caching performance is required. This cache selector is configurable in the service definition.

These CRDs are consumed by the module in CoreDNS. The module has to be configured accordingly:
```
        edgecdnxgeolookup {
            namespace ${namespace}
            recordttl 30
        }
```


### Service catalog
[edgecdnx-services](https://github.com/EdgeCDN-X/edgecdnx-services). This module is responsible for building the SOA and NS records and also enriches metadata with customer specific information for better routing decitions down the line. All thes services are auto loaded via the k8s client, so it is not required to reload the Configuration when a new service is configured.

#### Zone Configuration
It is possible to define customer specific Zones, if the customer wishes to bring their own domain. With a CRD it is possible to configure the necessary SOA and NS records. Once the zone is configured, it's the customer's responsibility to configure the corresponding NS records on the parent DNS server.

Example Zone CRD:
```yaml
apiVersion: infrastructure.edgecdnx.com/v1alpha1
kind: Zone
metadata:
  name: cdn.tbotech.sk
spec:
  email: noc.cdn.tbotech.sk
  zone: cdn.tbotech.sk
```

This CRD will create SOA and NS records for cdn.tbotech.sk

#### Service Configuration

If a service is configured, CoreDNS will load the service and respont do A and AAAA requests for the given domain.

Example Service:
```yaml
---
apiVersion: infrastructure.edgecdnx.com/v1alpha1
kind: Service
metadata:
  name: ytdemo.democdn.edgecdnx.com
spec:
  cache: ssd
  cacheKey:
    queryParams:
    - v
    - ver
    - version
  certificate: {}
  customer:
    id: 1
    name: tbotech
  domain: ytdemo.democdn.edgecdnx.com
  hostAliases:
  - name: cdn.tbotech.sk
  name: ytdemo.democdn.edgecdnx.com
  originType: static
  staticOrigins:
  - hostHeader: tbotech.sk
    port: 443
    scheme: Https
    upstream: tbotech.sk
  waf:
    enabled: true
```

This resource will configure CoreDNS to respond to `ytdemo.democdn.edgecdnx.com` and `cdn.tbotech.sk` domains. Zone configuration for both domains have to be present too.


# Full configuration
Since few of these services are dependent on each other, here is an example full config with all the required dependencies:


```yaml
.:53 {
    ready
    debug
    metadata
    log . "{combined}"
    errors {
        stacktrace
    }
    health {
        lameduck 5s
    }
    edgecdnxprefixlist edgecdnx
    geoip /etc/edgecdnx/geolookup/GeoLite2-City.mmdb {
        edns-subnet
    }
    edgecdnxgeolookup {
        namespace edgecdnx
        recordttl 30
    }
    edgecdnxservices {
        namespace edgecdnx
        soa ns1
        ns ns1 74.220.25.73
        ns ns2 54.44.35.25
    }
}
```

If multiple DNS servers are present, the NS configuration whould be available for each server in the edgecdnxservice modules. 

# Source code

The full fork with these modules is available at [CoreDNS](https://github.com/EdgeCDN-X/coredns). Images are published at [Docker HUb](https://hub.docker.com/repository/docker/fr6nco/coredns/tags)