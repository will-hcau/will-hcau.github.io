---
title: "14. From Render Passes and Framebuffers to Dynamic Rendering"
date: 2026-10-16 20:00:00 +0900
categories: [Vulkan]
tags: [vulkan, rendering, gpu, c++]
description: What render passes and framebuffers mean in Vulkan, how the legacy setup worked, and how Vulkan 1.3 made the common case much simpler with dynamic rendering.
---

I recently read the Vulkan Tutorial's chapter on [dynamic rendering](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/03_Drawing/00_Dynamic_rendering.html). Unlike some earlier chapters, this one did not ask me to implement a feature. It was more of a change in perspective: Vulkan 1.3 lets us describe a rendering operation at the point where we record commands, instead of building a collection of render-pass and framebuffer objects first.

That made me want to understand what those older objects were actually doing. The short version is:

* a **render pass** describes the shape of a rendering operation: its attachments, subpasses, and how attachment contents are loaded, stored, and used;
* a **framebuffer** supplies the concrete image views used by that render pass instance; and
* **dynamic rendering** describes the attachments directly with `vk::RenderingInfo` while recording the command buffer.

Dynamic rendering is not a new way to draw. It is a less indirect way to tell Vulkan where the draw commands are going to write.

## The mental model: a rendering contract and its concrete targets

Imagine a simple triangle rendered into a swap-chain image. The GPU needs to know at least:

1. which images are render targets;
2. the format and sample count of those images;
3. whether each attachment should be cleared, preserved, or discarded;
4. whether its final contents must be stored; and
5. the dimensions and layers of the actual images.

In the original Vulkan 1.0 model, that information was split between two objects.

The **render pass** was the reusable description. It contained attachment descriptions and one or more subpasses. An attachment description could say, for example, “this is a colour image, clear it at the beginning, store its result at the end, and use it as a colour attachment.” A subpass then said which attachment indices it used.

The **framebuffer** was the concrete binding. It connected those attachment slots to particular `VkImageView` objects and specified their width, height, and layer count. The framebuffer had to be compatible with the render pass it was created for.

The Vulkan specification describes the relationship well in its [render-pass and framebuffer reference](https://docs.vulkan.org/spec/latest/chapters/renderpass.html): the render pass describes the structure independently of specific image views, while the framebuffer provides the image views and dimensions.

This separation was useful when a renderer had carefully planned subpasses and predictable render targets. It also meant that a one-pass triangle needed several pieces of setup before its first draw.

## What the old setup looked like

The following is intentionally compact Vulkan-Hpp-style code. It is not a complete application, but it shows the information that an application had to assemble itself.

### 1. Describe the colour attachment

First, I described the swap-chain colour image as an attachment. The image would be cleared when the pass began and stored when the pass ended:

```cpp
vk::AttachmentDescription colorAttachment{
    .format = swapchainFormat,
    .samples = vk::SampleCountFlagBits::e1,
    .loadOp = vk::AttachmentLoadOp::eClear,
    .storeOp = vk::AttachmentStoreOp::eStore,
    .stencilLoadOp = vk::AttachmentLoadOp::eDontCare,
    .stencilStoreOp = vk::AttachmentStoreOp::eDontCare,
    .initialLayout = vk::ImageLayout::eUndefined,
    .finalLayout = vk::ImageLayout::ePresentSrcKHR};
```

`loadOp` and `storeOp` are important performance hints as well as correctness rules. If I do not need the old contents, `eDontCare` or `eClear` can avoid preserving them. If I do need the rendered result after the pass, I must store it.

### 2. Tell a subpass to use it

The attachment description is indexed. A subpass refers to that index through an attachment reference:

```cpp
vk::AttachmentReference colorReference{
    .attachment = 0,
    .layout = vk::ImageLayout::eColorAttachmentOptimal};

vk::SubpassDescription subpass{
    .pipelineBindPoint = vk::PipelineBindPoint::eGraphics,
    .colorAttachmentCount = 1,
    .pColorAttachments = &colorReference};
```

For a more advanced render pass, I could add a depth attachment, input attachments, resolve attachments, multiple subpasses, and explicit dependencies between them. For a triangle, however, this is already a lot of indirection: the subpass does not contain an image view, only the number `0`.

### 3. Create the render pass

Now those descriptions became a device object:

```cpp
vk::RenderPassCreateInfo renderPassInfo{
    .attachmentCount = 1,
    .pAttachments = &colorAttachment,
    .subpassCount = 1,
    .pSubpasses = &subpass};

vk::raii::RenderPass renderPass{device, renderPassInfo};
```

The graphics pipeline was then created against this render pass. In older Vulkan code, the pipeline create info commonly ended with something like:

```cpp
pipelineInfo.renderPass = *renderPass;
pipelineInfo.subpass = 0;
```

That tied the pipeline to the attachment and subpass compatibility described by the render pass.

### 4. Create one framebuffer for each swap-chain image

The swap chain normally has several images. Each image has its own image view, so I needed a matching framebuffer for each one:

```cpp
std::vector<vk::raii::Framebuffer> framebuffers;
framebuffers.reserve(swapchainImageViews.size());

for (const vk::raii::ImageView& imageView : swapchainImageViews) {
    vk::FramebufferCreateInfo framebufferInfo{
        .renderPass = *renderPass,
        .attachmentCount = 1,
        .pAttachments = &*imageView,
        .width = swapchainExtent.width,
        .height = swapchainExtent.height,
        .layers = 1};

    framebuffers.emplace_back(device, framebufferInfo);
}
```

This is why resizing or recreating the swap chain usually meant recreating framebuffers too: their dimensions and image-view attachments were no longer valid. Adding a depth image or multisampling meant adding more compatible attachments and updating the render-pass/framebuffer relationship.

### 5. Begin and end the render pass while recording

At draw time, I selected the framebuffer belonging to the acquired swap-chain image:

```cpp
vk::ClearValue clearValue =
    vk::ClearColorValue{std::array<float, 4>{0.02f, 0.02f, 0.02f, 1.0f}};

vk::RenderPassBeginInfo beginInfo{
    .renderPass = *renderPass,
    .framebuffer = *framebuffers[imageIndex],
    .renderArea = vk::Rect2D{vk::Offset2D{0, 0}, swapchainExtent},
    .clearValueCount = 1,
    .pClearValues = &clearValue};

commandBuffer.beginRenderPass(beginInfo, vk::SubpassContents::eInline);
commandBuffer.bindPipeline(vk::PipelineBindPoint::eGraphics, *pipeline);
commandBuffer.draw(3, 1, 0, 0);
commandBuffer.endRenderPass();
```

There is nothing wrong with this model. In fact, render passes could express subpass relationships and attachment behaviour in a way that helped drivers understand the whole operation. The cost was that even a straightforward pass required the application to maintain several compatible objects and recreate them when the render targets changed.

## Vulkan 1.3: describe the pass where it happens

[Dynamic rendering](https://docs.vulkan.org/spec/latest/chapters/renderpass.html#renderpass) was introduced by `VK_KHR_dynamic_rendering` and promoted to core Vulkan 1.3. It keeps the useful attachment information but removes the need to create `VkRenderPass` and `VkFramebuffer` for the common case.

Instead of attachment index `0` referring indirectly to an image view, I can describe the image view directly:

```cpp
vk::RenderingAttachmentInfo colorAttachment{
    .imageView = *swapchainImageViews[imageIndex],
    .imageLayout = vk::ImageLayout::eColorAttachmentOptimal,
    .loadOp = vk::AttachmentLoadOp::eClear,
    .storeOp = vk::AttachmentStoreOp::eStore,
    .clearValue = vk::ClearValue{
        vk::ClearColorValue{std::array<float, 4>{0.02f, 0.02f, 0.02f, 1.0f}}}};

vk::RenderingInfo renderingInfo{
    .renderArea = vk::Rect2D{vk::Offset2D{0, 0}, swapchainExtent},
    .layerCount = 1,
    .colorAttachmentCount = 1,
    .pColorAttachments = &colorAttachment};

commandBuffer.beginRendering(renderingInfo);
commandBuffer.bindPipeline(vk::PipelineBindPoint::eGraphics, *pipeline);
commandBuffer.draw(3, 1, 0, 0);
commandBuffer.endRendering();
```

That is the same operation expressed with fewer long-lived objects. The attachment still has a layout, load operation, store operation, and clear value. Vulkan did not stop caring about those details; it just lets me specify them next to the draw that needs them.

The pipeline also no longer points at a particular render pass. Its colour and depth formats are supplied through `vk::PipelineRenderingCreateInfo` in the `pNext` chain during pipeline creation:

```cpp
vk::Format colourFormats[] = {swapchainFormat};
vk::PipelineRenderingCreateInfo pipelineRendering{
    .colorAttachmentCount = 1,
    .pColorAttachmentFormats = colourFormats,
    .depthAttachmentFormat = vk::Format::eUndefined};

vk::GraphicsPipelineCreateInfo pipelineInfo{
    .pNext = &pipelineRendering,
    // shader stages, vertex input, rasterization, blending, and layout...
};
```

The pipeline still needs to agree with the formats it will render to. Dynamic rendering removes the render-pass handle from the relationship; it does not remove compatibility requirements.

## What became simpler?

For a single colour pass, the practical change is easy to feel:

| Legacy rendering | Dynamic rendering |
| --- | --- |
| Create attachment descriptions | Create `vk::RenderingAttachmentInfo` while recording |
| Create subpass descriptions | Put the attachments in `vk::RenderingInfo` |
| Create a render pass | No render-pass object for this pass |
| Create a framebuffer per swap-chain image | Use the selected image view directly |
| Begin/end a render pass | Begin/end rendering |
| Pipeline references a render pass | Pipeline declares attachment formats |

This is particularly convenient for engines with many render-target combinations: swap-chain images, off-screen textures, shadow maps, reflection targets, and post-processing images do not all need a separate family of framebuffer objects.

## What did not disappear?

Dynamic rendering is simpler, but it is not automatic rendering. I still need to:

* transition images into layouts suitable for their use;
* use the correct `loadOp` and `storeOp`;
* provide compatible colour, depth, and stencil formats when creating the pipeline;
* handle synchronization between image acquisition, rendering, and presentation; and
* end rendering before issuing commands that are not allowed inside a rendering instance.

There is also a design limitation to remember. Dynamic rendering originally did not reproduce every feature of legacy subpasses. If an application depends heavily on input attachments or tile-based subpass behaviour, the legacy model—or newer dynamic-rendering extensions—may still matter. The Vulkan documentation's [legacy functionality notes](https://docs.vulkan.org/spec/latest/appendices/legacy.html) explain this boundary.

## My takeaway

The old render-pass model taught Vulkan exactly what a rendering operation would look like before command recording began. A render pass described the plan; a framebuffer supplied the actual images. That explicit split made sense for Vulkan's original design, but it made a basic full-screen pass feel like a small object-management project.

Vulkan 1.3's dynamic rendering keeps the explicit GPU-facing information while putting it closer to the work that consumes it. For the triangle I am learning to draw, the code now reads more naturally: select the swap-chain image, describe its attachment, begin rendering, draw, and end rendering.

That is the important lesson for me. Dynamic rendering does not make Vulkan less explicit. It removes an extra layer of bookkeeping so that the explicit description is attached to the actual rendering operation.

### References

* [Khronos Vulkan Tutorial: Dynamic rendering](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/03_Drawing/00_Dynamic_rendering.html)
* [Vulkan specification: Render passes and framebuffers](https://docs.vulkan.org/spec/latest/chapters/renderpass.html)
* [Vulkan specification: `VK_KHR_dynamic_rendering`](https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_dynamic_rendering.html)
* [Vulkan specification: Legacy render-pass functionality](https://docs.vulkan.org/spec/latest/appendices/legacy.html)
