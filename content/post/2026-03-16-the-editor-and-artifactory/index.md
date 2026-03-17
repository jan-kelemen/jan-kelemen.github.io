---
author: Jan Kelemen
date: '2026-03-16T18:00:00Z'
tags:
- C++
- Vulkan
- niku
- Artifactory
title: The Editor and Artifactory
---

Back to writing about updates on my game engine [jan-kelemen/niku](https://github.com/jan-kelemen/niku)!

{{< youtube "cgVjCQWCKhg" >}}

# Editor
In the previous post, I mentioned that I started working on the game engine editor.
With the hope of making some forward progress on this topic, I've decided to use ImGui as the GUI framework in the editor.
Instead of reinventing a GUI framework on my own or using a different library for the GUI.

The grid implementation is pretty much a translation of the Unity grid shader shown in [The Best Darn Grid Shader (Yet)](https://bgolus.medium.com/the-best-darn-grid-shader-yet-727f9278b9d8).
It's sure better than anything I could come up with.

The grid is rendered as a 1000x1000 square of lines, but if the camera moves high up in the world, there were visible edges where the squares end.
So in the far distance of the grid, it's blended with the background to avoid seeing the edges:
```glsl
const float fragmentDistance = length(inCoords - camera.position.xz);
const float fadeDistance = clamp(abs(camera.position.y) * 25, 450, 500);
outColor.a *= smoothstep(1.0, 0.0, fragmentDistance / fadeDistance);
```
While I would figure out this blending by myself sooner or later, the contribution for this goes to [Simple "Infinite" Grid Shader](https://dev.to/javiersalcedopuyo/simple-infinite-grid-shader-5fah).

Thanks to the authors for making this available.

## I'm regret sure I won't this
When I started preparing for writing this post, I figured out that I'm missing a bit of content.
I've sprinkled in a bit of multithreading to the editor to mitigate this.

The editor now has a render thread that handles recording and execution of commands required to render the scene:
```cpp
render_thread_ = std::make_unique<std::jthread>(
    [this](std::stop_token const& token)
    {
        while (!token.stop_requested())
        {
            render();
        }
    });
```

And the main thread that handles the application loop, listening for events and updating the state:
```cpp
bool editor::application_t::update()
{
    if (uint64_t const steps{timestep_.pending_simulation_steps()})
    {
        std::unique_lock guard{state_mutex_};
        for (uint64_t i{}; i != steps; ++i)
        {
            camera_controller_.update(timestep_.update_interval);
        }
        projection_.update(camera_.view_matrix());
    }

    return true;
}
```

The threads are synchronized with a shared mutex. The rendering thread acquires a read lock when it needs to prepare the state for rendering.
The main thread acquires a write lock for state updates.

It was also a nice opportunity to try `std::jthread` and `std::stop_token`.
When the main editor window needs to be closed, the main thread requests a stop on the rendering threads' stop token:
```cpp
if (event.type == SDL_EVENT_WINDOW_CLOSE_REQUESTED)
{
    render_thread_->request_stop();
}
```

I'm aware that at one point this shared mutex will become a synchronization bottleneck, but let's see how far it takes me.

# Artifactory
So far, I've been using GitHub Actions cache for persisting compiled Conan packages of third party dependencies.
As the number of third party libraries and supported configurations grew, fitting all dependencies into the GitHub cache has become increasingly problematic.

It was time to pull the plug on this.
I've started hosting [Artifactory](https://jfrog.com/artifactory/) on my VPS.
JFrog offers a free variant of Artifactory for supporting the Conan package manager as a Docker container.

For one reason or another, this container has the JFConnect service enabled, a feature that isn't available in the free Artifactory license.
I was getting timeouts while connecting to the hosted Artifactory instance as it was trying to do something with this service.

Why it's enabled in the first place is a mystery to me, but it can be disabled in the system.yml via the following key:
```yaml
jfconnect:
    enabled: false
```

To be honest, it is obvious that these Artifactory containers aren't made to be hosted on small hardware. I'm running it below the official minimum specs.
Because of this, I'll probably look into replacing it with [Forgejo](https://forgejo.org) or some other lighter software.

## LNK4099
On the CI, some of the build configurations have debugging symbols enabled.
Since the symbol files are quite large, Conan recipes from the Conan Center don't collect the debug symbol files in the published packages.

This wasn't a problem when I had the Conan packages in the GitHub cache, as it was using the locally compiled packages and the symbol files were present in the cache.
With the switch to using published packages on my Artifactory instance, these symbol files were no longer available to the linker. 

To quote the wise words of MSVC compiler:
> volk.lib(volk.obj) : warning LNK4099: PDB 'volk.pdb' was not found with 'volk.lib(volk.obj)' or at 'D:\\...\\volk.pdb'; linking object as if no debug info

One way around it would be to compile the third party libraries with [/Z7](https://learn.microsoft.com/en-us/cpp/build/reference/z7-zi-zi-debug-information-format?view=msvc-170#z7) compiler flag.
With that option, MSVC would produce static libraries that contain the embedded debugging symbols.
The drawback of that is that the embedded symbols are larger than the self contained `.pdb` file.

That would be simpler, but I've decided to keep the symbol files separate ([/Zi](https://learn.microsoft.com/en-us/cpp/build/reference/z7-zi-zi-debug-information-format?view=msvc-170#zi)) and collect the `.pdb` files into the exported Conan package.
```python
def package(self):
  ...
  if self.settings.os == "Windows":
    for symbol_file in Path(self.build_folder).rglob("*.pdb"):
      shutil.copy2(symbol_file, os.path.join(self.package_folder, "lib", symbol_file.name))
```

This issue is only present on Windows builds, as the Linux libraries do contain the embedded debugging symbols.

# The great toolchain update
Recently, GitHub has finally made the Windows Runner with a Visual Studio 2026 installed available.
This also coincied with the release of GCC 15 to `ubuntu-toolchain-r/test` PPA repository and with the release of LLVM 22.

What this actually means is that I was able to update all of the compilers used on the CI to their latest versions.
I've left Visual Studio 2022 as a supported target for now. 
The GitHub Windows Runner image is technically still in a public beta.
I'll probably drop that soonish and switch over to C++26.

Though the language standard update isn't as exciting as it sounds.
I tend to avoid writing compiler / standard library specific code, so I stick to writing for the lowest common denominator of all supported compilers.
Until now, I've used [compiler support](https://en.cppreference.com/w/cpp/compiler_support.html) page to figure out what features I can use.
Recently, I've found [cppstat.org](https://cppstat.org). 
Nice to know there are alternatives, as cppreference.com has been in a read-only mode for some time now.

To enable usage of Visual Studio 2026 I've also had to bump the minimum version of CMake to 4.2.
This ended up being unexpectedly annoying to do on the CI.
For those that aren't following along with the updates to GitHub CI images, CMake on these images has been updated to 4.x and rolled back to 3.x at least once.
Coupled with the fact that CMake 4 raised some of the minimum requirements, the internet and GitHub issues are riddled with instructions how to pin the CMake version to 3.x.

Quite the opposite of what I needed, getting rid of the pinned version and updating it via the APT package manager.
So here is the magic:
```
rm /usr/local/bin/cmake || true
```
GitHub installs the pinned version to `/usr/local/bin`, this has predecedence over anything installed via APT packages.
On Windows images, I was able to simply install the new version via Chocolatey and it picked up the correct one.

I'll merge these changes to my template [melinda-sw/cpp-starter-template](https://github.com/melinda-sw/cpp-starter-template) once the GitHub runner image exits public beta.

# Final words
One of the things I also started working on but haven't touched upon in this post is writing unit tests for the engine.
This topic is still in its beginning stages. 
So far, I've noticed that Vulkan support on the GitHub runners isn't that great. I didn't expect it to be either, so the tests aren't run on the CI.
I'll see if I can do something meaningful by installing a CPU implementation of Vulkan, like [LLVMpipe](https://docs.mesa3d.org/drivers/llvmpipe.html), on the CI workflows, though this might turn out to be impractical.

The next step for the editor is to figure out the scene loading.
With the decision to have multiple threads, it will probably force me to finally implement asynchronous loading of assets.

For any questions or corrections to this post, leave a comment in [discussion thread](https://github.com/jan-kelemen/niku/discussions/1) on GitHub or any other place where you see this.

[Diff](https://github.com/jan-kelemen/niku/compare/3e2088a10a982cdfe1e61fb66102e05b2afb0c0b...a8f763ba7f307f0250075e786c7e5524695e7efc) compared to the state shown in the previous post.

Check out other posts in this series with [#niku](/tags/niku) tag.

