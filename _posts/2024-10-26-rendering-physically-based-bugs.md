---
title: Rendering Physically Based Bugs
author: Jan Kelemen
tags: [C++, Vulkan, graphics programming, niku, PBR]
---

I continued the work on the [PBR](https://en.wikipedia.org/wiki/Physically_based_rendering) rendering demo for my game engine [jan-kelemen/niku](https://github.com/jan-kelemen/niku). 
Although I thought most of the preparation work was already done, I guess it's never enough for things like this.

Here is a complete [diff](https://github.com/jan-kelemen/niku/compare/19489b3fc14df1d69e1b876e3849941ac104ec36...aa9c25f3440eee946501ec95de4dc155bebd00f2)
between the state shown previously and as it is now. 
The video demonstrates a couple of scenes and assets shown previously.
Illumination is done with point lights using linear attenuation.

<iframe width="1000" height="562" src="https://www.youtube.com/embed/288VO4wo24M" title="niku - Rendering Physically Based Bugs" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

I switched to using [PBR Neutral](https://github.com/KhronosGroup/ToneMapping/tree/main/PBR_Neutral) tone mapping and real linear to sRGB conversion instead of Reinhard tone mapping and gamma correction.
They look better, [here](https://graphics-programming.org/resources/tonemapping/index.html) is a nice comparison of various tone mapping functions.

As before all sample assets shown are from [KhronosGroup/glTF-Sample-Assets](https://github.com/KhronosGroup/glTF-Sample-Assets).

# Implementing your own PBR renderer
A couple of notes for implementing your own PBR renderer for glTF:
* [Normal texture](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html#_material_normaltexture) and the [emmisive texture](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html#_material_emissivetexture) should be loaded as they are in the sRGB color space, the others are in linear color space
* If a texture like [metallic-roughness](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html#_material_pbrmetallicroughness_metallicroughnesstexture) does not exist, that doesn't imply the [metallic](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html#_material_pbrmetallicroughness_metallicfactor) or [roughness](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html#_material_pbrmetallicroughness_roughnessfactor) factors won't be defined
* Make your shaders easy to debug, either by having switches for viewing the inputs and outputs for parts of the PBR equation or rendering them to multiple color attachments

And finally, pick one implementation as the basis for checking the correctness. [LearnOpenGL](https://learnopengl.com/PBR/Theory) or [SaschaWillems/Vulkan-glTF-PBR](https://github.com/SaschaWillems/Vulkan-glTF-PBR) are great starting points.
Google has also published good [documentation](https://google.github.io/filament/Filament.html) on the topic for their [Filament](https://github.com/google/filament) project.

# Rendering backend improvements
The rendering backend code held up quite nicely this time, not much refactoring was needed. 
I added support for [cubemaps](https://en.wikipedia.org/wiki/Cube_mapping) and exposed a way to perform one-off tasks.

One-off tasks use an RAII type which is tied to the `VkQueue` (`execution_port_t` in my implementation) on which it executes:
```
class [[nodiscard]] transient_operation_t final
{
public:
    transient_operation_t(execution_port_t& port, command_pool_t& pool);
...
public:
    ~transient_operation_t();
...
public:
    [[nodiscard]] constexpr VkCommandBuffer command_buffer() const noexcept;
};
```

The destructor submits the work to the GPU and waits for it to finish, here is a small sample of how it is used:
```
void vkrndr::backend_t::transfer_buffer(buffer_t const& source,
    buffer_t const& target)
{
    auto transient{request_transient_operation(false)};
    copy_buffer_to_buffer(transient.command_buffer(),
        source.buffer,
        source.size,
        target.buffer);
}
```
The parameter in the `request_transient_operation` specifies if it will be tied to a queue with graphical capabilities or just the transfer queue.

I've also extended the graphics pipeline builder with the support for multiple color attachments, so far I've used it only for debugging purposes. 
It will be more useful for implementing [bloom effect](https://en.wikipedia.org/wiki/Bloom_(shader_effect)) in the future.

# There and back again
This bug was particularly nasty and didn't want to go away even after several attempts at it.
I would fix it for one sample asset that broke something else, and so on, the cycle repeats.

The image shows my generated shading normals of the [Damaged Helmet](https://github.com/KhronosGroup/glTF-Sample-Assets/blob/main/Models/DamagedHelmet/README.md) model:
{:refdef: style="text-align: center;"}
![damaged-helmet-normals](/assets/posts/20241026/2024-10-26-normals.jpeg)
{:refdef}

In normal maps, the normals are in the tangent space coordinate system, while I've decided to keep my PBR calculations in world space for simplicity.
So one of the approaches for converting from tangent space to normal space is by partial derivatives in the fragment shader.
```
vec3 getNormalFromMap()
{
    vec3 tangentNormal = texture(normalMap, TexCoords).xyz * 2.0 - 1.0;

    vec3 Q1  = dFdx(WorldPos);
    vec3 Q2  = dFdy(WorldPos);
    vec2 st1 = dFdx(TexCoords);
    vec2 st2 = dFdy(TexCoords);

    vec3 N  = normalize(Normal);
    vec3 T  = normalize(Q1*st2.t - Q2*st1.t);
    vec3 B  = -normalize(cross(N, T));
    mat3 TBN = mat3(T, B, N);

    return normalize(TBN * tangentNormal);
}
```
The code above is taken from LearnOpenGL site, but some variation it is commonly used in other implementations online.
Turns out that this does not behave the same with Vulkan as it does with OpenGL, ending up with the result shown above.
By experimentation, this variant seems to give correct looking results:
```
    vec2 st2 = 1 - dFdy(TexCoords);
```
If it works in all cases or just this model, I'm not sure, I still don't quite understand why this is happening, if you know, please let me know.

Thanks to Ilija for helping me debug this one.

Since I still wanted to keep my PBR calculations in world space, I had to recalculate the tangent vectors in a different way.
I've decided to calculate the tangent vectors using the [MikkTSpace](http://www.mikktspace.com) library as suggested by the glTF specification.
This introduces a mild annoyance that it requires the model geometry to be unindexed.
This is fine, but not so much for rendering as it multiplies the vertex count by a factor of 3, so ..., [meshoptimizer](https://meshoptimizer.org) to the rescue!

When the tangents are missing from the model, the loader now creates an unindexed geometry, then calculates the tangents, and indexes it back with meshoptimizer.
I took the opportunity to perform lossless optimizations on the model geometry too, since all of the vertices were loaded to the CPU anyway.

At timestamp `01:18` you can see that it works correctly now. :)

# Interpolation is important
This one was completely my fault, I kept getting a grayish hue on the color but only when looking in one direction. 
Take note of the green curtains on the right side. 
{:refdef: style="text-align: center;"}
![sponza-before](/assets/posts/20241026/2024-10-26-before.png)
{:refdef}

This comes from the specular component of [image based lighting (IBL)](https://en.wikipedia.org/wiki/Image-based_lighting).
Reflections of the environment, or [aviation museum](https://polyhaven.com/a/aviation_museum) if you wish, are done by sampling a blurred version of the skybox image depending on the roughness.
The curtains in Sponza have a roughness factor at which the reflection is sampled from the last mip level, a completely blurred 1x1 pixel cubemap.
Therefore if one part of the model samples from one cube face, and a neighboring part of the model samples from another cube face, there can be a big change in values between them.

I mistakenly forgot to switch on linear filtering on the sampler for this texture. In the grand scheme of things, the difference is small but still visible.
{:refdef: style="text-align: center;"}
![sponza-after](/assets/posts/20241026/2024-10-26-after.png)
{:refdef}

As far as the flowers are concerned, they show a strong specular component too.
After spending hours comparing it with [Khronos' glTF Sample Viewer](https://github.khronos.org/glTF-Sample-Viewer-Release/?model=https://raw.GithubUserContent.com/KhronosGroup/glTF-Sample-Assets/main/./Models/Sponza/glTF/Sponza.gltf) and trying out different versions of the code, I think they are rendered correctly.
It's highly dependent on the environment image and on the math. I'm using a basic variant of IBL lighting so some improvements in this area are for sure possible. 

# Final words
The post didn't go into much detail on the math and shader implementation of PBR, they are currently mostly based on the reference implementations.
Basic stuff is in place, it can be expanded with other light types, directional for example, but directional lights require shadows to look correct, which is why the video uses point lights.

Compiling shaders upfront is becoming a tricky task, there's a lot of possible shader variations, so maybe it's finally time to get [glslang](https://github.com/KhronosGroup/glslang) and [SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross) working.

I also like the idea of adding scripting though. I have some experience in that area and I'm sure it won't be Python.

For any questions or corrections to this post, leave a comment in [discussion thread](https://github.com/jan-kelemen/niku/discussions/1) on GitHub or any other place where you see this.

Previous posts in the series:
* [Preparing For Physically Based Rendering](/2024/09/27/preparing-for-physically-based-rendering.html)
* [I See Spheres Now](/2024/08/25/i-see-spheres-now.html)
* [Rendering Medvednica From a Heightmap](/2024/08/10/rendering-medvednica-from-heightmap.html)
* [It Goes In The Square Hole](/2024/07/05/it-goes-in-the-square-hole.html)
* [Chess engine visualization](/2024/06/13/chess-engine-visualization.html)
* [Remaking World 1-1 with Vulkan, mostly](/2024/05/14/remaking-world-11-mostly.html)
