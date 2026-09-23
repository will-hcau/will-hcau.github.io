---
title: "8. Creating a Vulkan Swapchain"
date: 2026-09-23 12:00:00 +0900
categories: [Graphics]
tags: [vulkan, graphics, gpu, c++]
description: Understanding how a Vulkan swapchain connects fast GPU rendering to a display with a fixed refresh rate.
---

Today I continued with the [swapchain chapter of the Vulkan Tutorial](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/01_Presentation/01_Swap_chain.html). The matching sample is [07_swapchain](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/07_swapchain) in [The Journey of Learning Vulkan](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan).

The swapchain is the first Vulkan object that connects our renderer to the window on the screen. It answers a practical question:

> How can a GPU produce images as quickly as it can while a display consumes images at its own fixed rhythm?

## The display and the GPU do not run at the same speed

A display refreshes at a fairly regular frequency. A 60 Hz display checks for a new image about 60 times per second, or once every 16.67 milliseconds. A 30 Hz display has about 33.33 milliseconds per refresh. Some modern displays refresh at 120 Hz or more, but the basic problem is the same.

The GPU, however, does not naturally produce one image every 16.67 milliseconds. A simple scene might render in 4 milliseconds, while a more complicated scene might take 25 milliseconds. The rendering time can change from frame to frame.

The display also cannot use an image while the GPU is still writing to it. Presenting a partially rendered image would produce visible corruption. The application therefore needs several images that can take turns. These images form the swapchain. The application acquires an available image, renders into it, and presents it. The presentation engine later makes that image visible at a suitable display refresh.

<object type="image/svg+xml" data="{{ '/assets/img/vulkan-swapchain-flow.svg' | relative_url }}" width="100%" height="auto" style="max-width: 100%; aspect-ratio: 1200 / 430;" aria-label="Animated diagram of three buffers circulating between the GPU, swapchain queue, and display">
  <a href="{{ '/assets/img/vulkan-swapchain-flow.svg' | relative_url }}">View the animated swapchain flow diagram</a>
</object>

The swapchain does not make the GPU render at the display's frequency. Instead, it controls which images are available and how completed images are handed to the presentation engine. Synchronization objects such as semaphores and fences later make sure that an image is not reused too early.

## What must the surface support?

Before creating a swapchain, we need to ask the selected physical device what it can do with our particular surface. A surface represents the connection between Vulkan and the window-system window created by GLFW.

Swapchain support is surface-specific. A format or presentation mode supported by one window surface is not automatically supported by another surface or another GPU.

There are three groups of information we need:

1. **Surface capabilities** — limits and properties such as the allowed image count and image extent.
2. **Surface formats** — the image format and color space that can be presented.
3. **Presentation modes** — the rules used to place completed images on the display.

The sample collects all three in one small structure:

```cpp
struct SwapChainSupportDetails
{
    vk::SurfaceCapabilitiesKHR capabilities;
    std::vector<vk::SurfaceFormatKHR> formats;
    std::vector<vk::PresentModeKHR> presentModes;
};

SwapChainSupportDetails querySwapChainSupport(
    const vk::raii::PhysicalDevice& candidate) const
{
    return {
        candidate.getSurfaceCapabilitiesKHR(*surface),
        candidate.getSurfaceFormatsKHR(*surface),
        candidate.getSurfacePresentModesKHR(*surface)
    };
}
```

We should also reject a device or surface that cannot provide a usable format or presentation mode:

```cpp
const auto support = querySwapChainSupport(physicalDevice);

if (support.formats.empty() || support.presentModes.empty())
    throw std::runtime_error("surface does not support a usable swapchain");
```

This is different from checking whether the `VK_KHR_swapchain` device extension exists. The extension tells us that the device has swapchain functionality. The three queries tell us whether that functionality can be used with this particular window surface.

## Choosing the surface format

`vk::SurfaceFormatKHR` contains two values:

```cpp
struct SurfaceFormatKHR
{
    vk::Format format;
    vk::ColorSpaceKHR colorSpace;
};
```

The `format` describes how each pixel is stored. Common choices include:

- `eB8G8R8A8Srgb`: 8 bits each for blue, green, red, and alpha, using an sRGB encoding. This is a common choice for ordinary colour rendering.
- `eR8G8B8A8Srgb`: the same 8-bit channels in red-green-blue-alpha order. The order matters because the memory layout is different.
- `eB8G8R8A8Unorm` or `eR8G8B8A8Unorm`: 8-bit unsigned normalized values without sRGB encoding. The values are treated as a straightforward numeric range rather than an sRGB colour encoding.
- Other formats: some platforms expose 10-bit colour, floating-point channels, or different channel orders. They can be useful for HDR or specialised rendering, but they are not necessary for this first example.

The `colorSpace` describes how the stored colour values should be interpreted by the presentation system. `eSrgbNonlinear` is the usual choice for a normal sRGB monitor. “Nonlinear” refers to the encoding of colour values; it does not mean that lighting calculations should be performed in nonlinear space. Lighting and blending should generally happen in linear space, with sRGB conversion at the appropriate image boundaries.

The sample prefers a common sRGB combination and falls back to the first format if it is unavailable:

```cpp
vk::SurfaceFormatKHR chooseSwapSurfaceFormat(
    const std::vector<vk::SurfaceFormatKHR>& formats) const
{
    const auto preferred = std::ranges::find_if(
        formats,
        [](const auto& format)
        {
            return format.format == vk::Format::eB8G8R8A8Srgb &&
                   format.colorSpace == vk::ColorSpaceKHR::eSrgbNonlinear;
        });

    return preferred == formats.end() ? formats.front() : *preferred;
}
```

The lambda passed to `find_if` is a small test function. It returns `true` for the first format whose pixel format and colour space both match our preference. `preferred` is an iterator, so `*preferred` gives us the actual `SurfaceFormatKHR` value.

Some implementations can report a special `eUndefined` format when the surface places no restriction on the format. A more complete application may treat that case as permission to choose its preferred format. The sample keeps the selection simple because the returned format list is sufficient for the platforms being tested.

## Choosing the swap extent

The swap extent is the width and height of every swapchain image. In a normal windowed application it should match the window's framebuffer size, in pixels.

There is an important difference between window coordinates and framebuffer pixels. A high-density display may have a window that is 800 by 600 logical units but a framebuffer that is 1600 by 1200 pixels. GLFW's `glfwGetFramebufferSize` gives us the size Vulkan needs.

The surface capabilities provide the allowed minimum and maximum extents. The application must clamp the requested size to those limits:

```cpp
vk::Extent2D chooseSwapExtent(
    const vk::SurfaceCapabilitiesKHR& capabilities) const
{
    if (capabilities.currentExtent.width !=
        std::numeric_limits<uint32_t>::max())
    {
        return capabilities.currentExtent;
    }

    int width = 0;
    int height = 0;
    glfwGetFramebufferSize(window, &width, &height);

    return {
        std::clamp<uint32_t>(static_cast<uint32_t>(width),
                             capabilities.minImageExtent.width,
                             capabilities.maxImageExtent.width),
        std::clamp<uint32_t>(static_cast<uint32_t>(height),
                             capabilities.minImageExtent.height,
                             capabilities.maxImageExtent.height)
    };
}
```

If `currentExtent` is not the special maximum value, the window system has already chosen the extent and we must use it. Otherwise, we choose the framebuffer size ourselves and clamp it. This platform difference is why the code has both branches.

## Choosing the presentation mode

The presentation mode is one of the most important swapchain settings. It controls when a completed image becomes visible and what happens when the application produces images faster or slower than the display consumes them.

Vulkan defines four main modes:

### `eImmediate`

The image is submitted to the display immediately, without waiting for the next refresh. This gives the lowest possible presentation latency, but the display can switch to the new image halfway through a refresh. The top part of the screen may then show one frame while the bottom part shows another. This is **tearing**.

Immediate mode can be useful when latency is more important than visual smoothness, or for testing. It is not guaranteed to be available.

### `eFifo`

FIFO means “first in, first out”. Presented images enter a queue and are displayed in order at refresh boundaries. The presentation engine waits for the next refresh before replacing the visible image, so tearing is avoided.

This is the safe default and the only presentation mode that Vulkan requires every implementation to support. It behaves similarly to traditional vertical synchronization. If rendering is too slow to produce a new image before a refresh, the display continues showing the previous image and the frame rate may fall to a lower rhythm.

### `eFifoRelaxed`

This behaves like FIFO when the application is producing frames on time. If the application misses a refresh, however, the presentation engine may display the late image immediately instead of waiting for the next refresh. That can reduce the visible pause caused by missing a refresh, but it may introduce tearing for that late frame.

This mode is optional and is less commonly selected by a basic renderer.

### `eMailbox`

Mailbox also waits for a refresh boundary, so it avoids tearing. The difference is what happens when the application renders faster than the display. Instead of keeping every rendered image, the presentation engine replaces an older waiting image with the newest one.

This means the display receives the most recent completed frame rather than a long queue of old frames. It can reduce visible latency and prevents the renderer from blocking as often, but it may require more images and more memory. It is also optional.

The practical difference is easier to see as a timing example:

<object type="image/svg+xml" data="{{ '/assets/img/vulkan-presentation-modes.svg' | relative_url }}" width="100%" height="auto" style="max-width: 100%; aspect-ratio: 1200 / 520;" aria-label="Animated comparison of FIFO, Mailbox, and Immediate presentation modes">
  <a href="{{ '/assets/img/vulkan-presentation-modes.svg' | relative_url }}">View the animated presentation-mode comparison</a>
</object>

The sample prefers Mailbox and falls back to FIFO, because FIFO is guaranteed:

```cpp
vk::PresentModeKHR chooseSwapPresentMode(
    const std::vector<vk::PresentModeKHR>& modes) const
{
    const bool mailboxAvailable =
        std::ranges::find(modes, vk::PresentModeKHR::eMailbox) != modes.end();

    return mailboxAvailable
        ? vk::PresentModeKHR::eMailbox
        : vk::PresentModeKHR::eFifo;
}
```

The best mode depends on the application. A game may prefer Mailbox for lower latency. A simple application may use FIFO for predictable behaviour. A latency-sensitive application may consider Immediate if tearing is acceptable.

## Choosing the number of images

The capabilities include `minImageCount` and `maxImageCount`. The application must request a count in that range. A `maxImageCount` of zero means that there is no maximum limit exposed by the surface.

The sample requests at least three images, then respects the limits:

```cpp
uint32_t imageCount = std::max(3u, support.capabilities.minImageCount);

if (support.capabilities.maxImageCount != 0)
{
    imageCount = std::min(imageCount,
                          support.capabilities.maxImageCount);
}
```

Two images can be enough for double buffering: while one image is displayed, the GPU renders into the other. Three images provide an additional buffer, which can keep the GPU busy while one image is displayed and another is waiting to be presented. This is commonly called triple buffering.

More images are not automatically better. They consume more memory and can increase latency if old frames are allowed to wait in the queue. The presentation mode and synchronization strategy determine how much benefit additional images provide.

## The remaining create-info settings

The swapchain create structure contains a few other fields. They matter, but the defaults used by this first renderer are straightforward.

### `imageArrayLayers`

This is the number of layers in each swapchain image. Ordinary 2D rendering uses one layer:

```cpp
.imageArrayLayers = 1
```

Multiple layers can be used for stereoscopic or multiview rendering, where one swapchain image contains several views.

### `imageUsage`

This says how the application will use the swapchain images. The sample renders directly into them as colour attachments:

```cpp
.imageUsage = vk::ImageUsageFlagBits::eColorAttachment
```

Other usages, such as transfer source or transfer destination, may be available, but they must be supported by the surface's `supportedUsageFlags`.

### `imageSharingMode`

`eExclusive` means that the images belong to one queue family at a time. It is usually the best choice when the same graphics queue both renders and presents:

```cpp
.imageSharingMode = vk::SharingMode::eExclusive
```

`eConcurrent` allows several queue families to access the images without explicit ownership transfers. It is useful when graphics and presentation use different queue families, but it can have a performance cost and requires a list of queue-family indices.

### `preTransform`

This controls a transform applied by the presentation system, such as rotating the image on a device whose natural orientation differs from the window. Using `currentTransform` asks Vulkan to use the transform currently selected by the surface:

```cpp
.preTransform = support.capabilities.currentTransform
```

### `compositeAlpha`

This specifies how the alpha channel is used when the window is composited with other windows. For an ordinary opaque window, the opaque mode is the natural choice:

```cpp
.compositeAlpha = vk::CompositeAlphaFlagBitsKHR::eOpaque
```

The selected mode must appear in the surface's `supportedCompositeAlpha` flags.

### `clipped`

When true, pixels hidden behind another window or outside the visible region do not need to be preserved. This is normally a useful optimisation:

```cpp
.clipped = vk::True
```

## Creating the swapchain

Once the choices are made, they are assembled into `vk::SwapchainCreateInfoKHR`:

```cpp
const vk::SwapchainCreateInfoKHR createInfo{
    .surface = *surface,
    .minImageCount = imageCount,
    .imageFormat = swapChainSurfaceFormat.format,
    .imageColorSpace = swapChainSurfaceFormat.colorSpace,
    .imageExtent = swapChainExtent,
    .imageArrayLayers = 1,
    .imageUsage = vk::ImageUsageFlagBits::eColorAttachment,
    .imageSharingMode = vk::SharingMode::eExclusive,
    .preTransform = support.capabilities.currentTransform,
    .compositeAlpha = vk::CompositeAlphaFlagBitsKHR::eOpaque,
    .presentMode = chooseSwapPresentMode(support.presentModes),
    .clipped = vk::True
};

swapChain = vk::raii::SwapchainKHR(device, createInfo);
swapChainImages = swapChain.getImages();
```

The `vk::raii::SwapchainKHR` constructor creates the Vulkan swapchain using the logical device and the configuration we prepared. The swapchain owns the presentable images; `getImages()` retrieves their handles so that later rendering code can create image views and render into them.

At this point the application has not rendered a triangle yet, but it has created the path by which a rendered image can eventually reach the display. The next steps will use these swapchain images as colour attachments, acquire one for each frame, submit rendering work, and present the completed image.
