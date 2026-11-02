---
title: "Episode 17: Frames in Flight — Keeping the Work Moving"
date: 2026-11-02 00:00:00 +0900
categories: [Vulkan]
tags: [vulkan, synchronization, command-buffers, gpu, c++]
description: "Letting CPU preparation overlap GPU execution, while keeping frame slots and swapchain images separate."
---

Our triangle is finally visible. Now we can improve how we keep work moving.

Today I followed the Vulkan Tutorial's [Frames in flight chapter](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/03_Drawing/03_Frames_in_flight.html) and implemented [14_frame_in_flight](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/14_frame_in_flight).

The main idea is simple: **while the GPU renders one frame, let the CPU prepare another.** Understanding the code becomes easier once we distinguish a **frame slot** from a **swapchain image**.

The excerpts below come from my implementation, using Vulkan-Hpp RAII and dynamic rendering. Some comments are omitted for readability.

## Why the old loop made us wait

My previous example had one command buffer and one draw fence. Each iteration began with:

~~~cpp
if (_device->logicalDevice().waitForFences(
        *_drawFence, vk::True, UINT64_MAX) != vk::Result::eSuccess)
    throw std::runtime_error("failed to wait for draw fence");
~~~

The wait was necessary: we cannot record new commands into a command buffer while the GPU is still executing it.

But it meant the CPU could not record the next frame until the previous GPU submission finished. Then the GPU could have no new work while the CPU prepared that next frame.

![One frame slot: CPU preparation and GPU rendering take turns]({{ '/assets/img/vulkan-one-frame-timeline.svg' | relative_url }}?v=2)

_Illustrative scheduling, not measured timing. “Prepare” groups acquisition, command recording, and submission. Presentation is shown separately because it does not finish when the graphics fence signals._

The old loop did **not** necessarily wait for the monitor to finish displaying the previous image. Its fence tracked a graphics submission, not display completion.

That distinction matters. The chapter's primary improvement is **CPU/GPU overlap**, not simply “rendering while presenting.”

**We are not fully utilizing our CPU and GPU resources!**

## Let the CPU prepare the next frame while the GPU renders

We introduce two frame slots:

~~~cpp
constexpr uint32_t framesInFlight = 2;
~~~

A frame slot is a reusable set of resources for preparing and submitting work. It is not an image.

While the GPU executes slot 0's commands, the CPU can record slot 1's commands. When the CPU returns to slot 0, it waits only if slot 0's previous GPU submission is still running.

![Two frame slots: CPU preparation overlaps earlier GPU rendering]({{ '/assets/img/vulkan-two-frames-timeline.svg' | relative_url }})

_The GPU submissions are still drawn one after another. The new overlap is CPU preparation with GPU execution. Presentation of an earlier image may also overlap rendering into another image._

The benefit of multiple slots is that **the CPU and GPU no longer have to take turns using one set of resources**. Each slot has its own command buffer and synchronization objects, so preparing a later frame does not overwrite resources still used by an earlier submission.

Consider a simplified example: the CPU takes 4 ms to prepare each frame, and the GPU takes 6 ms to render it.

- **One slot:** prepare for 4 ms, render for 6 ms, then repeat. A frame takes roughly 10 ms of combined work before the next cycle can begin.
- **Two slots:** while the GPU renders A, the CPU spends 4 ms preparing B in the other slot. Once A finishes, B can already be queued and ready to render. After the pipeline fills, a completed frame can arrive roughly every 6 ms instead of every 10 ms.

These numbers ignore acquisition, presentation, and other overhead. They illustrate the benefit: **overlap can move the frame interval from the sum of CPU and GPU time toward whichever stage takes longer**. We have not made either stage faster; we have reduced the time they spend waiting for each other.

Additional slots can also absorb short timing variations. If one frame takes longer to prepare, having other prepared work queued may keep the GPU occupied during that delay. But more slots do not automatically improve performance: once two slots keep the slower stage supplied with work, extra slots may add no throughput benefit.

The tradeoff is more resources and potentially more latency. Each slot needs its own working resources, and allowing more frames to queue can increase the time between preparing a frame and seeing it on screen. Two slots are therefore a useful starting point: they allow overlap while limiting how far the CPU can get ahead of the GPU.

Multiple slots still do not guarantee continuous GPU activity. Acquisition can block, presentation can introduce delays, and the CPU or GPU may remain the limiting factor. We are removing an avoidable dependency, not every possible wait.

## Frame slots are not swapchain images

There are three participants:

- **CPU:** prepares and submits commands.
- **GPU:** executes those commands and writes pixels into an image.
- **Presentation system:** takes completed images and makes them visible.

Rendering proceeds as work and resources allow. Presentation follows the selected present mode and display system; it is not universally a simple fixed-rate handoff.

My code keeps two independent indices:

~~~cpp
_currentFrame  // Which reusable frame slot are we using?
imageIndex     // Which swapchain image did we acquire?
~~~

The frame slot cycles through 0, 1, 0, 1, and so on. The image index comes from acquisition. It need not match the frame slot, follow the same sequence, or even have the same range.

For example, with two slots and three images:

| Submission | Frame slot | Acquired image |
|---|---|---|
| A | 0 | 2 |
| B | 1 | 0 |
| C | 0 | 1 |
| D | 1 | 2 |

_An example only, not a promised acquisition order._

Thinking in terms of **N frame slots and M images** is useful, but we do not allocate an N-by-M grid of resources. Each resource belongs to the appropriate group:

| Resource | Count | Index |
|---|---|---|
| Command buffers | N | Frame slot |
| Image-available semaphores | N | Frame slot |
| Frame fences | N | Frame slot |
| Render-finished semaphores | M | Image |
| Last-submission fence references | M | Image |

## 1. Allocate command buffers per frame slot

Setup passes the slot count to my wrapper:

~~~cpp
_commandBuffers = std::make_unique<CommandBuffer>(*_device, framesInFlight);
~~~

Inside the wrapper, allocation becomes:

~~~cpp
const vk::CommandBufferAllocateInfo allocationInfo{
    .commandPool = *_pool,
    .level = vk::CommandBufferLevel::ePrimary,
    .commandBufferCount = framesInFlight};

_buffers = vk::raii::CommandBuffers(_device.logicalDevice(), allocationInfo);
~~~

Recording now receives both indices:

~~~cpp
void CommandBuffer::record(
    uint32_t frameIndex,
    uint32_t imageIndex,
    const Swapchain& swapchain,
    const GraphicsPipeline& graphicsPipeline)
~~~

The frame index selects the command buffer:

~~~cpp
const auto& commandBuffer = _buffers[frameIndex];
commandBuffer.begin({});
~~~

The image index selects the target image and image view inside the recorded commands.

My pool uses the flag allowing individual command-buffer resets. Beginning a fresh recording implicitly resets a previously recorded buffer, so my wrapper does not call reset separately. The fence wait ensures that buffer is no longer executing before we begin again.

The rendering commands themselves are unchanged: prepare the image layout, begin dynamic rendering, draw the triangle, then prepare the image for presentation.

## 2. Create synchronization objects for the right owner

Each frame slot gets an image-available semaphore and a fence:

~~~cpp
for (uint32_t frame = 0; frame < framesInFlight; ++frame)
{
    _imageAvailable.push_back(vk::raii::Semaphore(
        _device->logicalDevice(), vk::SemaphoreCreateInfo{}));
    _frameFences.push_back(vk::raii::Fence(
        _device->logicalDevice(),
        vk::FenceCreateInfo{.flags = vk::FenceCreateFlagBits::eSignaled}));
}
~~~

Their jobs are different:

- **Image-available semaphore:** tells submitted rendering work when the acquired image is ready.
- **Frame fence:** lets the CPU know when this slot's previous graphics submission has finished.

The fences start signaled because there is no previous submission on the first use.

Render-finished semaphores belong to images instead:

~~~cpp
for (std::size_t image = 0; image < _swapchain->images().size(); ++image)
    _renderFinished.emplace_back(
        _device->logicalDevice(), vk::SemaphoreCreateInfo{});
~~~

These connect rendering to presentation: presentation waits for the selected image's rendering to finish.

Why not index them by frame slot too? A graphics fence does not prove that presentation has finished using its wait semaphore. The [Vulkan Guide explains this reuse issue](https://docs.vulkan.org/guide/latest/swapchain_semaphore_reuse.html). The pattern here associates each semaphore with an image. Reacquiring that image, and waiting on acquisition in the next submission, orders reuse safely.

## 3. Wait for this slot, then acquire an image

At the beginning of the new loop:

~~~cpp
auto& frameFence = _frameFences[_currentFrame];

if (_device->logicalDevice().waitForFences(
        *frameFence, vk::True, UINT64_MAX) != vk::Result::eSuccess)
    throw std::runtime_error("failed to wait for frame fence");

auto [acquireResult, imageIndex] =
    _swapchain->handle().acquireNextImage(
        UINT64_MAX, *_imageAvailable[_currentFrame], nullptr);
if (acquireResult != vk::Result::eSuccess)
    throw std::runtime_error("failed to acquire a swap-chain image");
~~~

We wait for the submission that last used **this slot**, not necessarily the immediately previous submission.

After submitting slot 0, the CPU can prepare slot 1 without explicitly waiting on slot 0's fence. It checks slot 0 again when the slot index wraps around.

Acquisition returns the target image index. The image-available semaphore supplies the device-side readiness signal; receiving an index alone does not replace that synchronization.

## 4. Track the graphics submission associated with each image

My version also creates this table:

~~~cpp
_imagesInFlight.resize(_swapchain->images().size(), vk::Fence{});
~~~

It stores references to existing frame fences, not a new fence allocation for every image.

After acquisition:

~~~cpp
if (_imagesInFlight[imageIndex] &&
    _device->logicalDevice().waitForFences(
        _imagesInFlight[imageIndex], vk::True, UINT64_MAX)
        != vk::Result::eSuccess)
    throw std::runtime_error("failed to wait for swap-chain image fence");

_imagesInFlight[imageIndex] = *frameFence;
~~~

This is an extra CPU-side check for the previously tracked graphics submission.

It is additional bookkeeping in my implementation, not the mechanism that releases an image from presentation. Acquisition and its semaphore still provide that handoff.

For this simple sample, which has no additional CPU-modified per-image data, this check can be conservative. Fence objects are reused by frame slots, so an old image entry can even refer to a fence now tracking newer work. It remains a wait on graphics work, **not a presentation-completion signal**.

## 5. Record, reset the fence, and submit

We now have a slot to record into and an image to target:

~~~cpp
_commandBuffers->record(
    _currentFrame, imageIndex, *_swapchain, *_pipeline);

_device->logicalDevice().resetFences(*frameFence);
~~~

The fence is reset after acquisition and recording succeed. This ordering matters when adding recovery paths: resetting a fence and then returning without a submission could leave the next iteration waiting for a signal that will never arrive.

The submission connects the two resource groups:

~~~cpp
const vk::PipelineStageFlags waitStage =
    vk::PipelineStageFlagBits::eColorAttachmentOutput;
const auto& commandBuffer = _commandBuffers->buffer(_currentFrame);
auto& renderFinished = _renderFinished[imageIndex];

const vk::SubmitInfo submitInfo{
    .waitSemaphoreCount = 1,
    .pWaitSemaphores = &*_imageAvailable[_currentFrame],
    .pWaitDstStageMask = &waitStage,
    .commandBufferCount = 1,
    .pCommandBuffers = &*commandBuffer,
    .signalSemaphoreCount = 1,
    .pSignalSemaphores = &*renderFinished};

_device->graphicsQueue().submit(submitInfo, *frameFence);
~~~

Read it as:

1. Execute this frame slot's command buffer.
2. Wait for the image-available semaphore before color-output work proceeds.
3. Signal this image's render-finished semaphore when rendering is complete.
4. Signal this slot's fence when its submission is complete.

The semaphore wait is part of the submitted work. It does not make the CPU wait for the whole render to finish.

## 6. Request presentation and advance the slot

~~~cpp
const vk::PresentInfoKHR presentInfo{
    .waitSemaphoreCount = 1,
    .pWaitSemaphores = &*renderFinished,
    .swapchainCount = 1,
    .pSwapchains = &*_swapchain->handle(),
    .pImageIndices = &imageIndex};

if (_device->graphicsQueue().presentKHR(presentInfo)
    != vk::Result::eSuccess)
    throw std::runtime_error("failed to present a swap-chain image");

_currentFrame = (_currentFrame + 1) % framesInFlight;
~~~

Presentation waits on the same render-finished semaphore signaled by the graphics submission.

A successful presentation request does not mean the monitor has already displayed the pixels. The call can also introduce delays depending on the implementation and presentation behavior.

Once it returns, we advance to the next slot without explicitly waiting for the just-submitted fence.

The diagram below follows one iteration of my loop. **Time flows downward**, and the numbered CPU steps match the code above. Dashed arrows show which signal allows a wait to finish.

![Numbered frame-loop timeline showing CPU, GPU, and presentation wait dependencies]({{ '/assets/img/vulkan-frame-loop-dependencies.svg' | relative_url }}?v=5)

**Read the two kinds of waits separately:**

- **Steps 1 and 3:** the CPU checks fences before reusing resources. Step 3 is the additional per-image tracking check in my version.
- **Step 6:** the submitted GPU work waits for image availability. The CPU does not wait here for rendering to finish.
- **Step 7:** the presentation request carries a wait for rendering completion. The call returning does not mean the image is already on screen.
- **Step 8:** the CPU advances to the next slot without explicitly waiting for this new submission's fence. Preparing another frame can now overlap GPU work.

_The positions illustrate dependencies, not exact durations or guaranteed overlap. The initial fences may already be signaled, acquisition can block, and presentation calls can introduce delays._

## What actually changed from my previous example?

My previous sample already used one render-finished semaphore per image. That design stays.

The new pieces are:

- Command buffers, acquisition semaphores, and fences are allocated per frame slot.
- The current slot cycles independently of the acquired image index.
- Command recording receives both indices.
- A per-image table tracks frame-fence references.
- Fence reset happens after successful acquisition and recording.

The sample still treats non-success acquisition and presentation results as errors. Handling resizing and rebuilding the swapchain is separate work.

## The takeaway

Frames in flight do not mean “two frames must render simultaneously.”

They mean **several frames can be at different stages of preparation and execution without overwriting resources still in use**.

The main gain in this chapter is letting the CPU prepare the next frame while the GPU executes earlier commands. Rendering into another image may also overlap presentation, but the scheduling depends on image availability and the implementation.

Two questions keep the code clear:

> Which frame slot's working resources am I reusing?

> Which swapchain image am I rendering into?

Once those are separate, the synchronization objects have straightforward jobs: protect slot reuse, wait for image availability, and connect completed rendering to presentation.