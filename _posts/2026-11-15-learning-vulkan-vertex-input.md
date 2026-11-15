---
title: "Episode 19: Vertex Input — Giving the Triangle Its Data"
date: 2026-11-15 13:00:00 +0900
categories: [Vulkan]
tags: [vulkan, vertex-input, vertex-buffer, shaders, c++]
description: "Moving a triangle's positions and colors out of the shader and explaining how Vulkan reads vertex data from a buffer."
---

Today I followed the Vulkan Tutorial's [Vertex input description chapter](https://docs.vulkan.org/tutorial/latest/04_Vertex_buffers/00_Vertex_input_description.html) and implemented [16_vertex_input](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/16_vertex_input) in my learning repository.

Until now, the shader itself held the triangle's positions and colors. That was convenient for a first drawing, but it is not how we want to describe real objects. A model may contain thousands of vertices, and its data can change while the application runs. We should keep that data in the program and send it to the GPU as input to the shader.

In this post, we will build the idea from one **vertex**, see how our C++ program describes three of them, and then tell Vulkan how the data is laid out.

## Moving the data out of the shader

Our earlier vertex shader kept two arrays. The position at index `i` and color at index `i` together described one point:

~~~hlsl
static float2 positions[3] = float2[](
    float2(0.0, -0.5),
    float2(0.5, 0.5),
    float2(-0.5, 0.5)
);

static float3 colors[3] = float3[](
    float3(1.0, 0.0, 0.0),
    float3(0.0, 1.0, 0.0),
    float3(0.0, 0.0, 1.0)
);
~~~

The shader needed the built-in vertex ID to look up both arrays. This worked for a fixed triangle, but the geometry was baked into the shader binary. Changing a point meant editing and recompiling the shader.

Now the shader declares the data it expects for each invocation:

~~~hlsl
struct VertexInput
{
    [[vk::location(0)]] float2 position;
    [[vk::location(1)]] float3 color;
};
~~~

This structure still has two properties: a **position** and a **color**. The `vk::location` annotations give the fields input locations. Our C++ pipeline configuration will map data at location 0 to `position`, and data at location 1 to `color`.

The shader no longer chooses an array element using `SV_VertexID`. For every vertex the GPU processes, Vulkan supplies one `VertexInput` value. The vertex shader turns its position into clip space and passes its color along:

~~~hlsl
[shader("vertex")]
VertexOutput vertMain(VertexInput input)
{
    VertexOutput output;
    output.sv_position = float4(input.position, 0.0, 1.0);
    output.color = input.color;
    return output;
}
~~~

## One vertex has two properties

Think of a **vertex** as one record describing a point in the geometry. Our record has a two-number position and a three-number RGB color. The GPU reads the position and color together for each point.

![Three vertex records, each containing a position and color]({{ '/assets/img/vulkan-vertex-input-tree.svg' | relative_url }}?v=2)

Three vertex records form our triangle. The individual corner colors still get interpolated across the triangle by the rasterizer, just as before. We have changed where the vertex values come from, not what the triangle looks like.

## Defining the vertices in C++

Our implementation defines a C++ type representing one complete vertex in [`triangleExample.hpp`](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/blob/main/src/basic/16_vertex_input/include/triangleExample.hpp):

~~~cpp
#include <glm/glm.hpp>

struct TriangleVertex
{
    glm::vec2 position;
    glm::vec3 color;
};
~~~

The matching Slang types are `float2` and `float3`: two floats for position and three floats for color. Our implementation uses GLM's `glm::vec2` and `glm::vec3` to express those values naturally. The vertex attributes still describe the data as two and three 32-bit floats, and `offsetof` gets the field offsets from the actual structure. This relies on the GLM configuration used by the project matching the formats we declare; enabling aligned GLM gentypes can change vector alignment and structure layout, so the offsets and stride must always match the compiled types.

The example also links the GLM CMake target (`glm::glm`) so the compiler can find the library's headers.

Our triangle is an array of three `TriangleVertex` values in `main.cpp`:

{% raw %}
~~~cpp
inline static const std::array<TriangleVertex, 3> vertices = {{
    {{ 0.0F, -0.5F}, {1.0F, 0.0F, 0.0F}},
    {{ 0.5F,  0.5F}, {0.0F, 1.0F, 0.0F}},
    {{-0.5F,  0.5F}, {0.0F, 0.0F, 1.0F}}}};
~~~
{% endraw %}

The first record puts a red vertex at `(0.0, -0.5)`, the second puts a green vertex at `(0.5, 0.5)`, and the third puts a blue vertex at `(-0.5, 0.5)`. These are the same values as in the old shader arrays, now declared in our program.

The application uploads the array to a **vertex buffer**:

~~~cpp
_vertexBuffer = std::make_unique<VertexBuffer>(
    *_device, vertices.data(), sizeof(vertices));
~~~

The buffer is GPU input storage. The C++ array provides the initial values; the buffer holds the bytes that the GPU will read when drawing. In our sample, `VertexBuffer` creates a buffer with `eVertexBuffer` usage, allocates host-visible coherent memory, and copies the array into it.

## Telling Vulkan how to read the bytes

The bytes in a buffer have no built-in meaning. Vulkan needs a description of how to step through the records and where to find each shader input. Two pieces of that description are a **binding description** and **attribute descriptions**.

### `vk::VertexInputBindingDescription`

A `vk::VertexInputBindingDescription` describes a stream of vertex records:

~~~cpp
static vk::VertexInputBindingDescription bindingDescription()
{
    return {
        .binding = 0,
        .stride = sizeof(TriangleVertex),
        .inputRate = vk::VertexInputRate::eVertex};
}
~~~

* **`binding`** is the binding number for this stream. Our vertex attributes refer to binding `0`.
* **`stride`** is the byte distance from the start of one vertex record to the next. `sizeof(TriangleVertex)` means Vulkan advances by one complete position-and-color record.
* **`inputRate`** says how often to advance. `eVertex` advances for every vertex. Vulkan also has an instance rate for data that advances per instance when drawing instanced objects.

The binding description describes the whole record stream. It does not say which bytes are position or color; that is the job of the attribute descriptions.

### Attribute descriptions

Each `vk::VertexInputAttributeDescription` connects one shader location to a field in a bound record. Our implementation returns two:

{% raw %}
~~~cpp
static std::array<vk::VertexInputAttributeDescription, 2> attributeDescriptions()
{
    return {{
        {.location = 0, .binding = 0,
         .format = vk::Format::eR32G32Sfloat,
         .offset = offsetof(TriangleVertex, position)},
        {.location = 1, .binding = 0,
         .format = vk::Format::eR32G32B32Sfloat,
         .offset = offsetof(TriangleVertex, color)}}};
}
~~~
{% endraw %}

For **location 0**, Vulkan reads two 32-bit floats (`eR32G32Sfloat`) at the byte offset of `position`. For **location 1**, it reads three 32-bit floats (`eR32G32B32Sfloat`) at the offset of `color`. `offsetof` calculates those offsets from the actual C++ structure instead of making us guess a number.

Together, the stride and offsets tell Vulkan how the data is arranged:

~~~text
vertex 0: [ position (2 floats) ][ color (3 floats) ]
vertex 1: [ position (2 floats) ][ color (3 floats) ]
             <------ stride: sizeof(TriangleVertex) ------>
~~~

This arrangement is called **interleaved** data: each vertex's position and color are stored together before the next vertex begins.

### `vk::PipelineVertexInputStateCreateInfo`

`vk::PipelineVertexInputStateCreateInfo` brings the binding and attribute descriptions together as part of the graphics pipeline's vertex-input state. In our `graphicsPipeline.cpp`, we prepare the descriptions and point the Vulkan structure at them:

~~~cpp
const auto bindingDescription = TriangleVertex::bindingDescription();
const auto attributeDescriptions = TriangleVertex::attributeDescriptions();

const vk::PipelineVertexInputStateCreateInfo vertexInput{
    .vertexBindingDescriptionCount = 1,
    .pVertexBindingDescriptions = &bindingDescription,
    .vertexAttributeDescriptionCount =
        static_cast<uint32_t>(attributeDescriptions.size()),
    .pVertexAttributeDescriptions = attributeDescriptions.data()};
~~~

The structure says there is one vertex stream and two attributes described by those arrays. When creating the graphics pipeline, we pass it as `pVertexInputState`:

~~~cpp
.pVertexInputState = &vertexInput,
~~~

This configuration tells the pipeline **how to interpret vertex data**. At draw time, the command buffer binds the actual vertex buffer at binding 0. The binding and attribute descriptions then tell the GPU how to read each vertex from it and deliver the fields to the matching shader locations.

## The result

The vertex shader now takes its `position` and `color` from `VertexInput`; it no longer contains arrays of three hardcoded values. Our application defines three `TriangleVertex` records, uploads them to a vertex buffer, and the pipeline explains how those records match the shader inputs.

The resulting triangle looks as it did before: three colored corners with interpolated colors between them. The difference is that its vertex data now lives in the program and arrives through a **vertex buffer**. This gives us the foundation for supplying larger, changeable meshes without rebuilding the shader every time their data changes.

The next step is to explore how vertex buffers can be indexed and how a mesh can reuse vertices instead of listing each triangle's points repeatedly.
