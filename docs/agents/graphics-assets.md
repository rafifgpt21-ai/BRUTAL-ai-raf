# Graphics, windows, input, and assets

Read this guide when choosing a window backend, creating graphics resources, or loading textures/models. Package folder names differ from namespaces: GLFW uses `Brutal.GlfwApi`, Vulkan `Brutal.VulkanApi`, OpenGL `Brutal.OpenGlApi`, textures `Brutal.TextureApi`, and glTF `Brutal.GltfApi`.

The examples below are usage fragments checked against source, not complete applications. Prepare the native libraries and parent resources mentioned before running them; see [architecture and build limitations](architecture.md).

## Windows and input: choose GLFW or SDL

### GLFW

The [GLFW lifecycle](../../Brutal.Glfw/Manual/Libraries/Glfw.Initialization.cs) provides `Glfw.Init(): bool` and `Glfw.Terminate()`. Run initialization, window creation, and the event pump on the main thread as required by the source contract. [GlfwWindow](../../Brutal.Glfw/Manual/Libraries/Glfw.Window.cs) is created with `Glfw.CreateWindow(in GlfwWindow.CreateInfo)` and has `Dispose()`, `ShouldClose`, and events such as `OnKey`, `OnMouseButton`, and other window callbacks. `PollEvents()` also rethrows callback exceptions collected by the wrapper.

Correct — a fragment on the main thread; this window is for Vulkan/no OpenGL context:

```csharp
using Brutal.GlfwApi;
using Brutal.Numerics;

if (!Glfw.Init())
    throw new InvalidOperationException("GLFW initialization failed");
try
{
    Glfw.WindowHint(GlfwWindowHint.ClientApi, Glfw.Constants.GLFW_NO_API);
    using var window = Glfw.CreateWindow(new GlfwWindow.CreateInfo
    {
        Title = "Game",
        Size = new int2(1280, 720),
    });
    while (!window.ShouldClose)
    {
        Glfw.PollEvents();
        if (window.GetKey(GlfwKey.Escape) == GlfwKeyAction.Press)
            window.ShouldClose = true;
        // The game supplies updates and rendering.
    }
} // window.Dispose() runs before Terminate().
finally
{
    Glfw.Terminate();
}
```

`WindowHint`, `GetKey`, `MakeContextCurrent`, and `SwapBuffers` are available in the [generated window wrapper](../../Brutal.Glfw/Generated/Libraries/Glfw.Window.gen.cs). Hints apply when the window is created; set their values before `CreateWindow`.

Incorrect — skipping initialization and the event pump:

```csharp
var window = Glfw.CreateWindow(info); // Incorrect if Glfw.Init() has not succeeded.
while (!window.ShouldClose) { }       // Window/input events are not processed.
```

### SDL 3 and the SDL family

[Sdl.Init.Initialize(SdlInitFlags)](../../Brutal.Sdl/Manual/Libraries/Sdl.Init.cs) throws `SdlException` on failure. Pair it with `Sdl.Init.Quit()`. [Sdl.Video](../../Brutal.Sdl/Generated/Libraries/Sdl.Video.gen.cs) has `CreateWindow(string, int, int, SdlWindowFlags)` and `DestroyWindow(SdlWindow)`. [Sdl.Events.PollEvent(out SdlEvent)](../../Brutal.Sdl/Manual/Libraries/Sdl.Events.cs) drains events and checks callback exceptions. For keyboard input, use [Sdl.Keyboard](../../Brutal.Sdl/Manual/Libraries/Sdl.Keyboard.cs) and the [generated keyboard wrapper](../../Brutal.Sdl/Generated/Libraries/Sdl.Keyboard.gen.cs); polling functions use state updated by the event pump.

`AddEventWatch` retains its delegate through a `GCHandle`; release it with `RemoveEventWatch` when no longer needed. `Quit()` clears watches and the exception aggregator. Callbacks may still run on native threads; choose shared data according to the callback contract.

| Need | Existing API | Ownership and release |
|---|---|---|
| SDL CPU image | [`SdlImage.Load(string)`](../../Brutal.Sdl.Image/Generated/Libraries/SdlImage.Ergonomic.gen.cs) in `Brutal.SdlApi.Image` returns `SdlSurface*` | Caller-owned; use `Sdl.PInvoke.DestroySurface(surface)` from [SDL PInvoke](../../Brutal.Sdl/Generated/Sdl.PInvoke.gen.cs). |
| Texture for an SDL renderer | `SdlImage.LoadTexture(SdlRenderer, string)` | Caller-owned; use [`Sdl.Render.DestroyTexture`](../../Brutal.Sdl/Generated/Libraries/Sdl.Render.gen.cs) before destroying the renderer. |
| SDL_ttf font | [`SdlTtf.Init()`, `OpenFont(string, float)`](../../Brutal.Sdl.Ttf/Generated/Libraries/SdlTtf.Ergonomic.gen.cs) in `Brutal.SdlApi.Ttf` | `CloseFont(font)`, then `SdlTtf.Quit()`. |
| Audio mixer | [`Brutal.Sdl.Mixer`](../../Brutal.Sdl.Mixer/Manual/Libraries/SdlMixer.Init.cs) | See the [SDL Mixer lifecycle](physics-audio-jobs.md#sdl-mixer). |

`SdlSurface*`, `SdlTexture`, and `ITexture` are different types. Choose a loader for the target renderer; `TextureLoader.Unload` only accepts implementations recognized by BRUTAL's texture loader.

## Vulkan: resource helpers, not a game renderer

Start with [Vulkan.CreateInstance(Instance.CreateInfo)](../../Brutal.Vulkan/Manual/Partials/Vulkan.cs). [Instance](../../Brutal.Vulkan/Manual/Classes/Instance.cs) exposes `PhysicalDevices`; [Device](../../Brutal.Vulkan/Manual/Classes/Device.cs) and handle wrappers carry native function tables. Use suitable manual/abstraction wrappers before reaching for `PInvoke` for operations without wrappers.

For a GLFW surface, enable the extensions returned by [Glfw.GetRequiredInstanceExtensions](../../Brutal.Glfw/Generated/Libraries/Glfw.Vulkan.gen.cs) when creating the instance, create a window with `GLFW_NO_API`, then call [Glfw.CreateWindowSurface(nint, GlfwWindow, out ulong)](../../Brutal.Glfw/Manual/Libraries/Glfw.Vulkan.cs). It returns a Vulkan `int` result code rather than throwing automatically. The caller destroys the surface through Vulkan before the instance; GLFW does not do this. For SDL, create the window with `SdlWindowFlags.Vulkan`, then use [Sdl.Vulkan](../../Brutal.Sdl/Manual/Libraries/Sdl.Vulkan.cs), which provides `GetInstanceExtensions()`, `CreateSurface(SdlWindow, nint)`, and `DestroySurface(nint, nint)`, throwing if creation fails.

Helpers in `Brutal.VulkanApi.Abstractions` reduce the following resource work:

| Area | Helpers and responsibility boundaries |
|---|---|
| Device and allocation | [`PhysicalDeviceExtensions.CreateDevice(DeviceEx.CreateInfo)`](../../Brutal.Vulkan.Abstractions/Extensions/PhysicalDeviceExtensions.cs) returns [`DeviceEx`](../../Brutal.Vulkan.Abstractions/Classes/DeviceEx.cs), an image/buffer allocator that allocates memory directly from the device. The same extension file provides format and queue family selection. |
| VMA allocator | [`VmaAllocator`](../../Brutal.Vulkan.Vma/Manual/VmaAllocator.cs) in `Brutal.VulkanApi.Vma` implements `IImageAllocator`/`IBufferAllocator`. Create it through [`device.CreateVmaAllocator(...)`](../../Brutal.Vulkan.Vma/Manual/VmaDeviceExtensions.cs), free all its allocations, then call `device.DestroyVmaAllocator(allocator)`. |
| Resources and mapping | [`BufferEx`](../../Brutal.Vulkan.Abstractions/BufferEx.cs), [`ImageEx`](../../Brutal.Vulkan.Abstractions/ImageEx.cs), [`ImageViewEx`](../../Brutal.Vulkan.Abstractions/ImageViewEx.cs), and [`MappedMemory`](../../Brutal.Vulkan.Abstractions/MappedMemory.cs) retain metadata/allocators and provide explicit release. `MappedMemory.Dispose()` ends the mapping, not the buffer's lifetime. |
| Staging | [`allocator.CreateStagingPool(queue, count)`](../../Brutal.Vulkan.Abstractions/StagingPool.cs), `AddStagingBuffer`, `NextCommandBuffer`, `Submit`, and `Wait`/`WaitFor` manage transfer buffers, a command pool, and fences. The game still records copy commands and barriers. `Dispose()` waits for submissions, then frees pool resources. |
| Descriptors and vertices | [`DescriptorPoolEx`](../../Brutal.Vulkan.Abstractions/DescriptorPoolEx.cs), [`DescriptorSetLayoutEx`](../../Brutal.Vulkan.Abstractions/DescriptorSetLayoutEx.cs), [`VertexInput`](../../Brutal.Vulkan.Abstractions/VertexInput.cs), and [`Presets`](../../Brutal.Vulkan.Abstractions/Presets.cs) help with descriptor allocation and pipeline configuration. |
| Commands and debugging | [`CommandBufferExtensions`](../../Brutal.Vulkan.Abstractions/Extensions/CommandBufferExtensions.cs) wraps recording/copy operations and provides `DebugScope`; [`SetDebugName`](../../Brutal.Vulkan.Abstractions/DebugNameExtensions.cs) names resources for Vulkan tooling. |

Child resources and allocations must be released only after the GPU has finished using them; destroy resources/allocators before the device, then the surface and instance. `using` controls `Dispose` calls, while submission order, barriers, and synchronization remain the game's responsibility. `BufferEx` and `MappedMemory` are structs: copying their values does not create another allocation or mapping; assign a single owner for release.

## OpenGL: context first, function pointers second

[Glad.Initialize(GlGetProcAddressFunc)](../../Brutal.OpenGl/Glad.cs) loads GLAD and the `OpenGl` pointers. When using GLFW, create a window with an appropriate OpenGL client API, call `window.MakeContextCurrent()`, then initialize the loader. [Glfw.GetProcAddress(ReadOnlySpan<byte>)](../../Brutal.Glfw/Manual/Libraries/Glfw.Context.cs) requires a current context on the calling thread.

Correct — a fragment inside an `unsafe` scope; `window` already has an OpenGL context, and `System.Runtime.InteropServices` is imported:

```csharp
window.MakeContextCurrent();
if (!Brutal.OpenGlApi.Glad.Initialize(name =>
    Glfw.GetProcAddress(MemoryMarshal.CreateReadOnlySpanFromNullTerminated(name))))
    throw new InvalidOperationException("GLAD initialization failed");
```

Incorrect — calling an entry point before its pointer is populated:

```csharp
Brutal.OpenGlApi.OpenGl.ClearColor(new float4(0, 0, 0, 1)); // Incorrect before context + loader setup.
```

[OpenGl.cs](../../Brutal.OpenGl/OpenGl.cs) adds BRUTAL vector/matrix and span overloads; [OpenGl.gen.cs](../../Brutal.OpenGl/OpenGl.gen.cs) contains generated API operations. Choose versions/extensions supported by the context. This loader does not provide a render loop or convert models into draw calls.

## Textures: load on the CPU, inspect the format, then upload

[TextureLoader](../../Brutal.Texture/TextureLoader.cs) is the practical entry point: `LoadFromFile(string, params TextureLoadSettings?[])` or `LoadFromMemory(ReadOnlySpan<byte>, FormatType, params TextureLoadSettings?[])`. File dispatch checks extensions case-insensitively in KTX, STB, then GLI order:

| Backend | Formats selected by the dispatcher | Notes |
|---|---|---|
| [KTX](../../Brutal.Texture/Ktx/KtxTextureLoader.cs) | `.ktx`, `.ktx2` | `LoadSettings.LazyLoad` and `SuperCompressionTranscodeFormat`; the file path transcodes when needed, with BC7 RGBA as the default target. The memory path calls `KtxTexture2.CreateFromMemory` without the same transcode step; inspect the implementation for compressed assets. |
| [STB](../../Brutal.Texture/Stb/StbTextureLoader.cs) | `.png`, `.jpg`/`.jpeg`, `.tga`, `.bmp`, `.hdr`, `.psd`, `.gif`, `.pic`, `.ppm`, `.pgm` | 8-bit, 16-bit, or float data; `Loader.LoadSettings.ForceRgba8` forces four 8-bit channels. |
| [GLI](../../Brutal.Texture/Gli/GliTextureLoader.cs) | `.dds`, `.kmg` | This backend also supports KTX, but the dispatcher chooses the KTX backend first. |

[`ITexture`](../../Brutal.Texture/ITexture.cs) exposes CPU data: `Data`, `ImageData(level, layer, faceOrSlice)`, `Extent`, `Format`, mip/layer/face counts, and `RowPitch(level)`. The result is not yet a GPU resource. `ITexture` itself is not `IDisposable`; pair the loader with exactly one `TextureLoader.Unload(texture)` call or use [`texture.Using()`](../../Brutal.Texture/ScopedTexture.cs).

Correct — decode and plan staging while the data remains alive:

```csharp
using Brutal.TextureApi;
using Brutal.TextureApi.Abstractions;

using var owned = TextureLoader.LoadFromFile(
    "assets/albedo.png",
    Brutal.TextureApi.Stb.Loader.LoadSettings.ForceRgba8).Using();
ITexture texture = owned.Texture!;
StageUploadRegions regions = UploadRegions.Build(texture);
// Copy texture.Data according to regions.StageBufferRegions before owned leaves scope.
// Create the GPU image, record copy/barriers, and submit through the game's backend.
```

[`UploadRegions.Build`](../../Brutal.Texture.Abstractions/Upload/UploadRegions.cs) handles mip/layer/face ranges, row padding, and block alignment. Its [`StageUploadRegions`](../../Brutal.Texture.Abstractions/Upload/StageUploadRegions.cs) result contains `RequiredBufferSizeBytes`, `StageBufferRegions`, and `CopyToImageRegions`; these describe transfers rather than uploading automatically. Choose the GPU format and hardware support requirements before creating the image.

Incorrect — releasing data before copying it:

```csharp
var texture = TextureLoader.LoadFromFile("assets/albedo.png");
var pixels = texture.Data;
TextureLoader.Unload(texture);
byte first = pixels[0]; // Incorrect: the span points to freed native data.
```

For custom textures, [`GenericTexture`](../../Brutal.Texture.Abstractions/ITextureImplementations/GenericTexture.cs) allocates CPU memory and has `Destroy()`. Release it through that owner's API; `TextureLoader.Unload` rejects unrecognized types, including `GenericTexture`. `TextureLoader` statistics such as `CurrentLoadedCount` and `Diagnostics.EnableLoadTracing(true)` help track loaded textures that have not been released; constructing stack traces is not hot-path work.

## glTF/GLB: metadata, buffers, and images

[GltfUtility.LoadModel](../../Brutal.Gltf/GltfApi/GltfUtility.cs) reads `.gltf` JSON or a `.glb` JSON chunk. To access buffers and images as well, use [`GltfLoader(string filePath)`](../../Brutal.Gltf/GltfApi/GltfLoader.cs):

- `GltfJson` contains the schema for meshes, primitives, nodes, materials, skins, and animations. The constructor does not load every buffer/image.
- `GetOrLoadBuffer(int)` loads/caches a binary buffer; `GetBufferView(int)` returns a buffer range. External files resolve relative to `FilePath`; `ExternalResolveBuffer`/`ExternalResolveImage` can override resolution.
- `GetOrLoadImage(int, TextureLoadSettings? = null)` loads/caches an `ITexture`. The index refers to `GltfJson.Images`, not directly to `Textures` or a material. The buffer-view image path currently accepts PNG/JPEG MIME types; schema extension support does not imply support in every decoding path.
- `GetAccessorBufferSpan(int)` returns bytes based on offsets and strides. Interpret `ComponentType`, `Type`, `Count`, `Normalized`, and `ByteStride` yourself; it is not an automatically decoded vertex array. The helper does not apply sparse accessors. [`Accessor.ElementSize()`](../../Brutal.Gltf/GltfApi/Accessor.cs) rejects small matrix combinations that require column padding.

Correct — for an asset containing at least one image:

```csharp
using Brutal.GltfApi;

using var model = new GltfLoader("assets/ship.glb");
if (model.LoadedImages.Length > 0)
{
    var image = model.GetOrLoadImage(0);
    // Read/copy image.Data while model is alive.
}
// model.Dispose() frees all cached images.
```

Incorrect — two owners freeing the same cached image:

```csharp
using var model = new GltfLoader("assets/ship.glb"); // Assume image 0 exists.
var image = model.GetOrLoadImage(0);
TextureLoader.Unload(image); // Incorrect: model.Dispose() will unload this image again.
```

Load settings apply only before an image enters the cache. If game coordinates require handedness conversion, [`FlipHandedness()`/`FlipZAxis()`](../../Brutal.Gltf/GltfApi/GltfLoader.Transform.cs) modifies model data; call it before extracting assets and check accessor limitations in the implementation. Draw calls, vertex/index uploads, material shaders, animation evaluation, and scene structures are still built by the game on top of the loader's data.
