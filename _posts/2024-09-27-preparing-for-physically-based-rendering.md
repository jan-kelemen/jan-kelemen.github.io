---
title: Preparing For Physically Based Rendering
author: Jan Kelemen
tags: [C++, Vulkan, graphics programming, niku, PBR]
---

First official devlog of my game engine [jan-kelemen/niku](https://github.com/jan-kelemen/niku)!

Since I've moved the code to a dedicated repository, I've done some work on the rendering side of things.
The first goal is to get a [Physically Based Rendering (PBR)](https://en.wikipedia.org/wiki/Physically_based_rendering) working in at least somewhat reusable state for other projects.
I started by making a demo application that displays glTF files. The current state of the demo is shown in the video below.

<iframe width="1000" height="562" src="https://www.youtube.com/embed/WdiDkjNU1aw" title="niku - Preparing For Physically Based Rendering" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

It doesn't do any PBR for now, but most of the preparation work is done, I'll start working on the PBR soon™.

All of the sample assets shown in the video are taken from [KhronosGroup/glTF-Sample-Assets](https://github.com/KhronosGroup/glTF-Sample-Assets).

# Refactoring, cleanup and improvements

I started by doing some extensive cleanup of the existing code, which was mostly made for the purposes of individual projects.

Vulkan backend code no longer depends on some third party libraries that didn't belong there like ImGui, freetype or TinyGLTF.
Support for ImGui has been moved into the engine code, and TinyGLTF was replaced by [spnda/fastgltf](https://github.com/spnda/fastgltf).
While I didn't have any issues with TinyGLTF for now, fastgltf was a better fit. 

As far as freetype goes, I'll have to add that back later when I decide to support text rendering again.

I also switched to using the [zeux/volk](https://github.com/zeux/volk) to load the Vulkan libraries instead of using the Vulkan Loader directly.
This was painless and boiled down to changing the include directives and defining `VK_NO_PROTOTYPES` in the build process.
For complete details, you can check the [commit](https://github.com/jan-kelemen/niku/commit/610418c175be13d0c7c490ec0dc06cf65f2781e4).

In summary, the current (direct) dependencies of the engine are:
```
find_package(fmt REQUIRED)
find_package(imgui REQUIRED)
find_package(glm REQUIRED)
find_package(fastgltf REQUIRED)
find_package(SDL2 REQUIRED)
find_package(spdlog REQUIRED)
find_package(stb REQUIRED)
find_package(tl-expected REQUIRED)
find_package(volk REQUIRED)
find_package(vulkan-memory-allocator REQUIRED)
```

Although I plan to remove `tl-expected` as soon as I can update to Clang-19 and replace its usages with the standard equivalent `std::expected`.

One "small" change that also happened is I've decided to use `_t` suffix on all types. 
Up until now, I've believed that it's possible to come up with good names for a type without resorting to suffixes, but I'm also tired of coming up with names for better names for types like `push_constants`, so that they don't clash with variable names.

# New glTF loader

Loading the glTF files has been moved to a [separate library](https://github.com/jan-kelemen/niku/tree/19489b3fc14df1d69e1b876e3849941ac104ec36/src/vkgltf) which is built on top of the backend code.
My goal for this is to load the glTF file to a format that is mostly compatible with what the rendering code requires.
So that the applications that use glTF don't need to do much work on transforming the model for the GPU. 

This currently means that the images are loaded in the correct formats which are needed for PBR, while the vertices and indices are loaded into a transfer buffer that is host visible.
Not yet sure if this is the best approach, but I wanted to skip the step of loading the whole glTF file into the CPU memory and then transferring that over to GPU through a transfer buffer again in the application code.
Since the vertices and indices are loaded to a host visible buffer, the application can still access them if there is a need to do some postprocessing on them before they are transferred over to the actual vertex and index buffers.

I've left the materials on the CPU since the definitions for those are also required on the CPU side and I didn't want to impose a layout of the material buffer up front.

The interface of the loader is similar to the one I've had before:
```
class [[nodiscard]] loader_t final
{
public:
    explicit loader_t(vkrndr::backend_t& backend);

// ...

public:
    [[nodiscard]] tl::expected<model_t, std::error_code> load(
        std::filesystem::path const& path);
```

It just needs a path to the file and returns the loaded asset model in the format explained previously. 
Implementing it like this also means that I can add support for other file types in the future if there is a need for it.

# Normal mapping

One of the graphics techniques that wasn't mentioned in the previous posts is [normal mapping](https://en.wikipedia.org/wiki/Normal_mapping).
Normal mapping can be used to visually improve the look of the rendered image without adding a more detailed vertex geometry. 
Instead the fragment shader, uses a normal map texture to create the illusion of depth.

Below is an example of the lion's head without normal mapping.
{:refdef: style="text-align: center;"}
![lion-unmapped](/assets/posts/20240927/2024-09-27-lion.jpg)
{: refdef}

The intro video on timestamp `0:19` how it looks with normal mapping enabled. 
Since it only creates an illusion, it works best when the mesh is viewed from the front side.
When the viewing angle is steep, the illusion breaks, for example, compare the front facing and the side facing planes of the object on `0:22`, the side facing one is visibly flat.

# Gamma Correction and Tone Mapping
PBR is usually done in the linear color space, while the computer monitors display in non-linear color space.
Conversion between these two is [Gamma Correction](https://en.wikipedia.org/wiki/Gamma_correction).

[Tone Mapping](https://en.wikipedia.org/wiki/Tone_mapping) is a technique of transforming an image in a high dynamic range (HDR), to a low dynamic range, where the color values of individual fragments are in range `[0.0, 1.0]`.
PBR lighting calculations usually exceed the low dynamic range, so the results need to be tone mapped.

I decided to do exposure tone mapping. In the video on timestamp `0:32` you can see how tweaking the gamma factor and the exposure affects the displayed image.

This is currently implemented in a post processing shader which takes in a multisampled background image in linear color space and performs the conversion of color space and range, along with manual resolution of multiple samples to a single one.
It would be a nice candidate to do it only with a compute shader since the conversion does not require any graphics work, but I wanted to try a nice trick of [rendering a fullscreen quad without buffers](https://www.saschawillems.de/blog/2016/08/13/vulkan-tutorial-on-rendering-a-fullscreen-quad-without-buffers/).

When researching how to implement this I found out about [Vulkan's specialization constants](https://blogs.igalia.com/itoral/2018/03/20/improving-shader-performance-with-vulkans-specialization-constants/).
These constants are set when the pipeline shader is created so this approach should be a faster way of transferring data which rarely changes to the shader. 
The number of samples in the multisample image is passed in this way, while the gamma and exposure factors are sent via push constants since they can change in every frame.

```
layout (constant_id = 0) const int SAMPLES = 8;

layout(push_constant) uniform PushConsts {
    float gamma;
    float exposure;
} pc;
```

# Final words
As usual, the neverending list of TODO tasks required for this project was expanded. 
I'm a bit more satisfied with the backend code now that it deals only with Vulkan related things, but I'm not yet happy with how it handles device selection and descriptor pools.
The code in the demo project is a mess and will require some optimizations and refactoring after getting the basics of PBR working.

![pbr-equation](/assets/posts/20240927/2024-09-27-pbr.jpg)

For any questions or corrections to this post, leave a comment in [discussion thread](https://github.com/jan-kelemen/niku/discussions/1) on GitHub or any other place where you see this.

Previous posts in the series:
* [I See Spheres Now](/2024/08/25/i-see-spheres-now.html)
* [Rendering Medvednica From a Heightmap](/2024/08/10/rendering-medvednica-from-heightmap.html)
* [It Goes In The Square Hole](/2024/07/05/it-goes-in-the-square-hole.html)
* [Chess engine visualization](/2024/06/13/chess-engine-visualization.html)
* [Remaking World 1-1 with Vulkan, mostly](/2024/05/14/remaking-world-11-mostly.html)
