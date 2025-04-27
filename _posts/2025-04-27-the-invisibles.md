---
title: The Invisibles
author: Jan Kelemen
tags: [C++, Vulkan, graphics programming, Jolt Physics, niku]
---

It's quite easy to spend weeks making almost no visible progress on my game engine [jan-kelemen/niku](https://github.com/jan-kelemen/niku).

<iframe width="1000" height="562" src="https://www.youtube.com/embed/-syid1NaFCg" title="niku - The Invisibles" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

# Shadow mapping
After postponing working on shadows from the beginning of this project, I've finally implemented shadow mapping in the `gltfviewer`. 
At the start of the video, there is a comparison of the Sponza scene rendered just with the ambient lighting,
then with the added directional light and then with the shadows cast with that directional light.

When the scene is rendered with a directional light that doesn't cast a shadow, as shown at timestamp `00:10`, the results look weird.
The light also illuminates the area behind the curtains where the light source should be blocked.

This is done by first rendering the scene from the position of the light to a shadow map.
Only the depth information is required for the shadow map, so the scene is rendered to a depth buffer.
The shadow map will contain the depth value of the fragments visible by the light.

{:refdef: style="text-align: center;"}
![shadow-map](/assets/posts/20250427/2025-04-27-shadow-map.png)
{:refdef}

When doing the lighting calculations in the fragment shader this value is sampled from the texture and compared with the depth of the current fragment being rendered.
If the depth of the rendered fragment is larger than the depth value from the shadow map it means that the fragment is in the shadow, therefore its contribution is reduced.
```
color += NdotL * radiance * (1.0 - shadow) * (diffuseContribution + specularContribution);
```

# AngelScript & JPH::JobSystem
The previous post mentioned that I ran into a memory leak when calling scripting functions from the physics callbacks.
Jolt Physics executes the physics callbacks within a job system and has its abstraction [JPH::JobSystem](https://jrouwe.github.io/JoltPhysics/class_job_system.html).
There is a provided implementation of the [JPH::JobSystemThreadPool](https://jrouwe.github.io/JoltPhysics/class_job_system_thread_pool.html) that I've used until now.
Although the documentation states it's an example implementation, it's quite good.

Initially, I thought that I might be able to use [libuv](https://libuv.org) to implement a task system that would replace the thread pool implementation provided by Jolt.
Upon closer inspection of the documentation that wouldn't work out. Although the `libuv` does have a thread pool, it's global and doesn't give much control over it.
The event driven nature of the `libuv` interface also didn't fit the requirements for the implementation I needed, to submit a task and wait for it to be executed.

Scrapping that idea, I implemented the generic [work stealing](https://en.wikipedia.org/wiki/Work_stealing) thread pool and a wrapper around to adapt its interface to the `JPH::JobSystem` requirements.
The implementation of the thread pool is mostly based on the one found in [C++ Concurrency In Action](http://www.cplusplusconcurrencyinaction.com). 

This makes the implementation of the `JPH::JobSystem` fairly trivial. 
I took the same approach as the [Godot Jolt extension](https://github.com/godot-jolt/godot-jolt) developers and used [JPH::JobSystemWithBarrier](https://jrouwe.github.io/JoltPhysics/class_job_system_with_barrier.html).
This base class handles the dependencies between jobs created by Jolt and I didn't see much need for doing that part too by myself.

The actual submission of the Jolt job to the thread pool looks like this:
```
void ngnphy::job_system_t::QueueJob(JPH::JobSystem::Job* const inJob)
{
    inJob->AddRef();

    pool_->submit(
        [inJob]()
        {
            inJob->Execute();
            inJob->Release();
        });
}
```

As for the thread local storage used by AngelScript, the thread pool supports an exit function to be executed before joining the thread:
```
thread_pool_.set_thread_exit_function([]() { asThreadCleanup(); });
```

I'm keeping the idea of using `libuv` in the future though when I get to the networking and for things that are truly event driven and asynchronous.

## P(ath) controller
Staying on the topic of things done for the `galileo` demo. I needed to fix the PID controller used for moving the spheres when pathfinding.
I've fixed it by removing the ID parts of the controller, the proportional component is enough.
As the next point to which the sphere needs to move is evaluated each frame from the current position there isn't an accumulated error that should be corrected by the integral and derivative components.

The sphere now doesn't slam into the walls as much and its movement is smoothed out by the inertia of the physics body itself.
How the new pathfinding movement looks in the labyrinth is shown in the video at `00:50`.

# Other news
Looking around in the project properties in Visual Studio, I've noticed that the `/GL (Whole program optimization)` flag isn't turned on in the projects.
It makes sense, I use CMake to generate the projects and it has its own flag to control this `CMAKE_INTERPROCEDURAL_OPTIMIZATION`, that I didn't turn on.
So I've added options to turn on `Whole program optimization`, or `Link time optimization` as other compilers call it to the Conan profiles.

Great success, well almost, a surprise is hiding in the compiler output when compiling with GCC:
```
/home/runner/.conan2/p/b/glslade4429c0836b8/b/src/SPIRV/spirv.hpp:943: warning: type ‘spv::Scope’ violates the C++ One Definition Rule [-Wodr]
  943 | enum Scope {
/home/runner/.conan2/p/b/spirv0adda0305824e/p/include/spirv/unified1/spirv.hpp11:943: note: an enum with different value name is defined in another translation unit
  943 | enum class Scope : unsigned {
```

Turns out that GCC has a `-Wodr` warning that is enabled when LTO is active and it detected a One Definition Rule violation.
As ODR is undefined behavior it allows the compiler to do anything. 
I think I don't support compilers that summon dragons, but I don't like undefined behavior or disabling warnings. 
So I fixed it in the upstream version of the `glslang` library and temporarily suppressed the warning until a new version is released.

I've updated to Clang-20 without issues or changes required to the dependencies. 
Though a new version of GCC has been released, I'll have to handle that next.

The LTO changes and the Clang-20 update are propagated to the [cpp-starter-template](https://github.com/melinda-sw/cpp-starter-template) project.

# Final words
If you are wondering what is the performance benefit of LTO, well, I don't know. 
Neither `gltfviewer` nor `galileo` are CPU bound so the current difference is within the margin of error.
I did observe a 20% reduction in binary size with LTO enabled when compiled with Clang. 
Going by the rule of "less is more" this is good.
In theory, this should be the ideal case for LTO as I link all of the dependencies and libraries statically.

I feel like both the `gltfviewer` and `galileo` demos have a good enough scope to not expand them more from their current state.
That doesn't mean that the development work is finished either.
I'm aware that I'm seriously missing text rendering and sound support, event handling needs improvement.
On the Vulkan side, I want to support raytracing extensions, as well as try out bindless descriptors and buffer device addressing.

Working on the shadow mapping did make me realize that for the actual engine runtime, I should probably go with a full ECS style of handling things,
composition of different parts of the code felt kind of clunky.

I'll try to handle at least one of these things next.

For any questions or corrections to this post, leave a comment in [discussion thread](https://github.com/jan-kelemen/niku/discussions/1) on GitHub or any other place where you see this.

[Diff](https://github.com/jan-kelemen/niku/compare/5c168793d634d97ef5c04794d7ad03bfc53e69ba...8c2b6666fbc6dfa78253bd20186233318dc31452) compared to the state shown in the previous post.

Previous posts in the series:
* [Finding the Way Home](/2025/03/13/finding-the-way-home.html)
* [Navigating Bad Assumptions](/2025/02/07/navigating-bad-assumptions.html)
* [Eppur si muove](/2025/01/11/eppur-si-mouve.html)
* [Sidequests And Loose Ends](/2024/11/25/sidequests-and-loose-ends.html)
* [Rendering Physically Based Bugs](/2024/10/26/rendering-physically-based-bugs.html)
