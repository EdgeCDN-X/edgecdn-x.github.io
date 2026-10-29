# Defining Zones and Records


## Creating Zones
Create your first zone. e.g.:

```yaml
apiVersion: infrastructure.edgecdnx.com/v1alpha1
kind: Zone
metadata:
  name: edgecdnx.com
  namespace: edgecdnx
spec:
  project:
    name: edgecdnx
  email: noc.edgecdnx.com.
  zone: edgecdnx.com
```

Watch carefully the coredns logs. As soon as the Zone is applied, SOA records are automatically generated for you:
```
➜  kubectl logs -f -n edgecdnx-controller-system coredns-6dc565d7dc-9j47t --since=1m 
Defaulted container "coredns" out of: coredns, edgecdnx-mmdb-init (init)
[DEBUG] edgecdnx: Crafted SOA record for zone edgecdnx.com.: edgecdnx.com.      3600    IN      SOA     ns1.example.com. noc.edgecdnx.com. 2026102800 7200 3600 1209600 3600
[DEBUG] edgecdnx: Crafted NS record for zone edgecdnx.com.: edgecdnx.com.       3600    IN      NS      ns1.example.com.
```

If we query our DNS endpoint with a SOA record, we should now get a successful response:
```
➜  dig @127.0.0.1 -p 62747 NS edgecdnx.com +short
ns1.example.com.

➜  dig @127.0.0.1 -p 62747 SOA edgecdnx.com +short
ns1.example.com. noc.edgecdnx.com. 2026102800 7200 3600 1209600 3600
```


## Adding Records to Zone

We will start by adding simple DNS Records. Simple records do not go through the routing engine of the CoreDNS component, simple return how they're defined.
Define and apply a DNSRecord

```yaml
apiVersion: infrastructure.edgecdnx.com/v1alpha1
kind: DNSEndpoint
metadata:
  name: simple.edgecdnx.com-a
  namespace: edgecdnx
spec:
  project:
    name: edgecdnx
  zone:
    name: edgecdnx.com
  dnsName: simple.edgecdnx.com
  recordTTL: 300
  recordType: A
  routingPolicy: Simple
  targets:
  - 192.168.80.2
  - 192.168.80.3
```

You have to define both the project and zone refs and they must point to the existing project and zone name. Once applied, you will be able to query the endpoint:

```bash
➜  kubectl apply -f simple-record.yaml 
dnsendpoint.infrastructure.edgecdnx.com/simple.edgecdnx.com-a created

➜  dig @127.0.0.1 -p 62747 simple.edgecdnx.com +short
192.168.80.2
192.168.80.3
```

A, CNAME, AAAA and TXT record types are currently supported. To move forward for advanced routing features start by defining [Locations](defining-locations.md)