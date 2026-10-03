---
title: "12. Shader Modules: From Slang to a Vulkan Pipeline"
date: 2026-10-03 15:00:00 +0900
categories: [Vulkan]
tags: [vulkan, shaders, slang, spir-v, graphics, gpu, c++]
description: A beginner-friendly tour from shader basics and triangle coordinates to compiling Slang into SPIR-V and loading it as a Vulkan shader module.
---

Today I read the [Shader modules](https://docs.vulkan.org/tutorial/latest/03_Drawing_a_triangle/02_Graphics_pipeline_basics/01_Shader_modules.html) chapter of the Vulkan Tutorial. This chapter is where the graphics pipeline stops being only a diagram and starts receiving programs that actually run on the GPU.

The example for this post is the local [`10_shader_modules`](https://github.com/will-hcau/The_Journey_of_Learning_Vulkan/tree/main/src/basic/10_shader_modules) project from my Vulkan learning repository. It contains one Slang source file, a CMake build step that compiles that source into SPIR-V, and a `GraphicsPipeline` class that loads the resulting binary and prepares Vulkan shader stages.

There is a lot of new vocabulary here—shader, vertex, fragment, clip coordinates, SPIR-V, entry point, shader module, and pipeline stage. I will build the explanation from the beginning.

## What is a shader?

A **shader** is a small program that runs on the GPU. Unlike an ordinary C++ program, which usually runs once from top to bottom on the CPU, a shader is called many times as part of the graphics pipeline.

For example, a vertex shader may run once for every vertex in a model. A fragment shader may run once for every small piece of a triangle that covers the screen. The GPU is designed to run many of those invocations in parallel.

The word “shader” does not describe only one particular program. It describes a family of programs, each attached to a stage of the graphics pipeline. The two stages needed for our first triangle are:

```text
vertex data → vertex shader → rasterization → fragment shader → framebuffer
```

The **vertex shader** answers: “Where should this vertex appear?”

The **fragment shader** answers: “What should this covered screen location look like?”

The shader is programmable, while many other parts of the pipeline are configured through Vulkan structures. This division gives us both control over the appearance of objects and a well-defined route for processing them.

## OpenGL and Vulkan take different routes to the GPU

If you have used OpenGL, you may have written GLSL shader source such as this:

```glsl
#version 450

void main() {
    gl_Position = vec4(0.0, 0.0, 0.0, 1.0);
}
```

That source is human-readable text. In the traditional OpenGL workflow, the application passes the text to the driver, and the driver compiles and links it at runtime:

```text
GLSL source text → OpenGL driver compiler → GPU instructions
```

Vulkan uses a more explicit model. The application normally gives Vulkan shader code in **SPIR-V**, a binary intermediate representation designed for GPU programs:

```text
Slang / GLSL / HLSL source → SPIR-V binary → Vulkan shader module → GPU code
```

SPIR-V is not the final machine code for one particular GPU. It is a portable bytecode format. The graphics driver can validate it and translate it into instructions suitable for the installed GPU when the pipeline is created.

### Why use SPIR-V?

The biggest advantage is that shader compilation is split into two clearer jobs. A front-end compiler translates a shading language into a standardized intermediate format. The GPU vendor’s driver then translates that format into native instructions.

That gives Vulkan several useful properties:

* The application can compile shaders ahead of time and ship the resulting binaries.
* Vulkan receives a defined binary format instead of relying on every driver to interpret source syntax in exactly the same way.
* Shader validation and compilation errors can happen during the build rather than unexpectedly during a user’s first run.
* The driver compiler has a simpler, more focused input than a complete high-level language implementation.

There are trade-offs too. SPIR-V binaries are not pleasant to read or edit by hand. A project needs a shader build step, and the resulting binary still needs to be compatible with the Vulkan and SPIR-V versions supported by the target device. When debugging, we usually inspect the original shader source or disassemble the SPIR-V rather than reading the binary directly.

OpenGL’s source-based approach is convenient for experimentation: edit a string or file, compile it, and inspect the driver’s log. Its disadvantage is that compilation happens under the driver’s control at runtime. Compilation time, supported language features, and even the exact interpretation of edge cases can vary between vendors and driver versions.

Neither approach makes shaders magically portable. A shader still has to use features supported by the target API and hardware. Vulkan simply makes the intermediate representation and much of the pipeline contract more explicit.

## Compiling at build time or at runtime

The usual Vulkan workflow is to compile shader source into SPIR-V before the application runs:

```text
shader.slang --(build step)--> slang.spv --(application)--> vk::ShaderModule
```

This is what our example does. The application loads `slang.spv`; it does not parse or compile the Slang source itself.

Runtime compilation is also possible. A program can include a compiler library, generate Slang or GLSL source, and compile it while running. This can be useful for an editor, a shader playground, or a tool that generates specialized shaders. It also makes iteration convenient.

For a normal application, runtime compilation has costs: startup can be slower, compiler libraries increase the application’s footprint, and compilation failures now happen on the user’s machine. A practical engine often compiles shaders during development and packages SPIR-V binaries for release, possibly with a cache for specialized variants.

## Following the Tutorial's Use of Slang

The example uses **Slang**, a modern shading language with a C-like syntax. Slang can compile to SPIR-V and also supports other shader targets. It provides familiar types such as `float2`, `float3`, and `float4`, shader-stage annotations such as `[shader("vertex")]`, and semantics such as `SV_Position`.

One convenient feature is that a single source file can contain multiple entry points. An entry point is the function where a particular shader stage begins. Our file contains:

```hlsl
[shader("vertex")]
VertexOutput vertMain(...) { ... }

[shader("fragment")]
float4 fragMain(...) : SV_Target { ... }
```

The Vulkan pipeline later selects those functions by name: `vertMain` for the vertex stage and `fragMain` for the fragment stage. A larger file could contain several alternative vertex, fragment, ray-generation, miss, or closest-hit entry points.

## Vertex shader: starting with three points

To understand the vertex shader, we first need to understand a **vertex**. A vertex is a point in the data that describes geometry. A point may contain a position and additional attributes such as a colour, normal, or texture coordinate.

Three vertices can form a triangle. Triangles are the basic building blocks used by the rasterization pipeline. A model of a character, a landscape, or a building is represented by many triangles—sometimes millions of them. The GPU processes those triangles because triangles are simple, planar, and can be efficiently converted into covered screen locations.

For the first triangle, we can avoid creating a vertex buffer temporarily. Instead, we store three positions directly in the shader:

```hlsl
static float2 positions[3] = float2[](
    float2(0.0, -0.5),
    float2(0.5, 0.5),
    float2(-0.5, 0.5)
);
```

The three points describe a triangle in a two-dimensional coordinate system. The vertex shader receives `SV_VertexID`, which identifies the vertex currently being processed. We use that ID as the index into `positions`.

### Homogeneous coordinates and clip space

Graphics APIs do not send a raw 2D point directly to the screen. A vertex shader writes a position in **clip coordinates**. In this example, we use a four-component vector:

```text
(x, y, z, w)
```

The fourth component is part of the **homogeneous-coordinate** representation used by the graphics pipeline. For this first triangle, we set `w` to `1.0`, so the position stays unchanged when it is converted to normalized device coordinates.

The result is called **normalized device coordinates**. The visible x and y values are expressed in the range `[-1, 1]`. With the usual Vulkan viewport configuration, the positive Y direction is downward in the framebuffer. This is the convention that feels flipped if you are coming from OpenGL: `y = -1` is near the top and `y = 1` is near the bottom. The x direction still runs from left (`-1`) to right (`1`).

<object type="image/svg+xml" data="{{ '/assets/img/vulkan-homogeneous-coordinates.svg' | relative_url }}" width="100%" height="auto" style="max-width: 900px;" aria-label="Triangle in normalized device coordinates with its three vertex positions labeled">
  <a href="{{ '/assets/img/vulkan-homogeneous-coordinates.svg' | relative_url }}">View the normalized device coordinate diagram</a>
</object>

Our positions already lie in the desired normalized range. In this Vulkan convention, the first position `(0.0, -0.5)` is the top vertex and the other two positions form the lower base. We therefore set `z` to `0.0` and `w` to `1.0`:

```text
(x, y) → (x, y, 0.0, 1.0)
```

Because `w` is `1.0` in this example, the values remain the same. In a real 3D renderer, the vertex shader would normally calculate the clip-space position from model, view, and projection transformations before the GPU maps it into the normalized range.

### The vertex shader in our example

Here is the complete vertex shader from `shader/shader.slang`:

```hlsl
static float2 positions[3] = float2[](
    float2(0.0, -0.5),
    float2(0.5, 0.5),
    float2(-0.5, 0.5)
);

static float3 colors[3] = float3[](
    float3(1.0, 0.0, 0.0),
    float3(0.0, 1.0, 0.0),
    float3(0.0, 0.0, 1.0)
);

struct VertexOutput
{
    float3 color;
    float4 sv_position : SV_Position;
};

[shader("vertex")]
VertexOutput vertMain(uint vid : SV_VertexID)
{
    VertexOutput output;
    output.sv_position = float4(positions[vid], 0.0, 1.0);
    output.color = colors[vid];
    return output;
}
```

Let us read it from the bottom up. `[shader("vertex")]` tells Slang that `vertMain` is a vertex-shader entry point. The function returns a `VertexOutput` structure and accepts `vid`, a `uint` containing the built-in vertex ID.

`float4(positions[vid], 0.0, 1.0)` combines the two-dimensional position with z and w to produce the four-dimensional clip-space position. The `SV_Position` semantic tells the compiler that `sv_position` is the special output consumed as the vertex position by the rest of the pipeline.

The other output is `color`. It is ordinary data that travels to the next programmable stage. The rasterizer will interpolate it between the triangle’s vertices.

## Fragment shader: colouring the triangle

Once the vertex shader has produced three positions, Vulkan’s fixed-function rasterizer determines which screen locations are covered by the triangle. Each covered location produces a **fragment**.

A fragment is a candidate contribution to a pixel. It is not necessarily the final pixel: depth testing, stencil testing, and blending may still affect what eventually reaches the framebuffer.

The **fragment shader** runs on those fragments. Its job is to calculate values such as the colour that should be written for each one.

### A pure red triangle

The simplest fragment shader ignores all inputs and returns the same colour every time:

```hlsl
[shader("fragment")]
float4 fragMain() : SV_Target
{
    return float4(1.0, 0.0, 0.0, 1.0);
}
```

The four values are red, green, blue, and alpha. Therefore this returns opaque red. `SV_Target` identifies the colour output written to the first colour attachment.

The triangle shape still comes from the vertex shader and rasterizer. The fragment shader only decides that every covered fragment should have the same colour.

### A different colour at each vertex

Our actual example is more interesting. It stores one RGB colour for each vertex:

```hlsl
static float3 colors[3] = float3[](
    float3(1.0, 0.0, 0.0),
    float3(0.0, 1.0, 0.0),
    float3(0.0, 0.0, 1.0)
);
```

The vertex shader writes the matching colour into `VertexOutput`:

```hlsl
output.color = colors[vid];
```

The fragment shader receives the completed output from the vertex stage:

```hlsl
[shader("fragment")]
float4 fragMain(VertexOutput inVert) : SV_Target
{
    return float4(inVert.color, 1.0);
}
```

The important part is that the rasterizer does not copy only one vertex’s colour across the entire triangle. It interpolates the three values according to each fragment’s position. Near the red corner, the fragment is mostly red. Near an edge between red and green, it becomes a mixture of red and green. In the middle, all three colours contribute.

That automatic interpolation is one reason a triangle is such a useful first example. With only three input values, we can see the pipeline pass data from the vertex shader through rasterization into the fragment shader.

## Compiling Slang into SPIR-V

The Vulkan SDK provides `slangc`, the Slang compiler. We can compile the source file manually from the example directory with a command like this:

```bash
slangc shader/shader.slang \
    -target spirv \
    -profile spirv_1_4 \
    -emit-spirv-directly \
    -fvk-use-entrypoint-name \
    -entry vertMain \
    -entry fragMain \
    -o slang.spv
```

The important parameters are:

* `shader/shader.slang` is the input source file.
* `-target spirv` selects SPIR-V as the output format.
* `-profile spirv_1_4` selects the SPIR-V profile used by this example.
* `-emit-spirv-directly` asks Slang to emit SPIR-V directly.
* `-fvk-use-entrypoint-name` preserves the source entry-point names in the Vulkan-facing output.
* `-entry vertMain` and `-entry fragMain` tell the compiler which functions to include as entry points.
* `-o slang.spv` chooses the output file.

The entry-point arguments matter because the source file can contain multiple shader functions. They tell the compiler that this SPIR-V module should expose `vertMain` and `fragMain`. Later, Vulkan uses those same names in `PipelineShaderStageCreateInfo`.

We do not want to type this command manually every time `shader.slang` changes, so the example makes shader compilation a CMake build step.

## Compiling automatically with CMake

The example’s `CMakeLists.txt` first locates `slangc`:

```cmake
find_program(
    SLANGC_EXECUTABLE
    NAMES slangc
    HINTS
        "$ENV{VULKAN_SDK}/bin"
        "${Vulkan_ROOT}/bin"
)

if(NOT SLANGC_EXECUTABLE)
    message(FATAL_ERROR "slangc was not found. Install the Vulkan SDK or set VULKAN_SDK.")
endif()
```

It then describes the generated file and attaches a custom command to it:

```cmake
set(SHADER_SOURCE "${CMAKE_CURRENT_SOURCE_DIR}/shader/shader.slang")
set(SHADER_BINARY "${EXAMPLE_OUTPUT_DIRECTORY}/slang.spv")

add_custom_command(
    OUTPUT "${SHADER_BINARY}"
    COMMAND "${CMAKE_COMMAND}" -E make_directory
            "${EXAMPLE_OUTPUT_DIRECTORY}"
    COMMAND "${SLANGC_EXECUTABLE}" "${SHADER_SOURCE}"
            -target spirv
            -profile spirv_1_4
            -emit-spirv-directly
            -fvk-use-entrypoint-name
            -entry vertMain
            -entry fragMain
            -o "${SHADER_BINARY}"
    DEPENDS "${SHADER_SOURCE}"
    COMMENT "Compiling Slang shader to SPIR-V"
    VERBATIM
)
```

`DEPENDS` tells CMake that the generated binary needs to be rebuilt when the Slang source changes. The shader binary is also listed as a source of the executable, so building the example causes the compilation step to run before the program is linked.

The output is placed beside the executable. That detail matters because the C++ code resolves the shader path relative to the executable directory rather than assuming the process was launched from the project root.

## Loading the shader binary

Once the build has produced `slang.spv`, the application needs to read its bytes. The new `GraphicsPipeline` class owns the device reference and shader module:

```cpp
class GraphicsPipeline
{
public:
    GraphicsPipeline(const Device&, const std::filesystem::path& executableDirectory);

private:
    static std::vector<char> readShader(const std::filesystem::path&);
    static vk::raii::ShaderModule createShaderModule(
        const Device&, const std::vector<char>&);

    const Device& _device;
    vk::raii::ShaderModule _shaderModule = nullptr;
};
```

The implementation reads the file as binary data:

```cpp
std::vector<char> GraphicsPipeline::readShader(const std::filesystem::path& path)
{
    std::ifstream file(path, std::ios::ate | std::ios::binary);

    if (!file)
        throw std::runtime_error("failed to open shader binary: " + path.string());

    const auto size = file.tellg();
    std::vector<char> code(static_cast<size_t>(size));

    file.seekg(0, std::ios::beg);
    file.read(code.data(), size);

    if (!file)
        throw std::runtime_error("failed to read shader binary: " + path.string());

    return code;
}
```

Opening with `std::ios::binary` is important: SPIR-V is not text. The `ate` flag starts the file position at the end, which lets us ask for its size before allocating the vector. We then seek back to the beginning and read all bytes.

## Creating a Vulkan shader module

Vulkan wraps the SPIR-V bytes in a `VkShaderModule`. In this project, the C++ Vulkan-Hpp RAII version is:

```cpp
vk::raii::ShaderModule GraphicsPipeline::createShaderModule(
    const Device& device,
    const std::vector<char>& code)
{
    const vk::ShaderModuleCreateInfo createInfo{
        .codeSize = code.size(),
        .pCode = reinterpret_cast<const uint32_t*>(code.data())};

    return vk::raii::ShaderModule(device.logicalDevice(), createInfo);
}
```

`codeSize` is the number of bytes in the SPIR-V binary. Vulkan’s `pCode` field expects a pointer to 32-bit words, so the example casts the byte pointer to `const uint32_t*`. SPIR-V is made of 32-bit words, and the compiler output satisfies the alignment and size requirements expected here.

The constructor of `GraphicsPipeline` connects the pieces:

```cpp
GraphicsPipeline::GraphicsPipeline(
    const Device& device,
    const std::filesystem::path& executableDirectory)
    : _device(device)
{
    const auto shaderCode = readShader(executableDirectory / "slang.spv");

    _shaderModule = createShaderModule(_device, shaderCode);

    const vk::PipelineShaderStageCreateInfo vertexStage{
        .stage = vk::ShaderStageFlagBits::eVertex,
        .module = *_shaderModule,
        .pName = "vertMain"};

    const vk::PipelineShaderStageCreateInfo fragmentStage{
        .stage = vk::ShaderStageFlagBits::eFragment,
        .module = *_shaderModule,
        .pName = "fragMain"};

    (void)vertexStage;
    (void)fragmentStage;
}
```

A shader module is only the container for the compiled code. It does not yet say how the code should be used. `PipelineShaderStageCreateInfo` supplies that missing context:

* `.stage` says whether this is a vertex or fragment stage.
* `.module` identifies the SPIR-V shader module.
* `.pName` selects the entry-point function inside that module.

The strings in `.pName` must match the entry-point names written in the Slang source and included during compilation. `"vertMain"` selects:

```hlsl
[shader("vertex")]
VertexOutput vertMain(...) { ... }
```

and `"fragMain"` selects:

```hlsl
[shader("fragment")]
float4 fragMain(...) : SV_Target { ... }
```

The names are case-sensitive. A mismatch means Vulkan cannot find the requested entry point when it validates or creates the graphics pipeline.

In this particular learning example, the two stage structures are prepared but not yet passed into a complete graphics-pipeline creation call. That is deliberate: this chapter focuses on getting shader code from source into Vulkan. The following pipeline chapters will use these stage descriptions alongside the fixed-function configuration.

## The complete journey

The shader path now looks like this:

```text
shader/shader.slang
        │
        │  slangc, invoked by CMake
        ▼
slang.spv
        │
        │  readShader()
        ▼
std::vector<char>
        │
        │  createShaderModule()
        ▼
vk::raii::ShaderModule
        │
        │  PipelineShaderStageCreateInfo
        │  pName = "vertMain" / "fragMain"
        ▼
Vulkan graphics pipeline
```

The central idea is simple: a shader source file is not itself a Vulkan shader module. It must be compiled into SPIR-V, loaded as binary data, wrapped in a shader-module object, and then assigned to a particular pipeline stage and entry point.

The vertex shader places three vertices into clip space and passes colours forward. Rasterization turns the triangle into fragments and interpolates those colours. The fragment shader turns each interpolated colour into a framebuffer output. That is the first complete connection between a few lines of shader code and pixels on the screen.

There is still plenty left to configure—vertex input, viewport, rasterization, multisampling, colour blending, render passes, and pipeline creation—but the programmable part of the route is now much less mysterious.

## What I want to remember

* A shader is a GPU program executed many times as part of the graphics pipeline.
* Vulkan normally consumes SPIR-V bytecode, while traditional OpenGL accepts GLSL source and compiles it through the driver.
* Slang is the human-readable source language in this example; `slangc` turns it into `slang.spv`.
* A vertex is a point with attributes. Three vertices form the triangle primitive used in this first example.
* The vertex shader writes clip-space coordinates, including the homogeneous `w` component.
* Rasterization converts a triangle into fragments and interpolates values between vertices.
* The fragment shader decides the colour written for each fragment.
* `pName` selects the exact entry-point function inside the SPIR-V module.

The next step is to connect these shader stages to the rest of the graphics-pipeline configuration and finally submit a draw command that uses them.
