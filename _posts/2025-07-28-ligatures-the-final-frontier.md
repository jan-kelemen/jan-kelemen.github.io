---
title: Ligatures... The Final Frontier
author: Jan Kelemen
tags: [C++, Vulkan, graphics programming, freetype, harfbuzz, niku]
---

Text rendering update for my game engine [jan-kelemen/niku](https://github.com/jan-kelemen/niku), now without missing glyphs.

<iframe width="1000" height="562" src="https://www.youtube.com/embed/M0Soxc6BoRM" title="niku - Ligatures... The Final Frontier" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

For the demo video, I've increased the font size; YouTube renders it a bit better than the previous one.

# The missing glyph
In the previous [post](/2025/06/04/text-hates-us.html), I've shown that the rendering of variable spaced fonts is mostly working.
The missing part was ligatures, when multiple letters from the text are combined into a single glyph. 

A reminder with the missing `fl` ligature:
{:refdef: style="text-align: center;"}
![ligatures](/assets/posts/20250604/2025-06-04-missing-ligature.png)
{:refdef}

Initially, I've loaded the glyph bitmaps for ASCII characters into an image that is sampled by the fragment shader.
```
layout(binding = 0) uniform sampler2D texSampler;
...
    outColor = vec4(inTexColor.rgb,
        texelFetch(texSampler, ivec2(inTexCoord.x, inTexCoord.y), 0).r);
```

With the presence of ligatures in the font file, ligature glyphs aren't included in this initial bitmap, so they need to be loaded and made available to the GPU.
The new glyph bitmap can't be simply added to the image that is already loaded on the GPU, that would be a race condition between the CPU that is preparing the next frame to be rendered and the GPU that's rendering the current frame.

Let's create a new image that contains the missing glyphs, and change the fragment shader to use multiple bitmap samplers:
{:refdef: style="text-align: center;"}
![ligatures](/assets/posts/20250728/2025-07-28-ligature.png)
{:refdef}

```
layout(location = 2) in flat uint inBitmapIndex;

layout(binding = 0) uniform sampler2D texSamplers[];
...
    outColor = vec4(inTexColor.rgb,
        texelFetch(texSamplers[nonuniformEXT(inBitmapIndex)], ivec2(inTexCoord.x, inTexCoord.y), 0).r);
```

The `texSamplers` resource array is defined by a descriptor layout and the descriptor set.
The descriptor set describes which image is stored at which index in the array.

Creation of the descriptor set layout requires the size to be specified in [VkDescriptorSetLayoutBinding](https://registry.khronos.org/vulkan/specs/latest/man/html/VkDescriptorSetLayoutBinding.html). 
However, we don't know how many glyph images we'll have.

First option would be to recreate the descriptor set (and the layout) each time a bitmap image is added to the set, but this is clunky.

Adding `VK_DESCRIPTOR_BINDING_VARIABLE_DESCRIPTOR_COUNT_BIT` to the descriptor layout flags allows creation of descriptor sets from the same layout with different array sizes, in which case the `descriptorCount` is only used as an upper bound on the size.
The descriptor set would still need to be recreated. The alternative is to add `VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT`, which allows unallocated descriptors in the array to be present, for example, `[sampler1, sampler2, null]` if we allocate a descriptor set with 3 descriptors.

While we're adding flags, let's add `VK_DESCRIPTOR_BINDING_UPDATE_UNUSED_WHILE_PENDING_BIT` too. This flag allows the descriptor set to be updated while some parts of it are used in parallel.
If a shader invocation only uses samplers on indices 0 and 1, we can update the resource descriptor on index 2 in the meantime.

Now the bitmap images can be dynamically added. Final tweak, it appears that there are quite a few devices that have a fairly [low limit](https://vulkan.gpuinfo.org/displaydevicelimit.php?name=maxPerStageDescriptorSamplers) on the number of samplers in a shader stage.
Since all of the bitmap images can be sampled with the same sampler, we change it to N sampled images + sampler instead of N combined image samplers.
```
layout(binding = 0) uniform sampler texSampler;
layout(binding = 1) uniform texture2D texImages[];
...
    outColor = vec4(inTexColor.rgb,
        texelFetch(sampler2D(texImages[nonuniformEXT(inBitmapIndex)], texSampler), ivec2(inTexCoord.x, inTexCoord.y), 0).r);
```

This schema is, of course, wasteful and it would be better to merge these individual images after some time. That is a topic for a different day.

# Multiple windows
The next most reasonable thing to do was to add support for multiple windows, as that's a mandatory feature for all game engines.

Well, not really, but I did have a feeling that it would force me to improve upon parts of the code that were written quite a while ago and help towards the goal of making the event handling better.

To render multiple windows at once from the same application, aside from creating the native platform window, we need to create a surface (`VkSurfaceKHR`) and the swapchain (`VkSwapchainKHR`) for each window.

The surface is created by the windowing system integration, in this case SDL, so this isn't much of an issue. Except for one curious case, the surface is also needed to select a GPU device suitable for rendering, as it's required to query the capabilities of the swapchain.
Here, I've decided to go with the method of hopes and prayers; the surface of the first created window is used to pick a suitable device. Hopefully, the device selected in this manner will be compatible with the requirements of other surfaces created after the initial one.

Having that out of the way, the swapchain can be created for the window. 
My previous design of the `vkrndr::backend_t` class needed some refactoring, as it previously handled device initialization and swapchain creation.
Now with the device being shared between multiple windows and independent rendering loops, no longer fit there.

This was an opportunity to make the device feature selection a bit more generic, and was probably the first time that I've written code which accesses members of a struct through a member pointer:
```
auto const set_flags = [](auto&& instance, std::ranges::range auto&& range)
{
    return std::ranges::for_each(range,
        [&instance](auto const value) { instance.*value = VK_TRUE; });
};
```

Enabling device features in Vulkan requires setting members of a couple of structs, and I didn't want to do this for ~20ish different members by hand in various combinations.
```
set_flags(chain.device_10_features.features, flags.device_10_flags);
set_flags(chain.device_11_features, flags.device_11_flags);
set_flags(chain.device_12_features, flags.device_12_flags);
set_flags(chain.device_13_features, flags.device_13_flags);
```

The `device_*_flags` variables are a list of features that need to be enabled, while the `device_*_features` are feature structs like [VkPhysicalDeviceVulkan12Features](https://registry.khronos.org/vulkan/specs/latest/man/html/VkPhysicalDeviceVulkan12Features.html), that need to be passed during device creation.

## Changes to the event loop
Even with multiple windows being involved, the concept of the main event loop is pretty much the same:
```
    while (should_run())
    {
        begin_frame();

        SDL_Event event;
        while (SDL_PollEvent(&event))
        {
            handle_event(event);
        }

        for (uint64_t i{}; i != simulation_steps; ++i)
        {
            update(impl_->fixed_update_interval);
        }

        draw();

        end_frame();
    }
```

There is still a global event loop that pushes the events, they are just dispatched to the correct window inside the `handle_event` function.
It does imply however, that the loop can no longer block on the start of the frame until the image from the swapchain is acquired (VSYNC...), since there are independent swapchains now.
For now, I'm continuously polling for the available image without a timeout. 

The downside is that the CPU is running a busy loop most of the time.
This should probably be resolved by having a thread per window, but multithreading was out of the question for the initial implementation.

# Final words
Taking a short look back at the problem of combining multiple ligature bitmaps.
Let's say we want to combine images for ligatures `fl` and `ft` into one.
We can produce a new image containing these two at any time, but to safely delete the old ones, all rendering tasks submitted using them should be finished before swapping them out.
I don't expose this information out of the swapchain abstraction `vkrndr::swapchain_t`, and the responsibilities of that class are not yet 100% correct.
I'll have to do some experiments on how to do this, most likely using [deletion queues](https://vkguide.dev/docs/chapter-2/cleanup/) or something similar. 

That should also be valuable for handling swapchain resizing without resorting to sledgehammer approaches using `vk(Device|Queue)WaitIdle` functions.
Currently, resizing one window blocks the entire execution until the swapchain has been recreated. A similar issue occurs when closing windows.

For any questions or corrections to this post, leave a comment in [discussion thread](https://github.com/jan-kelemen/niku/discussions/1) on GitHub or any other place where you see this.

[Diff](https://github.com/jan-kelemen/niku/compare/28794148b1a2163876e02f53d4026c24a844efcc...4a25fa0ae5b3e97d5126c5293615d9faee8cbf6b) compared to the state shown in the previous post.

Previous posts in the series:
* [Text Hates Us](/2025/06/04/text-hates-us.html)
* [The Invisibles](/2025/04/27/the-invisibles.html)
* [Finding the Way Home](/2025/03/13/finding-the-way-home.html)
* [Navigating Bad Assumptions](/2025/02/07/navigating-bad-assumptions.html)
* [Eppur si muove](/2025/01/11/eppur-si-mouve.html)
