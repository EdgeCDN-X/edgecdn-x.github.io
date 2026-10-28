# Standalone Deployment

The standalone deployment can be deployed to a kind cluster or to any hyperscaler or K8S hosting provider. Before you begin to choose where to deploy there are a few notes which need to be taken into consideration. For the DNS to work well with GeoIP, the CoreDNS process must see the incoming IP address of the client the request is coming from. This is often an issue with many smaller managed kubernetes providers.
Running on baremetal with Cilium LB or MetalLB is tested to be working. AWS is one of the hyperscalers, which can preserve the source IP address on UDP protocol as well. Do your analysis before choosing where do you want to deploy the solution.

## Deploying to KIND for local development

Create a kind Cluster:
```bash
➜  kind create cluster --name edgecdnx-standalone-dns
Creating cluster "edgecdnx-standalone-dns" ...
 ✓ Ensuring node image (kindest/node:v1.36.1) 🖼
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-edgecdnx-standalone-dns"
You can now use your cluster with:

➜  kubectl cluster-info --context kind-edgecdnx-standalone-dns

Thanks for using kind! 😊
```

EdgeCDN-X Controller utilizes webhook validations, so you'll need cert manager. Install cert-manager with:

```bash
➜  helm install \
  cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.21.3 \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```


### Deploy EdgeCDN-X controller
Deploy the EdgeCDN-X Controller from the official EdgeCDN-X Helm Repository:

```bash
➜  helm repo add edgecdnx https://edgecdn-x.github.io/helm-charts/
➜  helm install edgecdnx-controller edgecdnx/edgecdnx-controller --namespace edgecdnx-controller-system --create-namespace
```

Check if the edgecdnx-controller is deployed successfully:
```bash
➜  kubectl get all -n edgecdnx-controller-system
NAME                                       READY   STATUS    RESTARTS   AGE
pod/edgecdnx-controller-696bd994b8-rr5ql   1/1     Running   0          70s

NAME                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/edgecdnx-controller   ClusterIP   10.96.235.228   <none>        443/TCP   71s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/edgecdnx-controller   1/1     1            1           71s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/edgecdnx-controller-696bd994b8   1         1         1       70s
```


### Deploy CoreDNS with EdgeCDN-X Plugin
Next you will need CoreDNS with the EdgeCDN-X plugin deployed. Create a values-coredns.yaml file and set the following values:

```yaml
image:
  repository: fr6nco/coredns
  tag: 1.14.2-edgecdnx-v2.2.0
  pullPolicy: Always
serviceType: LoadBalancer
service:
  externalTrafficPolicy: Local
isClusterService: false
servers:
  - plugins:
      - name: ready
      - name: debug
      - name: metadata
      - name: log
        parameters: . "{combined}"
      - name: errors
        configBlock: |-
          stacktrace
      - name: health
        configBlock: |-
          lameduck 5s
      - name: geoip
        parameters: /etc/edgecdnx/geolookup/GeoLite2-City.mmdb
        configBlock: |-
          edns-subnet
      - name: edgecdnx
        configBlock: |-
          recordttl 30
          soa ns1.example.com.
          ns ns1.example.com.
    zones:
      - zone: '.'
        scheme: dns://
        port: 53
initContainers:
  - name: edgecdnx-mmdb-init
    image: curlimages/curl:8.14.1
    volumeMounts:
      - name: geolookup-mmdb
        mountPath: /etc/edgecdnx/geolookup
    command:
      - sh
      - -c
      - | 
        set -eu
        cd /etc/edgecdnx/geolookup
        curl -fSL --retry 3 \
            "https://cdn.jsdelivr.net/npm/geolite2-city/GeoLite2-City.mmdb.gz" \
            -o GeoLite2-City.mmdb.gz
        gunzip GeoLite2-City.mmdb.gz
        test -s GeoLite2-City.mmdb
extraVolumes:
  - name: geolookup-mmdb
    emptyDir: {}
extraVolumeMounts:
  - name: geolookup-mmdb
    mountPath: /etc/edgecdnx/geolookup
```

Please note you should have your own GeoIP Database from Maxmind with proper licensing

Install it with:
```bash
➜  helm install coredns edgecdnx/coredns --namespace edgecdnx-controller-system --create-namespace -f values-coredns.yaml
```

The CoreDNS needs additional RBAC configuration so it can work with the specified CRDs by the EdgeCDN-X Controller. There's a small helm chart prepared for this purpose which sets the necessary RBAC fields
```bash
➜  helm install coredns-rbac edgecdnx/coredns-rbac --namespace edgecdnx-controller-system --create-namespace --set serviceAccount=default
```

The service is exposed as a LoadBalancer, for a Kind Cluster you'll need a kind cloud controller. Refer to the [documentation](https://kind.sigs.k8s.io/docs/user/loadbalancer/) how to run it. tldr on mac in another terminal;
```bash
➜  brew install cloud-provider-kind
➜  sudo cloud-provider-kind --enable-lb-port-mapping
```

wait until you get a load balancer IP
```bash
➜  kubectl get service
NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
coredns               LoadBalancer   10.96.238.197   172.18.0.5    53:31248/UDP   15m
edgecdnx-controller   ClusterIP      10.96.235.228   <none>        443/TCP        33m
```

to access it, examine the exported Ports (this is only mac specific)
```bash
➜  docker ps --format 'table {{.Names}}\t{{.Ports}}'
NAMES                                   PORTS
kindccm-62ec090a528a                    0.0.0.0:62747->53/udp, 0.0.0.0:65310->10000/tcp
```

Your CoreDNS port will be available at localhost port 62747 (please note that the **cloud-provider-kind** must be running in the background)


### Deploy HealthCheker
Healthchecker is necessary to run active healthchecks. Install it from the helm repository with helm:

```bash
➜  helm install healthchecker edgecdnx/healthchecker --namespace edgecdnx-controller-system --create-namespace 
```

## Next steps
Now you have the EdgeCDN-X DNS deployed locally in a kind cluster. Continue to defining [projects](user-guide/defining-projects.md) and [zones](user-guide/defining-zones-and-records.md)