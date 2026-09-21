---
title: "4. Adding Vulkan Validation Layers"
date: 2026-09-21 12:00:00 +0900
categories: [Graphics]
tags: [vulkan, graphics, validation, c++]
description: Enabling Vulkan's validation layer and registering a debug callback.
---

The next step in my Vulkan journey is based on the [validation layers chapter of the Vulkan Tutorial](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/00_Setup/02_Validation_layers.html). The sample is available in [The Journey of Learning Vulkan](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/03_validation_layer).

Vulkan works close to the hardware. That gives us control, but it also means that a small mistake can cause undefined behaviour, a crash, or a failure much later than the code that caused it. Vulkan therefore provides a development feature called a **validation layer**.

## What Is a Layer?

A layer is an optional component inserted into Vulkan's command call chain:

```text
Application
    ↓
Validation layer
    ↓
Vulkan loader and driver
    ↓
GPU
```

When the application calls a Vulkan function, an enabled layer can inspect its arguments before forwarding the call. It can also inspect the result afterward. Layers can provide:

- validation of Vulkan's valid-usage rules;
- useful warnings about incorrect or suspicious API usage;
- logging and API-call tracing;
- performance analysis and profiling;
- frame capture and debugging tools;
- additional API behaviour or extensions in specialised cases.

The validation layer is normally enabled during development and disabled in a release build. Its checks add overhead, but that overhead is valuable while we are finding mistakes.

### Instance and device layers

Layers are associated with different levels of the Vulkan object hierarchy. An **instance layer** is enabled when the `VkInstance` is created and can observe instance-level calls. Because a logical device is created from that instance, the enabled layer chain also applies to device-level calls made through it.

Historically, Vulkan also exposed separate **device layers**, enabled when creating a `VkDevice`. That mechanism is now legacy and should not be used for new applications. Modern validation is enabled as an instance layer, and the `VK_LAYER_KHRONOS_validation` layer then validates both instance and device usage.

## Enabling the Validation Layer

There are five steps in this sample:

1. Check whether `VK_LAYER_KHRONOS_validation` is supported.
2. Build the list of layers requested by the application.
3. Pass that list to `vk::InstanceCreateInfo` when creating the instance.
4. Create a debug messenger for the instance.
5. Handle messages in a callback.

### 1. Check whether the layer is supported

The Vulkan SDK normally installs the Khronos validation layer, but the application should not assume that it exists. It first enumerates the available layers and compares their names with the layer we want:

```cpp
constexpr const char* kValidationLayer =
    "VK_LAYER_KHRONOS_validation";

const auto availableLayers = context.enumerateInstanceLayerProperties();

const bool validationLayerAvailable = std::ranges::any_of(
    availableLayers,
    [](const auto& layer)
    {
        return std::strcmp(layer.layerName, kValidationLayer) == 0;
    });

if (!validationLayerAvailable)
{
    throw std::runtime_error("validation layer is not available");
}
```

This is similar to checking extensions. Vulkan exposes a list of capabilities, and the application requests only the names it actually needs.

### 2. Make a list of requested layers

The instance creation structure expects a count and a pointer to an array of layer names. Keeping the names in a vector makes the list easy to extend later:

```cpp
const std::vector<const char*> layers{
    kValidationLayer};
```

The layer name is a string supplied by the Vulkan implementation. It is not a C++ class that we construct ourselves.

### 3. Enable the layer during instance creation

We pass the layer list through the same `VkInstanceCreateInfo` structure that already contains the application information and required instance extensions:

```cpp
const vk::InstanceCreateInfo createInfo{
    .pApplicationInfo = &appInfo,
    .enabledLayerCount = static_cast<uint32_t>(layers.size()),
    .ppEnabledLayerNames = layers.data(),
    .enabledExtensionCount = static_cast<uint32_t>(extensions.size()),
    .ppEnabledExtensionNames = extensions.data()};

instance = vk::raii::Instance(context, createInfo);
```

At this point the validation layer is part of the instance's call chain. It can now observe Vulkan commands associated with this instance and the devices created from it.

## Registering a Debug Messenger

Enabling the validation layer makes validation possible, but we still need somewhere for its messages to go. `VK_EXT_debug_utils` provides a debug messenger that forwards selected messages to an application callback.

The messenger is connected to the instance and is configured with two filters:

- **message severity**: how serious the message is;
- **message type**: what kind of event produced it.

```cpp
const vk::DebugUtilsMessengerCreateInfoEXT messengerCreateInfo{
    .messageSeverity =
        vk::DebugUtilsMessageSeverityFlagBitsEXT::eVerbose |
        vk::DebugUtilsMessageSeverityFlagBitsEXT::eInfo |
        vk::DebugUtilsMessageSeverityFlagBitsEXT::eWarning |
        vk::DebugUtilsMessageSeverityFlagBitsEXT::eError,
    .messageType =
        vk::DebugUtilsMessageTypeFlagBitsEXT::eGeneral |
        vk::DebugUtilsMessageTypeFlagBitsEXT::eValidation |
        vk::DebugUtilsMessageTypeFlagBitsEXT::ePerformance,
    .pfnUserCallback = debugCallback};

debugMessenger =
    vk::raii::DebugUtilsMessengerEXT(instance, messengerCreateInfo);
```

`eVerbose` is detailed diagnostic information. `eInfo` describes useful state or events. `eWarning` may indicate a mistake, inefficient usage, or behaviour that deserves attention. `eError` normally means that the application has violated a Vulkan rule.

The message type describes the source or category of the event:

- `eGeneral`: an event not belonging to another specific category;
- `eValidation`: a Vulkan specification or valid-usage problem;
- `ePerformance`: an operation that may work but is inefficient or suboptimal.

## Understanding the Callback

The callback receives four arguments:

```cpp
VKAPI_ATTR VkBool32 VKAPI_CALL debugCallback(
    VkDebugUtilsMessageSeverityFlagBitsEXT messageSeverity,
    VkDebugUtilsMessageTypeFlagsEXT messageType,
    const VkDebugUtilsMessengerCallbackDataEXT* callbackData,
    void* userData)
{
    std::cerr << "Validation layer: "
              << callbackData->pMessage << '\n';

    return VK_FALSE;
}
```

`messageSeverity` tells us whether the message is verbose, informational, a warning, or an error. `messageType` tells us whether it is general, validation-related, or performance-related. `callbackData` contains the actual message and additional information such as the message ID and related Vulkan objects. `userData` is an optional application-defined pointer supplied when the messenger is created.

The return value is unusual. The application should normally return `VK_FALSE`. That tells the layer not to abort the Vulkan command that triggered the message. Returning `VK_TRUE` is reserved mainly for testing validation layers; it can cause the validation layer to stop the command and return `VK_ERROR_VALIDATION_FAILED_EXT`.

The callback must also not call Vulkan commands. It should perform small actions such as writing to a log, and then return.

## Seeing the Validation Layer Work

After the layer, debug messenger, and callback are connected, the flow looks like this:

```text
Vulkan function call
        ↓
Validation layer checks the call
        ↓
Debug messenger selects matching messages
        ↓
debugCallback receives the message
        ↓
The call continues to the driver
```

Now the program can capture runtime diagnostics while it creates and uses Vulkan objects. A message such as “object was destroyed while still in use” gives us a direct explanation of what went wrong instead of leaving us with an unexplained crash.

The validation layer does not make invalid Vulkan code valid. It gives us a development-time safety net that exposes mistakes close to where they occur. That makes it much easier to learn Vulkan's explicit rules before turning to the next part of the renderer.

*This post is based on the [validation layers chapter](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/00_Setup/02_Validation_layers.html) of the Vulkan Tutorial. The code is my own implementation in [The Journey of Learning Vulkan](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/03_validation_layer).*
