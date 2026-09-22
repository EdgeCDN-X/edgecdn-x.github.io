---
title: "Self-hosted Route53 alternative with advanced routing"
description: "Weighted, failover, geolocation, and round-robin DNS routing"
slug: routing-policies-in-edgecdnx-plugin
date: 2026-09-22
---

The EdgeCDN-X DNS plugin now includes advanced routing patterns directly in its DNSEndpoint model, making it easier to run a self-hostable Route53-style DNS layer from Kubernetes.

The plugin now implements the routing policies described in the GitHub README, including `Simple`, `Weighted`, `Failover`, `Geolocation`, and `RoundRobin`. You can review the full policy reference here: [EdgeCDN-X edgecdnx-plugin DNSEndpoint types](https://github.com/EdgeCDN-X/edgecdnx-plugin#dnsendpoint-types).

## Why this matters

DNS policy decisions often live in different places: static zone files, custom scripts, or external traffic-management systems. That makes operational changes slower and more fragile. With the EdgeCDN-X DNS plugin, the same Kubernetes workflow that manages locations, health checks, and node targeting can also describe how requests should be routed.

This gives platform and application teams a consistent way to express intent without depending on a managed DNS provider for every routing decision. A service can be pinned to a direct origin, prefer a primary region, rotate across healthy candidates, or use geographic context to keep users close to the most relevant edge location.

## The supported routing policies

The plugin now supports five core DNSEndpoint routing policies:

- `Simple`: Return the configured targets directly.
- `Weighted`: Prefer some matching locations or node groups based on weight metadata.
- `Failover`: Start with a primary location and move to healthy fallback targets when needed.
- `Geolocation`: Combine route matching, prefix routing, and geo metadata to pick the best location.
- `RoundRobin`: Cycle through matching targets in a deterministic sequence.

Each option is exposed through the `spec.routingPolicy` field and aligned with the wider EdgeCDN-X model for locations, route selectors, and health-aware node selection. That keeps the decision path consistent with the rest of the DNS controller rather than introducing a parallel routing system.

## A practical example

A service with several regional edge clusters can use `Weighted` routing to prefer one region while still distributing traffic across others. A global application can use `Geolocation` to select the closest suitable location, while a mission-critical API can use `Failover` to keep traffic on the primary site until it becomes unhealthy. For simple origin records, `Simple` continues to be the most direct option.

This is particularly useful when the service also benefits from the plugin’s node health filtering, fallback handling, and label-based route selection. The routing policy is not a standalone feature; it is part of the same decision engine that sees location health, maintenance state, and route metadata.

## Get started

If you are already running the EdgeCDN-X plugin, update to the version that includes these policy types and begin expressing DNS behavior as Kubernetes resources. If you are evaluating the project, start with the `DNSEndpoint` examples in the GitHub README and then map the policies to your own service topology and failover requirements.

The full routing-policy reference is available in the GitHub project page: [EdgeCDN-X edgecdnx-plugin DNSEndpoint types](https://github.com/EdgeCDN-X/edgecdnx-plugin#dnsendpoint-types).

This update makes the plugin easier to apply in real deployments where routing strategy is part of the same declarative workflow as the rest of the edge platform. For teams looking for a self-hostable Route53 alternative, the plugin now covers the core DNS behaviors needed for modern weighted, failover, and geolocation routing from Kubernetes.
