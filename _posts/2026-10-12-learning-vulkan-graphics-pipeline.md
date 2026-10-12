---
title: "13. Building a Vulkan Graphics Pipeline"
date: 2026-10-12 20:00:00 +0900
categories: [Vulkan]
tags: [vulkan, graphics, gpu, c++]
description: Configuring Vulkan fixed-function state, dynamic state, dynamic rendering, and the final graphics pipeline.
---

Today I continued the Vulkan Tutorial's graphics-pipeline chapter, reading the sections on [fixed functions](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/02_Graphics_pipeline_basics/02_Fixed_functions.html), [dynamic rendering](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/02_Graphics_pipeline_basics/03_Dynamic_rendering.html), and the [final pipeline creation](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/02_Graphics_pipeline_basics/04_Conclusion.html). This was a large lesson because it moved from understanding the pipeline conceptually to describing almost every part of it with Vulkan structures.

The sample program for this lesson is [available on GitHub](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/11_graphic_pipeline).

The graphics pipeline is the route that transforms vertex data into fragments and eventually writes colours into an attachment. Vulkan makes this route explicit. We need to describe the programmable shader stages, configure the fixed-function stages, define the shader interface, and tell Vulkan which attachment formats the pipeline supports.

This post follows the order in which I learned the pieces:

1. the fixed-function stages;
2. dynamic state;
3. dynamic rendering; and
4. the final `vk::GraphicsPipelineCreateInfo`.

## 1. Vertex input

Vertex input answers a simple question: how should data from vertex buffers be interpreted before it reaches the vertex shader?

The Vulkan structure is `vk::PipelineVertexInputStateCreateInfo`. It describes two related things:

* **bindings** describe the stride between elements and whether data advances per vertex or per instance;
* **attributes** describe the shader location, format, binding, and byte offset of each field.

For a conventional vertex containing a position and colour, the setup might look like this:

```cpp
const vk::VertexInputBindingDescription binding{
    .binding = 0,
    .stride = sizeof(Vertex),
    .inputRate = vk::VertexInputRate::eVertex};

const std::array attributes{
    vk::VertexInputAttributeDescription{
        .location = 0,
        .binding = 0,
        .format = vk::Format::eR32G32B32Sfloat,
        .offset = offsetof(Vertex, position)},
    vk::VertexInputAttributeDescription{
        .location = 1,
        .binding = 0,
        .format = vk::Format::eR32G32B32Sfloat,
        .offset = offsetof(Vertex, color)}};

const vk::PipelineVertexInputStateCreateInfo vertexInput{
    .vertexBindingDescriptionCount = 1,
    .pVertexBindingDescriptions = &binding,
    .vertexAttributeDescriptionCount = static_cast<uint32_t>(attributes.size()),
    .pVertexAttributeDescriptions = attributes.data()};
```

My current sample has not introduced a vertex buffer yet. The Slang vertex shader generates the triangle from the vertex ID, so there is no buffer data to describe:

```cpp
const vk::PipelineVertexInputStateCreateInfo vertexInput{};
```

This empty structure is intentional. It tells Vulkan that the vertex shader does not need attributes fetched from a vertex buffer.

## 2. Input assembly

Input assembly determines how vertices are grouped into primitives. Vulkan provides topologies for points, lines, and triangles. The common choices include:

* `ePointList`: one point per vertex;
* `eLineList`: one independent line per pair of vertices;
* `eLineStrip`: each new vertex continues the previous line;
* `eTriangleList`: one independent triangle per three vertices; and
* `eTriangleStrip`: adjacent triangles reuse vertices.

The structure also controls primitive restart. Restart allows a special index value to end one strip and begin another when strip topologies are being used.

This sample generates one triangle from every three vertices:

```cpp
const vk::PipelineInputAssemblyStateCreateInfo inputAssembly{
    .topology = vk::PrimitiveTopology::eTriangleList};
```

The input assembler does not transform the triangle or decide its final screen position. It only establishes the primitive structure consumed by the later stages.

## 3. Viewports and scissors

A viewport describes how the post-vertex-shader coordinates map into the framebuffer. It contains an origin, width, height, and depth range. For a full-window render, it normally covers the swap-chain image:

```cpp
const vk::Viewport viewport{
    0.0f,
    0.0f,
    static_cast<float>(swapchainExtent.width),
    static_cast<float>(swapchainExtent.height),
    0.0f,
    1.0f};
```

The extent should come from the swap chain rather than assuming that the window's requested size is the actual image size. Platforms can select or constrain the swap-chain extent.

A scissor rectangle is different from a viewport. It does not transform coordinates. Instead, it limits the region where fragments are allowed to be stored:

```cpp
const vk::Rect2D scissor{
    vk::Offset2D{0, 0},
    swapchainExtent};
```

Both viewports and scissors may be configured statically in the pipeline or declared as dynamic state. My sample chooses dynamic state, so the pipeline stores only their counts:

```cpp
const vk::PipelineViewportStateCreateInfo viewportState{
    .viewportCount = 1,
    .scissorCount = 1};
```

The actual values are supplied later while recording commands.

## 4. Rasterizer

The rasterizer converts primitives into fragments. It also controls how polygons are represented and which faces should be discarded.

`vk::PipelineRasterizationStateCreateInfo` provides settings such as:

* `polygonMode`: fill polygons, draw their edges, or draw their vertices;
* `cullMode`: keep all faces or cull front-facing and/or back-facing triangles;
* `frontFace`: define clockwise or counter-clockwise winding as front-facing;
* `rasterizerDiscardEnable`: stop geometry before it produces fragments;
* `depthClampEnable`: clamp depth outside the near/far range when supported;
* `depthBiasEnable`: offset depth values; and
* `lineWidth`: choose the thickness of line primitives.

The sample uses filled polygons, back-face culling, and clockwise front faces:

```cpp
const vk::PipelineRasterizationStateCreateInfo rasterizer{
    .polygonMode = vk::PolygonMode::eFill,
    .cullMode = vk::CullModeFlagBits::eBack,
    .frontFace = vk::FrontFace::eClockwise,
    .lineWidth = 1.0f};
```

The winding order matters. If the vertex transformation changes the orientation of a triangle, a configuration that culls back faces can accidentally remove geometry that was expected to be visible.

## 5. Multisampling

Multisampling is one way to reduce jagged edges. Instead of treating every pixel as one coverage sample, the rasterizer can evaluate multiple sample locations. This is most useful at polygon boundaries, where a triangle may cover only part of a pixel.

The main structure is `vk::PipelineMultisampleStateCreateInfo`. It controls the rasterization sample count, sample shading, the minimum fraction of samples shaded, and sample masks.

For now, the sample uses one sample and does not enable multisampling:

```cpp
const vk::PipelineMultisampleStateCreateInfo multisampling{
    .rasterizationSamples = vk::SampleCountFlagBits::e1};
```

Later, using multiple samples will also require compatible image attachments and supported device features. The simple `e1` configuration keeps this first pipeline focused on the basic path.

## 6. Depth and stencil testing

Colour alone does not tell us whether a fragment is visible. A farther object can be drawn after a nearer one, so a renderer needs a depth value to compare the fragments' distances from the camera.

A depth attachment stores those values. The pipeline can test a new fragment against the stored value and optionally write the new value when the test passes. Stencil testing uses an integer value associated with an attachment to include or reject fragments according to a mask and selected operations.

Both are configured through `vk::PipelineDepthStencilStateCreateInfo`. Important settings include:

* `depthTestEnable`;
* `depthWriteEnable`;
* `depthCompareOp`;
* `stencilTestEnable`; and
* front and back stencil operations, masks, and reference values.

This sample has no depth or stencil attachment yet, so it does not pass a depth/stencil state to the graphics pipeline. That is a deliberate omission: without those attachments there is nothing for a depth or stencil test to read or write.

## 7. Color blending

After the fragment shader produces a colour, Vulkan must combine it with the value already stored in the colour attachment. For opaque geometry, the new colour often replaces the old one. For transparent geometry, the source and destination colours are mixed.

The per-attachment settings are described by `vk::PipelineColorBlendAttachmentState`. They include whether blending is enabled, source and destination factors, blend operations, and a colour write mask.

For example, conventional alpha blending can be expressed as:

```cpp
const vk::PipelineColorBlendAttachmentState blendAttachment{
    .blendEnable = vk::True,
    .srcColorBlendFactor = vk::BlendFactor::eSrcAlpha,
    .dstColorBlendFactor = vk::BlendFactor::eOneMinusSrcAlpha,
    .colorBlendOp = vk::BlendOp::eAdd,
    .srcAlphaBlendFactor = vk::BlendFactor::eOne,
    .dstAlphaBlendFactor = vk::BlendFactor::eZero,
    .alphaBlendOp = vk::BlendOp::eAdd,
    .colorWriteMask = vk::ColorComponentFlagBits::eR |
                      vk::ColorComponentFlagBits::eG |
                      vk::ColorComponentFlagBits::eB |
                      vk::ColorComponentFlagBits::eA};
```

The global `vk::PipelineColorBlendStateCreateInfo` references the attachment configurations. It can also enable logic operations and provide blend constants.

The current sample draws an opaque triangle, so it writes every channel directly and leaves blending disabled:

```cpp
const vk::PipelineColorBlendAttachmentState blendAttachment{
    .colorWriteMask = vk::ColorComponentFlagBits::eR |
                      vk::ColorComponentFlagBits::eG |
                      vk::ColorComponentFlagBits::eB |
                      vk::ColorComponentFlagBits::eA};
const vk::PipelineColorBlendStateCreateInfo colorBlending{
    .attachmentCount = 1,
    .pAttachments = &blendAttachment};
```

## 8. Pipeline layout

The pipeline layout describes the interface between the shaders and application-provided resources. It contains descriptor-set layouts for buffers, images, and samplers, as well as push-constant ranges for small values updated directly in command buffers.

For example, a future vertex shader might read a transformation matrix from a uniform buffer, while a fragment shader might read a texture and sampler. The descriptor layouts tell Vulkan what those resources look like and how the shader accesses them.

The current shaders use neither descriptors nor push constants, but Vulkan still requires a pipeline layout:

```cpp
_pipelineLayout = vk::raii::PipelineLayout(
    _device.logicalDevice(),
    vk::PipelineLayoutCreateInfo{});
```

An empty layout is therefore the correct representation of the current shader interface.

## Dynamic state

Vulkan pipelines are mostly configured before creation. Once a graphics pipeline has been created, its baked settings cannot normally be changed in place. If we need a different shader, topology, rasterizer configuration, blending mode, or other baked property, we generally create another pipeline.

This design gives the driver a complete and reusable rendering plan. It also means that changing a setting at draw time is not automatically possible.

Vulkan provides dynamic state for a limited set of properties. A dynamic property is declared when the pipeline is created, but its actual value is supplied later while recording a command buffer.

The sample declares dynamic viewport and scissor state like this:

```cpp
const std::vector<vk::DynamicState> dynamicStates = {
    vk::DynamicState::eViewport,
    vk::DynamicState::eScissor};

const vk::PipelineDynamicStateCreateInfo dynamicState{
    .dynamicStateCount = static_cast<uint32_t>(dynamicStates.size()),
    .pDynamicStates = dynamicStates.data()};
```

This tells Vulkan not to bake the actual viewport and scissor values into the pipeline. The values must be set before the draw command:

```cpp
commandBuffer.setViewport(0, vk::Viewport{
    0.0f,
    0.0f,
    static_cast<float>(swapchainExtent.width),
    static_cast<float>(swapchainExtent.height),
    0.0f,
    1.0f});

commandBuffer.setScissor(0, vk::Rect2D{
    vk::Offset2D{0, 0},
    swapchainExtent});
```

The sample does not yet record command buffers, so these commands are a preview of the next stage of the project. The important connection is that `eViewport` and `eScissor` appear in `vk::PipelineDynamicStateCreateInfo`, while their values appear in command-buffer recording.

Dynamic state is especially useful for viewport and scissor values because the swap-chain extent can change when the window is resized. We can update those values without rebuilding the entire graphics pipeline.

Dynamic state does not mean that the complete pipeline is mutable. Only the states explicitly listed in the dynamic-state structure may be assigned later, and a required dynamic value must be set before it is used.

## Dynamic rendering

Before Vulkan 1.3, a graphics pipeline normally needed to be associated with a render pass. The render pass described the attachment formats, load and store operations, image layouts, and subpass relationships. Framebuffer objects then connected actual image views to those attachment descriptions.

For a swap-chain render loop, this created a group of responsibilities:

1. acquire a swap-chain image;
2. select the framebuffer containing that image's view;
3. begin a compatible render pass;
4. provide clear values in the expected order and formats;
5. record drawing commands inside the render-pass scope;
6. end the render pass; and
7. submit and present the image.

The render pass, framebuffers, pipeline, and image views also had to remain compatible. If the attachment arrangement changed, the application might need to create another render pass and another set of framebuffers.

Dynamic rendering, introduced as a core feature in Vulkan 1.3, simplifies this relationship. Instead of creating render-pass and framebuffer objects for the operation, we specify the attachment formats during pipeline creation and provide the actual colour, depth, and stencil attachments when beginning rendering.

The pipeline still needs a render-target contract. In this sample, that contract is one colour attachment with the swap-chain format:

```cpp
const auto swapchainFormat = swapchain.imageFormat();
const vk::PipelineRenderingCreateInfo rendering{
    .colorAttachmentCount = 1,
    .pColorAttachmentFormats = &swapchainFormat};
```

This describes the format, not a particular image. The acquired swap-chain image and image view will be supplied later in `beginRendering`.

When command recording is implemented, the rendering operation will conceptually look like this:

```cpp
commandBuffer.beginRendering(renderingInfo);
commandBuffer.bindPipeline(
    vk::PipelineBindPoint::eGraphics,
    *graphicsPipeline);
commandBuffer.draw(3, 1, 0, 0);
commandBuffer.endRendering();
```

The exact `vk::RenderingInfo` and image-transition code belongs to the drawing chapter. Dynamic rendering simplifies the attachment object model, but it does not remove synchronization or layout-transition responsibilities.

The sample enables the Vulkan 1.3 feature while creating the logical device:

```cpp
vk::StructureChain<vk::PhysicalDeviceFeatures2,
                   vk::PhysicalDeviceVulkan11Features,
                   vk::PhysicalDeviceVulkan13Features,
                   vk::PhysicalDeviceExtendedDynamicStateFeaturesEXT> features{
    {},
    {.shaderDrawParameters = vulkan_config::requireShaderDrawParameters},
    {.dynamicRendering = vulkan_config::requireDynamicRendering},
    {.extendedDynamicState = vulkan_config::requireExtendedDynamicState}};
```

The device-selection code queries these features before enabling them. That keeps feature discovery and feature activation consistent.

## Putting everything together

At this point the sample has prepared:

* shader stages;
* vertex-input state;
* input-assembly state;
* viewport and scissor counts;
* rasterization state;
* multisampling state;
* colour-blend state;
* dynamic-state declarations;
* a pipeline layout; and
* dynamic-rendering attachment formats.

The final `vk::GraphicsPipelineCreateInfo` connects the shader stages and fixed-function state. `vk::PipelineRenderingCreateInfo` is attached through the `pNext` chain:

```cpp
vk::StructureChain<vk::GraphicsPipelineCreateInfo,
                   vk::PipelineRenderingCreateInfo> info{
    {.stageCount = 2,
     .pStages = stages,
     .pVertexInputState = &vertexInput,
     .pInputAssemblyState = &inputAssembly,
     .pViewportState = &viewportState,
     .pRasterizationState = &rasterizer,
     .pMultisampleState = &multisampling,
     .pColorBlendState = &colorBlending,
     .pDynamicState = &dynamicState,
     .layout = *_pipelineLayout},
    rendering};
```

The `stages` array matches this sample's shader module and entry points:

```cpp
const vk::PipelineShaderStageCreateInfo stages[] = {
    {.stage = vk::ShaderStageFlagBits::eVertex,
     .module = *_shaderModule,
     .pName = "vertMain"},
    {.stage = vk::ShaderStageFlagBits::eFragment,
     .module = *_shaderModule,
     .pName = "fragMain"}};
```

The pipeline is finally created with Vulkan-Hpp RAII:

```cpp
_pipeline = vk::raii::Pipeline(
    _device.logicalDevice(),
    nullptr,
    info.get<vk::GraphicsPipelineCreateInfo>());
```

The `nullptr` represents the optional pipeline-cache argument. The sample has only one pipeline and does not use a cache yet. Vulkan also supports pipeline derivatives through `basePipelineHandle` and `basePipelineIndex`, but those are not useful for this first pipeline.

## What the sample creates

The construction order in `main.cpp` reflects the dependencies between the objects:

```text
window
  → instance
  → surface
  → logical device
  → swap chain
  → shader module
  → pipeline layout
  → graphics pipeline
```

The graphics-pipeline constructor receives both the device and swap chain:

```cpp
GraphicsPipeline::GraphicsPipeline(
    const Device& device,
    const Swapchain& swapchain,
    const std::filesystem::path& directory)
```

The device is needed to create Vulkan objects. The swap chain is needed because the pipeline's dynamic-rendering information uses the selected swap-chain image format.

This sample ends after successful pipeline creation. It does not yet acquire images, allocate command buffers, transition image layouts, begin dynamic rendering, bind the pipeline, draw, submit, or present. That is a useful boundary: the pipeline definition is complete even though the render loop is still the next milestone.

## Final summary

The graphics pipeline is a complete rendering plan assembled from several descriptions:

```text
vertex input
    + input assembly
    + viewport and scissor state
    + rasterization
    + multisampling
    + depth and stencil state
    + color blending
    + pipeline layout
    + shader stages
    + dynamic-rendering attachment formats
    = graphics pipeline
```

Most of these choices are baked into the pipeline before it is created. Dynamic state provides a controlled exception for values such as viewport and scissor, allowing them to be supplied during command recording.

Dynamic rendering changes how the pipeline connects to its targets. Instead of requiring a traditional render pass and framebuffer, Vulkan 1.3 lets the pipeline declare compatible attachment formats and lets the command buffer provide the actual attachments when rendering begins.

The final `vk::StructureChain` is therefore more than a collection of unrelated fields. It is the rendering contract for this pipeline: which shaders run, how their input is interpreted, how geometry becomes fragments, how colours are written, which values can change later, and which attachment format receives the result.

The next step is to use this contract in a real frame. Once command recording and synchronization are added, the triangle will finally travel through the pipeline and become pixels on the screen.
