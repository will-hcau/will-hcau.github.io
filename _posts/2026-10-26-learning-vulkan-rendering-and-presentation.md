---
title: "Episode 16: Rendering, Presentation, and Our First Triangle"
image:
  path: /assets/img/vulkan-first-triangle.png
date: 2026-10-26 00:00:00 +0900
categories: [Vulkan]
tags: [vulkan, rendering, synchronization, gpu, c++]
description: "Connecting command recording, graphics submission, and swapchain presentation with semaphores and a fence—and finally seeing our first Vulkan triangle."
---

Finally, the pieces come together.

Over the previous episodes, we created a window, selected a device, built a swapchain, compiled shaders, configured the graphics pipeline, and recorded drawing commands. That is a lot of infrastructure for a triangle we still could not see.

Today I followed the Vulkan Tutorial's [rendering and presentation chapter](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/03_Drawing/02_Rendering_and_presentation.html). The missing connection is the **frame loop**: acquire an image, submit work that renders into it, and ask the presentation engine to display the result.

My implementation is the [13_rendering_presentation sample](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/13_rendering_presentation). The excerpts below come from its `main.cpp` and supporting classes, with some comments omitted for readability. It uses **Vulkan-Hpp RAII**, **dynamic rendering**, and one queue family supporting both graphics and presentation.

## Rendering produces pixels; presentation makes them visible

The graphics pipeline describes **how** to render. A command buffer records **what work** to execute. Neither one, by itself, makes an image appear in the window.

We need two cooperating systems:

- **Rendering** executes the recorded commands and writes pixels into an image.
- **Presentation** hands that completed swapchain image to the presentation engine so it can eventually become visible.

The swapchain is the bridge between them:

<div style="width: 100%; aspect-ratio: 960 / 430;">
  <object type="image/svg+xml" data="{{ '/assets/img/vulkan-swapchain.svg' | relative_url }}" width="100%" height="100%" style="display: block; width: 100%; height: 100%;" aria-label="Animated swapchain diagram: rendering and presentation alternate between images">
    <a href="{{ '/assets/img/vulkan-swapchain.svg' | relative_url }}">View the animated swapchain diagram</a>
  </object>
</div>

_The swapchain diagram from our earlier episode: rendering and display use images at different times. Acquisition and presentation coordinate their reuse._

This is a conceptual diagram, not a promise of exactly two images or a fixed alternation. The actual image count and presentation behavior depend on the swapchain configuration and present mode.

The recurring flow is:

```text
Acquire a swapchain image
          |
          v
Record commands targeting that image
          |
          v
Submit the commands to the graphics queue
          |
          v
Present the completed image
          |
          v
Acquire another available image and repeat
```

The acquired **image index** connects these steps. It selects both the image we render into and the image we later present. We do not simply increment an index and assume that the next image is free.

Also, a successful presentation request is not a report that the pixels have already been scanned out to the monitor. Presentation is scheduled according to the platform and present mode.

## Why synchronization is necessary

The CPU and GPU do not execute our C++ statements in lockstep.

Returning from a queue submission does not mean the submitted drawing work is complete. Likewise, knowing which image we acquired is not permission to ignore its readiness synchronization.

Without explicit coordination, three hazards appear:

1. **Rendering too early:** the presentation engine may still be using the acquired image.
2. **Presenting too early:** the image may not contain the completed rendering yet.
3. **Reusing resources too early:** the CPU may start re-recording a command buffer while the GPU is still executing it.

The fact that my sample uses the same queue for graphics and presentation does not remove these responsibilities. Queue order is not a replacement for every required execution and memory dependency.

Vulkan makes synchronization an application responsibility. This chapter focuses on two tools: **binary semaphores** for the acquire/render/present dependencies, and a **fence** for host-side resource reuse. [Vulkan specification: Synchronization](https://docs.vulkan.org/spec/latest/chapters/synchronization.html)

## Semaphores coordinate asynchronous work

A **binary semaphore** provides a synchronization point between **device-side operations**, including the rendering and presentation dependencies in this example.

It has an unsignaled or signaled state. One operation signals it; a dependent operation waits on it. The wait consumes the signal, returning the semaphore to the unsignaled state.

For our frame, there are two dependencies to express:

1. **Image ready:** rendering must not use the swapchain image while the presentation engine is still using it.
2. **Rendering complete:** presentation must not use the image before its rendering commands have finished.

```text
Acquired image becomes available
      |
      | signal the image-ready semaphore
      v
Rendering waits at the required pipeline stage
      |
      v
Rendering commands execute and complete
      |
      | signal the rendering-complete semaphore
      v
Presentation's semaphore wait is satisfied
      |
      v
Image can proceed to presentation
```

These waits belong to the queued operations. **The CPU does not need to wait for rendering to finish before requesting presentation.** It can enqueue both operations and let the semaphore express their dependency.

Crucially, **submitting rendering work is not the same as completing it**. The rendering-complete semaphore is signaled after the submitted commands finish, not simply because the CPU has submitted them.

Acquiring a swapchain image can itself block, depending on availability and timeout. Semaphores do not make every API call non-blocking; they carry dependencies into the queued work. [Vulkan reference: vkAcquireNextImageKHR](https://docs.vulkan.org/refpages/latest/refpages/source/vkAcquireNextImageKHR.html)

Vulkan also has **timeline semaphores**, which support additional uses, including host waits and signals. Here, we are discussing the **binary semaphores** used by the swapchain flow.

## Fences let the CPU know when work has completed

A **fence** provides a synchronization point between **GPU work and the CPU**. It answers:

> **Has the GPU finished the submission whose resources I want to reuse?**

Suppose the CPU wants to record a new frame into a command buffer. If the GPU is still executing its previous contents, re-recording it would be unsafe. The host needs a completion notification before it proceeds.

We attach a fence when we submit the graphics work. **Once the device has completed that submitted work**, it signals the fence. This is not the CPU returning from the submission call. The CPU can wait for the fence before reusing the command buffer.

```text
CPU: submit work -> wait for fence ............... -> reuse resources
             |                              ^
             v                              |
GPU:      execute submitted work -> signal fence
```

A fence wait returns immediately if the work is already complete. Otherwise, it blocks the calling host thread until the fence is signaled or the wait times out.

Unlike a binary semaphore's consumed signal, **a fence remains signaled until the host resets it** before reuse.

## Semaphores versus fences

Both create synchronization points, but **the side that waits is different**:

| Tool in this example | Synchronization boundary | Who waits? | Question it answers |
|---|---|---|---|
| **Binary semaphore** | Between device-side operations, including swapchain dependencies | The dependent queued operation | May rendering or presentation proceed? |
| **Fence** | From completion of submitted work on the device to the CPU | The calling host thread, when it waits | May the CPU reuse this submission's resources? |

A useful shorthand is **device-to-device ordering versus device-to-host completion**. The semaphore dependencies keep the rendering/presentation flow ordered without inserting a CPU wait between each operation. The fence prevents the CPU from getting ahead of resources that the GPU still uses.

This distinction describes our use here, not every synchronization mechanism Vulkan offers. Neither tool replaces the image barriers in the command buffer: image layouts and access dependencies still need to be correct.

## Creating the objects in main.cpp

`GraphicPipelineExample::setup()` creates the resources in dependency order:

```text
Window -> instance -> surface -> device -> swapchain
       -> graphics pipeline -> command buffer -> synchronization objects
```

The synchronization objects are created after the swapchain because one set depends on its image count:

```cpp
_imageAvailable = vk::raii::Semaphore(
    _device->logicalDevice(), vk::SemaphoreCreateInfo{});

for (std::size_t image = 0; image < _swapchain->images().size(); ++image)
    _renderFinished.emplace_back(
        _device->logicalDevice(), vk::SemaphoreCreateInfo{});

_drawFence = vk::raii::Fence(
    _device->logicalDevice(),
    vk::FenceCreateInfo{.flags = vk::FenceCreateFlagBits::eSignaled});
```

Now that the objects have names, we can map them to the concepts above:

| Object | When it is signaled | Who waits? |
|---|---|---|
| `_imageAvailable` | When the acquired image becomes available for rendering | The queued rendering work, at its configured wait stage |
| `_renderFinished[imageIndex]` | After the submitted rendering commands have completed on the device | The queued presentation operation |
| `_drawFence` | When the device has completed the graphics work submitted with this fence | The CPU, before reusing the command buffer |

The `_renderFinished` array contains the same kind of binary semaphore as `_imageAvailable`; their different names describe their roles, not different Vulkan semaphore types.

The **fence starts signaled** for a practical reason: the first frame waits on it before any GPU submission has occurred. If it started unsignaled, nothing would signal it and that initial wait would never complete.

### Why one render-finished semaphore per swapchain image?

**Because rendering can finish before presentation has finished using the semaphore.**

Imagine that every frame reused one semaphore named `S`:

```text
Frame 1: GPU finishes rendering -> signals S
         Presentation is supposed to wait on S

Frame 2: CPU sees the graphics fence has signaled
         CPU submits another draw that will signal S again
```

What is missing? **The graphics fence does not tell the CPU whether presentation has consumed the first signal on `S`.** If it has not, the next draw could try to signal a binary semaphore that is still signaled or still in use. That is invalid reuse.

The fence is a receipt for the **rendering submission**, not for the later **presentation operation**.

My implementation avoids sharing that one semaphore across different images. Each swapchain image gets its own:

```text
Swapchain image 0 -> renderFinished[0]
Swapchain image 1 -> renderFinished[1]
Swapchain image 2 -> renderFinished[2]
```

When we acquire image 1, both rendering and presentation use `renderFinished[1]`. Acquiring image 2 selects `renderFinished[2]` instead.

Eventually, image 1 becomes available again. **Once its new acquisition's synchronization has been waited on, its previous presentation has finished using that image's semaphore.** We can then signal `renderFinished[1]` for the next rendering of image 1. In our code, that acquisition wait is part of the graphics submission, before the new render-finished signal.

The rule to remember is:

> **Reuse a render-finished semaphore with its swapchain image—not merely because the graphics fence signaled.**

This is the pattern recommended by the [Vulkan Guide on swapchain semaphore reuse](https://docs.vulkan.org/guide/latest/swapchain_semaphore_reuse.html).

The acquire semaphore has a different consumer: the graphics submission itself waits on `_imageAvailable`. Its fence therefore establishes that this wait has completed, allowing us to reuse the acquire semaphore in our single-submission-in-flight loop.

## Following one frame through drawFrame()

### 1. Wait before reusing the command buffer

The first part of `drawFrame()` waits for the previous submission and resets the fence:

```cpp
if (_device->logicalDevice().waitForFences(
        *_drawFence, vk::True, UINT64_MAX) != vk::Result::eSuccess)
    throw std::runtime_error("failed to wait for draw fence");

_device->logicalDevice().resetFences(*_drawFence);
```

`vk::True` requests waiting for all supplied fences; here there is only one. `UINT64_MAX` supplies a very large timeout rather than a short polling interval.

After this wait succeeds, the host knows that the previous graphics work associated with the fence has completed on the device. On the first iteration, there is no previous work: the initially signaled fence simply lets us proceed. We can now re-record the command buffer safely.

### 2. Acquire an image and select its semaphore

```cpp
auto [acquireResult, imageIndex] =
    _swapchain->handle().acquireNextImage(
        UINT64_MAX, *_imageAvailable, nullptr);

if (acquireResult != vk::Result::eSuccess)
    throw std::runtime_error("failed to acquire a swap-chain image");

auto& renderFinished = _renderFinished[imageIndex];
```

The returned `imageIndex` identifies the swapchain image to use. `_imageAvailable` will be signaled when the acquisition permits the dependent rendering operations to proceed.

Passing `nullptr` for the acquisition fence is deliberate: this dependency will be waited on by the graphics submission through a semaphore, rather than by a separate CPU fence wait.

Notice that the semaphore we select for presentation is tied to the **acquired image**, not a CPU frame counter.

### 3. Record the triangle for that image

```cpp
_commandBuffer->record(imageIndex, *_swapchain, *_pipeline);
```

**This call records instructions; it does not execute the transitions or draw on the GPU yet.** The recording implementation in `src/commandBuffer.cpp` builds the command sequence we prepared in the previous episode:

1. Transition the selected image to **`eColorAttachmentOptimal`**.
2. Select `swapchain.imageViews()[imageIndex]` as the color attachment.
3. Begin dynamic rendering and clear the attachment to black.
4. Bind the graphics pipeline, viewport, and scissor.
5. Draw three vertices.
6. End rendering and transition the image to **`ePresentSrcKHR`**.

The central draw is still only:

```cpp
_buffer.draw(3, 1, 0, 0);
```

The shader supplies the three positions and colors using `SV_VertexID`, so this sample does not yet need a vertex buffer.

The command pool was created with **`eResetCommandBuffer`**. Beginning a fresh recording on a completed executable command buffer can implicitly reset it; the crucial restriction is that it must no longer be pending. That is why the fence wait comes first. [Vulkan reference: vkBeginCommandBuffer](https://docs.vulkan.org/refpages/latest/refpages/source/vkBeginCommandBuffer.html)

### 4. Submit with the acquire dependency and completion signals

```cpp
const vk::PipelineStageFlags waitStage =
    vk::PipelineStageFlagBits::eColorAttachmentOutput;

const vk::SubmitInfo submitInfo{
    .waitSemaphoreCount = 1,
    .pWaitSemaphores = &*_imageAvailable,
    .pWaitDstStageMask = &waitStage,
    .commandBufferCount = 1,
    .pCommandBuffers = &*_commandBuffer->buffer(),
    .signalSemaphoreCount = 1,
    .pSignalSemaphores = &*renderFinished};

_device->graphicsQueue().submit(submitInfo, *_drawFence);
```

This submission connects three things:

- **Input dependency:** wait on `_imageAvailable` at the specified destination stage.
- **Work:** execute the recorded command buffer.
- **Completion notifications:** once the submitted rendering commands have completed, the device signals `renderFinished` for presentation; the submission fence `_drawFence` lets the host establish completion before reusing resources.

The wait stage is **color attachment output**, where this draw writes its attachment. It is not a blanket stall of the whole GPU: earlier independent processing may proceed while the acquired image becomes ready.

The fence and semaphore serve different consumers. The submission call can return while work is pending; either signal may also already have happened by the time the CPU checks. **The call returning is not the completion condition we rely on.** The fence does not mean the monitor has displayed the image.

### 5. Present the same image

```cpp
const vk::PresentInfoKHR presentInfo{
    .waitSemaphoreCount = 1,
    .pWaitSemaphores = &*renderFinished,
    .swapchainCount = 1,
    .pSwapchains = &*_swapchain->handle(),
    .pImageIndices = &imageIndex};

if (_device->graphicsQueue().presentKHR(presentInfo)
        != vk::Result::eSuccess)
    throw std::runtime_error("failed to present a swap-chain image");
```

The presentation request waits on the semaphore signaled by rendering, then presents **the same image index** we acquired and recorded into.

Calling `presentKHR()` on `graphicsQueue()` works here because `Device` explicitly chooses a queue family supporting **both graphics and presentation**. It is a property of this sample's device selection, not a guarantee that every graphics queue can present.

## The main loop and shutdown

With `drawFrame()` defined, the loop is pleasantly small:

```cpp
void run()
{
    while (!_window->shouldClose())
    {
        _window->pollEvents();
        drawFrame();
    }

    _device->logicalDevice().waitIdle();
}
```

The CPU processes window events and drives the acquire/record/submit/present cycle. The fence limits this implementation to **one graphics submission in flight**, though presentation can still have its own pending work.

On the normal exit path, the final `waitIdle()` waits for outstanding device queue work before cleanup. It is not called every frame: doing that would impose a much broader stall than the per-submission fence we actually need. Exception paths need their own safe cleanup strategy; RAII alone does not make destroying resources that are still in use safe.

### What this first version does not handle yet

The window is non-resizable, and the sample treats non-success acquisition and presentation results as errors. A non-resizable window does not guarantee the swapchain can never become out of date.

A more complete renderer must handle **out-of-date/suboptimal swapchains**, recreation, and safe resource retirement. When adding recovery paths, the fence-reset placement also matters: **do not reset the fence and then skip the submission that would signal it**, or a later frame can wait forever. The current sample throws on those results rather than recovering and continuing. `eSuboptimalKHR` is not the same as failed acquisition: an image can still have been acquired, so a recovery path must account for its synchronization as well.

Multiple frames in flight and more advanced synchronization are later improvements. This chapter's goal is the first complete path from recorded work to a visible image.

## Finally, our first Vulkan triangle!

After all that setup, this is the result:

![Our first Vulkan triangle rendered with interpolated vertex colors on a black background](/assets/img/vulkan-first-triangle.png)
_The screenshot from my rendering-and-presentation sample._

A red vertex at the top, green on the right, blue on the left, with interpolated colors filling the triangle. A tiny image—but a very satisfying milestone.

The pipeline, shaders, image views, command recording, and swapchain were not isolated exercises after all. They are now parts of one working frame:

```text
Wait for safe reuse -> acquire -> record -> submit -> present -> repeat
```

The two synchronization tools answer different questions:

- **Semaphores:** when may the next asynchronous operation use this image?
- **Fence:** when may the CPU reuse the previous submission's resources?

**Finally—we have put our first triangle on the screen using Vulkan.**
