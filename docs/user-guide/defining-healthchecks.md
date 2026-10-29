# Defining Health Checks

Health checks let the EdgeCDN-X healthchecker monitor nodes in a `Location`. Probe results are written to `Location.status.nodeStatus`; the CoreDNS plugin uses failed conditions to avoid unhealthy nodes when it answers routed `A` and `AAAA` queries.

This guide uses the `edgecdnx` namespace and project from [Defining Projects](defining-projects.md), and the `fra1-c1` location from [Defining Locations](defining-locations.md).

## Probe Types

| Type | What it checks | Success condition |
| --- | --- | --- |
| `HTTP` | Sends an HTTP `GET` request. Set `protocol: http` or `protocol: https`. | Response status is in the `2xx` range. Other status codes, connection errors and timeouts are failures. |
| `TCP` | Opens a TCP connection to the configured port. | Connection succeeds. It does not check application-level responses. |
| `ASSUME` | Reports a configured static status without connecting to the node. | `status: Healthy` reports healthy; `status: Unhealthy` reports unhealthy. |

For `HTTP` and `TCP`, `target` overrides the node address. If omitted, the probe uses the node IP for the selected stack. For HTTP probes, `host` sets the HTTP Host header and, for HTTPS, the TLS server name; the connection is still made to `target` or the node IP. HTTP ports default to 80 for `http` and 443 for `https` when omitted. Set the path explicitly and begin it with `/`.

## Create a HealthCheckProfile

A profile is namespaced and references the project just like other EdgeCDN-X resources. Each probe needs a unique name within its profile, a positive `interval`, and a positive `timeout`. The checker randomizes the first probe start within one interval, then repeats at the configured interval. Use a timeout shorter than the interval.

This profile checks both the cache service over HTTP and its listening port over TCP:

```yaml
apiVersion: infrastructure.edgecdnx.com/v1alpha1
kind: HealthCheckProfile
metadata:
  name: cache-probes
  namespace: edgecdnx
spec:
  project:
    name: edgecdnx
  probes:
    - name: cache-http
      type: HTTP
      interval: 10s
      timeout: 3s
      http:
        protocol: http
        port: 8080
        path: /health
        stack: IPv4
    - name: cache-tcp
      type: TCP
      interval: 10s
      timeout: 3s
      tcp:
        port: 8080
        stack: IPv4
```

Apply the profile:

```bash
kubectl apply -f cache-probes.yaml
```

### HTTP and HTTPS Options

The probe type remains `HTTP` for both HTTP and HTTPS. Use `protocol: https` for TLS. For a virtual host, set `host`; the healthchecker connects to the node IP (or `target`) but sends that hostname in the request and uses it for TLS server-name verification.

```yaml
- name: cache-https
  type: HTTP
  interval: 15s
  timeout: 5s
  http:
    protocol: https
    port: 443
    path: /health
    host: cache.edgecdnx.com
    stack: IPv4
```

The default HTTP transport verifies TLS certificates using the healthchecker runtime's normal trust roots. A certificate error therefore marks the probe failed. Do not use an HTTPS probe against a node certificate that does not cover the configured `host`.

### TCP Only

Use a TCP probe when a successful connection to a service port is sufficient. It confirms that something accepts connections on that port, not that the service can serve content correctly.

```yaml
- name: cache-listener
  type: TCP
  interval: 10s
  timeout: 2s
  tcp:
    port: 8080
    stack: IPv4
```

### Static Health

`ASSUME` is useful when active checks are not appropriate or while testing integration. It does not contact the node; it publishes the configured status on each interval. It is not a substitute for monitoring a real service.

```yaml
- name: manually-managed-status
  type: ASSUME
  interval: 30s
  timeout: 5s
  assume:
    status: Healthy
    stack: IPv4
```

## Attach the Profile to a Node Group

Reference the profile from `spec.nodeGroups[].healthCheck` on a location in the same namespace. The reference is a local object reference, so only the profile name is specified. The healthchecker starts the profile's probes for each node in that group.

```yaml
apiVersion: infrastructure.edgecdnx.com/v1alpha1
kind: Location
metadata:
  name: fra1-c1
  namespace: edgecdnx
  labels:
    region: eu
    tier: edge
spec:
  project:
    name: edgecdnx
  weight: 3
  fallbackLocations:
    - ams1-c1
  nodeGroups:
    - name: ssd
      flavor: ""
      labels:
        cache: ssd
      healthCheck:
        name: cache-probes
      nodes:
        - name: n1
          ipv4: 192.168.80.10
        - name: n2
          ipv4: 192.168.80.11
```

Apply or update the location:

```bash
kubectl apply -f fra1-c1.yaml
```

Profiles are looked up in the location's namespace. If the referenced profile does not exist or cannot be read, that node group will not get active health checks; check the healthchecker logs and ensure its Kubernetes permissions include reading `healthcheckprofiles`.

Only node groups are processed for active checks. Configure `healthCheck` on the node group rather than the legacy node-level field.

## IPv4, IPv6, and Dual Stack

Set `stack` explicitly on every probe:

- `IPv4` probes the node's `ipv4` address and reports an IPv4 condition.
- `IPv6` probes the node's `ipv6` address and reports an IPv6 condition.
- `Dual` creates separate IPv4 and IPv6 probes. A stack is skipped when that node has no corresponding address.

The DNS plugin evaluates health per query type: an unhealthy IPv4 condition excludes the node from `A` answers, while an unhealthy IPv6 condition excludes it from `AAAA` answers. For a dual-stack node, use `stack: Dual` if both address families should be checked. If you have only IPv4 nodes, as in the example, use `IPv4`.

## Verify Health Status

The first probe is delayed by a random amount up to one interval. Until a probe result has been reported, DNS routing assumes a node with no health conditions is healthy. Allow at least one interval for the initial result to appear.

Inspect the profile and location:

```bash
kubectl get healthcheckprofile cache-probes -n edgecdnx -o yaml
kubectl get location fra1-c1 -n edgecdnx -o yaml
```

Look under `status.nodeStatus` for each node's conditions. A successful IPv4 probe resembles:

```yaml
status:
  nodeStatus:
    n1:
      conditions:
        - type: HealthCheckSuccessful
          stack: IPv4
          healthCheckKey: cache-http
          status: true
          reason: OK
```

The reason contains the last probe message, such as `OK`, `Non-2xx response`, or a connection error. Each probe reports its own condition. When multiple probes target the same stack, a failed condition makes that node ineligible for that address family until the failure clears. The node can still be selected for the other family if its other stack is healthy.

Query a routed record to verify the resulting DNS answer:

```bash
dig @127.0.0.1 -p 62747 failover.edgecdnx.com A +short
dig @127.0.0.1 -p 62747 failover.edgecdnx.com AAAA +short
```

The selected location's fallback locations are tried if it has no healthy node that matches the endpoint's route selector. See [Defining Locations](defining-locations.md#how-labels-work) for node selection and fallback behavior.

## Update or Remove a Check

Editing a profile causes locations that reference it to be reconciled, and their running checks are restarted with the new profile. Update the profile in place:

```bash
kubectl apply -f cache-probes.yaml
```

To stop checking one node group, remove its `healthCheck` reference and apply the location. To remove the profile entirely, first remove its references, then delete it:

```bash
kubectl delete healthcheckprofile cache-probes -n edgecdnx
```

## Troubleshooting

| Symptom | Check |
| --- | --- |
| No health conditions appear | Confirm the profile and location are in the same namespace, the node group references the correct profile name, and the healthchecker is running. Wait through the randomized initial delay and one interval. |
| Probe fails although the service is up | Check the selected stack has a node IP, the target and port are reachable from the healthchecker pod, and the timeout is sufficient. For HTTP, verify the path returns a `2xx` status. |
| HTTPS probe fails | Verify the certificate chain is trusted by the healthchecker and that `host` matches the certificate's TLS name. |
| `A` works but `AAAA` does not (or vice versa) | Inspect the condition's `stack` and ensure the location node has that IP family. A health result only filters the corresponding query family. |
| DNS still returns a node after a failed check | Verify the status has a `HealthCheckSuccessful` condition with `status: false` for the queried stack, and that you are querying the expected location / CoreDNS instance. A node with no conditions is treated as healthy. |
| Profile changes do not take effect | Check healthchecker logs for profile lookup errors and confirm the controller can watch profiles and reconcile referencing locations. |