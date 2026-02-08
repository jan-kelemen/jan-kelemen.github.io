---
author: Jan Kelemen
date: '2025-02-07T00:00:00Z'
tags:
- C++
- Vulkan
- graphics programming
- niku
- VK_EXT_swapchain_maintenance1
aliases:
- /2025/02/07/navigating-bad-assumptions.html
title: Navigating Bad Assumptions
---

A regular update on the state of my game engine [jan-kelemen/niku](https://github.com/jan-kelemen/niku).
Some past mistakes were corrected, and hopefully, no new ones were made.

{{< youtube "fWpssvL9gOc" >}}

# Asset loading 2: Electric Boogaloo
The last time I implemented loading of glTF files, I loaded the model geometry to GPU buffers that can also be accessed from the CPU before finally transferring it to dedicated GPU buffers.
This approach worked well for `gltfviewer` as it didn't need to access the geometry. 

I ran into some drawbacks when adding the physics as the world geometry needs to be loaded into the physics engine.
The second drawback was that the model loading had a direct dependency on the graphics code making it less reusable.

The new approach to loading the glTF files is to first load everything to the CPU memory and let the application code decide when/if/what needs to be transferred to the GPU.

## Navigation mesh
Having blue spheres that spawn out of nowhere is fun, but since the world of `galileo` consists of simple geometry, it seemed like a good idea to try integrating the generation of navigation meshes into the demo too.

After some investigation, I went with the [Recast Navigation](https://recastnav.com) library.
This was a very short investigation, as Recast is the only one that's open source and can be integrated easily into a custom engine.

Navigation mesh generation was the other reason why I needed the world geometry to be easily accessible by the CPU.

For now, I've followed the example code provided by the Recast library to generate the navigation mesh and add a debug renderer for it.
The process involves rasterizing the world mesh into individual cells. 
A more detailed overview of how this looks like in the Unreal Engine which also uses Recast is available [here](https://www.unrealdoc.com/p/navigation-mesh).

The intro video on timestamp `0:30` shows what the rendering of a `rcPolyMesh` looks like. 
Light blue color marks the areas that are walkable. This view is not that useful for now as there are no non-walkable areas like water.

The next thing demonstrated in the video is the rendering of `rcPolyMeshDetail`. 
This one matches the world geometry more closely and shows the effect of some parameters that can be tuned for the mesh generation process.

Although the video shows the mesh generation process is done in real time, it should be done ahead of time.
Even for this simple case, it takes around 60 milliseconds to generate it from scratch with the default parameters. 
It does depend highly on the settings as it can go into the 10 second range with smaller cell sizes.

I've used the same parameters as they are in the Recast example, they work nicely for the world proportions that I use, where the character is a bit less than 2 units tall.

# Dynamic present mode change
In Vulkan, the images are presented to the screen through a swapchain. 
When the swapchain is created a present mode for the images is also specified. 
The present mode controls [vertical synchronization](https://en.wikipedia.org/wiki/Screen_tearing#Vertical_synchronization) with the screen refresh rate.
There are 4 basic [present modes](https://registry.khronos.org/vulkan/specs/latest/man/html/VkPresentModeKHR.html) that match Vsync on/off and if image tearing is allowed or not. 

I usually run the demo applications with Vsync turned on (`VK_PRESENT_MODE_FIFO_KHR`) to avoid running the graphics card at full usage all of the time.
Sometimes, I want to check if the changes I made to the code negatively affect performance, so I would have to change the present mode of the swapchain.

There was an issue with this, I had the used present mode hardcoded, therefore changing it would imply recompiling the code.

To support changing the present mode without recompiling the swapchain needs to be recreated.
Recreating the swapchain is a basic operation that all renderers have to support, as it is common to recreate it when the window is resized.
It's still a bit of an expensive operation for only changing the Vsync state.

There is a Vulkan extension [VK_EXT_swapchain_maintenance1](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_swapchain_maintenance1.html).
One of the issues this extension addresses is changing the present mode without recreating the swapchain.

To change the present mode, `VkSwapchainPresentModeInfoEXT` needs to be specified when issuing the present command on the swapchain.
```cpp
VkSwapchainPresentModeInfoEXT const present_mode_info{
    .sType = VK_STRUCTURE_TYPE_SWAPCHAIN_PRESENT_MODE_INFO_EXT,
    .swapchainCount = 1,
    .pPresentModes = &desired_present_mode_};
```
The desired present mode would then be applied for the current and subsequent images presented on the swapchain (until the next change).

That was the easy part. The `VK_EXT_swapchain_maintenance1` extension depends on two others `VK_KHR_get_surface_capabilities2` and `VK_EXT_surface_maintenance1`, they need to be enabled too.
If any of these can't be enabled the code needs to fall back to recreating the swapchain.

Finally, the current present mode needs to be compatible with the one, if they aren't compatible, the implementation should fall back to recreating the swapchain.

In the end, dynamically changing the present mode now works, as shown in the intro video on timestamp `1:17`, observe the FPS counter on the ImGui window.
At one point I'll have to implement the rest of the functionality provided by the swapchain extension.

## Spreading the disease
On the topic of changes in `gltfviewer`, I fixed a nasty bug. The black pixel in the middle of the image below is a pixel with `NaN` value.
![NaN disease 1](disease1.jpeg#center)

It happens, there was a division by 0 in the PBR shading code. The issue wouldn't be obvious without the blur shader doing its thing.
![NaN disease 2](disease2.jpeg#center)

That one pixel would sometimes end up being sampled by the blurring process. 
As the blur shader essentially averages an area of pixels and doing any operations with `NaN` value also results in a `NaN` value, the result is a flickering black square.

Initially, I thought this was a synchronization issue, but after checking and fixing all of the memory barriers the issue persisted.
After being annoyed by it for several days I finally figured out that it happens when looking at the model from specific angles.
That made it easier to capture in RenderDoc and fix it. 

# Other news
Vulkan 1.4 specification has been released and NVIDIA has released a driver which supports it. 
While I've updated all of the third party dependencies to support the new version, niku will continue to target the Vulkan 1.3 version for now.
It is a pragmatic choice between using the latest features and having a broader platform and tooling support. 

Expecting the release of [SDL3](https://www.libsdl.org/index.php), I was conservative with the actual usage of SDL in the code.
A stable version of SDL3 was also released recently.
I'm quite sure that event handling could use some improvements, so maybe the time has come to improve on that after I update from SDL2.

# Final words
Although the static maze in the world is an amazing addition, with physics, scripting, and navigation meshes now in place,
it would be reasonable to make the world a bit more interactive unless I get sidetracked by something else.

For any questions or corrections to this post, leave a comment in [discussion thread](https://github.com/jan-kelemen/niku/discussions/1) on GitHub or any other place where you see this.

[Diff](https://github.com/jan-kelemen/niku/compare/760b7e6231d363328396b3eeb9a3562b8041ad60...28c9c89b7b7dece34280d0a6b9181f38ce68f265) compared to the state shown in the previous post.

Previous posts in the series:
* [Eppur si muove](/post/2025-01-11-eppur-si-mouve)
* [Sidequests And Loose Ends](/post/2024-11-25-sidequests-and-loose-ends)
* [Rendering Physically Based Bugs](/post/2024-10-26-rendering-physically-based-bugs)
* [Preparing For Physically Based Rendering](/post/2024-09-27-preparing-for-physically-based-rendering)
* [I See Spheres Now](/post/2024-08-25-i-see-spheres-now)
* [Rendering Medvednica From a Heightmap](/post/2024-08-10-rendering-medvednica-from-heightmap)
* [It Goes In The Square Hole](/post/2024-07-05-it-goes-in-the-square-hole)
* [Chess engine visualization](/post/2024-06-13-chess-engine-visualization)
* [Remaking World 1-1 with Vulkan, mostly](/post/2024-05-14-remaking-world-11-mostly)
