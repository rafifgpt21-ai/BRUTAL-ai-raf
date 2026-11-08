# BRUTAL Agent Guide

BRUTAL is a C# framework for building your own game or engine, with native bindings and utilities for memory, mathematics, text, and jobs. The game defines its own loop, rendering, and gameplay systems. Start with the [README](README.md); this snapshot does not yet include project/solution files for building from source.

## Core rules

- Read implementations and overloads before using an API; package folder names often differ from C# namespaces.
- Choose suitable public helpers or `.Abstractions`, then public bindings; use `PInvoke` when you need to manage ABI details yourself.
- Extend handwritten code through suitable partial types/helpers. Do not edit `.gen.cs` manually; lasting changes belong in the generator, which is not included in this snapshot.
- Track who owns each handle/buffer, when it is borrowed, and who releases it. Copying a handle struct does not duplicate the native resource.
- Use the ABI types and UTF-8 text required by the signature. Keep `bool`, `string`, and .NET collections when they fit the API and game requirements.
- Avoid unnecessary per-frame allocations; inspect the implementation before assuming collections or text conversions are allocation-free.
- Respect library threading and lifecycle rules: windows/UI, callbacks, GPUs, and jobs have different requirements; read the relevant guide.
- Follow [CONTRIBUTING.md](CONTRIBUTING.md), including commit sign-off, when contributing. Refer to [LICENSE.txt](LICENSE.txt) and [NOTICE.txt](NOTICE.txt) when distributing.

## Choose a guide for your task

| When working on | Read before writing code |
| --- | --- |
| Bindings, partial types, namespaces, project setup | [Architecture and snapshot limitations](docs/agents/architecture.md) |
| Basic types, pointers, memory, collections, mathematics, text, logging, localization | [Foundations](docs/agents/foundation.md) |
| Windows/input, Vulkan/OpenGL, textures and glTF models | [Graphics and assets](docs/agents/graphics-assets.md) |
| Physics, audio, ImGui consoles/commands, parallel jobs | [Physics, audio, debug UI, and jobs](docs/agents/physics-audio-jobs.md) |

## All modules in this snapshot

This table covers all 48 `Brutal.*` folders. Links point to source; source availability does not guarantee that all build tools or native binaries are included.

| Module | Purpose / limitations to know |
| --- | --- |
| [Brutal.Core](Brutal.Core/), [Brutal.Interop](Brutal.Interop/) | ABI types, pointers, attributes/platform; native string and handle marshallers |
| [Brutal.Memory](Brutal.Memory/), [Brutal.Collections](Brutal.Collections/) | Native memory/allocators; native and strided collections |
| [Brutal.Numerics](Brutal.Numerics/), [Brutal.Maths](Brutal.Maths/) | Vectors/matrices/colors; `Random` extensions |
| [Brutal.Strings](Brutal.Strings/), [Brutal.Logging](Brutal.Logging/), [Brutal.Localization](Brutal.Localization/) | UTF-8/builders, category logging, translated text |
| [Brutal.Glfw](Brutal.Glfw/), [Brutal.Sdl](Brutal.Sdl/) | Windows, input, events, graphics API integration |
| [Brutal.Sdl.Image](Brutal.Sdl.Image/), [Brutal.Sdl.Mixer](Brutal.Sdl.Mixer/), [Brutal.Sdl.Ttf](Brutal.Sdl.Ttf/) | SDL add-ons for images, audio mixing, and fonts |
| [Brutal.Vulkan](Brutal.Vulkan/), [Brutal.Vulkan.Abstractions](Brutal.Vulkan.Abstractions/), [Brutal.Vulkan.Vma](Brutal.Vulkan.Vma/) | Vulkan bindings/handles, resource/rendering helpers, VMA allocator |
| [Brutal.OpenGl](Brutal.OpenGl/), [Brutal.ShaderC](Brutal.ShaderC/), [Brutal.Slang](Brutal.Slang/) | OpenGL bindings; shader compilation |
| [Brutal.Texture](Brutal.Texture/), [Brutal.Texture.Abstractions](Brutal.Texture.Abstractions/) | Texture data and format readers backed by decoders |
| [Brutal.Gli](Brutal.Gli/), [Brutal.Ktx](Brutal.Ktx/), [Brutal.Stb](Brutal.Stb/) | GLI/KTX textures and STB decoding |
| [Brutal.Gltf](Brutal.Gltf/), [Brutal.MeshOptimizer](Brutal.MeshOptimizer/), [Brutal.Render.Mesh](Brutal.Render.Mesh/) | glTF data/access, mesh optimization, storage and procedural meshes |
| [Brutal.ImGui](Brutal.ImGui/), [Brutal.ImGui.Extensions](Brutal.ImGui.Extensions/), [Brutal.ImGui.Abstractions](Brutal.ImGui.Abstractions/) | Immediate-mode UI, overloads/helpers, consoles and terminals |
| [Brutal.ImGui.Backends](Brutal.ImGui.Backends/), [Brutal.ImGui.FreeType](Brutal.ImGui.FreeType/), [Brutal.ImPlot](Brutal.ImPlot/) | Platform/rendering backends, FreeType font integration, plotting |
| [Brutal.Jolt](Brutal.Jolt/), [Brutal.Box2D](Brutal.Box2D/) | 3D and 2D physics |
| [Brutal.Fmod](Brutal.Fmod/), [Brutal.OpenAl](Brutal.OpenAl/) | Native audio |
| [Brutal.Concurrency](Brutal.Concurrency/), [Brutal.RakNet](Brutal.RakNet/) | Job/threading; networking |
| [Brutal.Monitor.Common](Brutal.Monitor.Common/), [Brutal.Monitor.Host](Brutal.Monitor.Host/) | Monitor options/protocol; host integration, logs and crash dumps |
| [Brutal.RenderDoc](Brutal.RenderDoc/), [Brutal.RenderDoc.ImGui](Brutal.RenderDoc.ImGui/) | RenderDoc capture and an ImGui control window |
| [Brutal.NativeFileDialog](Brutal.NativeFileDialog/), [Brutal.NotCurses](Brutal.NotCurses/) | Native file dialogs and terminal UI |
| [Brutal.FreeType](Brutal.FreeType/), [Brutal.ZLib](Brutal.ZLib/) | Third-party license folders only in this snapshot; no C# API yet |

## Agent skills


### Issue tracker

Specs and tickets are stored in GitHub Issues for rafsidqi/BRUTAL-raf.
Before reading, creating, or resolving tickets, read the
[issue tracker rules](docs/agents/issue-tracker.md).

### Domain docs

Single-context layout: GLOSSARY.md and docs/adr/ at the root.
Before exploring or changing the domain, read the
[domain rules](docs/agents/domain.md).
