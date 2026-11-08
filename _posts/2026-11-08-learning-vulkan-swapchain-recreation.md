---
image:
  path: /assets/img/vulkan-first-triangle.png
title: "Episode 18: Resizing the Window — Recreating the Swapchain"
date: 2026-11-08 00:00:00 +0900
categories: [Vulkan]
tags: [vulkan, swapchain, window, c++]
description: "Making the triangle follow a changing window size by rebuilding the swapchain and the resources that depend on it."
---

Our triangle has a fixed-size window. Today, we will make the window resizable and update the swapchain when its size changes.

I followed the Vulkan Tutorial's [swapchain recreation chapter](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/04_Swap_chain_recreation.html) and implemented the [15_swapchain_recreation sample](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/15_swapchain_recreation).

The swapchain holds the images used to show our frames. Those images have a size, so when the window changes size, the old swapchain may no longer fit. We need to create a new swapchain and rebuild the resources that refer to the old one.

## Let the window be resized

GLFW windows are fixed-size by default. My `Window` class makes the window resizable and registers a callback:

~~~cpp
glfwWindowHint(GLFW_RESIZABLE, GLFW_TRUE);

_window = glfwCreateWindow(
    static_cast<int>(width),
    static_cast<int>(height),
    title,
    nullptr,
    nullptr);

glfwSetWindowUserPointer(_window, this);
glfwSetFramebufferSizeCallback(_window, framebufferResizeCallback);
~~~

The callback records that the framebuffer size changed:

~~~cpp
void Window::framebufferResizeCallback(
    GLFWwindow* window, int width, int height)
{
    auto* instance =
        static_cast<Window*>(glfwGetWindowUserPointer(window));

    if (instance != nullptr)
    {
        instance->_framebufferResized = true;
        LOG_INFO("Framebuffer resized to "
                 << width << 'x' << height << " pixels.");
    }
}
~~~

The callback does not rebuild the swapchain itself. It only leaves a note for the frame loop. The loop can finish dealing with the current frame, then rebuild the resources in one place.

We use the **framebuffer size**, measured in pixels, because that is the size Vulkan uses for the images it draws into. On a high-resolution display, this can differ from the window size reported in screen coordinates.

## Notice when the old swapchain no longer fits

There are two useful signals:

1. The window callback tells us the framebuffer size changed.
2. Acquiring or presenting an image can tell us the swapchain is out of date or no longer ideal.

Here is the acquisition part of `drawFrame()`:

~~~cpp
auto [acquireResult, imageIndex] =
    _swapchain->handle().acquireNextImage(
        UINT64_MAX, *_imageAvailable[_currentFrame], nullptr);

if (acquireResult == vk::Result::eErrorOutOfDateKHR)
{
    // No image was acquired, so skip submission and rebuild.
    recreateSwapchain();
    return;
}

if (acquireResult != vk::Result::eSuccess &&
    acquireResult != vk::Result::eSuboptimalKHR)
    throw std::runtime_error("failed to acquire a swap-chain image");
~~~

An out-of-date result means we did not get an image to draw into. We skip recording and submitting this frame, recreate the swapchain, and try again on the next loop iteration.

A suboptimal result is different: the acquired image can still be used, but the swapchain should be refreshed. My code lets the current frame finish before rebuilding.

After submitting the drawing work, we ask the queue to present the image. Presentation can also report that the swapchain needs to be refreshed:

~~~cpp
const auto presentResult =
    _device->graphicsQueue().presentKHR(presentInfo);

if (presentResult != vk::Result::eSuccess &&
    presentResult != vk::Result::eSuboptimalKHR &&
    presentResult != vk::Result::eErrorOutOfDateKHR)
    throw std::runtime_error("failed to present a swap-chain image");

if (acquireResult == vk::Result::eSuboptimalKHR ||
    presentResult == vk::Result::eSuboptimalKHR ||
    presentResult == vk::Result::eErrorOutOfDateKHR ||
    _window->framebufferResized())
    recreateSwapchain();
~~~

Both acquisition and presentation matter: either can reveal that the old swapchain no longer matches the window. We also check the resize flag, since the callback may have told us about the change before Vulkan reports it.

## Rebuild the swapchain and its related resources

The swapchain is not the only object tied to its images and size. Our pipeline and recorded commands refer to swapchain details too, and the number of images can change. We need to wait for queued work to finish, release the old dependent objects, and make replacements.

Here is the recreation function:

~~~cpp
void recreateSwapchain()
{
    _device->logicalDevice().waitIdle();

    _pipeline.reset();
    _commandBuffers->reset();

    _swapchain->recreate();
    if (_window->shouldClose())
        return;

    _pipeline = std::make_unique<GraphicsPipeline>(
        *_device, *_swapchain, _shaderDirectory);

    createSwapchainSynchronization();
    _window->clearFramebufferResized();
}
~~~

`waitIdle()` makes sure queued GPU work has finished before we release resources it might still use.

We discard the pipeline and command buffers that refer to the old swapchain setup. Then `Swapchain::recreate()` builds the new swapchain and image views. With those ready, we create a pipeline for the new swapchain and reset synchronization objects that are sized according to the swapchain's image count.

The swapchain creation code queries the surface again and chooses a new extent. If the platform supplies a fixed extent, we use it. Otherwise, we read the current framebuffer size and clamp it to the range supported by the surface:

~~~cpp
vk::Extent2D Swapchain::chooseExtent(
    const vk::SurfaceCapabilitiesKHR& capabilities) const
{
    if (capabilities.currentExtent.width !=
        std::numeric_limits<uint32_t>::max())
        return capabilities.currentExtent;

    int width = 0;
    int height = 0;
    _window.framebufferSize(width, height);

    return {
        std::clamp<uint32_t>(
            static_cast<uint32_t>(width),
            capabilities.minImageExtent.width,
            capabilities.maxImageExtent.width),
        std::clamp<uint32_t>(
            static_cast<uint32_t>(height),
            capabilities.minImageExtent.height,
            capabilities.maxImageExtent.height)};
}
~~~

## Handle a minimized window

When a window is minimized, its framebuffer can have a width or height of zero. There is no useful swapchain size to create yet.

Before rebuilding, my code waits for the framebuffer to become drawable again:

~~~cpp
void Window::waitForDrawableSize() const
{
    int width = 0;
    int height = 0;
    framebufferSize(width, height);

    while ((width == 0 || height == 0) && !shouldClose())
    {
        glfwWaitEvents();
        framebufferSize(width, height);
    }
}
~~~

`glfwWaitEvents()` lets GLFW wait for a window event instead of spinning through the loop while the window is minimized. Once the framebuffer has a nonzero size, swapchain creation can continue. If the user closes the window while it is minimized, the loop ends.

## The resize flow

Putting it together:

1. GLFW reports a framebuffer resize, or acquisition/presentation reports an outdated swapchain.
2. We finish the current frame when possible.
3. We wait for queued work to finish and discard resources tied to the old swapchain.
4. We wait for a nonzero framebuffer size if the window is minimized.
5. We create a new swapchain, image views, pipeline, and image-count-based synchronization objects.
6. The next frame uses the new setup.

Now the triangle can follow the window as it changes size. The key is to treat resizing as a small rebuild: the swapchain changes, and the resources that depend on it must be refreshed too.
