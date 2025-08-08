---
tags:
  - performance
  - networking
  - cache
---
# Cache Prerequisites
Cache locations have to be prepared upfront on the Cache servers. Considering the performance it is recommended to use local disks, ideally SSDs. Your disk performance depends on the available uplink. For servers on beefier bandwidth links consider using NVMe or TMPFS disks. Ideally there will be a support for mixed caches withing a single region and cluster.

**Setup Steps**

* Prepare the storage locations, e.g. `/var/cache/ssd`
* Mark your nodes with a label `edgecdnx.com/cache: ssd`
* Deploy the Ingress [controller](ingress-nginx.md) as a Daemonset

## Cache types
Each cache server should be designated only to a single cache type. In the future we will put effort in having multiple cache types on a single server.

# Networking
We wan't to lower the overhead as much as possible. No thorough performance tests were made yet, but consider the following measures:

* If possible use HostNetworking
* Consider CNI with eBPF support (e.g. Cilium)
* If possible, assign Public IPs directly to the hosts.
