---
title: "7. Creating a Vulkan Window Surface"
date: 2026-09-22 15:00:00 +0900
categories: [Graphics]
tags: [vulkan, graphics, gpu, c++, glfw]
description: Connecting Vulkan to a platform window with a window surface and a presentation-capable queue.
---

So far, Vulkan has been mostly platform-agnostic. We created an instance, inspected physical devices, and created a logical device without writing code specifically for Windows, Linux, or macOS.

Now we want to show the result of rendering in an operating-system window. This is where Vulkan's platform-independent design meets the platform-specific window system.

The matching sample is [06_window_surface](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/06_window_surface) in [The Journey of Learning Vulkan](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan).

## What is a window surface?

Vulkan is a platform-agnostic API. It does not assume that the application is running on Windows, Linux, macOS, Android, or an embedded device. That is useful for a graphics API, but it means Vulkan cannot simply guess how to communicate with the local window manager.

A **window surface** is Vulkan's abstract representation of a place where rendered images can be presented. It is represented by a `VkSurfaceKHR` handle.

The surface is not the window itself, and it is not a rendered image. The window is created and owned by the operating system or a windowing library. The surface is the Vulkan-side connection to that window:

```text
Vulkan image
     │
     ▼
VkSurfaceKHR  ──────  operating-system window
     │
     ▼
presentation to the screen
```

Later, a swapchain will provide a collection of images that can be rendered into and presented through this surface. For now, the important idea is that the surface tells Vulkan where presentation is supposed to happen.

## Every platform has its own window system

The operating system is not one uniform abstraction. Each platform has its own native window handles and presentation mechanisms. Even a single platform can have more than one window system available.

For example, Vulkan exposes platform-specific Window System Integration (WSI) extensions such as:

- Windows: `VK_KHR_win32_surface`, which works with Win32 window handles;
- Linux/X11: `VK_KHR_xcb_surface` or `VK_KHR_xlib_surface`;
- Linux/Wayland: `VK_KHR_wayland_surface`;
- macOS: commonly `VK_EXT_metal_surface`, used with a Metal layer through MoltenVK;
- Android: `VK_KHR_android_surface`;
- other embedded or device-specific systems: their own native WSI extensions and window types.

These are not interchangeable. A Win32 `HWND`, an X11 window, a Wayland `wl_surface`, and a macOS Metal layer are different native objects. The correct surface extension must be enabled and the correct native handle must be passed when creating the surface.

## Creating a surface directly with Vulkan

Vulkan provides native surface creation through extensions. The exact function depends on the platform. A simplified Windows example looks like this:

```cpp
#define VK_USE_PLATFORM_WIN32_KHR
#include <vulkan/vulkan.h>

VkWin32SurfaceCreateInfoKHR surfaceInfo{
    .sType = VK_STRUCTURE_TYPE_WIN32_SURFACE_CREATE_INFO_KHR,
    .hinstance = GetModuleHandle(nullptr),
    .hwnd = nativeWindowHandle
};

VkSurfaceKHR surface = VK_NULL_HANDLE;
vkCreateWin32SurfaceKHR(instance, &surfaceInfo, nullptr, &surface);
```

Linux, macOS, Android, and other platforms use different structures and functions. The application would need platform-specific code, platform-specific headers, and a way to obtain each platform's native window handle.

That work is sometimes exactly what an engine or a windowing layer needs. However, it would distract from the GPU-rendering concepts in this tutorial. We want to concentrate on Vulkan itself, not maintain a separate window implementation for every operating system.

## GLFW hides the platform differences

GLFW has already done this portability work for us. It provides one cross-platform API for creating a window and one function for creating the Vulkan surface associated with that window.

First, we create a GLFW window without asking GLFW to create an OpenGL context:

```cpp
if (glfwInit() == GLFW_FALSE)
    throw std::runtime_error("failed to initialize GLFW");

glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);

GLFWwindow* window = glfwCreateWindow(
    800, 600, "Vulkan Triangle", nullptr, nullptr);
```

Before creating the Vulkan instance, GLFW also provides the platform-specific instance extensions required for Vulkan window-system integration. The sample adds those names to the instance creation information. After the instance exists, GLFW creates the surface with a pleasantly platform-independent call:

```cpp
VkSurfaceKHR rawSurface = VK_NULL_HANDLE;

if (glfwCreateWindowSurface(
        instance, window, nullptr, &rawSurface) != VK_SUCCESS)
{
    throw std::runtime_error("failed to create window surface");
}
```

Internally, GLFW selects the appropriate Vulkan WSI function for the platform. On Windows it can use the Win32 surface extension; on Linux it can use X11 or Wayland; on macOS it can use the Vulkan portability implementation and its Metal surface integration. Our tutorial code stays the same.

In the sample, the raw C handle is wrapped in the Vulkan-Hpp RAII type:

```cpp
surface = vk::raii::SurfaceKHR(instance, rawSurface);
```

The surface must be created after the instance, because it is an instance-level Vulkan object. It is created before selecting the physical device because presentation support is part of deciding whether a GPU is suitable for this application.

## A rendering queue must also be able to present

Creating a surface does not mean that every GPU or every queue can present to it. A physical device may expose several queue families, and presentation support is checked for a particular queue family and a particular surface.

For this sample, we look for one queue family that supports both graphics commands and presentation:

```cpp
uint32_t findGraphicsAndPresentQueueFamily(
    const vk::raii::PhysicalDevice& physicalDevice,
    const vk::raii::SurfaceKHR& surface)
{
    const auto queueFamilies = physicalDevice.getQueueFamilyProperties();

    for (uint32_t index = 0; index < queueFamilies.size(); ++index)
    {
        const bool supportsGraphics =
            (queueFamilies[index].queueFlags & vk::QueueFlagBits::eGraphics) !=
            vk::QueueFlags{};

        const bool supportsPresentation =
            physicalDevice.getSurfaceSupportKHR(index, *surface);

        if (supportsGraphics && supportsPresentation)
            return index;
    }

    throw std::runtime_error(
        "failed to find a queue family for graphics and presentation");
}
```

The sample uses this check both while choosing the physical device and while creating the logical device. A GPU is not suitable for this windowed renderer unless it can perform the required rendering work and present images to this particular surface.

It is possible for graphics and presentation to be provided by different queue families. That can be useful in more advanced applications, but this first renderer keeps the setup simple and looks for one family supporting both operations.

## A surface is optional

Although this chapter creates a window surface, Vulkan does not require one. Vulkan can render entirely off-screen when there is no screen to update.

Some examples include:

- cloud gaming, where a server renders frames and sends encoded video to a client;
- AI and machine-learning workloads that use Vulkan compute without displaying pixels;
- offline image or video generation;
- server-side rendering and automated testing;
- rendering into textures that another system consumes.

In those cases, the application can render to its own images or use Vulkan for compute work without creating a window or a `VkSurfaceKHR`. A surface is specifically the bridge to a window-system presentation target.

For more detail, see the [Window surface chapter of the Vulkan Tutorial](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/01_Presentation/00_Window_surface.html) and the [06_window_surface sample](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/06_window_surface).
