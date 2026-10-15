# Grafis, window, input, dan aset

Baca panduan ini saat memilih backend window, membuat resource grafis, atau memuat tekstur/model. Nama folder paket berbeda dari namespace: GLFW memakai `Brutal.GlfwApi`, Vulkan `Brutal.VulkanApi`, OpenGL `Brutal.OpenGlApi`, tekstur `Brutal.TextureApi`, dan glTF `Brutal.GltfApi`.

Contoh berikut adalah fragmen penggunaan API yang diperiksa dari sumber, bukan aplikasi lengkap. Siapkan pustaka native dan resource induk yang disebutkan sebelum menjalankannya; lihat [arsitektur dan batas build](architecture.md).

## Window dan input: pilih GLFW atau SDL

### GLFW

[Lifecycle GLFW](../../Brutal.Glfw/Manual/Libraries/Glfw.Initialization.cs) menyediakan `Glfw.Init(): bool` dan `Glfw.Terminate()`. Jalankan inisialisasi, pembuatan window, dan event pump pada main thread sesuai kontrak sumber. [GlfwWindow](../../Brutal.Glfw/Manual/Libraries/Glfw.Window.cs) dibuat lewat `Glfw.CreateWindow(in GlfwWindow.CreateInfo)` dan memiliki `Dispose()`, `ShouldClose`, serta event `OnKey`, `OnMouseButton`, dan callback window lain. `PollEvents()` juga melempar kembali exception callback yang ditampung wrapper.

Benar — fragmen pada main thread; window ini khusus Vulkan/tanpa context OpenGL:

```csharp
using Brutal.GlfwApi;
using Brutal.Numerics;

if (!Glfw.Init())
    throw new InvalidOperationException("GLFW gagal diinisialisasi");
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
        // Update dan rendering milik game.
    }
} // window.Dispose() berjalan sebelum Terminate().
finally
{
    Glfw.Terminate();
}
```

`WindowHint`, `GetKey`, `MakeContextCurrent`, dan `SwapBuffers` tersedia di [wrapper generated window](../../Brutal.Glfw/Generated/Libraries/Glfw.Window.gen.cs). Hint berlaku saat pembuatan window; pilih nilai sebelum `CreateWindow`.

Salah — melewati init dan event pump:

```csharp
var window = Glfw.CreateWindow(info); // Salah jika Glfw.Init() belum sukses.
while (!window.ShouldClose) { }       // Event window/input tidak diproses.
```

### SDL 3 dan keluarga SDL

[Sdl.Init.Initialize(SdlInitFlags)](../../Brutal.Sdl/Manual/Libraries/Sdl.Init.cs) melempar `SdlException` saat gagal. Pasangkan dengan `Sdl.Init.Quit()`. [Sdl.Video](../../Brutal.Sdl/Generated/Libraries/Sdl.Video.gen.cs) memiliki `CreateWindow(string, int, int, SdlWindowFlags)` dan `DestroyWindow(SdlWindow)`. [Sdl.Events.PollEvent(out SdlEvent)](../../Brutal.Sdl/Manual/Libraries/Sdl.Events.cs) menguras event dan memeriksa exception callback. Untuk input keyboard gunakan [Sdl.Keyboard](../../Brutal.Sdl/Manual/Libraries/Sdl.Keyboard.cs) dan [wrapper keyboard generated](../../Brutal.Sdl/Generated/Libraries/Sdl.Keyboard.gen.cs); fungsi polling memakai state yang diperbarui event pump.

`AddEventWatch` menyimpan delegate melalui `GCHandle`; lepaskan dengan `RemoveEventWatch` ketika watch tidak diperlukan. `Quit()` membersihkan watch dan aggregator exception. Callback masih dapat berjalan dari thread native; pilih data bersama sesuai kontrak callback.

| Kebutuhan | API yang sudah ada | Pemilik dan pelepasan |
|---|---|---|
| Gambar CPU SDL | [`SdlImage.Load(string)`](../../Brutal.Sdl.Image/Generated/Libraries/SdlImage.Ergonomic.gen.cs) di `Brutal.SdlApi.Image` menghasilkan `SdlSurface*` | Pemanggil; `Sdl.PInvoke.DestroySurface(surface)` di [PInvoke SDL](../../Brutal.Sdl/Generated/Sdl.PInvoke.gen.cs). |
| Tekstur untuk SDL renderer | `SdlImage.LoadTexture(SdlRenderer, string)` | Pemanggil; [`Sdl.Render.DestroyTexture`](../../Brutal.Sdl/Generated/Libraries/Sdl.Render.gen.cs), sebelum renderer dihancurkan. |
| Font SDL_ttf | [`SdlTtf.Init()`, `OpenFont(string, float)`](../../Brutal.Sdl.Ttf/Generated/Libraries/SdlTtf.Ergonomic.gen.cs) di `Brutal.SdlApi.Ttf` | `CloseFont(font)`, lalu `SdlTtf.Quit()`. |
| Mixer audio | [`Brutal.Sdl.Mixer`](../../Brutal.Sdl.Mixer/Manual/Libraries/SdlMixer.Init.cs) | Lihat [panduan audio dan runtime](physics-audio-jobs.md). |

`SdlSurface*`, `SdlTexture`, dan `ITexture` adalah tipe berbeda. Pilih loader sesuai renderer tujuan; `TextureLoader.Unload` hanya menerima implementasi yang dikenal loader tekstur BRUTAL.

## Vulkan: helper resource, bukan renderer game

Mulai dari [Vulkan.CreateInstance(Instance.CreateInfo)](../../Brutal.Vulkan/Manual/Partials/Vulkan.cs). [Instance](../../Brutal.Vulkan/Manual/Classes/Instance.cs) menyediakan `PhysicalDevices`; [Device](../../Brutal.Vulkan/Manual/Classes/Device.cs) dan handle wrapper membawa function table native. Gunakan wrapper manual/abstraksi yang sesuai sebelum turun ke `PInvoke` untuk operasi yang belum dibungkus.

Untuk surface GLFW, aktifkan extension dari [Glfw.GetRequiredInstanceExtensions](../../Brutal.Glfw/Generated/Libraries/Glfw.Vulkan.gen.cs) ketika membuat instance, buat window dengan `GLFW_NO_API`, lalu panggil [Glfw.CreateWindowSurface(nint, GlfwWindow, out ulong)](../../Brutal.Glfw/Manual/Libraries/Glfw.Vulkan.cs). Hasilnya kode Vulkan `int`, bukan exception otomatis. Pemanggil menghancurkan surface melalui Vulkan sebelum instance; GLFW tidak melakukannya. Untuk SDL, window memakai `SdlWindowFlags.Vulkan`, lalu [Sdl.Vulkan](../../Brutal.Sdl/Manual/Libraries/Sdl.Vulkan.cs) menyediakan `GetInstanceExtensions()`, `CreateSurface(SdlWindow, nint)`, dan `DestroySurface(nint, nint)` dengan exception saat pembuatan gagal.

Helper di namespace `Brutal.VulkanApi.Abstractions` mengurangi pekerjaan resource berikut:

| Area | Helper dan batas tanggung jawab |
|---|---|
| Device dan alokasi | [`PhysicalDeviceExtensions.CreateDevice(DeviceEx.CreateInfo)`](../../Brutal.Vulkan.Abstractions/Extensions/PhysicalDeviceExtensions.cs) menghasilkan [`DeviceEx`](../../Brutal.Vulkan.Abstractions/Classes/DeviceEx.cs), allocator image/buffer yang mengalokasikan memori langsung dari device. File extension yang sama menyediakan pemilihan format dan queue family. |
| Allocator VMA | [`VmaAllocator`](../../Brutal.Vulkan.Vma/Manual/VmaAllocator.cs) di `Brutal.VulkanApi.Vma` mengimplementasikan `IImageAllocator`/`IBufferAllocator`. Buat lewat [`device.CreateVmaAllocator(...)`](../../Brutal.Vulkan.Vma/Manual/VmaDeviceExtensions.cs), bebaskan seluruh alokasinya, lalu `device.DestroyVmaAllocator(allocator)`. |
| Resource dan mapping | [`BufferEx`](../../Brutal.Vulkan.Abstractions/BufferEx.cs), [`ImageEx`](../../Brutal.Vulkan.Abstractions/ImageEx.cs), [`ImageViewEx`](../../Brutal.Vulkan.Abstractions/ImageViewEx.cs), serta [`MappedMemory`](../../Brutal.Vulkan.Abstractions/MappedMemory.cs) menyimpan metadata/allocator dan menyediakan pelepasan eksplisit. `MappedMemory.Dispose()` mengakhiri mapping, bukan masa hidup buffer. |
| Staging | [`allocator.CreateStagingPool(queue, count)`](../../Brutal.Vulkan.Abstractions/StagingPool.cs), `AddStagingBuffer`, `NextCommandBuffer`, `Submit`, `Wait`/`WaitFor` mengelola buffer transfer, command pool, dan fence. Game tetap merekam command copy serta barrier. `Dispose()` menunggu submission lalu membebaskan resource pool. |
| Descriptor dan vertex | [`DescriptorPoolEx`](../../Brutal.Vulkan.Abstractions/DescriptorPoolEx.cs), [`DescriptorSetLayoutEx`](../../Brutal.Vulkan.Abstractions/DescriptorSetLayoutEx.cs), [`VertexInput`](../../Brutal.Vulkan.Abstractions/VertexInput.cs), dan [`Presets`](../../Brutal.Vulkan.Abstractions/Presets.cs) membantu alokasi descriptor dan konfigurasi pipeline. |
| Command dan debugging | [`CommandBufferExtensions`](../../Brutal.Vulkan.Abstractions/Extensions/CommandBufferExtensions.cs) membungkus recording/copy dan menyediakan `DebugScope`; [`SetDebugName`](../../Brutal.Vulkan.Abstractions/DebugNameExtensions.cs) memberi nama resource untuk tooling Vulkan. |

Resource anak dan alokasi harus selesai dipakai GPU sebelum dilepas; hancurkan resource/allocator sebelum device, kemudian surface dan instance. `using` mengatur pemanggilan `Dispose`, sedangkan urutan submission, barrier, dan sinkronisasi tetap tanggung jawab game. `BufferEx` dan `MappedMemory` berupa struct: salinan nilainya bukan alokasi atau mapping baru; tentukan satu pemilik pelepasan.

## OpenGL: context dahulu, function pointer kemudian

[Glad.Initialize(GlGetProcAddressFunc)](../../Brutal.OpenGl/Glad.cs) memuat GLAD dan pointer `OpenGl`. Jika memakai GLFW, buat window dengan client API OpenGL yang sesuai, panggil `window.MakeContextCurrent()`, baru inisialisasi loader. [Glfw.GetProcAddress(ReadOnlySpan<byte>)](../../Brutal.Glfw/Manual/Libraries/Glfw.Context.cs) memerlukan context current pada thread pemanggil.

Benar — fragmen dalam scope `unsafe`; `window` sudah memiliki context OpenGL, dan `System.Runtime.InteropServices` diimpor:

```csharp
window.MakeContextCurrent();
if (!Brutal.OpenGlApi.Glad.Initialize(name =>
    Glfw.GetProcAddress(MemoryMarshal.CreateReadOnlySpanFromNullTerminated(name))))
    throw new InvalidOperationException("GLAD gagal diinisialisasi");
```

Salah — memanggil entry point sebelum pointer diisi:

```csharp
Brutal.OpenGlApi.OpenGl.ClearColor(new float4(0, 0, 0, 1)); // Salah sebelum context + loader siap.
```

[OpenGl.cs](../../Brutal.OpenGl/OpenGl.cs) menambah overload vektor/matriks BRUTAL dan span; [OpenGl.gen.cs](../../Brutal.OpenGl/OpenGl.gen.cs) berisi operasi API generated. Pilih versi/extension yang didukung context. Loader ini tidak menyediakan loop render atau konversi model menjadi draw call.

## Tekstur: load CPU, baca format, lalu upload

[TextureLoader](../../Brutal.Texture/TextureLoader.cs) adalah pintu masuk praktis: `LoadFromFile(string, params TextureLoadSettings?[])`, atau `LoadFromMemory(ReadOnlySpan<byte>, FormatType, params TextureLoadSettings?[])`. Dispatch file memeriksa extension tanpa membedakan huruf besar/kecil, dengan urutan KTX, STB, lalu GLI:

| Backend | Format yang dipilih dispatcher | Catatan |
|---|---|---|
| [KTX](../../Brutal.Texture/Ktx/KtxTextureLoader.cs) | `.ktx`, `.ktx2` | `LoadSettings.LazyLoad` dan `SuperCompressionTranscodeFormat`; jalur file mentranscode jika diperlukan, default target BC7 RGBA. Jalur memori memanggil `KtxTexture2.CreateFromMemory` dan tidak menjalankan langkah transcode yang sama; periksa implementasi untuk aset terkompresi. |
| [STB](../../Brutal.Texture/Stb/StbTextureLoader.cs) | `.png`, `.jpg`/`.jpeg`, `.tga`, `.bmp`, `.hdr`, `.psd`, `.gif`, `.pic`, `.ppm`, `.pgm` | Data 8-bit, 16-bit, atau float; `Loader.LoadSettings.ForceRgba8` memaksa empat kanal 8-bit. |
| [GLI](../../Brutal.Texture/Gli/GliTextureLoader.cs) | `.dds`, `.kmg` | Backend ini juga mendukung KTX, tetapi dispatcher memilih backend KTX lebih dulu. |

[`ITexture`](../../Brutal.Texture/ITexture.cs) mengekspos data CPU: `Data`, `ImageData(level, layer, faceOrSlice)`, `Extent`, `Format`, mip/layer/face count, dan `RowPitch(level)`. Hasil ini belum menjadi resource GPU. `ITexture` sendiri bukan `IDisposable`; pasangkan loader dengan `TextureLoader.Unload(texture)` tepat sekali atau pakai [`texture.Using()`](../../Brutal.Texture/ScopedTexture.cs).

Benar — decode dan rencanakan staging selagi data masih hidup:

```csharp
using Brutal.TextureApi;
using Brutal.TextureApi.Abstractions;

using var owned = TextureLoader.LoadFromFile(
    "assets/albedo.png",
    Brutal.TextureApi.Stb.Loader.LoadSettings.ForceRgba8).Using();
ITexture texture = owned.Texture!;
StageUploadRegions regions = UploadRegions.Build(texture);
// Salin texture.Data sesuai regions.StageBufferRegions sebelum owned keluar scope.
// Buat image GPU, rekam copy/barrier, dan submit melalui backend game.
```

[`UploadRegions.Build`](../../Brutal.Texture.Abstractions/Upload/UploadRegions.cs) menangani mip/layer/face, row padding, dan alignment blok. Hasil [`StageUploadRegions`](../../Brutal.Texture.Abstractions/Upload/StageUploadRegions.cs) berisi `RequiredBufferSizeBytes`, `StageBufferRegions`, serta `CopyToImageRegions`; ini deskripsi transfer, bukan perintah upload otomatis. Pilih format GPU dan batas dukungan hardware sebelum membuat image.

Salah — melepas data sebelum menyalinnya:

```csharp
var texture = TextureLoader.LoadFromFile("assets/albedo.png");
var pixels = texture.Data;
TextureLoader.Unload(texture);
byte first = pixels[0]; // Salah: span menunjuk data native yang telah dibebaskan.
```

Untuk tekstur buatan sendiri, [`GenericTexture`](../../Brutal.Texture.Abstractions/ITextureImplementations/GenericTexture.cs) mengalokasikan memori CPU dan memiliki `Destroy()`. Lepaskan dengan API pemilik tersebut; `TextureLoader.Unload` menolak tipe yang tidak dikenalnya, termasuk `GenericTexture`. Statistik `CurrentLoadedCount` dan `Diagnostics.EnableLoadTracing(true)` pada `TextureLoader` membantu melacak hasil loader yang belum dilepas; pembentukan stack trace bukan pekerjaan hot path.

## glTF/GLB: metadata, buffer, dan gambar

[GltfUtility.LoadModel](../../Brutal.Gltf/GltfApi/GltfUtility.cs) membaca JSON `.gltf` atau chunk JSON `.glb`. Untuk mengambil buffer dan gambar sekaligus, gunakan [`GltfLoader(string filePath)`](../../Brutal.Gltf/GltfApi/GltfLoader.cs):

- `GltfJson` berisi schema mesh, primitive, node, material, skin, dan animasi. Constructor belum memuat semua buffer/gambar.
- `GetOrLoadBuffer(int)` memuat/cache binary buffer; `GetBufferView(int)` mengambil rentang buffer. File eksternal di-resolve relatif terhadap `FilePath`; `ExternalResolveBuffer`/`ExternalResolveImage` dapat menggantikannya.
- `GetOrLoadImage(int, TextureLoadSettings? = null)` memuat/cache `ITexture`. Index ini menunjuk `GltfJson.Images`, bukan langsung index `Textures` atau material. Jalur gambar buffer view saat ini menerima MIME PNG/JPEG; dukungan schema extension tidak berarti setiap jalur decode didukung.
- `GetAccessorBufferSpan(int)` memberi byte berdasarkan offset dan stride. Terapkan interpretasi `ComponentType`, `Type`, `Count`, `Normalized`, dan `ByteStride` sendiri; ini bukan array vertex yang otomatis terurai. Helper tidak menerapkan sparse accessor. [`Accessor.ElementSize()`](../../Brutal.Gltf/GltfApi/Accessor.cs) menolak kombinasi matriks kecil yang membutuhkan padding kolom.

Benar — hanya untuk aset yang memiliki setidaknya satu gambar:

```csharp
using Brutal.GltfApi;

using var model = new GltfLoader("assets/ship.glb");
if (model.LoadedImages.Length > 0)
{
    var image = model.GetOrLoadImage(0);
    // Baca/salin image.Data selama model masih hidup.
}
// model.Dispose() membebaskan seluruh gambar yang di-cache.
```

Salah — dua pemilik membebaskan gambar cache yang sama:

```csharp
using var model = new GltfLoader("assets/ship.glb"); // Asumsikan ada gambar 0.
var image = model.GetOrLoadImage(0);
TextureLoader.Unload(image); // Salah: model.Dispose() akan meng-unload gambar ini lagi.
```

Load setting hanya berpengaruh ketika gambar belum ada di cache. Bila koordinat game memerlukan konversi handedness, [`FlipHandedness()`/`FlipZAxis()`](../../Brutal.Gltf/GltfApi/GltfLoader.Transform.cs) memodifikasi data model; panggil sebelum mengekstrak aset dan periksa batas accessor pada implementasinya. Bentuk draw call, upload vertex/index, shader material, evaluasi animasi, dan struktur scene tetap dibangun oleh game di atas data loader.
