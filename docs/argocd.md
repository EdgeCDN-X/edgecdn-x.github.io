---
tags:
  - argocd
---
# Control Plane
The control plane is central component where the rollout logic is happening. EdgeCDN-X Platform relies on ArgoCD to effectively distribute the resources to the desired locations.

# Prerequisites
* Cert-manager - for certificate creation.
* Domain name - Pointing to ArgoCD endpoint

## ArgoCD Deployment
ArgoCD is deployed via Helm. As of writing these docs, the latest helm release for ArgoCD is **8.0.13** from repository **https://argoproj.github.io/argo-helm**.

Install `argo-cd` with the following `values.yaml` files:



```yaml
configs:
  params:
    server.insecure: true
    applicationsetcontroller.namespaces: "*"
    applicationsetcontroller.enable.scm.providers: "false"
    application.namespaces: "*"
  cm:
    exec.enabled: true
server:
  ingress:
    enabled: true
    ingressClassName: {ingressclass-name}
    annotations:
      cert-manager.io/cluster-issuer: {cluster-issuer-name}
    tls: true
    hostname: {argocd-hostname}
controller:
  resources:
    requests:
      memory: 512Mi
    limits:
      memory: 1024Mi
```

* We disallow scm providers, due to incompatibility with applicationsets in any namespace
* we allow applications to be placed in any namespace
* we allow applicationsets to be placed in any namespace
* adjust your hostname and cluster-issuer if necessary.
* adjust resources based on your cluster size.

For a working SSL cert endpoint, you must have cert-manager installed.

```shell
kubectl create ns argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm -n argocd install my-argo-cd argo/argo-cd --version 8.0.13 -f values.yaml
```

This setup is enough for a basic ArgoCD deployment. For scaling and further tweaks please refer to the ArgoCD documentation.

ArgoCD discourages this setup, where we can host applicationsets in any namespace, due to this, we have to slightly adjust some RBAC fields. 

Apply the following additional yamls so the application and applicationset controller can read applications and applicationsets in any namespace.

**appset-rbac.yaml**:
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/name: argocd-applicationset-controller-cluster-apps
    app.kubernetes.io/part-of: argocd
    app.kubernetes.io/component: server
  name: argocd-applicationset-controller-cluster-apps
rules:
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
- apiGroups:
  - "argoproj.io"
  resources:
  - "applicationsets"
  - "applications"
  - "appprojects"
  - "applicationsets/status"
  verbs:
  - create
  - delete
  - update
  - patch
  - watch
  - list
- apiGroups:
  - ""
  resources:
  - secrets
  - configmaps
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/name: argocd-applicationset-controller-cluster-apps
    app.kubernetes.io/part-of: argocd
    app.kubernetes.io/component: server
  name: argocd-applicationset-controller-cluster-apps
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argocd-applicationset-controller-cluster-apps
subjects:
- kind: ServiceAccount
  name: argocd-applicationset-controller
  namespace: argocd
```

**app-rbac.yaml**
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/name: argocd-notifications-controller-cluster-apps
    app.kubernetes.io/part-of: argocd
    app.kubernetes.io/component: notifications-controller
  name: argocd-notifications-controller-cluster-apps
rules:
- apiGroups:
  - "argoproj.io"
  resources:
  - "applications"
  verbs:
  - get
  - list
  - watch
  - update
  - patch
- apiGroups:
  - ""
  resources:
  - secrets
  - configmaps
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/name: argocd-notifications-controller-cluster-apps
    app.kubernetes.io/part-of: argocd
    app.kubernetes.io/component: notifications-controller
  name: argocd-notifications-controller-cluster-apps
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argocd-notifications-controller-cluster-apps
subjects:
- kind: ServiceAccount
  name: argocd-notifications-controller
  namespace: argocd
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/name: argocd-server-cluster-apps
    app.kubernetes.io/part-of: argocd
    app.kubernetes.io/component: server
  name: argocd-server-cluster-apps
rules:
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
- apiGroups:
  - "argoproj.io"
  resources:
  - "applications"
  verbs:
  - create
  - delete
  - update
  - patch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/name: argocd-server-cluster-apps
    app.kubernetes.io/part-of: argocd
    app.kubernetes.io/component: server
  name: argocd-server-cluster-apps
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argocd-server-cluster-apps
subjects:
- kind: ServiceAccount
  name: argocd-server
  namespace: argocd
```

Further [reading](https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/Appset-Any-Namespace/).