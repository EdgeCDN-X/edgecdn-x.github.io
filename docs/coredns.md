# CoreDNS
CoreDNS is used as a routing engine of the EdgeCDN-X Platform. The DNS resolves to IP address based on the user's location. CoreDNS is extended with several custom plugins for enhanced functionality.

## Routing
Routing component routes the individual requests via the following steps:

* Prefix static routing to individual location (sourced from static prefix list)
* GeoLookup to locations if static routing returns no destination
* Consistent hashing in location to maximize cache-hit ratio
* Active healthchecks to make sure destinations are healthy and available
* Fallback routing to different location if location has no active nodes


Routing engine is rolled out to each location with **edgecdnx.com/routing** label in the cluster [metadata](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Cluster/)


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
  name: eu-west-1
spec:
  fallbackLocations:
  - us-west-1
  nodes:
  - name: n1
    ipv4: 192.168.100.35
    caches:
    - ssd
  geoLookup:
    weight: 50
    attributes:
      geoip/continent/code:
        weight: 1000
        values:
          - value: "EU"
            weight: 10
          - value: "AF"
```

The nodes for the specific location have to be defined here. These nodes are automatically healthchecked and evaluated when routing decisions are made. Consul is used to store the endpoints and performing the healthchecks, thus the consul endpoint has to be configured.

These CRDs are consumed by the module in CoreDNS. The module has to be configured accordingly:
```
        edgecdnxgeolookup {
            namespace ${namespace}
            consulEndpoint ${consul-endpoint}
            consulcachettl 5s
            recordttl 30
        }
```


### Service catalog
[edgecdnx-services](https://github.com/EdgeCDN-X/edgecdnx-services). This module is responsible for building the SOA and NS records and also enriches metadata with customer specific information for better routing decitions down the line. All thes services are auto loaded via the k8s client, so it is not required to reload the Configuration when a new service is configured.
This module enriches the metadata in the DNS lookup chain and passes down the service specs to the routing decision.

Example CRDs:
```yaml
---
apiVersion: infrastructure.edgecdnx.com/v1alpha1
kind: Service
metadata:
  name: akldjgsofheiwu.cdn.edgecdnx.com
spec:
  name: akldjgsofheiwu.cdn.edgecdnx.com
  domain: akldjgsofheiwu.cdn.edgecdnx.com
  originType: static
  cacheKey:
    queryParams:
      - "v"
      - "ver"
      - "version"
  staticOrigins:
    - upstream: tbotech.sk
      hostHeader: tbotech.sk
      port: 443
      scheme: Https
  certificate: {}
  customer:
    id: 2
    name: TboTech
  cache: ssd
```

Example configuration:
```
        edgecdnxservices {
            namespace ${namespace}
            soa ${current ns - e.g. ns1}
            email ${soa email}
            ns ns1 ${ NS IP1 }
            ns ns2 ${ NS IP2 }
        }
```


# Full configuration
```

    cdn.edgecdnx.com.:53 {
        ready
        debug
        metadata
        health {
            lameduck 5s
        }
        edgecdnxprefixlist edgecdnx
        geoip /etc/edgecdnx/geolookup/GeoLite2-City.mmdb {
            edns-subnet
        }
        edgecdnxgeolookup {
            namespace edgecdnx
            consulEndpoint http://edgecdnx-consul-consul-server:8500
            consulcachettl 5s
            recordttl 30
        }
        edgecdnxservices {
            namespace edgecdnx
            soa ns1
            email noc.edgecdnx.com
            ns ns1 188.167.203.183
            ns ns2 189.167.203.182
        }
    }

```