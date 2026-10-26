---
title: "Episode 6: Creating a Vulkan Logical Device"
date: 2026-09-22 13:00:00 +0900
categories: [Vulkan]
tags: [vulkan, graphics, gpu, c++]
description: Creating a logical device, enabling application features, and retrieving a graphics queue.
---

This should be the last missing piece of the puzzle before we actually render something using the GPU.

The previous step selected a suitable physical device by checking its Vulkan version, queue families, extensions, and features. The next step is to create a **logical device**: the interface through which our application will use that physical device.

The matching sample is [05_logical_device](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/05_logical_device) in [The Journey of Learning Vulkan](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan).

## Physical device versus logical device

A physical device represents the real Vulkan-capable hardware discovered through the instance. It describes what the GPU can do: its properties, limits, queue families, extensions, and optional features.

It is useful to think of it as a catalogue of capabilities. Querying it answers questions such as:

- Does this GPU support Vulkan 1.3?
- Does it have a queue family capable of graphics commands?
- Does it support `VK_KHR_swapchain`?
- Does it support dynamic rendering?

The physical device is not yet an application-facing execution context. We do not submit command buffers directly to it. Instead, we create a logical device from it.

<object type="image/svg+xml" data="{{ '/assets/img/vulkan-logical-device.svg' | relative_url }}" width="100%" height="auto" style="max-width: 100%; aspect-ratio: 1100 / 650;" aria-label="One physical device can provide several logical devices, each with different enabled features and queues">
  <a href="{{ '/assets/img/vulkan-logical-device.svg' | relative_url }}">View the physical-device and logical-device diagram</a>
</object>

The logical devices above are separate application interfaces with separate enabled features and queues. They all refer to the same physical GPU, but each one is created with a particular configuration. In a normal application we usually create one logical device, but Vulkan permits multiple logical devices when their different requirements are useful.

This also explains why checking and enabling features are separate operations. The physical device tells us what is possible. Logical-device creation selects what this application will use.

## What goes into `DeviceCreateInfo`?

The central structure is `vk::DeviceCreateInfo`. It describes the logical device we want to create. The important inputs for this example are:

1. queue-create information;
2. the features to enable;
3. required device extensions;
4. optionally, allocation callbacks and other extension-specific structures through `pNext`.

The final structure looks like this:

```cpp
vk::DeviceCreateInfo deviceCreateInfo{
    .pNext = &featureChain.get<vk::PhysicalDeviceFeatures2>(),
    .queueCreateInfoCount = 1,
    .pQueueCreateInfos = &deviceQueueCreateInfo,
    .enabledExtensionCount =
        static_cast<uint32_t>(requiredDeviceExtensions.size()),
    .ppEnabledExtensionNames = requiredDeviceExtensions.data()
};

device = vk::raii::Device(physicalDevice, deviceCreateInfo);
```

The logical device is created from the already-selected `physicalDevice`. If we request an unsupported feature or extension, device creation fails.

## Queues: where GPU work is submitted

Vulkan does not submit work directly to a GPU as one undifferentiated stream. Work is submitted to **queues**. A queue accepts command buffers and sends their commands to the GPU for execution.

Queues are grouped into **queue families**. Each family supports a particular set of operations, advertised with flags such as:

- `eGraphics` for drawing commands;
- `eCompute` for compute work;
- `eTransfer` for copy operations;
- `eSparseBinding` for sparse-memory operations.

The previous step only checked that a suitable family exists. During logical-device creation we must say which family we want to use and how many queues to create from it.

```cpp
const auto queueFamilies = physicalDevice.getQueueFamilyProperties();

const auto graphicsFamily = std::ranges::find_if(
    queueFamilies,
    [](const auto& family)
    {
        return (family.queueFlags & vk::QueueFlagBits::eGraphics) !=
               vk::QueueFlags{};
    });

const uint32_t graphicsIndex = static_cast<uint32_t>(
    std::distance(queueFamilies.begin(), graphicsFamily));

float queuePriority = 0.5f;

vk::DeviceQueueCreateInfo deviceQueueCreateInfo{
    .queueFamilyIndex = graphicsIndex,
    .queueCount = 1,
    .pQueuePriorities = &queuePriority
};
```

`queueFamilyIndex` selects the family. `queueCount` requests one queue from that family. The queue priority is a value between `0.0f` and `1.0f` that can influence scheduling when several queues compete for execution. It is required even when there is only one queue.

Creating more queues is not automatically faster. The number available is limited by the family, and command buffers can be recorded on several CPU threads before being submitted through a single queue. One graphics queue is enough for this first renderer.

After the device is created, we retrieve the queue handle:

```cpp
graphicsQueue = vk::raii::Queue(device, graphicsIndex, 0);
```

The final `0` is the index of the queue within the selected family. The queue was created together with the logical device; this call obtains the handle we use later for command submission.

## Enabling features

When we inspected the physical device, we queried whether required features were supported. That query did not enable them. It only answered whether enabling them would be valid.

For example, the sample checks these features:

- `shaderDrawParameters`, introduced in Vulkan 1.1;
- `dynamicRendering`, introduced in Vulkan 1.3;
- `extendedDynamicState`, provided by `VK_EXT_extended_dynamic_state`.

The application enables the features it intends to use by putting them into a feature chain:

```cpp
vk::StructureChain<
    vk::PhysicalDeviceFeatures2,
    vk::PhysicalDeviceVulkan11Features,
    vk::PhysicalDeviceVulkan13Features,
    vk::PhysicalDeviceExtendedDynamicStateFeaturesEXT
> featureChain = {
    {},
    {.shaderDrawParameters = true},
    {.dynamicRendering = true},
    {.extendedDynamicState = true}
};
```

`vk::PhysicalDeviceFeatures2` is the starting container for the feature chain. It is not itself an extra GPU capability that we need to understand; it provides the modern, extensible way to connect the original Vulkan 1.0 features with newer feature structures.

The structures are linked through their `pNext` fields. Assigning the first structure to `DeviceCreateInfo::pNext` lets Vulkan follow the complete chain while creating the device.

```text
DeviceCreateInfo
      │ pNext
      ▼
PhysicalDeviceFeatures2
      │ pNext
      ▼
Vulkan11Features
      │ pNext
      ▼
Vulkan13Features
      │ pNext
      ▼
ExtendedDynamicStateFeaturesEXT
```

This is why the feature query and feature enablement look similar but occur at different times: one reads the physical device's capability catalogue, while the other writes the selected application configuration into the new logical device.

## Enabling device extensions

Extensions are enabled on the logical device in the same general pattern. The sample needs the swapchain extension so that rendered images can be presented to the window system:

```cpp
const std::vector<const char*> requiredDeviceExtensions = {
    vk::KHRSwapchainExtensionName
};
```

On platforms such as macOS, MoltenVK may also expose `VK_KHR_portability_subset`. The sample adds that extension when the physical device advertises it. This is a portability detail rather than a capability that every Vulkan implementation must provide:

```cpp
const bool supportsPortabilitySubset = std::ranges::any_of(
    available,
    [](const auto& extension)
    {
        return std::strcmp(extension.extensionName,
                           "VK_KHR_portability_subset") == 0;
    });

if (supportsPortabilitySubset)
    extensions.push_back("VK_KHR_portability_subset");
```

The physical-device selection step first verifies that the extension is available. The logical-device creation step then enables it. Without `VK_KHR_swapchain`, Vulkan can execute rendering commands but does not provide the device-side swapchain functionality needed to present images to the window.

## The complete setup

Putting the pieces together, logical-device creation has this shape:

```text
Selected physical device
          │
          ├── choose a graphics queue family
          ├── select supported features to enable
          ├── list required device extensions
          └── create VkDevice
                       │
                       └── retrieve VkQueue
```

At this point the application has a usable logical device and a graphics queue. The initialization flow is now:

```text
Create window
    ↓
Create Vulkan instance
    ↓
Enable validation and debug messenger
    ↓
Select a suitable physical device
    ↓
Create logical device and queues
    ↓
Create rendering resources
```

The physical device told us what the GPU could provide. The logical device now represents the subset of that capability that this application has explicitly requested. With the queue handle in hand, we are finally ready to create command buffers and submit actual work to the GPU.

For more detail, this step follows the [logical device and queues chapter of the Vulkan Tutorial](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/00_Setup/04_Logical_device_and_queues.html).
