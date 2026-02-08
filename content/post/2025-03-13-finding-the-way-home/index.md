---
author: Jan Kelemen
date: '2025-03-13T00:00:00Z'
tags:
- C++
- Vulkan
- graphics programming
- niku
aliases:
- /2025/03/13/finding-the-way-home.html
title: Finding The Way Home
---

I didn't intend to be making so much my game engine [jan-kelemen/niku](https://github.com/jan-kelemen/niku) devlog about spheres.
They keep showing up like rocks from [KNOW1NG (2009)](https://www.imdb.com/title/tt0448011/).

{{< youtube "EWkgoKozUJA" >}}

# Pathfinding
Once the navigation mesh of the world is generated, we can use it for its intended purpose, pathfinding between different points of the world.
For this, I've used the other part of the [Recast Navigation](https://recastnav.com) library, Detour, which handles pathfinding and navigation mesh querying.

The basic idea is to:
* Find the point nearest to the current position on the navigation mesh
* Find the point nearest to the target position on the navigation mesh
* Find the shortest (cheapest) path to the target position using an algorithm like [A\*](https://en.wikipedia.org/wiki/A*_search_algorithm)

The search algorithm gives a list of navmesh polygons between two points that need to be traversed.
These polygons are marked with red in the intro video. 
As for the yellow lines, they are nodes of the navmesh query. 

The path between navmesh nodes is then smoothed out by a [string-pulling algorithm](https://idm-lab.org/bib/abstracts/papers/socs20c.pdf). 
This gives a more natural-looking movement, than using the query node positions directly.

In the physics simulation, the spheres are moved along the path by adding impulses to them using a [PID controller](https://en.wikipedia.org/wiki/Proportional–integral–derivative_controller).
The implementation of this controller is very incorrect and untuned, which is one reason why the sphere flies off the path.
It should also detect sharp turns between path points and slow down.

## Scripting actions 
Pathfinding stops when the sphere touches the spawner cube. 
Each sphere now has a scripting object with a `on_hit` action, that is invoked from a physics collision event.
```
class sphere
{
    void on_hit(uint32 other)
    {
        if (is_spawner(other)) 
        {
            stop_pathfinding(id);
        }
    }
}
```

This was a nice opportunity to try out how to create objects in AngelScript and do interactions between different parts of the code using scripting.

## Batch renderer
Developing the navmesh generation and pathfinding required yet another layer of debug renderers.
My previous approach of doing them individually didn't scale anymore. 
Therefore I've refactored the code of the physics debug renderer and the navmesh debug renderer to a shared batch renderer with a simple interface.
```cpp
class [[nodiscard]] batch_renderer_t final
{
    void add_triangle(batch_vertex_t const& p1,
        batch_vertex_t const& p2,
        batch_vertex_t const& p3);

    void add_line(batch_vertex_t const& p1, batch_vertex_t const& p2);

    void add_point(batch_vertex_t const& p1);
```

The debug state can now be rendered through it and drawn on top of the scene geometry.
This renderer currently uses batches of 30000 vertices rendered together in a single draw call.
Why 30000? It's divisible by 3, 2 and 1, the number of points each of the supported topology primitives has and it sounded "large enough" without wasting too much memory if left unused.

I'm sure the buffer allocation scheme could be done in a smarter way. Currently allocates a new buffer for when the batch limit is filled and never releases it.

# Contributing to local warming
While debugging a completely unrelated thing in NVIDIA Nsight, I've noticed in the SPIR-V assembly code that the code below actually does three loads to `lights.v` buffer.
```glsl
const vec3 lightDir = normalize(lights.v[i].position - position);
const vec3 diffuse = max(dot(normal, lightDir), 0.0) * albedo * lights.v[i].color.rgb;
const float distance = length(lights.v[i].position - position);
```

I would have expected it to be optimized away into one load, but for some reason it doesn't, even though the light buffer is readonly.
Doing the trivial optimization by hand generates the code with only one load statement from the buffer.
```glsl
Light light = lights.v[i];

const vec3 lightDir = normalize(light.position - position);
const vec3 diffuse = max(dot(normal, lightDir), 0.0) * albedo * light.color.rgb;
const float distance = length(light.position - position);
```

GPUs don't like waiting for the data, and this was the bottleneck in the rendering pipeline. 
The change allowed for a 100% increase in the limit of lights which can be rendered before dropping below 60FPS.

So, here is the scene rendered with 2000 lights.
![Local warming](local-warming.png#center)

The video was recorded with the previous limit, I figured this out after recording and the GPU doesn't make happy noises using this limit, so I don't suggest using it for a longer period.

# Other news
The migration from SDL2 to SDL3 went smoothly, the procedure is well documented in the [migration guide](https://wiki.libsdl.org/SDL3/README/migration).
In my case, it boiled down to resolving the compiler errors and searching for the replacement usage in the guide.
I've noticed that the API is somewhat cleaner than it used to be. 
SDL in niku is used only for basic window and input handling, so I can't judge how big of an effort is to migrate projects that have a more extensive usage.

On the toolchain side, niku and [cpp-starter-template](https://github.com/melinda-sw/cpp-starter-template) now support compiling with Clang-19 on Windows.
This is a nice step towards ensuring that the code is actually cross-platform as I've supported MSVC, which was previously the only compiler supported on Windows.
Clang-20 was released in the meantime though, I'll have to switch to that soon.

Finally, install targets have been added to the CMake build and the Conan script. 
With this change in place, niku can now be used as a Conan package dependency in other projects.
I'm considering creating a dedicated samples repository. I don't want to keep binary assets in the same repository as the engine code.

# Final words
I've mentioned that the physics simulation is now calling scripting functions. 
Physics callbacks are running on their dedicated thread pool. 
AngelScript requires cleanup of the thread local data on each thread where it was used. 
Due to me not reading the documentation, this created a memory leak.
It was an easy fix, but I don't like this direct coupling of physics, threads and scripting. 
In the future, I would also like to handle asset loading asynchronously.

As most of niku development is based on self-inflicted opportunities™ I'm thinking about a task system.
That would tie in nicely with abstracting away the direct dependency on SDL events.

Fixing the PID controller and pathfinding are a priority, of course.

For any questions or corrections to this post, leave a comment in [discussion thread](https://github.com/jan-kelemen/niku/discussions/1) on GitHub or any other place where you see this.

[Diff](https://github.com/jan-kelemen/niku/compare/28c9c89b7b7dece34280d0a6b9181f38ce68f265...5c168793d634d97ef5c04794d7ad03bfc53e69ba) compared to the state shown in the previous post.

Previous posts in the series:
* [Navigating Bad Assumptions](/post/2025-02-07-navigating-bad-assumptions)
* [Eppur si muove](/post/2025-01-11-eppur-si-mouve)
* [Sidequests And Loose Ends](/post/2024-11-25-sidequests-and-loose-ends)
* [Rendering Physically Based Bugs](/post/2024-10-26-rendering-physically-based-bugs)
* [Preparing For Physically Based Rendering](/post/2024-09-27-preparing-for-physically-based-rendering)
