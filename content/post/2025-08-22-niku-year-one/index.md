---
author: Jan Kelemen
date: '2025-08-22T00:00:00Z'
tags:
- C++
- Vulkan
- niku
aliases:
- /2025/08/22/niku-year-one.html
title: 'niku: Year One'
---

After working for a year on my game engine [jan-kelemen/niku](https://github.com/jan-kelemen/niku), time has come to publish an anniversary release [0.1](https://github.com/jan-kelemen/niku/releases/tag/0.1).

{{< youtube "7eTE_94SNc0" >}}

# What was done?
The engine development is primarily done with demo applications that reuse the same codebase. Currently, there are three of them:
* `gltfviewer` - Forward 3D renderer for glTF files implementing a Physically Based Rendering workflow
* `galileo` - Deferred 3D renderer with physics and scripting
* `reshed` - Text editor with bitmap font rendering and GLSL syntax highlighting

The complete list of features for each is shown in the intro video.

I quite like this approach as it allows me to figure out how things should work before promoting them into the core parts of the engine.
Having a wildly different set of features between the demos also helps with making sure I don't make the core parts too restrictive.

# How did we get here?

{{< youtube "suRviCycwFk" >}}

It was raining on a Sunday. 
That was pretty much the moment when I decided to open the [Vulkan Tutorial](https://vulkan-tutorial.com), having almost no prior experience with graphics or game development.

Following that rainy night (well, actually it took me a week to go through the tutorial), I continued by doing graphics demos until I had gotten the basic architecture of the code to a stable state.
The video above features previously unseen footage of [vkpong](https://github.com/jan-kelemen/vkpong) and [vkchip8](https://github.com/jan-kelemen/vkchip8) demos.

I've gotten familiar with glTF loading and basic lighting model with [pawn](/post/2024-06-13-chess-engine-visualization).
Learned how to use a physics engine with [geos](/post/2024-07-05-it-goes-in-the-square-hole) and terrain rendering with [soil](/post/2024-08-10-rendering-medvednica-from-heightmap).

I've also did a compute shader raytracer with [beam](/post/2024-08-25-i-see-spheres-now), which unfortunately I wasn't able to capture on video now. Recording it with OBS didn't want to cooperate and the footage ended up jittery.

# Contributing back to open source
I don't want to reinvent the wheel completely for this project, so the engine requires quite a [few](https://github.com/jan-kelemen/niku/blob/950c779d77bcf89e532d9f0212565b4b2afc7311/COPYRIGHT.md#third-party-attributions) open source libraries.

At the time of writing, working on `niku` has directly or indirectly contributed to:
* 19 package versions to [Conan Center](https://conan.io/center)
* Bugfixes to [SPIRV-Tools](https://github.com/KhronosGroup/SPIRV-Tools), [glslang](https://github.com/KhronosGroup/glslang), [fastgltf](https://github.com/spnda/fastgltf), [Vulkan-ValidationLayers](https://github.com/KhronosGroup/Vulkan-ValidationLayers) and [HarfBuzz](https://github.com/harfbuzz/harfbuzz)

Along with some issues being reported to other libraries.

It's funny, in the years prior to this, I was looking for open source projects to contribute to. I guess it's true to simply contribute to projects you are actually using.

# Development update
Some [time](/post/2025-02-07-navigating-bad-assumptions) has passed since I started to implement support for the [VK_EXT_swapchain_maintenance1](https://docs.vulkan.org/features/latest/features/proposals/VK_EXT_swapchain_maintenance1.html) Vulkan extension.

I finally got around to implementing the rest of the features provided by the extension.
The swapchain can now be recreated by reusing the old one and the swapchain images use a deferred allocation.
These changes should allow for a smoother experience when resizing the windows and a deterministic way of knowing when to release GPU resources used in commands that have been submitted to the GPU.

The code for this is mostly based on the official [Swapchain Recreation](https://github.com/KhronosGroup/Vulkan-Samples/tree/main/samples/api/swapchain_recreation) sample.
I won't go into much detail here, but it almost made me draw UML sequence diagrams, probably for the first time in many years.

I still need to implement the nonblocking resize for the `gltfviewer` and `galileo` demos.

# Final words
To be fair, according to the modern definition of the term `game engine`, `niku` would currently be classified as a `game framework`.
I should probably add support for audio and make an editor for it. 
Though the editor part requires some experimentation on my side on how it should work.

Thanks to my friends who discussed development topics with me, and to folks from [Vulkan Discord](https://discord.com/invite/vulkan) and [Graphics Programming](https://discord.com/invite/graphicsprogramming) for help along the way.

For any questions or corrections to this post, leave a comment in [discussion thread](https://github.com/jan-kelemen/niku/discussions/1) on GitHub or any other place where you see this.

[Diff](https://github.com/jan-kelemen/niku/compare/4a25fa0ae5b3e97d5126c5293615d9faee8cbf6b...950c779d77bcf89e532d9f0212565b4b2afc7311) compared to the state shown in the previous post.

Check out other posts in this series with [#niku](/tags/niku) tag.

