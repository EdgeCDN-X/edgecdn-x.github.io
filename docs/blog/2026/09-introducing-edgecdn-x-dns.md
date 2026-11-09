---
title: "Introducing EdgeCDN-X DNS"
description: "Managed and self-hosted DNS with Kubernetes-native routing."
slug: introducing-edgecdn-x-dns
date: 2026-11-09
---

DNS is a small part of an application stack, but it is often the first step in
reaching a service. Managing records, routing traffic, and responding to
unhealthy endpoints should fit the way your infrastructure is operated.

EdgeCDN-X brings these capabilities together in a Kubernetes-native DNS
platform. You can use the hosted service, run the open-source components in
your own infrastructure, or discuss an on-premises deployment.

## Start with managed DNS

The hosted EdgeCDN-X portal provides a place to manage DNS zones and records
without operating the underlying DNS infrastructure yourself. A free plan is
available for personal and commercial use.

Visit the [portal](https://portal.edgecdnx.com/) to get started, or explore the
[demo](https://portal.demo.edgecdnx.com/) to see the interface.

## Go beyond static records

EdgeCDN-X DNS can return static answers or select targets using routing
policies. Depending on your configuration, you can distribute traffic with
weighted or round-robin routing, prefer a primary location with failover, or
use geolocation and IP-prefix information to select a suitable location.

Health checks, maintenance state, and active alerts help determine which nodes
are eligible to receive traffic. The DNS routing layer can also operate
independently of the caching component, so it can be used with your existing
infrastructure.

## Built on Kubernetes and CoreDNS

The platform uses Kubernetes custom resources to describe DNS and routing
configuration. The EdgeCDN-X controller manages the resources and validates
them, while CoreDNS with the EdgeCDN-X plugin answers queries using that
configuration. A healthchecker monitors configured endpoints and reports
health changes for routing decisions.

For teams that prefer to run their own infrastructure, the core controller,
CoreDNS plugin, and healthchecker are open source under AGPLv3. Start with the
[documentation](https://edgecdn-x.github.io/) or the
[standalone DNS deployment guide](https://edgecdn-x.github.io/standalone-dns-deployment/).

## What's next

EdgeCDN-X is designed as a broader platform for DNS and content delivery.
Today, DNS can be used on its own; the caching component is still a work in
progress and is not intended for heavy production use.

Alongside the CDN, the longer-term product portfolio is intended to include
container hosting for customer applications. The goal is to bring application
workloads together with DNS and content delivery on the EdgeCDN-X platform.
Container hosting is a future offering, not part of the service available
today; availability and details will be shared as the product develops.

Explore the [hosted portal](https://portal.edgecdnx.com/), read the
[documentation](https://edgecdn-x.github.io/), or
[contact us](https://edgecdn-x.github.io/contact/) to discuss an on-premises
deployment.
