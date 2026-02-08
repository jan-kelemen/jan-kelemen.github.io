---
author: Jan Kelemen
date: '2024-07-05T00:00:00Z'
tags:
- C++
- Vulkan
- graphics programming
- bullet3 physics
aliases:
- /2024/07/05/it-goes-in-the-square-hole.html
title: It Goes In The Square Hole
---

I gave a hint that I wanted to study how to do mouse picking in a 3D environment properly. 
For that, I've chosen to go with the route of raycasting from the mouse position into the scene and remake the scene from the popular video [The Original Square Hole Girl Video + The Redemption](https://www.youtube.com/watch?v=cUbIkNUFs-4).
Link to the source code: [jan-kelemen/geos](https://github.com/jan-kelemen/geos)

You can see the results in the video below.

{{< youtube "vkjBM03vehw" >}}

# Mouse picking
As mentioned, mouse picking is implemented by raycasting. For this, I've started with a single cube with 2x2x2 dimensions, also known as the default Blender cube.
This was much easier to start since the dimensions it uses are trivial to debug if some part of the calculation isn't correct.

The process starts by projecting the current mouse position on the window into the World. 
The first thing needed is to calculate the coordinates in the world space that match the mouse position on the window. 
We do this on both the near and the far plane of view space, depth in Vulkan is in the range [0, 1], therefore the near coordinate has a Z value of 0, and the far coordinate has a Z value of 1. 
Which are passed to the [glm::unprojectZO](https://glm.g-truc.net/0.9.9/api/a00666.html#gade5136413ce530f8e606124d570fba32) function taking into account the current view and projection matrices of the camera view.

```cpp
glm::fvec3 const window{cppext::as_fp(x_position),
    cppext::as_fp(y_position),
    0};

glm::fvec4 const viewport{0, 0, width_, height_};

auto const near{glm::unProjectZO(glm::fvec3{window.x, window.y, 0},
    view_matrix_,
    projection_matrix_,
    viewport)};

auto const far{glm::unProjectZO(glm::fvec3{window.x, window.y, 1},
    view_matrix_,
    projection_matrix_,
    viewport)};
```

Once we have that, we can cast a ray passing through these two points and see on which point it intersects with some object in the world, more on that later when we get to the physics simulation.

# Creating the model
To create the model I've used Blender. Square, rectangle and thin rectangle models were trivial, those are just cube models with applied scaling on X, Y and Z axes.
The cylinder is also a default shape available in Blender. For the semicircle shape, I used a circle and cut it in half, the triangle was copied from the semicircle and removed most of the vertices.
The arch was a bit trickier, I created a rectangle and then intersected it with a cylinder shape using a [boolean modifier](https://docs.blender.org/manual/en/latest/modeling/modifiers/generate/booleans.html)
to remove the intersecting part of those shapes. The sorter first started as an intersection of 2 cylinders of different diameters to create a hole in the middle.
To make the holes on the top I've taken all of the shapes, scaled their dimensions up 20%, again removed the intersections between the sorter and the scaled shapes.
Cutting out the arch shape needed a bit more work, I scaled the whole rectangle up by 20% and then intersected it with a cylinder shape which was scaled down by 20%. 
As scaling up the arch model directly also shrinks the radius of the cutout cylinder.

# Physics simulation
Now that we have the model of the scene, back to the mouse picking. To check if the mouse ray collides with some object, we test it against a collision shape of the object.
This could be an axis aligned bounding box. I've used the [Bullet Physics SDK](https://github.com/bulletphysics/bullet3), which thankfully has a [rayTest](https://pybullet.org/Bullet/BulletFull/classbtCollisionWorld.html#aaac6675c8134f6695fecb431c72b0a6a) function which does exactly that.
```cpp
btCollisionWorld::ClosestRayResultCallback callback{near, far};
world_->rayTest(near, far, callback);
if (callback.hasHit())
{
    return std::make_pair(callback.m_collisionObject,
        callback.m_hitPointWorld);
}
```
From this, we get back the object which intersects with the mouse ray and the intersection point. Afterwards to move the picked object around a point to point constraint is created:
```cpp
auto const local_pivot{
    rigid_body->getCenterOfMassTransform().inverse() * point};
pick_constraint_ =
    std::make_unique<btPoint2PointConstraint>(*rigid_body, local_pivot);
physics_simulation_->add_constraint(pick_constraint_.get());
pick_constraint_->m_setting.m_impulseClamp = 100.0f;
pick_distance_ =
	(point - btVector3{near.x, near.y, near.z}).length();
```

To move the object closer or away from the camera, I've used the mouse scroll wheel event which changes the distance between points of the constraint.
This gives the possibility to lift the object into the sorter and track it along as the mouse moves.
```cpp
auto const direction{glm::normalize(far - near) * pick_distance_};
auto const new_pivot{near + direction};
pick_constraint_->setPivotB({new_pivot.x, new_pivot.y, new_pivot.z});
```

Finally releasing the mouse button removes the point to point constraint and allows the object to fall back down.

Collision shapes of objects are defined as following Bullet collision shapes:
* Square, rectangle and thin rectangle - [btBoxShape](https://pybullet.org/Bullet/BulletFull/classbtBoxShape.html)
* Cylinder - [btCylinderShape](https://pybullet.org/Bullet/BulletFull/classbtCylinderShape.html)
* Triangle and semicircle - [btConvexHullShape](https://pybullet.org/Bullet/BulletFull/classbtConvexHullShape.html)
* Arch and sorter - [btGImpactMeshShape](https://pybullet.org/Bullet/BulletFull/classbtGImpactMeshShape.html)

Each of these shape types has different computational complexities so it's best to use the simplest shape possible which fits the object.
Collision shapes are created directly from the vertices loaded from the glTF model, for the box and cylinder shapes dimensions of the shape can be calculated from the axis aligned bounding boxes.
Thankfully, [Sascha Willems' Vulkan physically-Based Renderer](https://github.com/SaschaWillems/Vulkan-glTF-PBR) implements the calculation of AABB so I was able to use that as a reference implementation again. 

Interestingly I didn't have any issues with compiling Bullet, usually I have to tinker around with Conan recipes to make it work on the GitHub Actions pipeline when compiling with sanitizers or toolchain hardening options enabled.
Although I think that there are some false positives coming from the Address Sanitizer when running on Linux, on Windows I didn't observe any.
Also, save yourself the trouble, don't try to use move constructors on Bullet objects, the library does use quite a bit of inheritance and move semantics aren't really working.
Which is understandable for a library developed before move semantics were a thing.

# Vulkan Memory Allocator
I had a bit of time left after finishing with the physics simulation, so I decided to integrate [VulkanMemoryAllocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator) into my rendering code.
Previously I used to do all of the memory allocations for the graphics buffers and images individually, while this didn't cause any issues for now, there are limits to the number of individual active allocations.
In general memory from the GPU should be allocated in chunks and then subdivided by the application to individual resources, there are also alignment constraints for images and buffers.
I've already had my helper functions which allocate buffers and images. Hence, I just needed to initialize the VMA allocator and replace the calls from `vkCreateBuffer` / `vkCreateImage` to `vmaCreateBuffer` and `vmaCreateImage`.
If I knew it was this easy I would have done it sooner. :)

# Final words
Other than the integration with Vulkan Memory Allocator I didn't make improvements on the graphics side of things in this project.
I definitely want to continue improving on that, I'm thinking about doing shadow mapping next, or trying out compute shaders since I haven't touched those yet.
There is also some refactoring work to be done on the glTF model loading or maybe trying to make it work on ARM.

For any questions or corrections to this post, leave a comment in [discussion thread](https://github.com/jan-kelemen/niku/discussions/1) on GitHub or any other place where you see this.

Previous posts in the series:
* [Chess engine visualization](/post/2024-06-13-chess-engine-visualization)
* [Remaking World 1-1 with Vulkan, mostly](/post/2024-05-14-remaking-world-11-mostly)

