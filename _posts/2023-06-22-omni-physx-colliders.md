---
layout:     post
title:      Omni.PhysX Colliders
summary:    Types of colliders available in OmniPhysX
categories: omniverse
published: false
tags:
 - usd
 - omniverse
 - physx
 - physics
 - collider
---

In PhysX a collider is a component that handles the calculation of collisions between objects. When you add a collider to an object, you're effectively giving it a physical presence within the physics scene of the simulation so that it can interact with other objects.

To visually add a collider component from the GUI (in a Kit application which uses the [omni.physx extension](https://docs.omniverse.nvidia.com/prod_extensions/prod_extensions/ext_physics.html)), you can just right click on a prim and select `Physics->Colliders Preset` to add a default collider.

![collider_menu](/assets/images/add_collider_menu.png)

You can later tweak the type of generated collider and other properties in the added rollout menu `Properties->Physics->Collider`

![collider_rollout_menu](/assets/images/physics_collider_rollout_menu.png)

Here's how you would add a collider in Omniverse USD Python

```python
from pxr import UsdPhysics

stage = omni.usd.get_context().get_stage()
cubePrim = stage.GetPrimAtPath('/World/Cube')
UsdPhysics.CollisionAPI.Apply(cubePrim)
```

Adding a collider in USD involves adding a `CollisionAPI` to a USD prim.