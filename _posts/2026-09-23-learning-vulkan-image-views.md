---
title: "Episode 9: Creating Vulkan Image Views"
date: 2026-09-23 13:00:00 +0900
categories: [Vulkan]
tags: [vulkan, graphics, gpu, c++]
description: Understanding how Vulkan image views describe access to swapchain images.
---

Today I continued with the [image views chapter of the Vulkan Tutorial](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/01_Presentation/02_Image_views.html). The matching sample is [08_image_view](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/08_image_view) in [The Journey of Learning Vulkan](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan).

The swapchain has given us a collection of `vk::Image` objects. These are the actual GPU images that will eventually be presented on screen. The next question is: how do rendering commands access one of those images?

The answer is an **image view**.

## An image and an image view are different things

A `vk::Image` represents image storage. It has a format, a size, mip levels, array layers, and memory managed by Vulkan or the application. It is the resource that contains the pixels.

An `vk::ImageView` is a description of how to access that image. It selects the image type, format interpretation, and the part of the image that a rendering operation should use.

```text
vk::Image
    actual GPU image resource
          ↓
vk::ImageView
    a view describing how to access it
          ↓
rendering attachment or sampled texture
```

The image view does not copy the image, create another image, or register an image with the swapchain. It is more like a window into an existing image.

For a swapchain with three images, we normally create three image views:

```text
Swapchain
 ├── Image A ── Image View A
 ├── Image B ── Image View B
 └── Image C ── Image View C
```

Later, a framebuffer or dynamic-rendering attachment will refer to one of these views. That tells Vulkan which swapchain image should receive the rendering for the current frame.

## Creating an image view

The central structure is `vk::ImageViewCreateInfo`:

```cpp
const vk::ImageViewCreateInfo createInfo{
    .image = image,
    .viewType = vk::ImageViewType::e2D,
    .format = swapChainSurfaceFormat.format,
    .subresourceRange = {
        .aspectMask = vk::ImageAspectFlagBits::eColor,
        .levelCount = 1,
        .layerCount = 1
    }
};

vk::raii::ImageView imageView(device, createInfo);
```

The sample creates one view for every swapchain image:

```cpp
for (const vk::Image image : swapChainImages)
{
    auto imageViewCreateInfo = createInfo;
    imageViewCreateInfo.image = image;
    swapChainImageViews.emplace_back(device, imageViewCreateInfo);
}
```

The common settings are shared because every swapchain image has the same size and format. Only the `.image` field changes for each iteration.

## The fields of `ImageViewCreateInfo`

`vk::ImageViewCreateInfo` contains several fields. Some are important for every view, while others are usually left at their default values for a basic swapchain.

### `pNext`

```cpp
.pNext = nullptr
```

`pNext` links optional extension structures to the create-info structure. Vulkan uses this pattern throughout its API so extensions can add information without changing the original structure.

The basic swapchain image view does not need any extension structure, so `nullptr` is appropriate. With Vulkan-Hpp's designated initializers, leaving the field out has the same effect.

### `flags`

```cpp
.flags = {}
```

This contains image-view creation flags. Most ordinary image views use no flags.

Some extensions define special flags for specialised view types. For a normal 2D swapchain image, the default empty value is correct.

### `image`

```cpp
.image = image
```

This is the actual `vk::Image` that the view describes. In the swapchain example, the images come from:

```cpp
swapChainImages = swapChain.getImages();
```

The swapchain owns these images. We do not destroy them ourselves, but we do create and destroy the image views that refer to them.

Each image needs its own view because each view points to a particular image object:

```cpp
for (const vk::Image image : swapChainImages)
{
    imageViewCreateInfo.image = image;
    swapChainImageViews.emplace_back(device, imageViewCreateInfo);
}
```

### `viewType`

```cpp
.viewType = vk::ImageViewType::e2D
```

This describes the shape through which the image will be accessed. Common choices are:

- `e1D` for a one-dimensional image;
- `e2D` for an ordinary two-dimensional image;
- `e3D` for a three-dimensional image;
- `eCube` for a cube map with six faces;
- `e1DArray`, `e2DArray`, and `eCubeArray` for arrays of those view types.

Swapchain images are ordinary 2D images, so `e2D` is the correct choice. The view type must agree with the image's dimensions and the number of layers selected by the view.

### `format`

```cpp
.format = swapChainSurfaceFormat.format
```

This specifies how the view interprets the pixels when they are accessed. For our example, it is the same format selected while creating the swapchain, such as:

```cpp
vk::Format::eB8G8R8A8Srgb
```

The format appears in both structures because the image and the view are separate Vulkan objects. The swapchain selects the format of the image storage. The image view states the format used when rendering through that view.

For an ordinary swapchain, the values should match. Vulkan can support compatible alternative view formats for specially created mutable-format images, but that is not needed here. The colour space is not repeated: `vk::ColorSpaceKHR` belongs to presentation, while `vk::Format` belongs to image access.

### `components`

```cpp
.components = {
    .r = vk::ComponentSwizzle::eIdentity,
    .g = vk::ComponentSwizzle::eIdentity,
    .b = vk::ComponentSwizzle::eIdentity,
    .a = vk::ComponentSwizzle::eIdentity
}
```

This controls how the image's colour channels are mapped through the view. The possible swizzles include:

- `eIdentity`: use the original channel;
- `eR`, `eG`, `eB`, `eA`: use a particular source channel;
- `eZero`: always return zero;
- `eOne`: always return one.

For example, a view could make every output channel use the red channel, or force alpha to one. A normal swapchain view does not need channel remapping, so the default identity mapping is used:

```cpp
.components = {}
```

Leaving the field at its default means that red remains red, green remains green, blue remains blue, and alpha remains alpha.

### `subresourceRange`

This is the most important part of the view description. It selects which subresources of the image are visible through the view:

```cpp
.subresourceRange = {
    .aspectMask = vk::ImageAspectFlagBits::eColor,
    .baseMipLevel = 0,
    .levelCount = 1,
    .baseArrayLayer = 0,
    .layerCount = 1
}
```

An image can contain several kinds of subresources:

- mipmap levels, from the full-resolution image to smaller versions;
- array layers, such as the six faces of a cube map;
- aspects, such as colour, depth, and stencil data.

The fields select a range rather than one individual value.

#### `aspectMask`

```cpp
.aspectMask = vk::ImageAspectFlagBits::eColor
```

This selects which part of the image is being viewed. Swapchain images contain colour data, so we use `eColor`.

Depth/stencil images may use `eDepth`, `eStencil`, or both. A multi-planar image may use a plane aspect. The aspect must match the image format and the intended use.

#### `baseMipLevel` and `levelCount`

```cpp
.baseMipLevel = 0,
.levelCount = 1
```

These select mipmap levels. `baseMipLevel = 0` starts with the full-resolution level, and `levelCount = 1` exposes only that level.

Swapchain images do not have a mipmap chain, so one level is the complete image. A texture with several mip levels could create a view covering all of them or a view covering only one particular level.

#### `baseArrayLayer` and `layerCount`

```cpp
.baseArrayLayer = 0,
.layerCount = 1
```

These select array layers. A normal swapchain image has one layer, so the view starts at layer zero and includes one layer.

An array texture or cube map would use different values. A cube map has six layers, for example, while a view of one cube face selects a single layer.

## The complete swapchain image-view code

The sample keeps the common configuration in one structure and assigns each image inside the loop:

```cpp
void createImageViews()
{
    swapChainImageViews.clear();

    const vk::ImageViewCreateInfo createInfo{
        .viewType = vk::ImageViewType::e2D,
        .format = swapChainSurfaceFormat.format,
        .subresourceRange = {
            .aspectMask = vk::ImageAspectFlagBits::eColor,
            .levelCount = 1,
            .layerCount = 1
        }
    };

    for (const vk::Image image : swapChainImages)
    {
        auto imageViewCreateInfo = createInfo;
        imageViewCreateInfo.image = image;
        swapChainImageViews.emplace_back(device, imageViewCreateInfo);
    }
}
```

The omitted fields use their zero or default values: no extension chain, no special flags, identity channel mapping, and base mip/layer values of zero. This is enough to create a full-colour 2D view of each complete swapchain image.

The resulting relationship is:

```text
Swapchain image
    format: B8G8R8A8Srgb
    size:   swapchain extent
    layers: 1
        ↓
Image view
    type:   2D
    aspect: colour
    levels: mip 0 only
    layers: layer 0 only
        ↓
Rendering attachment
```

The image view is therefore the bridge between the image resource provided by the swapchain and the rendering setup used by the application. Once all views exist, the next step is to describe which view, together with which other attachments, belongs to each framebuffer or rendering operation.
