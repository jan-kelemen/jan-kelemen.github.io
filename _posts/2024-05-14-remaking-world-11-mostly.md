---
title: Remaking World 1-1 with Vulkan, mostly
author: Jan Kelemen
tags: [C++, Vulkan, VK_KHR_dynamic_rendering, ImGui, graphics programming, game programming]
---

A couple of months ago (2 to be exact), I started learning graphics programming with [Vulkan API](https://www.vulkan.org).
I went through the [Vulkan Tutorial](https://vulkan-tutorial.com), then made a [Pong](https://github.com/jan-kelemen/vkpong)
clone and a [CHIP-8 emulator](https://github.com/jan-kelemen/vkchip8).

Sidenote, before this I'd never done any serious graphics programming at all, so use the information given here as a learning resource at your own risk.

After that, it was time to increase the complexity a bit, so I decided to remake the first level from Super Mario Bros, well, mostly.
You can take a look at the gameplay footage of how far I've gotten in the video below.

<iframe width="1000" height="562" src="https://www.youtube.com/embed/h1GldMexiZM" title="Ceramio - Gameplay" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

This is far from accurate recreation, nor does it attempt to be.
Due to the fact that I'm not an owner of assets used for making this project, the source code is not available publicly. 
This post serves as an overview of the project.

# The basics

As already mentioned, this project uses Vulkan, for the window creation and handling of the events I've used SDL2. 
And a handful of other dependencies for which I've used Conan2 to integrate with the CMake build of the code. Here's the complete list of used dependencies:
```
self.requires("glm/cci.20230113")
self.requires("entt/3.13.0")
self.requires("fmt/10.2.1")
self.requires("freetype/2.13.2")
self.requires("imgui/1.90.4-docking")
self.requires("sdl/2.30.1")
self.requires("spdlog/1.13.0")
self.requires("stb/cci.20230920")
self.requires("vulkan-headers/1.3.268.0")
self.requires("vulkan-loader/1.3.268.0")
```

Since Vulkan is available on multiple platforms, this remake runs both on Windows and on Linux. I did have to make changes to a couple of public Conan recipes,
mostly to make them compile with all of the combinations compilers and settings that I use, modifications to these are available in [jan-kelemen/conan-recipes](https://github.com/jan-kelemen/conan-recipes).

# Main loop

## Events
This part of the code is pretty simple, the main loop polls the events and forwards them to the game instance.
```
while (!done)
{
    SDL_Event event;
    while (SDL_PollEvent(&event) != 0)
    {
        ...
        if (is_quit_event(event, window.native_handle()))
        {
            done = true;
        }
        else if (!game.handle_event(event))
        {
            spdlog::debug("Unhandled event: {}", event.type);
        }
    }
```
Afterwards [SDL_GetKeyboardState](https://wiki.libsdl.org/SDL2/SDL_GetKeyboardState) is used to get the current state of all keys on the keyboard.
An alternative to this would have been using the `SDL_Event` directly, but this handles a single key at the time,
therefore one would have to remember and listen for all key up/down events and remember the state, which is basically the same as the value returned by `SDL_GetKeyboardState`.

State of multiple keys is needed for implementing the correct movement during jumping.

The only other event that is handled is the window resize event for which the graphics swapchain is recreated when the window is resized.
Handled similarly as explained in [Vulkan Tutorial - Swap chain recreation](https://vulkan-tutorial.com/Drawing_a_triangle/Swap_chain_recreation)

## Cyclic update

On every frame in the main loop, after handling the user input, the frame is started, the game state is updated, and then drawn to the screen:

```
uint64_t const current_tick{SDL_GetPerformanceCounter()};
float const delta{static_cast<float>(current_tick - last_tick) /
    static_cast<float>(SDL_GetPerformanceFrequency())};
last_tick = current_tick;

game.begin_frame();
renderer.begin_frame();

game.update(delta);			
...
renderer.draw(&game);
renderer.end_frame();
```			

To be independent of the frame rate, [delta time](https://en.wikipedia.org/wiki/Delta_timing) is calculated and afterward used during physics calculation.
Explicit start and ending of a frame is performed mainly to indicate to graphics components to swap resources like buffers and such, forgetting to do this
usually leads to some funny bugs with flickering on some parts of the screen. The game uses multiple [frames in flight](https://vulkan-tutorial.com/Drawing_a_triangle/Drawing/Frames_in_flight)
therefore resources used to render a single frame are cyclically swapped on each update.

```
void ceramio::game::update(float const delta_time)
{
    update_physics(registry_, level_, delta_time);
    update_collisions(registry_, level_, delta_time);
}
```

The game update consists of updating the physics and collisions. Although in the current state, these only handle the player sprite, they are separated as collisions could be used
to implement block breaking. This is organized into an entity-component-system pattern and uses [EnTT](https://skypjack.github.io/entt/index.html) library.
Most likely an overkill because the entity registry contains just two entities, the player, and the display text overlay.

# Graphics

## Vulkan specific parts

Vulkan features used are kept to a minimum, here are the ones that are required from the device:
```
constexpr std::array const device_extensions = {
    VK_KHR_SWAPCHAIN_EXTENSION_NAME,
    VK_KHR_DYNAMIC_RENDERING_EXTENSION_NAME};

constexpr VkPhysicalDeviceFeatures device_features{
    .sampleRateShading = VK_TRUE,
    .samplerAnisotropy = VK_TRUE};

constexpr VkPhysicalDeviceVulkan13Features device_13_features{
    .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_VULKAN_1_3_FEATURES,
    .synchronization2 = VK_TRUE,
    .dynamicRendering = VK_TRUE};
```

Optionally, validation layers are also required when compiling in debug mode.

To avoid managing render passes and frame buffers explicitly [VK_KHR_dynamic_rendering](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_dynamic_rendering.html) extension is used.
The original Vulkan Tutorial doesn't use it, but I switched to using it when I made the Pong clone because it simplifies the amount of code necessary by a lot.
Setting it up correctly in combination with [Multisample anti-aliasing](https://docs.vulkan.org/samples/latest/samples/performance/msaa/README.html) took a couple of tries to get right, and staring at a blank screen while validation layer messages passed by.
[Lesley Lai's tutorial](https://lesleylai.info/en/vk-khr-dynamic-rendering/) explains how to use dynamic rendering in general, but doesn't mention MSAA, nor have I been able to find an example online that mentions it in this combination.

Therefore the missing piece of the puzzle is, when using multisampling, resolve the output of the color attachment into the swap chain image, here is a snippet of how to do it:
```
VkRenderingAttachmentInfo color_attachment_info{};
color_attachment_info.sType = VK_STRUCTURE_TYPE_RENDERING_ATTACHMENT_INFO;
color_attachment_info.imageLayout =
    VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
color_attachment_info.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
color_attachment_info.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
color_attachment_info.clearValue = scene->clear_color();
if (is_multisampled())
{
    color_attachment_info.imageView = color_image_.view;
    color_attachment_info.resolveMode = VK_RESOLVE_MODE_AVERAGE_BIT;
    color_attachment_info.resolveImageView =
        swap_chain_->image_view(image_index);
    color_attachment_info.resolveImageLayout =
        VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
}
else
{
    color_attachment_info.imageView = swap_chain_->image_view(image_index);
}

VkRenderingInfo render_info{};
...
render_info.colorAttachmentCount = 1;
render_info.pColorAttachments = &color_attachment_info;

vkCmdBeginRendering(command_buffer, &render_info);
```

Renderer code is independent of the game code and separated by a virtual interface:
```
class [[nodiscard]] vulkan_scene
{
public: // Destruction
    virtual ~vulkan_scene() = default;

public: // Interface
    virtual VkClearValue clear_color() = 0;

    virtual void draw(VkCommandBuffer command_buffer,
        VkExtent2D extent) = 0;

    virtual void draw_imgui() = 0;
};
```

The game only needs to implement `clear_color()` and `draw_()` calls to render graphics on the screen. Command buffers passed in the draw call are secondary command buffers, the primary command buffer is managed by the renderer.
In this case, using primary command buffers is sufficient, but this gives somewhat better isolation between rendering code and game code. I've also done this as a preparation for more complex scenarios in the future.

Other than that, the renderer is single threaded and uses one graphic queue and one transfer queue. Transfer queue is used to transfer texture images used by tiles, sprites and fonts to the graphics card.

## World rendering 

The world is rendered to a space of 32x16 tiles where each tile is 32x32 pixels in size, this is one of the deviations from the original game which used much smaller screen space and 16x16 tiles.

To render the whole level in a single draw call, instanced rendering is used in combination with a texture atlas, vertex shader calculates correct texture coordinates based on the tile id that needs to be rendered.
```
layout(push_constant) uniform PushConsts {
    int horizontal_tiles;
    int vertical_tiles;
} pushConsts;

layout(location = 0) in vec3 inPosition;
layout(location = 1) in vec2 inTexCoord;
layout(location = 2) in int inTileId;
layout(location = 3) in mat4 inModel;

layout(location = 0) out vec2 fragTexCoord;

layout(binding = 0) uniform Transform {
    mat4 view;
    mat4 projection;
} transform;

void main() {
    gl_Position = transform.projection * transform.view * inModel * vec4(inPosition, 1.0);

    float horizontalTextureOffset = float(inTileId) / pushConsts.horizontal_tiles;
    float verticalTextureOffset = float(inTileId / pushConsts.horizontal_tiles) / pushConsts.vertical_tiles;

    fragTexCoord = vec2(
        inTexCoord.x / pushConsts.horizontal_tiles + horizontalTextureOffset,
        inTexCoord.y / pushConsts.vertical_tiles + verticalTextureOffset);
}
```

There were probably better ways to implement the texture atlas, but this works. 
One thing I did learn is that texture atlases are prone to texture bleed, there are multiple ways to avoid this, I took the lazy one and slightly reduced `inTexCoord` to be in range `[0, 1)` instead of `[0, 1]`.
Considering the simplicity of the textures used here this works correctly.

### Camera
The game uses an orthographic camera and follows the player. Previously when making the CHIP-8 emulator I positioned the tiles on the screen by using the [normalized device coordinates](https://learnopengl.com/Getting-started/Coordinate-Systems).
That approach wasn't scalable here so the camera and positioning of the tiles on the screen is handled by model, view and projection matrices.
```
constexpr glm::fvec3 front_face{0, 0, -1};
constexpr glm::fvec3 up_direction{0, 1, 0};

auto const camera_x{std::clamp<float>(player_position.x,
    half_width,
    static_cast<float>(level_.width()) - half_width)};
auto const camera_y{std::clamp<float>(player_position.y,
    half_height,
    static_cast<float>(level_.height()))};
camera_ = glm::fvec3(camera_x, camera_y, 0.f);
camera.view = glm::lookAt(camera_, camera_ + front_face, up_direction);
camera.projection = glm::ortho<float>(-half_width,
    half_width,
    half_height,
    -half_height);
}
```	

### Frustum culling
In this case, it was trivial to implement [frustum culling](https://learnopengl.com/Guest-Articles/2021/Scene/Frustum-Culling) by rendering only blocks that are visible on the screen.
This didn't make any difference in the optimized builds, but some improvement was visible in debug mode with an uncapped frame rate.
```
auto const from_width{std::min(
    static_cast<size_t>(std::max(0.f, player_position.x - half_width)),
    level_.width() - horizontal_tiles)};

auto const to_width{static_cast<size_t>(
    std::min(from_width + horizontal_tiles + 1, level_.width()))};

auto const from_height{std::min(
    static_cast<size_t>(std::max(0.f, player_position.y - half_height)),
    level_.height() - vertical_tiles)};

auto const to_height{
    std::min(from_height + vertical_tiles + 1, level_.height())};

for (size_t height{from_height}; height != to_height; ++height)
{
    for (size_t width{from_width}; width != to_width; ++width)
    {
        if (auto level_tile{level_.at(width, height)};
            level_tile != tile::empty)
        {
            positions.emplace_back(
                static_cast<int>(std::to_underlying(level_tile)),
                glm::fvec2{width, height});
        }
    }
}
```

## Sprite rendering
Sprite rendering reuses the world rendering shader and code with a texture atlas of one tile. 
It would be better to make a batched renderer that deals only with sprites, in particular if support for different size sprites is added, as the world shader assumes uniform tile sizes.
 
## Text rendering
Text rendering loads a {{.ttf}} font file using `freetype` library, which is rendered to an image and sent to the graphics card as a texture. Text is rendered in a single batch using a batched renderer.
This is the prime candidate to use as a base code for sprite rendering batching.

# Debug UI
[Dear ImGui](https://github.com/ocornut/imgui) is used as a debug UI for the game renderer. After rendering the scene additional `draw_imgui()` callback is made to the application to render the debug UI.

This post [Integrating Dear ImGui in a custom Vulkan renderer](https://frguthmann.github.io/posts/vulkan_imgui/) by François Guthmann, explains quite well how to integrate ImGui
into a custom renderer. Using dynamic rendering simplifies the initialization code a lot though, the following shows the correct initialization of `ImGui_ImplVulkan_InitInfo` struct when using dynamic rendering.
```
VkPipelineRenderingCreateInfoKHR rendering_create_info{};
rendering_create_info.sType =
    VK_STRUCTURE_TYPE_PIPELINE_RENDERING_CREATE_INFO_KHR;
rendering_create_info.colorAttachmentCount = 1;
VkFormat const image_format{swap_chain->image_format()};
rendering_create_info.pColorAttachmentFormats = &image_format;

ImGui_ImplVulkan_InitInfo init_info{};
init_info.Instance = context->instance;
init_info.PhysicalDevice = device->physical;
init_info.Device = device->logical;
init_info.QueueFamily = device->present_queue->family;
init_info.Queue = device->present_queue->queue;
init_info.PipelineCache = VK_NULL_HANDLE;
init_info.DescriptorPool = descriptor_pool_;
init_info.RenderPass = VK_NULL_HANDLE;
init_info.Subpass = 0;
init_info.MinImageCount = swap_chain->min_image_count();
init_info.ImageCount = swap_chain->image_count();
init_info.MSAASamples = device->max_msaa_samples;
init_info.Allocator = VK_NULL_HANDLE;
init_info.CheckVkResultFn = imgui_vulkan_result_callback;
init_info.UseDynamicRendering = true;
init_info.PipelineRenderingCreateInfo = rendering_create_info;
ImGui_ImplVulkan_Init(&init_info);
```

Where the used descriptor pool just allocates one image sampler, this is enough for ImGui in a using a standalone descriptor pool:
```
VkDescriptorPoolSize imgui_sampler_pool_size{};
imgui_sampler_pool_size.type =
    VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
imgui_sampler_pool_size.descriptorCount = 1;

VkDescriptorPoolCreateInfo pool_info{};
pool_info.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
pool_info.flags = VK_DESCRIPTOR_POOL_CREATE_FREE_DESCRIPTOR_SET_BIT;
pool_info.poolSizeCount = 1;
pool_info.pPoolSizes = &imgui_sampler_pool_size;
pool_info.maxSets = 1;
```

# Level representation
One thing that wasn't previously explained is how the level is represented in the game, the answer is ASCII art.
```
R"(................................................................................................................................................................................................................................)"
R"(...................456..............4556...........................456..............4556..........................456..............4556............................456..............4556..............o............456..........)"
R"(........456........789.....45556....7889................456........789.....45556....7889................456.......789.....45556....7889.................456........789.....45556....7889.............[F.456........789.....45556)"
R"(........789................78889........................789................78889........................789...............78889.........................789................78889......................T.789................78889)"
R"(......................?.........................................................BBBBBBBB...BBB?..............?..........BBB....B??B.........................................................CC........|.........................)"
R"(...........................................................................................................................................................................................CCC........|.........................)"
R"(..........................................................................................................................................................................................CCCC........|.........................)"
R"(.........................................................................................................................................................................................CCCCC........|....VVV..................)"
R"(................?...B?B?B.....................QP.........QP..................B?B..............B.....BB....?..?..?....B..........BB......C..C...........CC..C............BB?B............CCCCCC........|....(b)..................)"
R"(.._...................................QP......qp.._......qp......................................._....................................CC..CC....._...CCC..CC..........................CCCCCCC...._...|...VWWWV.................)"
R"(./H\............._..........QP........qp......qp./H\.....qp......_.............................../H\............._....................CCC..CCC.../H\.CCCC..CCC..._.QP..............QP.CCCCCCCC.../H\..|...b]Abb.._..............)"
R"(/HGH\......12223/H\....123..qp........qp.1223.qp/HGH\....qp12223/H\......................1223.../HGH\......12223/H\...123............CCCC22CCCC./HGHCCCCC..CCCC3/H\qp..123.........qpCCCCCCCCC../HGH\.C...b]abb3/H\....123......)"
R"(#####################################################################..###############...################################################################..#####################################################################)"
R"(#####################################################################..###############...################################################################..#####################################################################)"
```

This is converted into a 2D array of `enum class` instances on startup of the game, to extend it one could load it from the disk, but it's hardcoded currently in the source. Thanks to Rick N. Burns for publishing the whole world on NESMaps.com which I've used as a [reference](https://nesmaps.com/maps/SuperMarioBrothers/SuperMarioBrosWorld1-1Map.html).

# The missing parts

To make this into a complete remake of World 1-1 one would have to implement animations, enemies (and other sprites), block breaking and the underground part of the level.
Maybe at some later point in time. :) 

For any questions or corrections to this post, contact me via email at <jkeleme@gmail.com>.