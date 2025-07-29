# Overview

## What is EdgeCDN-X

![Logo](assets/logo.png)

EdgeCDN-X is an open source CDN platform built on top of CNCF projects. It's core orchestration is built on top of kubernetes using CRDs and Custom controllers. For caching, nginx-ingress is used and for routing CoreDNS is used.

## Why EdgeCDN-X
EdgeCDN-X is using a robust control plane built on top of Kubernetes API server using CRDs. Deploy services declaratively, using an UI or via GitOps. EdgCDN-X under the hood is using ArgoCD, which is mainly using Cluster Generators to roll out the deployment to the desired locations.


# Components

* Control-Plane - GitOps / Declarative control plane using K8S CRDs helps to roll out the services across the desired locations. ArgoCD is used in the background for effective resource distribution
* Routing - [CoreDNS](https://coredns.io/) based DNS server with improved logic. Effectively a GSLB, which can redirec the requests to the closest edge based on the client's location. **Features**:
    * Prefix Routing (IPv4 or IPv6 based) prefixes can be defined
    * Location based routing using GeoLookup
    * Active Healthchecks
    * Weight based routing, e.g. A/B testing, unequal load-balancing
    * Fallback routing. Fallback to altenative region if no caapcity is available in the given region
    * Dynamic reconfiguration without reloads. DNS server actively wach changes on k8s API
    * Support NS record configuration for zone delegation (e.g. cdn.edgecdnx.com)
* Caching - Nginx Ingress based caching engine. The [ingress-nginx](https://kubernetes.github.io/ingress-nginx/) controller is used as it is without any forking. Currently all the functionality is achieved purely using customizations and __annotations__.
* Secure-URLs - Custom component supporting URL signatures. To avoid access to certain objects publicly it is possible to use URL signatures to prevent unauthorized access to the resources. These signatures are often used for signing Stream (HLS or MPEG) playlists. Further down the line, once the signature is verified a session cookie is issued which the client can use to access the stream without having to Sign each segment's request. The session is only valid for a specific stream.
* S3-Gateway - S3 gateway connector ensures that we can use private or public S3 buckets for our content origin.


Read further on the [Architecture](architecture.md) here.