---
title: "2. Building My Own Vulkan Learning Environment"
date: 2026-09-20 12:00:00 +0900
categories: [Vulkan]
tags: [vulkan, graphics]
description: Setting up a small CMake-based Vulkan project that builds its tutorial dependencies from source.
---

Today I continued with the [Vulkan Tutorial](https://docs.vulkan.org/tutorial/latest/02_Development_environment.html) and reached the development-environment chapter.

The official tutorial provides a repository and explains how to install the libraries needed by the later chapters. I decided to take a slightly different approach: I will follow the tutorial by writing the code myself, rather than reusing the tutorial's Git repository.

To support that approach, I created my own learning repository:

[The Journey of Learning Vulkan](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan)

The project is intentionally small at this stage. It is not trying to be a production-ready rendering framework. Its purpose is to give me a clean place to implement each Vulkan concept, one step at a time, and to make the build process as repeatable as possible.

## My Goal: A Minimal and Repeatable Setup

Coming from an embedded-software background, I naturally prefer a development environment with a small number of assumptions. If a project depends on several libraries, I would rather describe those dependencies in the build system than ask every reader to find, install, and configure them separately.

That is why my project uses CMake's `FetchContent` module. During configuration, CMake fetches GLFW, GLM, and tinyobjloader from their upstream repositories. It then builds them as part of the project. They do not need to be installed globally, and they do not need to be committed into my repository.

The project currently has this structure:

```text
src/
└── basic/
    └── 00_hello_world/
```

As the tutorial progresses, each new concept can become a separate, numbered example. This makes the history of the learning process visible instead of hiding everything inside one large application.

## The Four Pieces We Need

The development environment consists of four important dependencies. They do different jobs, so understanding their roles is useful before writing any Vulkan code.

### GLFW: Creating a Window

Vulkan is a graphics and compute API, not a windowing toolkit. It does not create an application window or provide a portable way to connect that window to the operating system.

That is where [GLFW](https://www.glfw.org/) comes in. It creates the window and handles the platform-specific details needed to interact with it. It also helps us obtain the Vulkan surface extensions required to present rendered images in that window.

Using GLFW means that the tutorial code can keep the windowing layer relatively small while still working across desktop platforms such as Windows, Linux, and macOS. The important limitation is that GLFW is primarily a desktop solution; mobile applications normally use platform-specific windowing systems or another library such as SDL.

### GLM: Mathematics for Graphics

Vulkan deliberately stays close to the graphics API itself. It does not include a general-purpose mathematics library for vectors, matrices, transformations, or projections.

[GLM](https://github.com/g-truc/glm) fills that gap. It provides familiar graphics-math types and functions, including vectors and matrices used for model, view, and projection transformations. GLM is header-only and designed with graphics programming in mind, which makes it a convenient fit for a small C++ Vulkan project.

Without a library such as GLM, we would need to write and maintain these mathematical operations ourselves before we could even begin to transform a model on screen.

### tinyobjloader: Reading 3D Models

Eventually, the tutorial needs to load geometry from a model file. A model format contains more than just a list of positions: it can include vertex normals, texture coordinates, material information, and relationships between faces.

[tinyobjloader](https://github.com/tinyobjloader/tinyobjloader) provides a small loader for Wavefront OBJ files. It lets us focus on the Vulkan side of transferring vertex data and drawing a model instead of spending the early chapters implementing an OBJ parser.

It is a deliberately simple choice. More feature-rich formats such as glTF are often a better foundation for a larger renderer, but tinyobjloader is appropriate for a focused learning project where the model-loading code should remain easy to understand.

### Vulkan SDK: The Vulkan Development Toolkit

The [Vulkan SDK](https://vulkan.lunarg.com/) is different from the other three dependencies. It is not just a small library that the project can download and build as part of its normal CMake configuration.

The SDK supplies the Vulkan headers, loader, validation layers, debugging tools, and shader-related utilities needed to develop and test a Vulkan application. It also provides the files that allow CMake to locate Vulkan during configuration.

For this reason, the Vulkan SDK remains the one dependency that I expect the developer to install separately. The [development-environment chapter of the Vulkan Tutorial](https://docs.vulkan.org/tutorial/latest/02_Development_environment.html) already provides detailed installation instructions for the supported platforms, so I will follow those instructions rather than duplicate them here. Running the SDK's `vkcube` sample is also a useful first check that the driver and Vulkan installation are working.

## How My CMake Project Handles the Dependencies

The root `CMakeLists.txt` does three main things:

1. It defines the project and its C++20 requirements.
2. It uses `FetchContent` to download and build GLFW, GLM, and tinyobjloader.
3. It accepts a Vulkan SDK path through `VULKAN_SDK_PATH`, or uses the `VULKAN_SDK` environment variable when it is already available.

The first configuration downloads the third-party source code into CMake's dependency area. The dependencies are therefore kept out of the Git repository while still being available to the build. Later configurations can reuse the downloaded content.

A new checkout can be configured with:

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --parallel
```

If the Vulkan SDK environment variable is not set, the SDK location can be supplied explicitly:

```bash
cmake -S . -B build \
  -DVULKAN_SDK_PATH=/path/to/vulkan-sdk \
  -DCMAKE_BUILD_TYPE=Debug
```

The path is stored in the build directory's CMake cache, so it does not need to be typed again for every build.

## Why Make a Separate Learning Repository?

The main advantage of [The Journey of Learning Vulkan](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan) is that it separates learning from copying.

The official tutorial is an excellent guide, but copying its completed code would make it easy to miss the decisions and problems that the code is solving. By creating each example myself, I can stop at every stage and ask questions such as:

- What does this Vulkan object represent?
- Why does it need to be created before the next object?
- Which information belongs in CMake, and which belongs in the application?
- What error or validation message appears when I make a deliberate mistake?

The repository also has a few practical advantages:

- **Repeatable setup:** the CMake project describes how the non-SDK dependencies are obtained and built.
- **No global library installation:** GLFW, GLM, and tinyobjloader are managed as part of the project.
- **Small, ordered examples:** each tutorial step can remain understandable and easy to compare with the previous one.
- **Visible progress:** the Git history records how the renderer grows from a minimal program into a Vulkan application.
- **Room for experiments:** I can try an alternative implementation without modifying the tutorial's source code.

There is still one honest tradeoff: fetching dependencies requires network access during the initial configuration, and building them can make the first build slower. I consider that a reasonable cost for a project whose goal is to make its setup explicit and reproducible.

## Ready for the First Vulkan Objects

After this setup, the project has a compiler, CMake, the Vulkan SDK, a portable window layer, graphics mathematics, and a small model loader. More importantly, I have a build system that explains where each dependency comes from and why it is present.

The next step is no longer environment preparation. It is time to create the first Vulkan instance and begin turning the concepts from the overview chapter into code.

*This post is based on the development-environment chapter of the [Vulkan Tutorial](https://docs.vulkan.org/tutorial/latest/02_Development_environment.html). The project discussed here is my own implementation: [The Journey of Learning Vulkan](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan).*
