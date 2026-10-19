# Architecture, bindings, and project setup

Return to [AGENTS.md](../../AGENTS.md). For game APIs, continue with [foundations](foundation.md), [graphics/assets](graphics-assets.md), or [physics/audio/jobs](physics-audio-jobs.md).

## Start with the repository's actual state

[README: Getting started / Building from source](../../README.md#getting-started) specifies the .NET 10 SDK, Windows/Linux, and the `Ahwoo.Brutal.*` NuGet packages. Vulkan rendering also requires a suitable GPU/driver; the Vulkan SDK is mentioned for validation layers. This repository contains source without `.csproj`, `.sln`, `.slnx`, or build/test configuration. Do not promise that `dotnet build`/`dotnet test` work at the root or guess dependency versions from `using` directives.

For a separate game experiment, the following commands come from the README:

```powershell
dotnet new console -n HelloBrutal
cd HelloBrutal
dotnet add package Ahwoo.Brutal.Core
dotnet add package Ahwoo.Brutal.Glfw
```

A complete window example is in the [graphics guide](graphics-assets.md). Choose additional packages for the APIs actually used. Pointer code requires an `unsafe` context and `AllowUnsafeBlocks` in the consuming project. Examples in these guides were checked against source, rather than compiled in the unavailable project; examples marked as fragments require the objects/context described around them.

Native binaries are still required. For example, the [Vulkan resolver](../../Brutal.Vulkan/ImportResolver.cs) loads the library for the current OS and obtains `vkGetInstanceProcAddr`. Copying `.cs` files does not automatically supply DLLs, generators, or package dependencies. `Brutal.FreeType/` and `Brutal.ZLib/` contain only `LICENSES/` in this snapshot; do not invent C# API examples from package names.

## Generated and Manual form the same API

| Location | Role visible in the source |
| --- | --- |
| `Generated/`, `*.gen.cs` | Generator output: enums, struct layouts, delegates, handles, P/Invoke bindings, and public wrappers. Generated does not necessarily mean a raw API only. |
| `Manual/` and helpers at the module root | Handwritten implementations, special overloads, lifecycle management, and partial types that complement generated types. |
| `Generated-Internal/` in ImGui | ImGui's internal surface; prefer public APIs when available because internal details couple the game to the library implementation. |

[Generated Glfw.Context](../../Brutal.Glfw/Generated/Libraries/Glfw.Context.gen.cs) is marked `do not edit by hand`. The [manual partial](../../Brutal.Glfw/Manual/Libraries/Glfw.Context.cs) explains that `GetCurrentContext` and `GetProcAddress` require special handling, while other members are combined into the same `Brutal.GlfwApi.Glfw` type. [Manual Glfw.Vulkan](../../Brutal.Glfw/Manual/Libraries/Glfw.Vulkan.cs) adds `CreateWindowSurface`, pinning the output and passing a null allocator.

A small, directly traceable example: [generated VkExtent2D](../../Brutal.Vulkan/Generated/Structs/VkExtent2D.gen.cs) defines `Width`/`Height`, while its [manual partial](../../Brutal.Vulkan/Manual/Partials/VkExtent2D.cs) adds a two-argument constructor.

```csharp
using Brutal.VulkanApi;

// Correct: one public type combining generated and manual code.
var extent = new VkExtent2D(1280, 720);

// Incorrect: the namespace does not follow source folder names.
// var extent = new Brutal.Vulkan.Generated.VkExtent2D(1280, 720);
```

The headers name generators, but generator projects are absent from this snapshot. For binding changes, locate the generator source before regenerating; for game helpers, use public APIs or appropriate handwritten partial types/helpers rather than patching generator output.

## Abstractions and Extensions

Start with helpers that fit the game's needs, then use public bindings when more control is required. Choose the layer per feature; there is no universal initialization order across all modules.

| Layer | Concrete examples and reasons to use it |
| --- | --- |
| `.Abstractions` | [DeviceEx](../../Brutal.Vulkan.Abstractions/Classes/DeviceEx.cs), [StagingPool](../../Brutal.Vulkan.Abstractions/StagingPool.cs), and [FrameResources](../../Brutal.Vulkan.Abstractions/FrameResources.cs) help manage GPU resources; [ConsoleWindow](../../Brutal.ImGui.Abstractions/ConsoleWindow/ConsoleWindow.cs) provides a console UI. These are components to assemble, not a complete renderer/game loop. |
| `.Extensions` | [ImGuiEx](../../Brutal.ImGui.Extensions/ImGui.extensions.cs) adds enum and color operations; `CheckboxFlags<T>` checks that the enum has the same size as `int`. Extensions also appears as a helper folder in other modules. |
| Public bindings | `Glfw`, `ImGui`, and Vulkan/Jolt handle types expose C# signatures that handle some marshaling. Read the signatures you use, including ref/out/span parameters and ownership. |
| `PInvoke` | Native contracts include pointers, sizes, and callbacks. Use them when public wrappers are insufficient and the game correctly handles lifetime and ABI requirements. |

Avoid old overloads chosen solely because they resemble tutorials for other C# libraries. [ImGui compatibility](../../Brutal.ImGui.Extensions/Compatibility/ImGui.Compatibility.cs) marks `InputText(ref string)` obsolete and allocates a temporary `ImInputString`. Choose a persistent `ImInputString` for UI input; see [foundations](foundation.md).

## Namespaces, ABI, and lifecycle

Folder names are not namespace contracts: GLFW uses `Brutal.GlfwApi`, Vulkan `Brutal.VulkanApi`, ImGui `Brutal.ImGuiApi`, Jolt `Brutal.JoltApi`, and interop `Brutal.Interop`. Check each module's `namespace` declarations and `GlobalUsings.cs`. A library's global using directives do not automatically become imports in the game project.

[InteropStringMarshaller](../../Brutal.Interop/InteropStringMarshaller.cs) converts `string` to null-terminated UTF-8, using a small buffer or native allocation as needed. [IInteropHandle](../../Brutal.Interop/IInteropHandle.cs) and the [handle marshaller](../../Brutal.Interop/InteropHandleMarshaller.cs) map struct wrappers to unmanaged handles. Type conversion does not establish that a wrapper owns the resource or that retaining it is safe.

Some Vulkan structs have important initializers. [VkInstanceCreateInfo](../../Brutal.Vulkan/Generated/Structs/VkInstanceCreateInfo.gen.cs) sets `sType` through its constructor; `default` only zeroes memory.

```csharp
using Brutal.VulkanApi;

unsafe
{
    // Correct: runs the sType initializer; fill other fields as needed.
    var info = new VkInstanceCreateInfo();

    // Incorrect as a ready-to-use create-info: sType is not initialized.
    // VkInstanceCreateInfo info = default;
}
```

This is an initialization fragment, not complete instance creation. For each native resource type, verify creation/initialization, ownership, use, synchronization, and destruction/disposal. `IDisposable`, `Managed...` wrappers, borrowed handle structs, and COM proxies behave differently; the runtime and graphics guides provide examples.

## Updating the guides

When APIs change, read the current implementation and update examples and links in the relevant topic guide. Keep the complete module list in `AGENTS.md` aligned with the repository folders. Check link targets, namespaces, overloads, and lifecycles; run builds/tests when the necessary projects and dependencies are available, and report what was verified.
