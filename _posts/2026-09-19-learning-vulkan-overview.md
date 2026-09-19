---
title: "[Lesson 1] Learning Vulkan — Why It Exists and How a Triangle Gets Drawn"
date: 2026-09-19 12:00:00 +0900
categories: [Graphics]
tags: [vulkan, graphics]
description: A beginner-friendly overview of Vulkan's design tradeoffs and the high-level flow of rendering a triangle.
---

Today I started reading the [Vulkan Tutorial](https://docs.vulkan.org/tutorial/latest/01_Overview.html). The first chapter does not immediately ask us to write a complete rendering program. Instead, it explains why Vulkan exists and gives us a high-level view of what is required to render a triangle.

That overview helped me understand Vulkan as an API design choice rather than simply a newer version of OpenGL.

## Why Vulkan?

**Vulkan is not always the best graphics API. The right choice depends on the problem we are trying to solve.**

OpenGL provides a relatively high-level and convenient programming model. Many features feel like turning a switch on or off. We can configure a small number of things, issue a command, and let the driver handle much of the underlying work.

This convenience has an important benefit: it can reduce development time and help a team deliver an application more quickly. For many projects, that is exactly what we want.

The tradeoff is that the driver has to guess more about our intentions. It must translate our relatively abstract commands into operations that work efficiently on the specific GPU. As hardware becomes more complex and different vendors use different architectures, this translation can introduce overhead and inconsistencies.

Vulkan takes a different approach. It exposes more of the underlying GPU workflow and asks the programmer to describe more things explicitly. Instead of hiding most of the details behind a convenient abstraction, Vulkan gives the application more responsibility and more control.

One way to imagine the difference is starting a car.

With a simple interface, we might only need to turn on the engine, choose a direction, and press the accelerator. The car handles the gearbox, engine settings, and temperature control for us.

With Vulkan, it is more like being asked to configure several of those systems before driving. An ordinary driver might feel overwhelmed, but an experienced driver may appreciate the additional flexibility and control.

**OpenGL prioritizes convenience, while Vulkan prioritizes explicit control.**

Vulkan is designed for modern graphics hardware. It aims to reduce driver overhead, support more effective multithreading, make shader compilation more consistent, and unify graphics and compute operations in one API.

The cost is verbosity. Vulkan requires us to be much more precise about what we want the GPU to do.

## The Big Picture: Rendering a Triangle

At first glance, drawing a triangle sounds like one simple operation. At the GPU level, however, many pieces must be prepared before the draw command can run.

**The rendering flow can be understood as four broad stages: prepare the GPU, prepare the window images, describe the drawing process, then record and submit commands.**

### 1. Prepare the GPU

First, the application creates a Vulkan instance and checks which physical devices are available.

A physical device represents an actual GPU. We can inspect its capabilities, supported features, and memory before choosing which device to use.

After choosing a physical device, we create a logical device. This is the interface that our application will use to communicate with the selected GPU.

We also select queues. Queues are where commands are submitted for execution. Different queue families may support graphics, computation, or memory-transfer operations.

At this stage, we are answering an important question: **which GPU will we use, and which operations can it perform for us?**

### 2. Prepare the Window and Images

If we want the triangle to appear in a window, Vulkan needs a connection between the GPU and the window system. The application creates a window surface and a swap chain.

The swap chain contains a collection of images that can be rendered to and eventually displayed on the screen. While one image is being shown, another image can be used for rendering. This prevents the screen from displaying an incomplete frame.

<p align="center">
  <object type="image/svg+xml" data="{{ '/assets/img/vulkan-swapchain.svg' | relative_url }}" width="100%" height="auto" style="max-width: 100%; aspect-ratio: 960 / 430;" aria-label="Swap chain: the GPU renders to one buffer while the display reads another">
    <a href="{{ '/assets/img/vulkan-swapchain.svg' | relative_url }}">View the swap-chain diagram</a>
  </object>
</p>

The basic cycle is:

1. Acquire an available image from the swap chain.
2. Render the next frame into that image.
3. Return the completed image for presentation.

This is why the swap chain is an important part of real-time rendering. It manages the relationship between images being rendered and images being displayed.

### 3. Describe How Drawing Should Work

Next, we describe the rendering process itself.

The graphics pipeline contains the configuration required to transform drawing commands into pixels. It includes programmable stages such as shaders, as well as settings such as the viewport and depth behavior.

One important characteristic of Vulkan is that much of this state must be specified in advance. If we change a shader or modify the way vertex data is laid out, we may need to create a new pipeline.

This may seem inconvenient, but it also makes the rendering process more explicit and predictable. The driver has more information ahead of time and can optimize the pipeline accordingly.

Modern Vulkan also supports dynamic rendering. Instead of always preparing predefined render passes and framebuffers, we can specify the rendering attachments when recording commands. This simplifies some parts of the setup while preserving Vulkan's explicit model.

### 4. Record, Submit, and Synchronize Commands

Vulkan does not usually execute a drawing operation immediately when we call a function. Instead, we record operations into a command buffer.

A command buffer describes what the GPU should do. For a simple triangle, it may contain instructions to:

1. Begin rendering.
2. Bind the graphics pipeline.
3. Draw three vertices.
4. Finish rendering.

After recording the command buffer, we submit it to a queue.

Because GPU operations are asynchronous, we also need synchronization. The application must ensure that the swap-chain image is available before rendering begins, and that rendering is complete before the image is presented.

The main loop therefore looks roughly like this:

1. Acquire an image from the swap chain.
2. Submit the recorded drawing commands.
3. Wait for rendering to finish.
4. Present the completed image.
5. Repeat for the next frame.

## The Main Idea

The triangle itself is simple. **Preparing the GPU workflow around it is the real introduction to Vulkan.**

Before issuing one draw command, we need to choose and configure a GPU, select queues, connect Vulkan to a window, create a swap chain, prepare image views, describe the graphics pipeline, record command buffers, submit work to the GPU, and synchronize rendering with presentation.

This is why Vulkan can feel more complicated than OpenGL. OpenGL hides more of these decisions behind a higher-level abstraction. Vulkan exposes them so that the application has more direct control over the hardware.

For me, the most useful way to understand Vulkan is not to think of it as “better OpenGL.” It is better understood as a feature-full but verbose API designed for situations where control, predictability, and efficient use of modern hardware are important.

The first triangle is only a small drawing operation. The real lesson is learning how to prepare the entire GPU workflow that makes that operation possible.

*This post is based on the overview chapter of the [Vulkan Tutorial](https://docs.vulkan.org/tutorial/latest/01_Overview.html). The detailed explanations of each step will come in later posts.*
