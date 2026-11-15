---
title: "Vulkan Triangle: A Concept Map"
date: 2026-11-15 12:00:00 +0900
categories: [Vulkan]
tags: [vulkan, graphics, learning-notes]
description: A categorized map of the concepts behind drawing and resizing a Vulkan triangle, with links to each lesson.
---

It took a surprising amount of code to draw a triangle in a resizable window. That code is easier to revisit when we group the ideas by the job they do. This table is a map of the concepts covered so far, with a few key terms for each one and a link to the lesson that explains it.

## Concept map

| Category | Concept and key words | Related blog |
|---|---|---|
| Foundations | **Why Vulkan**<br>Explicit control, driver responsibility, graphics API tradeoffs | [Why Vulkan and the rendering overview]({{ '/posts/learning-vulkan-overview/' | relative_url }}) |
| Foundations | **Development environment**<br>Vulkan SDK, CMake, GLFW, dependencies, build | [Building the learning environment]({{ '/posts/learning-vulkan-development-environment/' | relative_url }}) |
| Window and instance | **Window and event loop**<br>GLFW, OS window, input, window lifetime | [Creating a window and Vulkan instance]({{ '/posts/learning-vulkan-window-and-instance/' | relative_url }}) |
| Window and instance | **Vulkan instance**<br>Loader, application info, instance extensions, object lifetime | [Creating a window and Vulkan instance]({{ '/posts/learning-vulkan-window-and-instance/' | relative_url }}) |
| Debugging | **Validation layers**<br>`VK_LAYER_KHRONOS_validation`, valid usage, debug messenger, callback | [Adding Vulkan validation layers]({{ '/posts/learning-vulkan-validation-layers/' | relative_url }}) |
| Device selection | **Physical device**<br>GPU enumeration, properties, features, limits, suitability | [Checking physical-device capabilities]({{ '/posts/learning-vulkan-physical-device-capabilities/' | relative_url }}) |
| Device selection | **Queue families**<br>Graphics queue, presentation support, queue index | [Checking physical-device capabilities]({{ '/posts/learning-vulkan-physical-device-capabilities/' | relative_url }}) |
| Device setup | **Logical device**<br>Requested features, enabled device extensions, queue handles | [Creating a logical device]({{ '/posts/learning-vulkan-logical-device-and-queues/' | relative_url }}) |
| Window presentation | **Window surface**<br>`VkSurfaceKHR`, WSI, platform-specific surface extensions, GLFW bridge | [Creating a Vulkan window surface]({{ '/posts/learning-vulkan-window-surface/' | relative_url }}) |
| Window presentation | **Surface support**<br>Presentation-capable queue, surface format, present mode | [Creating a Vulkan window surface]({{ '/posts/learning-vulkan-window-surface/' | relative_url }}) |
| Window presentation | **Swapchain**<br>Presentable images, image format, extent, image count, FIFO | [Creating a Vulkan swapchain]({{ '/posts/learning-vulkan-swapchain/' | relative_url }}) |
| Window presentation | **Image views**<br>Image resource, view format, color aspect, subresource range | [Creating Vulkan image views]({{ '/posts/learning-vulkan-image-views/' | relative_url }}) |
| Code structure | **Refactoring setup**<br>RAII, ownership, class responsibilities, separation of concerns | [Refactoring the Vulkan setup]({{ '/posts/learning-vulkan-refactor/' | relative_url }}) |
| Graphics pipeline | **Pipeline stages**<br>Vertex input, vertex shader, rasterization, fragment shader, attachment | [Graphics pipeline basics]({{ '/posts/learning-vulkan-graphics-pipeline-basics/' | relative_url }}) |
| Shaders and compilation | **Shader**<br>Vertex and fragment stages, inputs, outputs, entry point | [Shader modules: Slang to Vulkan]({{ '/posts/learning-vulkan-shader-modules/' | relative_url }}) |
| Shaders and compilation | **Shader compilation**<br>Slang source, build step, SPIR-V bytecode, shader module | [Shader modules: Slang to Vulkan]({{ '/posts/learning-vulkan-shader-modules/' | relative_url }}) |
| Graphics pipeline | **Pipeline configuration**<br>Fixed-function state, dynamic state, viewport, rasterization | [Building a Vulkan graphics pipeline]({{ '/posts/learning-vulkan-graphics-pipeline/' | relative_url }}) |
| Rendering | **Render pass and framebuffer**<br>Attachments, subpasses, load/store operations | [Render passes and dynamic rendering]({{ '/posts/learning-vulkan-render-passes-and-dynamic-rendering/' | relative_url }}) |
| Rendering | **Dynamic rendering**<br>`vk::RenderingInfo`, attachment views, begin/end rendering | [Render passes and dynamic rendering]({{ '/posts/learning-vulkan-render-passes-and-dynamic-rendering/' | relative_url }}) |
| GPU work | **Command pool and command buffer**<br>Record, reset, submit, GPU work batch | [Understanding command buffers]({{ '/posts/learning-vulkan-command-buffers/' | relative_url }}) |
| GPU work | **Drawing and presentation**<br>Acquire image, record draw, submit, present | [Rendering, presentation, and the first triangle]({{ '/posts/learning-vulkan-rendering-and-presentation/' | relative_url }}) |
| Synchronization | **Semaphores and fences**<br>GPU-to-GPU ordering, CPU wait, image availability | [Rendering, presentation, and the first triangle]({{ '/posts/learning-vulkan-rendering-and-presentation/' | relative_url }}) |
| Synchronization | **Frames in flight**<br>Frame slots, swapchain images, per-frame synchronization | [Frames in flight]({{ '/posts/learning-vulkan-frames-in-flight/' | relative_url }}) |
| Resizing | **Swapchain recreation**<br>Framebuffer size, out-of-date result, dependent resources, rebuild | [Resizing the window and recreating the swapchain]({{ '/posts/learning-vulkan-swapchain-recreation/' | relative_url }}) |

## The whole flow in one line

**Window → instance and extensions → physical device and queues → logical device → surface and swapchain → image views → shaders and pipeline → command buffer → submit and synchronize → present.**

When the window size changes, recreate the swapchain and the resources that depend on its images. The [swapchain recreation lesson]({{ '/posts/learning-vulkan-swapchain-recreation/' | relative_url }}) follows that dependency chain.
