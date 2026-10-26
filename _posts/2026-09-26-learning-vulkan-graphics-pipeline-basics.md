---
title: "Episode 11: Graphics Pipeline Basics"
date: 2026-09-26 15:00:00 +0900
categories: [Vulkan]
tags: [vulkan, graphics, gpu, c++]
description: A beginner-friendly tour of the graphics pipeline and the problem each stage solves.
---

Today I read the [Introduction to Graphics Pipeline Basics](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/02_Graphics_pipeline_basics/00_Introduction.html) from the Vulkan Tutorial. The next part of the tutorial starts building a pipeline that can draw our first triangle.

Before writing that code, I wanted to understand the larger picture. The **graphics pipeline** is a sequence of operations that turns **vertex data** into **pixels** in a **framebuffer**. That sentence sounds simple, but the diagram contains many unfamiliar stages:

<object type="image/svg+xml" data="{{ '/assets/img/vulkan-graphics-pipeline.svg?v=2' | relative_url }}" width="100%" height="auto" style="max-width: 100%;" aria-label="Diagram showing the Vulkan graphics pipeline from vertex buffers to the framebuffer">
  <a href="{{ '/assets/img/vulkan-graphics-pipeline.svg?v=2' | relative_url }}">View the graphics pipeline diagram</a>
</object>

The pipeline is easier to understand if we keep asking two questions:

1. What problem does this stage solve?
2. What information does it pass to the next stage?

Some stages have a predefined job and are configured with Vulkan structures. These are usually called **fixed-function stages**. Other stages run shader programs that we provide. Those are the **programmable stages**.

## The problem: a GPU needs a repeatable route from data to pixels

A 3D model is usually stored as data: positions, colours, texture coordinates, normals, and indices. A display does not understand any of those things. It needs a grid of pixels, with a final colour for each pixel.

The pipeline is the route between those two forms:

```text
description of objects  ── graphics pipeline ──  pixels on screen
```

The GPU performs this work for many vertices and fragments in parallel. That parallelism is why a GPU can process a large number of small operations efficiently. It also explains why the stages have such specific responsibilities: each stage is designed to do one kind of work repeatedly and in parallel.

## 1. **Input assembly**: how should the data be grouped?

The **input assembler** reads vertex data from **buffers**. It uses the information in the graphics pipeline to decide how those vertices form **primitives** such as points, lines, or triangles.

For example, if the topology is `eTriangleList`, every group of three vertices becomes one independent triangle:

```text
vertices:  0  1  2  3  4  5
triangles: [0  1  2] [3  4  5]
```

### What problem does it solve?

The GPU receives a stream of values, but it needs to know what those values mean. Are three positions one triangle? Are the vertices connected as a strip? Is an index buffer being used to reuse a vertex?

The input assembler supplies that interpretation. An index buffer can also avoid repeating identical vertex data:

```text
vertex buffer:  A  B  C  D
index buffer:  0  1  2  2  3  0
triangles:    [A  B  C] [C  D  A]
```

Without indices, the shared vertices would need to be copied into the vertex buffer.

### Is it programmable?

No shader code runs here. We configure the input binding, attribute descriptions, and primitive topology. The hardware then fetches and groups the data according to those settings.

## 2. **Vertex shader**: where should each vertex go?

The **vertex shader** runs once for every vertex. Its most important job is to transform a vertex position through the **coordinate systems** used by the renderer.

A model may begin with positions in **model space**, where the coordinates describe the object around its own origin. The renderer then applies transformations for the object's placement, the camera, and the projection:

```text
model space
     ↓ model transform
world space
     ↓ view / camera transform
view space
     ↓ projection transform
clip space
```

The **vertex shader** reports the **transformed position** of each vertex to the later stages. It can also pass along values such as **colour**, **texture coordinates**, and **surface direction** for later use.

### What problem does it solve?

The original vertex data does not yet know where the camera is or how the 3D scene should be projected onto the 2D screen. The vertex shader performs those calculations and can also prepare data needed for lighting and texturing.

### Is it programmable?

Yes. This is one of the stages where we provide our own shader code. The GPU runs that code for many vertices in parallel.

## 3. **Tessellation shaders**: can the mesh be made denser?

**Tessellation** takes a coarse patch of geometry and subdivides it into smaller pieces. A simple mesh can therefore become more detailed near the camera without storing every small triangle in the original model.

Imagine a large quad representing a surface:

```text
before tessellation        after tessellation

    A──────B                    A──┬──┬──B
    │      │                    ├──┼──┼──┤
    │      │                    ├──┼──┼──┤
    D──────C                    D──┴──┴──C
```

The additional vertices can then be moved to create a curved or detailed surface. Tessellation is useful for terrain, curved surfaces, and displacement mapping.

### What problem does it solve?

There is a trade-off between mesh detail and memory or processing cost. A permanently dense mesh has enough triangles everywhere, even where they are not visible or not needed. Tessellation allows the application to start with fewer control points and increase detail according to a rule.

### Is it programmable?

Yes, when enabled. Tessellation uses programmable shader stages, but it is optional. A simple triangle does not need it, so we can leave these stages out of the pipeline.

## 4. **Geometry shader**: can each primitive be changed or expanded?

The **geometry shader** runs once for each primitive: a point, line, or triangle. It can discard that primitive, modify its output, or produce additional primitives.

For example, a point could be expanded into a small screen-facing quad, or a triangle could be duplicated and emitted in several directions.

```text
one input primitive  ── geometry shader ──  zero, one, or many primitives
```

### What problem does it solve?

Sometimes an application wants to make a decision about a whole primitive rather than one vertex. The geometry shader provides a flexible place to do that. It can also generate simple geometry from a smaller input.

### Is it programmable?

Yes, but it is optional. Although it is flexible, geometry shaders are not always fast on modern GPUs, and many use cases are better handled by other techniques, including mesh-shader pipelines. For now, our triangle does not need a geometry shader.

## 5. **Rasterization**: which fragments does the primitive cover?

At this point the GPU has transformed vertices and assembled primitives. **Rasterization** converts those mathematical primitives into **fragments**.

A fragment is a candidate piece of a final pixel. It contains a location and interpolated values from the primitive's vertices. For a triangle with different colours at its corners, rasterization can calculate a smoothly changing colour across its surface:

<object type="image/svg+xml" data="{{ '/assets/img/vulkan-rasterization-triangle.svg' | relative_url }}" width="100%" height="auto" style="max-width: 480px;" aria-label="A triangle with different colours at its vertices and interpolated fragment colours across its surface">
  <a href="{{ '/assets/img/vulkan-rasterization-triangle.svg' | relative_url }}">View the rasterization triangle diagram</a>
</object>

The stage also removes geometry outside the view and participates in tests such as culling and depth testing, depending on the pipeline configuration.

### What problem does it solve?

The screen is a grid of pixels, but a triangle is continuous geometry. Rasterization answers: which screen locations does this triangle cover, and what values should be interpolated at those locations?

It is also wasteful to shade parts of a primitive that are outside the viewport or hidden behind another object. Rasterization and related tests help reduce that work.

### Is it programmable?

The core operation is fixed-function, but we configure important behaviour such as the viewport, scissor rectangle, polygon mode, face culling, and depth testing. We do not upload a rasterization algorithm of our own.

## 6. **Fragment shader**: what colour should each fragment have?

The **fragment shader** runs for fragments that survive the earlier stages. It determines the values written to the **framebuffer**, usually including colour and sometimes depth.

For the simplest triangle, a fragment shader can assign the same colour to every fragment.

<object type="image/svg+xml" data="{{ '/assets/img/vulkan-fragment-shader.svg' | relative_url }}" width="100%" height="auto" style="max-width: 600px;" aria-label="Diagram showing fragments entering a fragment shader and leaving as framebuffer colours">
  <a href="{{ '/assets/img/vulkan-fragment-shader.svg' | relative_url }}">View the fragment shader diagram</a>
</object>

In a more realistic renderer, it might sample a texture, calculate lighting from a normal, combine material properties, or perform more advanced effects. It receives interpolated values from the vertex shader, so a texture coordinate or colour can vary smoothly across a triangle.

### What problem does it solve?

The rasterizer tells us where a primitive covers the screen, but it does not know what the surface should look like. The fragment shader turns each covered location into a colour and gives the application its main opportunity to describe the appearance of an object.

### Is it programmable?

Yes. This is the other shader stage required for the first triangle in the Vulkan Tutorial. The GPU runs it many times, often once for every covered fragment.

## 7. **Color blending**: what happens when several fragments share a pixel?

More than one primitive can produce a **fragment** for the same pixel. The final stage combines a new fragment with the colour already stored in the **framebuffer**.

For opaque objects, the new colour may simply replace the old colour after depth testing. For transparent objects, the result can be a mixture:

<object type="image/svg+xml" data="{{ '/assets/img/vulkan-color-blending.svg' | relative_url }}" width="100%" height="auto" style="max-width: 600px;" aria-label="Diagram showing source and destination colours combined into a final framebuffer colour">
  <a href="{{ '/assets/img/vulkan-color-blending.svg' | relative_url }}">View the color blending diagram</a>
</object>

An intuitive alpha-blending rule is:

```text
result = source × alpha + destination × (1 - alpha)
```

### What problem does it solve?

The pipeline can produce several candidate colours for one pixel, but the framebuffer stores one final value for that location. Blending defines how those values are combined. Without it, drawing a transparent object would usually just overwrite whatever was already there.

### Is it programmable?

The blending operation is fixed-function, but we configure its parameters: whether blending is enabled and how source and destination colours are combined. In Vulkan, these settings are part of the graphics pipeline configuration.

## Fixed-function versus programmable stages

The distinction is useful, but “fixed-function” does not mean “unimportant” or “unchangeable.” It means that Vulkan provides the algorithm and we choose from its configuration options.

| Stage | Type | Our role |
|---|---|---|
| **Input assembly** | Fixed-function | Describe buffer layout and topology |
| **Vertex shader** | Programmable | Provide shader code |
| **Tessellation shaders** | Programmable | Provide shaders when needed |
| **Geometry shader** | Programmable | Provide a shader when needed |
| **Rasterization** | Fixed-function | Configure viewport, culling, depth, and more |
| **Fragment shader** | Programmable | Provide shader code |
| **Color blending** | Fixed-function | Configure blend operations |

The **programmable stages** are where we write code that runs on the GPU. The **fixed-function stages** are where we describe how Vulkan's built-in operations should behave.

## Vulkan pipelines are configured ahead of time

In some older graphics APIs, it is common to change rendering state with individual calls while drawing. Vulkan takes a more explicit approach. A **graphics pipeline** packages together the shaders and much of the fixed-function configuration needed to draw.

The purpose is to give the GPU a complete rendering plan before the draw call begins. The plan includes how vertex data is read, which shader programs run, how geometry is converted into fragments, how depth and culling behave, how colours are blended, and which render targets receive the results. Vulkan can then prepare and validate that combination instead of discovering the settings one change at a time while rendering.

If we want a substantially different combination—for example, a different shader or blend mode—we generally create another pipeline. This requires more setup, but it also gives the driver more information in advance and can make the rendering work more predictable.

Not every pipeline needs every optional stage. A basic triangle can skip tessellation and geometry shaders. A depth-only pass can omit the fragment shader. The pipeline should contain the stages required by the rendering task, not every stage that Vulkan makes available.

## What I need to remember for now

I am not a computer graphics professional, and this was a lot of information to absorb at once. For now, I think it is enough to remember the broad movement:

```text
vertices
  → transform them
  → optionally create more geometry
  → turn geometry into covered fragments
  → calculate fragment colours
  → combine colours into the framebuffer
```

The next hands-on chapters will make these ideas less abstract. We will create the vertex and fragment shader modules first, then configure the fixed-function parts of the pipeline, and finally connect the pipeline to the framebuffer.

The important thing is not to memorize every field of every Vulkan structure yet. It is to understand which problem each stage is solving and which stages are programmable. The implementation details will make much more sense once we start configuring the pipeline ourselves.
