---
title: "[Lesson 3] Creating a Window and Vulkan Instance"
date: 2026-09-20 12:00:00 +0900
categories: [Graphics]
tags: [vulkan, graphics, glfw, c++]
description: Creating a GLFW window, then connecting the application to Vulkan with a RAII instance.
---

I have started the [drawing-a-triangle chapter of the Vulkan Tutorial](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/00_Setup/00_Base_code.html). This time, I implemented the first two setup steps in my own repository:

- [Creating a window](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/01_creating_window)
- [Creating a Vulkan instance](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/02_creating_instance)

These steps may look small, but they establish two important boundaries: the window belongs to the operating system, while the Vulkan instance is the connection between our application and the Vulkan library.

## Vulkan Does Not Create Windows

Vulkan is a graphics and compute API. It knows how to communicate with a GPU, but it does not provide window-management features such as creating a desktop window, processing mouse and keyboard events, or handling the operating system's event loop.

We could use Vulkan for off-screen rendering, where the result is written to an image without ever being shown in a window. However, if we want to see our triangle, we first need a canvas to present it on.

For this project, that canvas comes from [GLFW](https://www.glfw.org/).

## What Is GLFW?

GLFW is a small, cross-platform library for creating windows and receiving input and window events. It hides the platform-specific details behind one consistent C API, so our application can use the same basic code on macOS, Windows, and Linux.

GLFW was originally designed with OpenGL in mind, but it also supports Vulkan. In a Vulkan application, GLFW creates the window and later helps us create the platform-specific Vulkan surface that connects Vulkan presentation to that window.

For now, the goal is simpler: create a window that stays open until the user closes it. We explicitly tell GLFW not to create an OpenGL context because Vulkan will be responsible for the graphics API.

## The Smallest Useful GLFW Setup

The setup is intentionally short. `glfwInit` initializes the library, the window hints configure the kind of window we want, and `glfwCreateWindow` creates it. The loop keeps the application alive and processes events.

```cpp
namespace {
constexpr uint32_t kWidth = 800;
constexpr uint32_t kHeight = 600;
}

void initWindow()
{
    if (glfwInit() == GLFW_FALSE)
    {
        throw std::runtime_error("failed to initialize GLFW");
    }

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);

    window = glfwCreateWindow(kWidth, kHeight, "Vulkan Triangle", nullptr, nullptr);
    if (window == nullptr)
    {
        glfwTerminate();
        throw std::runtime_error("failed to create GLFW window");
    }
}

void mainLoop()
{
    while (!glfwWindowShouldClose(window))
    {
        glfwPollEvents();
    }
}

void cleanup()
{
    if (window != nullptr)
    {
        glfwDestroyWindow(window);
    }
    glfwTerminate();
}
```

The application does not draw anything yet. It only creates the window and waits for events. That is enough to confirm that the first layer of the program is working.

## The `SimpleTriangle` Structure

My sample puts the application inside a `SimpleTriangle` class. This gives us one place to store the window and, later, the Vulkan objects that depend on it. Each setup step gets its own function, and `run` controls the order in which they happen.

```cpp
class SimpleTriangle
{
public:
    void run()
    {
        initWindow();
        mainLoop();
        cleanup();
    }

private:
    // Definitions for initWindow, mainLoop, and cleanup are in the class.
    GLFWwindow* window = nullptr;
};
```

The `initVulkan` function is empty in the window-only example. In the next step, it calls `createInstance`.

If everything is configured correctly, running the window example should produce a blank window titled **Vulkan Triangle**. A blank window is expected: there is no swap chain, render pass, or draw command yet.

<div style="text-align: center;">
  <img src="{{ '/assets/img/vulkan-window-created.png' | relative_url }}" width="400" alt="Blank Vulkan window created by GLFW">
</div>

This is the first visible milestone. The operating system has given us a window, GLFW is processing its events, and the application can now provide Vulkan with the information needed to present images there.

## Connecting the Application to Vulkan

Now that we have a window, we can begin creating Vulkan objects. The first one is the **Vulkan instance**.

The instance is the connection between our application and the Vulkan library. Creating it tells the Vulkan implementation some basic information about our application and which global features we want to use. It happens before we select a physical GPU or create a logical device.

In my example, the class stores two objects:

```cpp
vk::raii::Context context;
vk::raii::Instance instance = nullptr;
```

The context loads the Vulkan functions available to the application. The instance represents the connection that we create from that context.

## A Note About RAII

RAII stands for **Resource Acquisition Is Initialization**. It is a common C++ pattern in which an object's constructor acquires a resource and its destructor releases that resource automatically.

This is the same general idea behind smart pointers. Instead of remembering to call `delete` manually, we let an object clean up what it owns when it goes out of scope.

The traditional Vulkan C API requires manual cleanup:

```cpp
VkInstance instance;

if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
    throw std::runtime_error("failed to create instance");
}

// Use the instance...

vkDestroyInstance(instance, nullptr);
```

The `vk::raii` API wraps that lifetime management for us:

```cpp
instance = vk::raii::Instance(context, createInfo);
```

When `instance` is destroyed, its destructor calls the appropriate Vulkan cleanup function. We still need to understand the lifetime of our resources and destroy them in the right order, but we no longer need to write every `vkDestroy...` call ourselves. RAII saves time and makes early returns and exceptions much safer.

## Preparing `vk::InstanceCreateInfo`

Vulkan passes much of its configuration through structures. Before creating the instance, we prepare an `appInfo` structure and then use it from `createInfo`.

There are four practical pieces of information to think about here:

1. `pApplicationInfo` describes the application and the Vulkan API version it expects.
2. `enabledExtensionCount` says how many global extensions we are requesting.
3. `ppEnabledExtensionNames` points to the names of those extensions.
4. The layer fields describe optional global validation layers. I am leaving them empty for now and will cover them in the next lesson.

The application information is supplied by us. It is useful metadata for the driver, although it does not describe the rendering pipeline itself.

```cpp
constexpr vk::ApplicationInfo appInfo{
    .pApplicationName = "Vulkan Triangle",
    .applicationVersion = VK_MAKE_VERSION(1, 0, 0),
    .pEngineName = "Example Engine",
    .engineVersion = VK_MAKE_VERSION(1, 0, 0),
    .apiVersion = vk::ApiVersion14};
```

The window-system extensions are not something we should hard-code ourselves. GLFW knows which extensions are needed on the current platform, so we ask GLFW for them:

```cpp
uint32_t glfwExtensionCount = 0;
const char* const* glfwExtensions =
    glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

if (glfwExtensions == nullptr)
{
    throw std::runtime_error("failed to find GLFW Vulkan extensions");
}

std::vector<const char*> extensions(
    glfwExtensions, glfwExtensions + glfwExtensionCount);

const vk::InstanceCreateInfo createInfo{
    .pApplicationInfo = &appInfo,
    .enabledExtensionCount = static_cast<uint32_t>(extensions.size()),
    .ppEnabledExtensionNames = extensions.data()};

instance = vk::raii::Instance(context, createInfo);
```

This is the handoff between GLFW and Vulkan. GLFW tells us what the current window system requires, and we pass those extension names into Vulkan when creating the instance.

## What Is a Vulkan Extension?

A Vulkan extension is an optional feature that adds functionality beyond Vulkan's core API.

Vulkan is designed to work across different operating systems, GPUs, and hardware vendors. Instead of putting every platform-specific feature into the core API, Vulkan makes additional functionality available through named extensions.

The Vulkan implementation may support many extensions, but they are not automatically enabled for our application. We must explicitly request the extensions that we want to use when creating the relevant Vulkan object.

```text
Supported extensions
        ↓
Application requests selected extensions
        ↓
Vulkan enables them for the instance or device
```

There are two important categories:

- **Instance extensions** affect the whole application. The window-system extensions returned by GLFW belong to this category.
- **Device extensions** affect a specific physical device or logical device. The swap-chain extension used for presentation is an example.

The extensions returned by `glfwGetRequiredInstanceExtensions` are therefore GLFW's way of telling us which instance extensions are required to connect Vulkan to the window we created. GLFW provides the names, but Vulkan enables them when the instance is created.

Extensions are identified by text names, for example:

```text
VK_KHR_surface
VK_EXT_debug_utils
VK_KHR_swapchain
```

Core Vulkan functionality is different: it is available according to the Vulkan version and the capabilities of the selected device. It does not need to be enabled through the extension mechanism.

## Checking the Extensions

Extensions are identified by text names. That makes Vulkan flexible, but it also means that we should check whether the names we need are actually supported before asking Vulkan to enable them.

The Vulkan context can enumerate the extensions exposed by the current implementation:

```cpp
const auto extensionProperties =
    context.enumerateInstanceExtensionProperties();

for (const auto& extension : extensionProperties)
{
    LOG_INFO("   " << extension.extensionName);
}

for (const char* extensionName : extensions)
{
    if (std::ranges::none_of(extensionProperties,
            [extensionName](const auto& extensionProperty)
            {
                return std::strcmp(extensionProperty.extensionName, extensionName) == 0;
            }))
    {
        throw std::runtime_error(
            "Required Vulkan extension not supported: " + std::string(extensionName));
    }
}
```

We can compare this list with the names returned by `glfwGetRequiredInstanceExtensions`. If a required extension is missing, instance creation should fail with a useful error instead of failing later when we try to create a surface.

This is also a useful debugging technique: when Vulkan reports that an extension is not present, the extension name gives us a direct clue about which part of the platform connection is unavailable.

## The First Vulkan Connection

At the end of this lesson, the program has two foundations:

- GLFW has created a window and is processing window events.
- Vulkan has created an instance using the platform extensions required by GLFW.

We still have not selected a GPU or rendered a triangle. That is fine. Vulkan is deliberately explicit, so each small step gives us another piece of the rendering system to understand and verify.

The next step is to inspect the physical devices available through the instance and choose one that can do the work we need.

*This post is based on the [base-code](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/00_Setup/00_Base_code.html) and [instance](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/00_Setup/01_Instance.html) chapters of the Vulkan Tutorial. The code is my own implementation in [The Journey of Learning Vulkan](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan).*
