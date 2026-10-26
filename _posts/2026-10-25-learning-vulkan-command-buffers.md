---
title: "Episode 15: Understanding Vulkan Command Buffers"
date: 2026-10-24 20:00:00 +0900
categories: [Vulkan]
tags: [vulkan, rendering, gpu, c++]
description: What command pools and command buffers are, how they relate, and how my triangle sample records a complete dynamic-rendering command sequence.
---

Today I read the Vulkan Tutorial's chapter on [command buffers](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/03_Drawing/01_Command_buffers.html) and implemented it in the [12_command_buffer sample](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/12_command_buffer).

This chapter answered a question that had been sitting underneath all the earlier Vulkan setup: when do the drawing operations actually happen?

The answer is not when I call a Vulkan drawing function from C++. Vulkan records those operations into a command buffer first. Later, the command buffer is submitted to a queue, where the GPU can execute the recorded work as a batch.

That indirection is one of Vulkan's defining ideas. The CPU builds an explicit sequence of GPU work, and the GPU receives that sequence in a form it can process efficiently. It also means command recording can be organized across multiple threads when a renderer becomes more advanced.

The sample for this post uses Vulkan-Hpp RAII objects and dynamic rendering. The code is in `src/basic/12_command_buffer/src/commandBuffer.cpp` in my [Vulkan learning repository](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/12_command_buffer).

## The confusing part: command pool versus command buffer

The names are similar, but they describe two different layers:

* A **command pool** owns the memory used to allocate command buffers. It belongs to one queue family.
* A **command buffer** is the actual container of recorded commands: image barriers, rendering commands, pipeline bindings, viewport state, and draw calls.

An analogy helps. The command pool is like a workshop or an allocator. The command buffer is like a work order created inside that workshop. The workshop does not contain the drawing instructions itself; it provides the storage and lifetime management for the work orders.

The relationship is therefore:

```text
logical device
    └── command pool for a queue family
            └── one or more command buffers
```

The pool must be created first because command buffers are allocated from it. When the pool is destroyed, the command buffers allocated from it are released as well.

## Creating the command pool

Here is the command-pool part of my implementation:

```cpp
const vk::CommandPoolCreateInfo poolInfo{
    .flags = vk::CommandPoolCreateFlagBits::eResetCommandBuffer,
    .queueFamilyIndex = *_device.queueFamilies().graphicsAndPresent};

_pool = vk::raii::CommandPool(_device.logicalDevice(), poolInfo);
```

The `queueFamilyIndex` is important. A command pool is tied to one queue family, and command buffers allocated from it are intended for queues of that family. This sample records graphics work, so it uses the queue family selected for graphics and presentation.

The `eResetCommandBuffer` flag allows an individual command buffer to be reset and recorded again. That is useful for a renderer that records new commands every frame. Without it, command buffers from the pool would have to be reset together.

The pool is not a queue and it does not execute anything. It is only the memory-management context from which command buffers are allocated.

## Allocating the command buffer

Once the pool exists, my code allocates one primary command buffer:

```cpp
const vk::CommandBufferAllocateInfo allocationInfo{
    .commandPool = *_pool,
    .level = vk::CommandBufferLevel::ePrimary,
    .commandBufferCount = 1};

auto buffers = vk::raii::CommandBuffers(
    _device.logicalDevice(), allocationInfo);
_buffer = std::move(buffers.front());
```

`commandPool` identifies the source pool. `commandBufferCount` is one because this small sample uses a single command buffer. In a real application, it is common to have several buffers—for example, one per frame in flight or one per worker thread.

The `level` is `ePrimary`. A primary command buffer can be submitted directly to a queue. A secondary command buffer cannot be submitted on its own; it is called from a primary command buffer. Secondary buffers are useful when different parts of a renderer record reusable pieces of work independently, but this triangle does not need that extra layer.

`vk::raii::CommandBuffers` returns a collection. Since I requested one buffer, I move its first element into `_buffer`. The RAII objects also make ownership clear: the command pool owns the allocation, and destruction is handled automatically.

## Recording commands

Creating a command buffer does not put any work into it. It starts empty. The `record` function writes a fresh sequence for the selected swap-chain image:

```cpp
_buffer.begin({});
```

The call starts recording. The empty `CommandBufferBeginInfo` means that none of the optional usage flags are needed here. The command buffer is recorded, submitted once by the application's main loop, and recorded again for later frames.

There is one more piece of preparation before drawing: the swap-chain image must be in a layout suitable for use as a colour attachment. My sample records this transition with `pipelineBarrier2`, changing the image from `eUndefined` to `eColorAttachmentOptimal`. After rendering, it records the reverse transition to `ePresentSrcKHR`, which makes the image ready for presentation.

## Starting dynamic rendering

This sample uses Vulkan's dynamic-rendering path, so it does not create a render-pass or framebuffer object. Instead, it describes the attachment directly while recording:

```cpp
vk::ClearValue clearColor;
clearColor.color = vk::ClearColorValue{
    std::array<float, 4>{0.0F, 0.0F, 0.0F, 1.0F}};

const vk::RenderingAttachmentInfo colorAttachment{
    .imageView = *swapchain.imageViews()[imageIndex],
    .imageLayout = vk::ImageLayout::eColorAttachmentOptimal,
    .loadOp = vk::AttachmentLoadOp::eClear,
    .storeOp = vk::AttachmentStoreOp::eStore,
    .clearValue = clearColor};

const vk::RenderingInfo renderingInfo{
    .renderArea = {.offset = {0, 0}, .extent = swapchain.extent()},
    .layerCount = 1,
    .colorAttachmentCount = 1,
    .pColorAttachments = &colorAttachment};

_buffer.beginRendering(renderingInfo);
```

`imageView` selects the swap-chain image that this frame will write to. The layout matches the transition recorded earlier. `loadOp = eClear` tells Vulkan to clear the attachment before the draw, while `storeOp = eStore` tells it to keep the rendered result afterward.

`renderArea` covers the swap-chain extent, and `layerCount = 1` describes an ordinary, non-layered image. Finally, `beginRendering` opens the dynamic-rendering section. Graphics commands can now be recorded inside it.

## Binding the pipeline and setting dynamic state

The next command selects the graphics pipeline:

```cpp
_buffer.bindPipeline(
    vk::PipelineBindPoint::eGraphics,
    *graphicsPipeline.pipeline());
```

The pipeline contains the shaders and fixed-function state that describe how the triangle should be processed. My pipeline declares the swap-chain format through `vk::PipelineRenderingCreateInfo`, which makes it compatible with the dynamic-rendering attachment above.

The pipeline was created with viewport and scissor as dynamic state, so their actual values must be supplied in the command buffer:

```cpp
{% raw %}
_buffer.setViewport(0, vk::Viewport{
    0.0F, 0.0F,
    static_cast<float>(swapchain.extent().width),
    static_cast<float>(swapchain.extent().height),
    0.0F, 1.0F});

_buffer.setScissor(
    0, vk::Rect2D{{0, 0}, swapchain.extent()});
{% endraw %}
```

The viewport maps the vertex shader's output into the swap-chain image. The scissor rectangle limits the region where fragments may be written. Both cover the entire image in this example.

## Drawing the triangle

Now the actual draw call is very small:

```cpp
_buffer.draw(3, 1, 0, 0);
```

The four arguments are:

* `3`: draw three vertices;
* `1`: draw one instance;
* `0`: start at vertex zero; and
* `0`: start at instance zero.

There is no vertex buffer yet. The vertex shader generates the triangle from the vertex ID, so three generated vertices are enough to form one triangle. Most of the information needed for this call—the shaders, rasterizer, render target, viewport, and scissor—has already been recorded or configured.

## Ending the recording

After the draw, the rendering section is closed:

```cpp
_buffer.endRendering();
```

Then the sample transitions the image from `eColorAttachmentOptimal` to `ePresentSrcKHR` and finishes the command buffer:

```cpp
_buffer.end();
```

At this point, the commands have been recorded but the GPU has not necessarily executed them yet. The application's main loop can submit this primary command buffer to the graphics queue, together with the synchronization needed for image acquisition and presentation.

The complete recorded sequence is therefore:

```text
begin recording
    transition image for colour attachment use
    begin dynamic rendering
    bind graphics pipeline
    set viewport and scissor
    draw three vertices
    end dynamic rendering
    transition image for presentation
end recording
```

## Summary

The most useful distinction I learned is that a command pool and a command buffer are not interchangeable:

* the **command pool** manages the memory and queue-family relationship;
* the **command buffer** stores the commands that will eventually run on the GPU.

The workflow is deliberately explicit: create a pool, allocate a primary buffer from it, begin recording, prepare the image, describe dynamic rendering, bind the pipeline, set the dynamic state, issue the triangle draw, restore the image for presentation, and end recording.

The final `draw` call is simple because the command buffer has already collected the context around it. That is the central idea of Vulkan command buffers: build a complete, ordered description of GPU work first, then submit that description to a queue for execution.

### References

* [Khronos Vulkan Tutorial: Command buffers](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/03_Drawing/01_Command_buffers.html)
* [My Vulkan learning sample: `12_command_buffer`](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/12_command_buffer)
