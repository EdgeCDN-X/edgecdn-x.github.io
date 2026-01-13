---
tags:
  - edgecdnx-controller
  - routing
---
# EdgeCDN-X Controller - Routing
The controller is the same one as [this](edgecdnx-controller.md) but running in a `router` role. In such case the controller is responsible for managing the objects related to Routing.

Currently there is no extra functionality on of the controller on the Routing Nodes except for setting the Resources to Healthy.

[CRDs](crds.md) are deployed alongside this helm deployment.
In the values you have to specify the **role** to **router**.