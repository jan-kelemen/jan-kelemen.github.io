---
title: Eppur si mouve
author: Jan Kelemen
tags: [C++, Vulkan, graphics programming, niku]
---

New year, new demo application for my game engine [jan-kelemen/niku](https://github.com/jan-kelemen/niku).
The new demo application is named `galileo`.
Its continuing mission to integrate physics simulation and scripting language support into the engine.

I also took the opportunity to clean up some things. 
Here is the [diff](https://github.com/jan-kelemen/niku/compare/3adf002453d55a2b654402def252812185982417...760b7e6231d363328396b3eeb9a3562b8041ad60) compared to the state shown in the previous post.

<iframe width="1000" height="562" src="https://www.youtube.com/embed/PhTNq3U2Q8M" title="niku - Eppur si muove" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

# Physics
I went over the basics of using a physics engine in [It Goes In The Square Hole](/2024/07/05/it-goes-in-the-square-hole.html) and [Rendering Medvednica From a Heightmap](/2024/08/10/rendering-medvednica-from-heightmap.html).
Compared to that post I've decided to switch from using the [Bullet Physics SDK](https://github.com/bulletphysics/bullet3) to [Jolt Physics](https://github.com/jrouwe/JoltPhysics).

The main motivation behind making the switch was that the maintenance status of Bullet is a bit questionable at the moment. 
Another alternative would have been NVIDIA's PhysX, although I'm quite certain it would be a pain to compile on GitHub's CI.

In general, I would recommend starting out with Bullet if you're unfamiliar with physics engines as it's simpler to use and compile.

## Debug renderer
One of the nitpicks I had with Bullet's debug renderer was its callback interface definition.
It has no concept of a mesh and no way to filter which lines belong to some body.
Therefore it requires the whole vertex buffer for the debug lines to be updated each frame.

In comparison, Jolt's [DebugRenderer](https://jrouwe.github.io/JoltPhysics/class_debug_renderer.html) is a bit more involved to implement:
```
void DrawLine(...) override;
void DrawTriangle(...) override;
Batch CreateTriangleBatch(...) override;
void DrawGeometry(RMat44Arg inModelMatrix, const AABox &inWorldSpaceBounds, ...) override;
void DrawText3D(...) override;
```

It has a concept of triangle batches. A triangle batch is a mesh of the physics collision object created for all levels of detail for the mesh.
This batch can be put into a vertex buffer on the GPU without the need to update each frame.

Another nice thing about Jolt's definition is that the `DrawGeometry` callback has an AABB and the model matrix parameters.
These should allow for frustum culling to exclude the meshes that aren't visible in the rendered frame.

However, I was lazy and implemented the whole debug renderer as a batch renderer, which updates the vertex buffer for each frame. 
`DrawGeometry` and `DrawTriangle` methods can be implemented in terms of `DrawLine`, so implementing a debug renderer for Jolt isn't much more complicated than Bullet.

## Contact listeners
To detect a collision between a rigid body in the world and a character a contact listener has to be registered with Jolt.
```
class character_contact_listener_t final : public JPH::CharacterContactListener
{
    void OnContactAdded(JPH::CharacterVirtual const* inCharacter,
        JPH::BodyID const& inBodyID2, ...) override;
};
```

When a character collides with a body, `OnContactAdded` will be called. 
Through this I've implemented the possibility for the character to push a sphere, by adding a force to the colliding object:
```
        auto& interface{physics_engine_->body_interface()};
        interface.AddForce(inBodyID2,
            inContactNormal * inCharacter->GetMass() * 100,
            inContactPosition);
```

Another functionality implemented with the contact listener is the collision with the box in the world, colliding with the box spawns a new sphere on random coordinates.

# Scripting
I've decided to try using [AngelScript](https://www.angelcode.com/angelscript/) as a scripting language. Spawning a sphere is done by executing a script:
```
void main()
{
  spawn_sphere();
}
```

`spawn_sphere()` is a C++ function which is registered as a global function in the scripting engine:
```
scripting_engine_.engine().RegisterGlobalFunction(
    "void spawn_sphere()",
    asMETHOD(application_t, spawn_sphere),
    asCALL_THISCALL_ASGLOBAL,
    this)};
```
Executing this function adds a new sphere to the physics world. 
The contact listener has a cooldown period of 5 seconds so that the script is not executed on each collision with the box.

My experience with AngelScript for now is fairly limited, but I've seen it used in a couple of games and think it will be a viable choice.
So far, it seems that it does not leak memory and the interpreter can be restarted, I'm happy with that already.

I was considering using Lua with Sol3 bindings, though the Sol3 bindings are currently a bit inactive with the maintenance.
I'm not a huge fan of 1-based indexing either.
Writing my bindings around more C APIs is not a goal currently, so I went with an alternative. 
AngelScript on the other hand has native C++ bindings and 0-based indexing.

# Deferred rendering
The `gltfviewer` demo uses a forward renderer, for `galileo` I wanted to write a [deferred renderer](https://learnopengl.com/Advanced-Lighting/Deferred-Shading).
Most important difference between these two approaches is the amount of calculations that need to be performed with each light.
When using forward rendering, worst case complexity is `O(meshes) * O(lights)`, while with the deferred approach, it's `O(meshes) + O(lights)`.

This is achieved by first rendering the whole scene in a geometry pass to the g-buffer.
The g-buffer consists of multiple color attachments corresponding to attributes required for the lighting pass done in the second step.
For this demo, the g-buffer contains position, normals and albedo color attachments:
```
layout(location = 0) out vec3 outPosition;
layout(location = 1) out vec3 outNormal;
layout(location = 2) out vec4 outAlbedo;
```

A full screen lighting pass is then performed for all lights sampling from the previously written color attachments to the target image.
```
const vec3 position = texture(positionTexture, inUV).rgb;
const vec3 normal = texture(normalTexture, inUV).rgb;
const vec3 albedo = texture(albedoTexture, inUV).rgb;

vec3 lighting = albedo * 0.1;
for(int i = 0; i < frame.lightCount; ++i) { ... }

outColor = lighting;
```

The lighting model in this demo is intentionally bare bones, containing only ambient lighting and point lights.
The benefits of using deferred rendering are demonstrated at the end of the intro video.
On my hardware, I can render 1000 point lights and still have stable 60 FPS.

## Fast approximate anti-aliasing (FXAA)
Deferred rendering does come with a memory cost and requires `O(attributes) + 1` images to render the scene.
This makes multisample anti-aliasing impractical as MSAA requires even more memory per image and increases the computational cost of the lighting pass.

Therefore I chose to use FXAA as the anti-aliasing solution for this demo. 
Thanks to Timothy Lottes and NVIDIA implementation of FXAA shader code is publicly available under a permissive license.
This [blog post](https://blog.codinghorror.com/fast-approximate-anti-aliasing-fxaa/) compares FXAA and MSAA approaches 

# Other news
While on the topic of expensive fragment shaders, I implemented a [depth-only prepass](https://www.youtube.com/watch?v=2ZK527cQbeo) to the rendering pipeline in the `gltfviewer`.
The idea behind this pass is to reduce the number of times the fragment shader is invoked by doing depth testing. 
This pass includes only the opaque geometry of the scene, similarly to deferred rendering it's also a trade-off as it requires an additional traversal of the scene.

# Final words
This demo is still a work in progress, one obvious improvement would be to add jumping to the character controller and smarter ways of interacting with the world. 
My knowledge of how to use Blender is lacking, I guess I'll finally have to go through the whole donut tutorial.

For any questions or corrections to this post, leave a comment in [discussion thread](https://github.com/jan-kelemen/niku/discussions/1) on GitHub or any other place where you see this.

Previous posts in the series:
* [Sidequests And Loose Ends](/2024/11/25/sidequests-and-loose-ends.html)
* [Rendering Physically Based Bugs](/2024/10/26/rendering-physically-based-bugs.html)
* [Preparing For Physically Based Rendering](/2024/09/27/preparing-for-physically-based-rendering.html)
* [I See Spheres Now](/2024/08/25/i-see-spheres-now.html)
* [Rendering Medvednica From a Heightmap](/2024/08/10/rendering-medvednica-from-heightmap.html)
* [It Goes In The Square Hole](/2024/07/05/it-goes-in-the-square-hole.html)
* [Chess engine visualization](/2024/06/13/chess-engine-visualization.html)
* [Remaking World 1-1 with Vulkan, mostly](/2024/05/14/remaking-world-11-mostly.html)
