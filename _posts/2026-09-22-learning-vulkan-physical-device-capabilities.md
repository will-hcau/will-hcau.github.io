---
title: "Episode 5: Checking Vulkan Physical Device Capabilities"
date: 2026-09-22 12:00:00 +0900
categories: [Vulkan]
tags: [vulkan, graphics, gpu, c++]
description: Enumerating GPUs and selecting one that supports the Vulkan features, queues, and extensions our application needs.
---

Today I continued with the [physical devices and queue families chapter of the Vulkan Tutorial](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/00_Setup/03_Physical_devices_and_queue_families.html). The matching sample is [04_device_capability_check](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/04_device_capability_check) in [The Journey of Learning Vulkan](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan).

So far, the application has created a window, connected to Vulkan through an instance, and enabled a validation layer. Those are software-side setup steps. The next question is different: **can the physical GPU actually provide the capabilities that our renderer needs?**

## From the Instance to a Physical Device

The Vulkan instance is the connection between the application and the Vulkan implementation. Through that instance, we can enumerate the physical devices visible to the system:

```cpp
const auto physicalDevices = instance.enumeratePhysicalDevices();

if (physicalDevices.empty())
{
    throw std::runtime_error("failed to find GPUs with Vulkan support");
}
```

`physicalDevices` contains handles to the GPUs that expose Vulkan support. At this point we have not created a logical device and we are not submitting any work. We are only asking the Vulkan implementation what hardware is available.

The object we select is a `vk::raii::PhysicalDevice`:

```cpp
vk::raii::PhysicalDevice physicalDevice = nullptr;
```

A physical device represents the real GPU and its immutable capabilities. Later, we will use it to create a logical device, which is the interface through which the application actually submits commands.

## Inspecting Device Properties and Features

Vulkan separates descriptive properties from optional features. Properties tell us about the device; features tell us which parts of Vulkan the device can support.

```cpp
const auto properties = candidate.getProperties();
const auto features = candidate.getFeatures();

std::cout << "GPU: " << properties.deviceName << '\n';
std::cout << "API version: " << properties.apiVersion << '\n';
std::cout << "Geometry shader: "
          << (features.geometryShader ? "yes" : "no") << '\n';
```

Properties include the device name, device type, supported API version, and implementation limits such as the maximum 2D image dimension. Features include optional functionality such as geometry shaders, texture compression, and other capabilities.

The application must not simply assume that a feature exists because the GPU supports Vulkan. It needs to query the feature and then enable it when the logical device is created.

## Choosing Between Multiple GPUs

Real machines can have more than one Vulkan-capable GPU. A laptop might expose an integrated GPU and a discrete GPU, for example. Selecting the first device is easy, but it may not select the best device.

A simple scoring system gives desirable devices a higher score while still allowing a fallback:

```cpp
uint32_t score = 0;

if (properties.deviceType == vk::PhysicalDeviceType::eDiscreteGpu)
    score += 1000;

score += properties.limits.maxImageDimension2D;

if (!features.geometryShader)
    score = 0;
```

The discrete-GPU bonus favours a dedicated graphics card. The image-dimension limit is a rough proxy for capability. A missing mandatory feature makes the candidate unusable, regardless of its score.

My sample keeps the selection logic deliberately simpler: it chooses the first GPU that satisfies every required condition. A scoring system can be added later when there are meaningful preferences between otherwise suitable devices.

## The Capabilities Required by This Example

The tutorial and this sample require four kinds of support:

1. Vulkan 1.3 support;
2. a queue family that supports graphics commands;
3. the device swapchain extension;
4. several optional features used by later rendering steps.

### Vulkan 1.3

The sample uses functionality introduced by Vulkan 1.3, so the device API version must be at least 1.3:

```cpp
bool supportsVulkan13(const vk::raii::PhysicalDevice& candidate) const
{
    return candidate.getProperties().apiVersion >= vk::ApiVersion13;
}
```

This is a version check, not a feature check. It tells us whether the implementation exposes the Vulkan 1.3 API level. Individual features still need to be queried separately.

### A graphics queue family

Vulkan work is submitted to queues. Queues are grouped into queue families, and each family advertises the kinds of commands it accepts. A graphics queue can process drawing commands:

```cpp
bool supportsGraphicsQueue(const vk::raii::PhysicalDevice& candidate) const
{
    const auto families = candidate.getQueueFamilyProperties();

    return std::ranges::any_of(families,
        [](const auto& family)
        {
            return (family.queueFlags & vk::QueueFlagBits::eGraphics) !=
                   vk::QueueFlags{};
        });
}
```

This does not select a queue yet. It only confirms that at least one family can support graphics operations. Queue selection and logical-device creation will use this information in the next step.

### `VK_KHR_swapchain`

The Vulkan API itself does not know how to present an image to our GLFW window. The `VK_KHR_swapchain` device extension provides the swapchain functionality used to acquire images for rendering and present completed images to the window system.

The application keeps its required device extensions in one list:

```cpp
const std::vector<const char*> kRequiredDeviceExtensions = {
    vk::KHRSwapchainExtensionName};
```

Before selecting a device, it compares that list with the extensions exposed by the candidate:

```cpp
bool supportsRequiredExtensions(
    const vk::raii::PhysicalDevice& candidate) const
{
    const auto available = candidate.enumerateDeviceExtensionProperties();

    return std::ranges::all_of(kRequiredDeviceExtensions,
        [&available](const char* required)
        {
            return std::ranges::any_of(available,
                [required](const auto& extension)
                {
                    return std::strcmp(extension.extensionName, required) == 0;
                });
        });
}
```

The important detail is that every required extension must be found. One missing extension is enough to reject the GPU.

### Required features

The sample also checks three features that will be used by later tutorial steps:

- `shaderDrawParameters` from Vulkan 1.1: allows shader stages to access draw-related parameters supplied by the command;
- `dynamicRendering` from Vulkan 1.3: allows rendering attachments to be specified without creating a traditional render pass object;
- `extendedDynamicState` from `VK_EXT_extended_dynamic_state`: allows more pipeline state to be changed dynamically rather than baked into the pipeline.

These features are returned through a chain of feature structures:

```cpp
bool supportsRequiredFeatures(
    const vk::raii::PhysicalDevice& candidate) const
{
    const auto features = candidate.template getFeatures2<
        vk::PhysicalDeviceFeatures2,
        vk::PhysicalDeviceVulkan11Features,
        vk::PhysicalDeviceVulkan13Features,
        vk::PhysicalDeviceExtendedDynamicStateFeaturesEXT>();

    const auto& vulkan11 =
        features.template get<vk::PhysicalDeviceVulkan11Features>();
    const auto& vulkan13 =
        features.template get<vk::PhysicalDeviceVulkan13Features>();
    const auto& dynamicState =
        features.template get<vk::PhysicalDeviceExtendedDynamicStateFeaturesEXT>();

    return vulkan11.shaderDrawParameters &&
           vulkan13.dynamicRendering &&
           dynamicState.extendedDynamicState;
}
```

Querying a feature does not enable it. The query only tells us whether the device can provide it. When we create the logical device, we must put the supported features we intend to use into the device-create feature chain.

## Combining the Checks

A device is suitable only when it passes every mandatory test:

```cpp
bool isDeviceSuitable(const vk::raii::PhysicalDevice& candidate) const
{
    return supportsVulkan13(candidate) &&
           supportsGraphicsQueue(candidate) &&
           supportsRequiredExtensions(candidate) &&
           supportsRequiredFeatures(candidate);
}
```

Finally, `pickPhysicalDevice` enumerates the candidates and stores the first suitable one:

```cpp
void pickPhysicalDevice()
{
    const auto devices = instance.enumeratePhysicalDevices();

    if (devices.empty())
        throw std::runtime_error("failed to find GPUs with Vulkan support");

    const auto selected = std::ranges::find_if(devices,
        [this](const auto& candidate)
        {
            return isDeviceSuitable(candidate);
        });

    if (selected == devices.end())
        throw std::runtime_error("failed to find a suitable GPU");

    physicalDevice = *selected;
    std::cout << "Selected GPU: "
              << physicalDevice.getProperties().deviceName << '\n';
}
```

The initialization order now looks like this:

```text
Create window
    ↓
Create Vulkan instance
    ↓
Set up validation messenger
    ↓
Enumerate physical devices
    ↓
Check version, queues, extensions, and features
    ↓
Select a suitable physical device
```

This step changed the program from merely having access to Vulkan into having a verified piece of GPU hardware for the renderer. The next step is to create a logical device and the queues that will receive our commands.

*This post is based on the [physical devices and queue families chapter](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/00_Setup/03_Physical_devices_and_queue_families.html) of the Vulkan Tutorial. The code is my own implementation in [The Journey of Learning Vulkan](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/04_device_capability_check).*
