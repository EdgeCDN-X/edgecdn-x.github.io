# Architecture

Since the whole ecosystem is built around K8S, each location and location type in the architecture is a separate K8S cluster. Each cluster has a different role, and it is encouraged, that a single cluster only server a single role.

## Roles
- [Control Plane](argocd.md)
- [Routing](coredns.md)
- [Caching](caching.md)

Deploy each of these locations as a separate kubernetes cluster to achieve HA. We recommend having at least 2-3 Routing nodes distributed based on your needs. These nodes host DNS and routing engines. Caching clusters are responsible for caching the content. Deploy these in various regions based on your needs. There's only a single control plane, and this control-plane connects to the other clusters via service account tokens. Read through for deployment details for each location type.
