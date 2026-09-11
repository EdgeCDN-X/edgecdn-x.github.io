# Overview

## What is EdgeCDN-X

![Logo](assets/logo.png)

EdgeCDN-X is an open source CDN platform built on top of CNCF projects. It's core orchestration is built on top of kubernetes using CRDs and Custom controllers. For caching, nginx-ingress is used and for routing CoreDNS is used.

## Why EdgeCDN-X
EdgeCDN-X is using a robust control plane built on top of Kubernetes API server using CRDs. Deploy services declaratively, using an UI or via GitOps. EdgCDN-X under the hood is using ArgoCD, which is mainly using Cluster Generators to roll out the deployment to the desired locations.


# Components

* Control-Plane - GitOps / Declarative control plane using K8S CRDs helps to roll out the services across the desired locations. ArgoCD is used in the background for effective resource distribution
* Routing - [CoreDNS](https://coredns.io/) based Global Server Load Balancer (GSLB) with EdgeCDN-X DNS controller. Routes requests to the closest edge location based on client location, IP prefix, and health status. **Features**:
    * Prefix-based routing (IPv4/IPv6) with EDNS client subnet support
    * Geolocation-based routing with weighted location balancing
    * Deterministic hash-based node selection within locations for cache affinity
    * Active health checks with alert-aware node filtering
    * Hierarchical location fallback with parent and sibling location support
    * Dynamic reconfiguration without reloads—DNS controller watches K8S API for real-time changes
    * Direct node access via DNS queries (node.location.node.service pattern)
    * Configurable response modes (A/AAAA or CNAME) per request origin (DNS vs gRPC)
    * Zone-authoritative DNS behavior for managed domains and records
* Caching - Nginx Ingress based caching engine. The [ingress-nginx](https://kubernetes.github.io/ingress-nginx/) controller is used as it is without any forking. Currently all the functionality is achieved purely using customizations and __annotations__.
* Secure-URLs - Custom component supporting URL signatures. To avoid access to certain objects publicly it is possible to use URL signatures to prevent unauthorized access to the resources. These signatures are often used for signing Stream (HLS or MPEG) playlists. Further down the line, once the signature is verified a session cookie is issued which the client can use to access the stream without having to Sign each segment's request. The session is only valid for a specific stream.
* S3-Gateway - S3 gateway connector ensures that we can use private or public S3 buckets for our content origin.
* UI - User interface Prototype is in progress and will be rolled out soon.


Read further on the [Architecture](architecture.md) here.