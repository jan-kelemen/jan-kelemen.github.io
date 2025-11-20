---
title: "I Still Struggle With Triangles"
author: Jan Kelemen
tags: [C++, Vulkan, niku, VK_KHR_ray_tracing_pipeline, VK_KHR_acceleration_structure] 
---

Life sometimes gets in the way of game engine development. 
Thanks to Hrvoje for reminding me that I need to publish a new update on my game engine [jan-kelemen/niku](https://github.com/jan-kelemen/niku).

<iframe width="1000" height="562" src="https://www.youtube.com/embed/6sTx6-KbQ7Y" title="niku - I Still Struggle With Triangles" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

Video quality has been massacred by YouTube's compression algorithm.

# Winter is coming
Heating bills are expensive, but thankfully, I have a 220W space heater in my PC.
I've decided it's time to add ray tracing support to the rendering engine.
For this purpose, I've created a new demo `heatx`. 
I'll admit, I've spent way too much time playing Factorio in the meantime. The demo name is a reference to [Heat exchanger](https://wiki.factorio.com/Heat_exchanger).

Prerequisite for using raytracing in Vulkan is to enable [VK_KHR_acceleration_structure](https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_acceleration_structure.html) and [VK_KHR_ray_tracing_pipeline](https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_ray_tracing_pipeline.html) extensions.
While the steps required for this are the same as for any other extension, I ran into a very curious issue.
On Windows it worked; on Linux the Vulkan device refused to be created with the `VK_ERROR_INITIALIZATION_FAILED` error code. 
Even though my graphics card and the installed driver support it.
I've also tried running official Vulkan samples that use ray tracing, and those worked as well.

It took me a while to remember that I usually have Address Sanitizer enabled in the build when I'm doing development on Linux.
Sure enough, the combination of enabling `VK_KHR_acceleration_structure` with ASAN doesn't work.
I'm quite certain that this is a bug in the driver.

I've created a standalone reproduction [acceleration-structure-repro](https://github.com/jan-kelemen/acceleration-structure-repro), if you want to try it out.
Thanks to Marijan for confirming the reproducibility of this.

Back to the topic of ray tracing. 
The first step of rendering a scene with ray tracing is to build acceleration structures of the scene.
Implementation of acceleration structures is left to the graphics driver, but the general concept is similar to [bounding volume hierarchies](https://en.wikipedia.org/wiki/Bounding_volume_hierarchy).

Vulkan API distinguishes between top and bottom level acceleration structures (from now on referred to as TLAS and BLAS).
TLAS consists of multiple BLAS. To build the BLAS we need to convert the geometry of the meshes in the scene into triangles.

I've already had this part sorted out as part of loading the scene from the glTF files.
Most of the work here was to declare the structure of this vertex data in a way that `vkCmdBuildAccelerationStructuresKHR` understands it correctly:
```
geometries.push_back({
    .sType = vku::GetSType<VkAccelerationStructureGeometryKHR>(),
    .geometryType = VK_GEOMETRY_TYPE_TRIANGLES_KHR,
    .geometry =
        {.triangles =
                {
                    .sType = vku::GetSType<
                        VkAccelerationStructureGeometryTrianglesDataKHR>(),
                    .vertexFormat = VK_FORMAT_R32G32B32_SFLOAT,
                    .vertexData = {.deviceAddress =
                                       rv.vertex_buffer.device_address},
                    .vertexStride = sizeof(vertex_t),
                    .maxVertex = primitive_counts.back() - 1,
                }},
    .flags = VK_GEOMETRY_OPAQUE_BIT_KHR,
});
```

For now, I've decided that each of the mesh primitives from the glTF file corresponds to one BLAS.
Once the BLAS are created, the TLAS is created by declaring instances of the individual BLAS.
The [VkAccelerationStructureInstanceKHR](https://docs.vulkan.org/refpages/latest/refpages/source/VkAccelerationStructureInstanceKHR.html) links together an instance of the BLAS and its position/orientation inside a TLAS.
The TLAS is then built with an array of these instance objects.

Here's how the TLAS looks when viewed from NVIDIA NSight's Ray Tracing Inspector:
{:refdef: style="text-align: center;"}
![tlas](/assets/posts/20251120/2025-11-20-TLAS.jpeg)
{:refdef}

With the TLAS built, we can perform ray tracing against it to render the scene.
The ray generation shader is used to generate rays from the point of view of the camera:
```
traceRayEXT(
    topLevelAS, // Top level acceleration structure
    gl_RayFlagsOpaqueEXT,
    0xff,
    0,
    0, 
    0, 
    origin.xyz, // Camera origin
    tmin, // Near plane
    direction.xyz, // Ray direction
    tmax, // Far plane 
    0
);
```
The `origin`, `direction`, `tmin` and `tmax` define the space that is checked for ray intersections with the TLAS (and containing BLAS).
For the rays generated by the ray generation shader, either a hit shader or a miss shader will be invoked.
There's a closest hit shader that's called for the first BLAS instance that the ray intersects and an any hit shader that is used when a ray continues passing the object once hit. A use case for it would be a transparent object, like a window.
The miss shader is called when the ray didn't hit any geometry.
In the intro video, the miss shader returns a hardcoded dark blue color, while the closest hit shader returns the barycentric coordinates of the place where the ray intersects the individual triangle.

That's it for the current state of the `heatx` demo. 
My plan for ray tracing inside the engine is to have it as an optional component, probably for shadows.
I took a look at what's needed next to do ray-traced shadows and figured out that the raw vertex data should also be available on the GPU.

Currently, I don't have this data loaded in a way that would be compatible for accessing it from the hit/miss shaders, so I'll be looking into that.
The idea is that if I implement [programmable vertex pulling](https://voxel.wiki/wiki/vertex-pulling/) for the `gltfviewer` demo, it will magically resolve this issue too.

# Deletion queues
A couple of wishes from the previous posts:
> I'll have to do some experiments on how to do this, most likely using [deletion queues](https://vkguide.dev/docs/chapter-2/cleanup/) or something similar. 

> That should also be valuable for handling swapchain resizing without resorting to sledgehammer approaches using `vk(Device|Queue)WaitIdle` functions.

> I still need to implement the nonblocking resize for the `gltfviewer` and `galileo` demos.

This works now! The engine now supports a deterministic way of releasing resources once the frame has been rendered on the GPU.

The cyclic structure of [frames in flight](https://vulkan-tutorial.com/Drawing_a_triangle/Drawing/Frames_in_flight) now has a `cleanup` queue.
```
    struct [[nodiscard]] frame_in_flight_t final
    {
        uint32_t index{};

        VkFence submit_fence{VK_NULL_HANDLE};

        boost::container::deque<std::function<void()>> cleanup;
        boost::container::deque<std::function<void()>> old_cleanup_queue;
    };
```
Resources that are used in commands submitted to the GPU, but no longer needed after the current frame is executed, can be pushed to `cleanup` queue.
The cleanup functions are invoked once the frame has been acquired again.

There are still some intricacies of smooth resizing on Windows remaining that I'll handle once I find the will for it.

# Development update
When I went through the old demos of the engine to make a video for the previous post, I did have a bit of "fun" trying to match the commit that should be used for the [conan-recipes](https://github.com/jan-kelemen/conan-recipes) repository that contains Conan recipes of third party dependencies
with the actual demo that I tried to compile.

To avoid this issue in the future, I've decided to include the `conan-recipes` repository as a git subtree into the niku repository.
This way, the correct version of Conan recipes to use when building the engine is always in the engine source code under [conan/index](https://github.com/jan-kelemen/niku/tree/master/conan/index).

The problem could have been avoided by maintaining the recipes in a backwards compatible way, tagging them or something else, but I don't really care about it that much.

# Final words
If someone knows how to convert a huge SVN repository to Git in a history preserving way, please let me know.
I'm looking at converting a repository with ~40k revisions and my attempts so far weren't really successful. I would like to avoid just making incremental copies of the trunk.

For any questions or corrections to this post, leave a comment in [discussion thread](https://github.com/jan-kelemen/niku/discussions/1) on GitHub or any other place where you see this.

[Diff](https://github.com/jan-kelemen/niku/compare/950c779d77bcf89e532d9f0212565b4b2afc7311...df1a696b72ba63fb6d16131adad66afb04b18dc7) compared to the state shown in the previous post.

Check out other posts in this series with [#niku](https://jan-kelemen.github.io/archive.html#niku) tag.

