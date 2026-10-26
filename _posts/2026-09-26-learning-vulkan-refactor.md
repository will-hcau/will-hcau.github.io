---
title: "Episode 10: Refactoring the Vulkan Setup"
date: 2026-09-26 13:00:00 +0900
categories: [Vulkan]
tags: [vulkan, graphics, gpu, c++]
description: Splitting Vulkan initialisation into small classes with clear responsibilities.
---

So far, the Vulkan examples have followed the tutorial one step at a time. That approach made each new API concept easy to introduce, but it also left us with a problem: by the [image views chapter](https://will-hcau.github.io/posts/learning-vulkan-image-views/), almost the entire setup lived in `main.cpp` and the `SimpleTriangle` class.

The matching refactored sample is [09_refactor](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/09_refactor) in [The Journey of Learning Vulkan](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan). This is the last cleanup chapter before moving on to the GPU pipeline.

The goal is not to hide Vulkan. It is to give each part of the program one job. If we want to understand something about the device, we should be able to open `device.cpp`. If we want to understand the swapchain, we should not need to search through window creation and validation-layer code first.

## The problem with the earlier implementation

The implementation from the [image views chapter](https://will-hcau.github.io/posts/learning-vulkan-image-views/), available in the [`08_image_view` GitHub example](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/08_image_view), already does several logically different things:

```text
SimpleTriangle
 ├── creates the GLFW window
 ├── creates the Vulkan instance
 ├── creates the debug messenger
 ├── creates the window surface
 ├── selects a physical device
 ├── checks extensions and features
 ├── creates the logical device and queue
 ├── selects swapchain settings
 ├── creates the swapchain
 └── creates image views
```

This is manageable while the example is small. It becomes uncomfortable when every new Vulkan chapter adds another group of methods and member variables. The class also becomes responsible for unrelated rules: GLFW lifetime, instance extensions, device capabilities, queue-family selection, and presentation formats.

There is another source of duplication: Vulkan capability checks appear at several levels. We need to check instance extensions, physical-device features and extensions, queue families, and swapchain capabilities. It is technically possible to hard-code these checks wherever they are needed, but that becomes difficult to manage as the application evolves. When we add or remove a feature or extension, a human has to remember every place that must change: instance setup, device selection, logical-device creation, or swapchain setup. Missing one of those changes can lead to confusing initialization failures or a device being accepted without actually supporting everything the application expects. Keeping each group of requirements with the component that owns it makes the policy easier to find and maintain.

The refactor gives each responsibility a home.

## The new structure

The new sample has this shape:

```text
09_refactor/
 ├── main.cpp
 ├── include/
 │   ├── window.hpp
 │   ├── vulkan_instance.hpp
 │   ├── device.hpp
 │   ├── swapchain.hpp
 │   └── vulkan_config.hpp
 └── src/
     ├── window.cpp
     ├── vulkan_instance.cpp
     ├── device.cpp
     └── swapchain.cpp
```

The header describes the public interface and the source file contains the implementation. This is a useful boundary even in a small project: callers can use a `Device` without knowing how its physical-device checks are implemented.

### `Window`: GLFW ownership

`Window` owns the GLFW window and the process-wide GLFW lifetime:

```cpp
class Window
{
public:
    Window(uint32_t width, uint32_t height, const char* title);
    ~Window();

    bool shouldClose() const;
    void pollEvents() const;
    void framebufferSize(int& width, int& height) const;
    GLFWwindow* handle() const { return _window; }

private:
    GLFWwindow* _window = nullptr;
};
```

The constructor calls `glfwInit`, creates a window without an OpenGL context, and throws if either operation fails. The destructor destroys the native window and calls `glfwTerminate`.

This is a good fit for RAII: the object owns a resource, and its lifetime tells us exactly when that resource should be released. The class is also non-copyable because two objects must not try to own the same `GLFWwindow*`.

`Window` exposes only the operations that the rest of the application needs. In particular, swapchain extent selection can ask for the framebuffer size without knowing how GLFW stores the native handle.

### `VulkanInstance`: instance-level Vulkan setup

`VulkanInstance` owns the `vk::raii::Context`, the Vulkan instance, and the optional debug messenger. It is also the place where instance extensions are assembled and validated.

```cpp
class VulkanInstance
{
public:
    VulkanInstance();

    vk::raii::Instance& instance() { return _instance; }
    vk::raii::SurfaceKHR createSurface(const Window& window) const;

private:
    std::vector<const char*> requiredExtensions() const;
    void checkValidationLayerSupport() const;
};
```

GLFW contributes platform-specific instance extensions at runtime, so this class combines those extensions with the configured requirements. Validation-layer support and the debug callback also belong here because they are instance-level facilities.

Creating a surface still requires a window, but the surface itself is a Vulkan object. `createSurface` is therefore a useful bridge between the two components instead of making either class responsible for the other one.

### `Device`: physical-device selection and logical-device creation

The device class contains the rules for deciding whether a GPU is suitable:

```cpp
struct QueueFamilyIndices
{
    std::optional<uint32_t> graphicsAndPresent;

    bool complete() const { return graphicsAndPresent.has_value(); }
};

class Device
{
public:
    Device(const VulkanInstance& instance,
           const vk::raii::SurfaceKHR& surface);

    QueueFamilyIndices queueFamilies() const;
    const vk::raii::PhysicalDevice& physicalDevice() const;
    const vk::raii::Device& logicalDevice() const;
};
```

During construction, `Device` enumerates the physical devices exposed by the instance. For each candidate it checks:

1. the required Vulkan API version;
2. a queue family that supports both graphics and presentation;
3. the required device extensions; and
4. the features needed by this chapter.

Those checks are now named methods in `device.cpp`: `findQueueFamilies`, `supportsExtensions`, `supportsFeatures`, and `suitable`. The selection policy is consequently visible in one place:

```cpp
return device.getProperties().apiVersion >= vulkan_config::apiVersion &&
       indices.complete() &&
       supportsExtensions(device) &&
       supportsFeatures(device);
```

This also fixes an important conceptual separation. Querying a feature is not the same as enabling it. `supportsFeatures` reads the feature chain from the physical device. Only after a candidate passes that test does the constructor create the logical device with the corresponding feature chain requested.

The class returns the two objects that later work needs: the selected physical device for capability queries, and the logical device for creating Vulkan resources. The swapchain does not need to know how the selection was performed.

## Configuration is data, not scattered policy

The refactor also expands `vulkan_config.hpp`. Values that describe this application are collected there:

```cpp
inline constexpr uint32_t windowWidth = 800;
inline constexpr uint32_t windowHeight = 600;
inline constexpr const char* applicationName = "Vulkan Triangle";
inline constexpr uint32_t apiVersion = vk::ApiVersion14;

inline constexpr std::array<const char*, 1> requiredDeviceExtensions = {
    vk::KHRSwapchainExtensionName};

inline constexpr bool requireDynamicRendering = true;
```

This avoids repeating the same extension names, dimensions, and feature decisions across several `.cpp` files. The code that consumes the configuration remains responsible for applying it: `VulkanInstance` uses the instance requirements, while `Device` uses the device extensions and feature requirements.

Validation is selected at build time with `VULKAN_ENABLE_VALIDATION`. That keeps the debug messenger code available in debug builds without forcing release builds to request the validation layer.

## `Swapchain`: presentation-specific setup

The swapchain now depends on three already-created objects:

```cpp
Swapchain(const Device& device,
          const vk::raii::SurfaceKHR& surface,
          const Window& window);
```

It queries the selected physical device for surface capabilities, formats, and present modes. It then chooses the format, present mode, extent, and image count before creating the swapchain.

The image views have moved here as well. They describe how the swapchain's images are accessed, so they belong with the swapchain rather than with the general application class. The constructor creates a view for every image returned by the swapchain:

```cpp
for (const vk::Image image : images)
{
    const vk::ImageViewCreateInfo viewInfo{
        .image = image,
        .viewType = vk::ImageViewType::e2D,
        .format = format.format,
        .subresourceRange = {
            .aspectMask = vk::ImageAspectFlagBits::eColor,
            .levelCount = 1,
            .layerCount = 1}};

    _imageViews.emplace_back(_device.logicalDevice(), viewInfo);
}
```

The class can access `Device::physicalDevice()` for capability queries and `Device::logicalDevice()` for resource creation, but it does not need to duplicate any device-selection logic.

## `main.cpp` becomes an application diagram

After moving the implementation out, `SimpleTriangle` is much smaller:

```cpp
void setup()
{
    _window = std::make_unique<Window>(
        vulkan_config::windowWidth,
        vulkan_config::windowHeight,
        vulkan_config::applicationName);

    _instance = std::make_unique<VulkanInstance>();
    _surface = std::make_unique<vk::raii::SurfaceKHR>(
        _instance->createSurface(*_window));

    _device = std::make_unique<Device>(*_instance, *_surface);
    _swapchain = std::make_unique<Swapchain>(
        *_device, *_surface, *_window);
}
```

This order is not arbitrary:

```text
Window
  ↓
VulkanInstance
  ↓
Surface
  ↓
Device
  ↓
Swapchain and image views
```

Each layer needs something created by the layer above it. The reverse destruction order is just as important. `Swapchain` must disappear before `Device`, the surface before the instance, and the window before GLFW terminates. The `unique_ptr` members make that dependency order explicit: members are destroyed in reverse declaration order.

The render loop is still intentionally simple:

```cpp
while (!_window->shouldClose())
    _window->pollEvents();
```

There is no rendering yet. That is exactly why this is a useful stopping point for the refactor. The application now has a clean place to add command pools, command buffers, synchronisation, render passes, pipelines, and frame rendering without turning one class into another 500-line file.

## Why this structure is worth keeping

The refactor does not reduce the number of Vulkan concepts. It reduces the number of concepts that must be held in mind at the same time.

When debugging a missing device extension, open `device.cpp`. When debugging a validation message or instance extension, open `vulkan_instance.cpp`. When debugging an incorrect swapchain format or extent, open `swapchain.cpp`. The file layout now follows the Vulkan ownership and dependency model.

There is also a practical benefit for the next chapters. New GPU-pipeline code can depend on `Device` and `Swapchain` without knowing how they were constructed. The setup code becomes a stable foundation, while the rendering code can grow separately.

The project has now reached the point where the scaffolding is tidy enough to support actual rendering. Next, we can start building the GPU pipeline.
