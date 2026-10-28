# Defining Projects

EdgeCDN-X controller system can work in 2 modes, namespaced and single namespace mode. We highly encourage you to use namespaced mode, where zones, dnsrecords and other resources are tied to a project. The Project CRD is not necessary in the community version, although the Commercial edition uses these to enable multi-tenancy and the webhooks are configured to validate these so we have to start by creating a project.

When in namespaced mode (by default) project name must be the same as the namespace name where the project is deployed.

Apply the following manifests:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: edgecdnx
---
apiVersion: infrastructure.edgecdnx.com/v1alpha1
kind: Project
metadata:
  name: edgecdnx
  namespace: edgecdnx
spec:
  description: Project Space for EdgeCDN-X
  name: EdgeCDN-X
  rbac:
    groups: []
    rules: []
```

This will create a namespace edgecdnx and a project name edgecdnx. In namespaced mode the project name must match the namespace name, thus multiple projects are not allowed in the same namespace. In case we want to create a new project in the same namespace with a different name, the validating webhook will reject the request:

```bash
➜  kubectl apply -f project.yaml 
Error from server (Forbidden): error when creating "project.yaml": admission webhook "tenancy.infrastructure.edgecdnx.com" denied the request: project name must match namespace "edgecdnx"
```


Go ahead and create your first zone and records to test the [DNS ecosystem](defining-zones-and-records.md).