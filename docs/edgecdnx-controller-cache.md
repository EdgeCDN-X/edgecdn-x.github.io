---
tags:
  - edgecdnx-controller
  - caching
---
# EdgeCDN-X Controller - Cache
The controller is the same one as [this](edgecdnx-controller.md) but running in a `cache-controller` role. In such case the controller is responsible for managing the objects related to Caching.

## Responsibilities
In `cache-controller` mode the reconcile loop serves the following tasks:

* Manages origin servers
* Manages S3 Origins
* Manages ingress configuration

The main caching logic is happening here. Detailed use cases is visible in the User Guide section.

[CRDs](crds.md) are deployed alongside this helm deployment.
In the values you have to specify the **role**.