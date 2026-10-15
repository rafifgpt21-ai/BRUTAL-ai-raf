# Arsitektur, binding, dan persiapan proyek

Kembali ke [AGENTS.md](../../AGENTS.md). Untuk API game, lanjutkan ke [fondasi](foundation.md), [grafis/aset](graphics-assets.md), atau [fisika/audio/job](physics-audio-jobs.md).

## Mulai dari kondisi repo yang sebenarnya

[README: Getting started / Building from source](../../README.md#getting-started) menyebut .NET 10 SDK, Windows/Linux, dan penggunaan paket NuGet `Ahwoo.Brutal.*`. Rendering Vulkan juga memerlukan GPU/driver yang sesuai; Vulkan SDK disebut untuk validation layers. Repo ini berisi source tanpa `.csproj`, `.sln`, `.slnx`, atau konfigurasi build/test. Jangan menjanjikan `dotnet build`/`dotnet test` di root atau menebak versi dependensi dari `using`.

Untuk eksperimen game terpisah, perintah berikut berasal dari README:

```powershell
dotnet new console -n HelloBrutal
cd HelloBrutal
dotnet add package Ahwoo.Brutal.Core
dotnet add package Ahwoo.Brutal.Glfw
```

Contoh window lengkap ada di [panduan grafis](graphics-assets.md). Pilih paket tambahan sesuai API yang benar-benar dipakai. Kode dengan pointer memerlukan konteks `unsafe` dan `AllowUnsafeBlocks` pada project konsumen. Contoh dalam panduan ini diverifikasi terhadap source, bukan hasil kompilasi project yang belum tersedia; contoh bertanda fragmen memerlukan objek/konteks yang disebutkan di sekitarnya.

Binary native tetap diperlukan. Misalnya [resolver Vulkan](../../Brutal.Vulkan/ImportResolver.cs) memuat library sesuai OS dan memperoleh `vkGetInstanceProcAddr`. Menyalin `.cs` tidak otomatis menyediakan DLL, generator, atau dependensi paket. `Brutal.FreeType/` dan `Brutal.ZLib/` pada snapshot hanya berisi `LICENSES/`; jangan menyusun contoh API C# dari nama paketnya.

## Generated dan Manual menyusun API yang sama

| Lokasi | Peran yang terlihat pada source |
| --- | --- |
| `Generated/`, `*.gen.cs` | Hasil generator: enum, layout struct, delegate, handle, binding P/Invoke, dan wrapper publik. Generated tidak selalu berarti hanya API mentah. |
| `Manual/` dan helper di root modul | Implementasi tulisan tangan, overload khusus, pengelolaan lifecycle, dan partial yang melengkapi tipe generated. |
| `Generated-Internal/` di ImGui | Permukaan internal ImGui; pilih API publik bila tersedia karena detail internal mengikat game ke implementasi library. |

[Glfw.Context generated](../../Brutal.Glfw/Generated/Libraries/Glfw.Context.gen.cs) bertanda `do not edit by hand`. [Partial manual](../../Brutal.Glfw/Manual/Libraries/Glfw.Context.cs) menjelaskan bahwa `GetCurrentContext` dan `GetProcAddress` ditangani khusus, sedangkan anggota lain digabungkan ke `Brutal.GlfwApi.Glfw` yang sama. [Glfw.Vulkan manual](../../Brutal.Glfw/Manual/Libraries/Glfw.Vulkan.cs) menambahkan `CreateWindowSurface` dengan pinning output dan allocator null.

Contoh kecil yang bisa dilacak langsung: [VkExtent2D generated](../../Brutal.Vulkan/Generated/Structs/VkExtent2D.gen.cs) mendefinisikan `Width`/`Height`, sedangkan [partial manual](../../Brutal.Vulkan/Manual/Partials/VkExtent2D.cs) menambahkan konstruktor dua argumen.

```csharp
using Brutal.VulkanApi;

// Benar: satu tipe publik, gabungan generated + manual.
var extent = new VkExtent2D(1280, 720);

// Salah: namespace tidak mengikuti nama folder source.
// var extent = new Brutal.Vulkan.Generated.VkExtent2D(1280, 720);
```

Generator disebut dalam header, tetapi project generator tidak ikut dalam snapshot. Untuk perubahan binding, temukan sumber generator sebelum regenerasi; untuk helper game gunakan API publik atau partial/helper tulisan tangan yang sesuai, bukan patch hasil generator.

## Abstractions dan Extensions

Mulai dengan helper yang cocok dengan kebutuhan game, lalu turun ke binding publik saat kontrol tambahan diperlukan. Pemilihan lapisan dilakukan per fitur; tidak ada urutan inisialisasi universal untuk semua modul.

| Lapisan | Contoh nyata dan alasan pemakaian |
| --- | --- |
| `.Abstractions` | [DeviceEx](../../Brutal.Vulkan.Abstractions/Classes/DeviceEx.cs), [StagingPool](../../Brutal.Vulkan.Abstractions/StagingPool.cs), dan [FrameResources](../../Brutal.Vulkan.Abstractions/FrameResources.cs) membantu resource GPU; [ConsoleWindow](../../Brutal.ImGui.Abstractions/ConsoleWindow/ConsoleWindow.cs) menyediakan UI console. Ini komponen untuk dirangkai, bukan renderer/game loop lengkap. |
| `.Extensions` | [ImGuiEx](../../Brutal.ImGui.Extensions/ImGui.extensions.cs) menambahkan operasi enum dan warna; `CheckboxFlags<T>` memeriksa ukuran enum harus sama dengan `int`. Nama Extensions juga muncul sebagai folder helper di modul lain. |
| Binding publik | `Glfw`, `ImGui`, serta tipe handle Vulkan/Jolt menawarkan signature C# yang menangani sebagian marshaling. Baca signature yang dipakai, termasuk ref/out/span dan ownership. |
| `PInvoke` | Kontrak native berisi pointer, ukuran, dan callback. Gunakan hanya saat wrapper publik tidak mencukupi dan game menangani lifetime serta ABI dengan benar. |

Jangan memilih overload lama hanya karena mirip tutorial library C# lain. [ImGui compatibility](../../Brutal.ImGui.Extensions/Compatibility/ImGui.Compatibility.cs) menandai `InputText(ref string)` obsolete dan mengalokasikan `ImInputString` sementara. Pilih `ImInputString` persisten untuk input UI; penjelasan ada di [fondasi](foundation.md).

## Namespace, ABI, dan lifecycle

Nama folder bukan kontrak namespace: GLFW memakai `Brutal.GlfwApi`, Vulkan `Brutal.VulkanApi`, ImGui `Brutal.ImGuiApi`, Jolt `Brutal.JoltApi`, dan interop `Brutal.Interop`. Periksa deklarasi `namespace` serta `GlobalUsings.cs` modul. Global using milik library tidak otomatis menjadi import dalam project game.

[InteropStringMarshaller](../../Brutal.Interop/InteropStringMarshaller.cs) mengonversi `string` ke UTF-8 berterminator null, memakai buffer kecil atau alokasi native saat diperlukan. [IInteropHandle](../../Brutal.Interop/IInteropHandle.cs) dan [marshaller handle](../../Brutal.Interop/InteropHandleMarshaller.cs) memetakan wrapper struct ke handle unmanaged. Konversi tipe bukan bukti bahwa resource dimiliki wrapper atau aman disimpan lebih lama.

Beberapa struct Vulkan mempunyai initializer penting. [VkInstanceCreateInfo](../../Brutal.Vulkan/Generated/Structs/VkInstanceCreateInfo.gen.cs) mengisi `sType` melalui konstruktor; `default` hanya men-zero-kan memori.

```csharp
using Brutal.VulkanApi;

unsafe
{
    // Benar untuk menjalankan initializer sType; isi field lain sesuai kebutuhan.
    var info = new VkInstanceCreateInfo();

    // Salah bila diteruskan sebagai create-info siap pakai: sType tidak diinisialisasi.
    // VkInstanceCreateInfo info = default;
}
```

Ini fragmen inisialisasi, bukan pembuatan instance lengkap. Untuk native resource, verifikasi create/init, pemilik, penggunaan, sinkronisasi, dan destroy/dispose pada tipe masing-masing. `IDisposable`, wrapper `Managed...`, struct handle pinjaman, dan proxy COM mempunyai perilaku berbeda; panduan runtime dan grafis merinci contohnya.

## Memperbarui panduan

Saat API berubah, baca implementasi aktual lalu perbarui contoh dan tautan pada panduan topik terkait. Pertahankan daftar semua modul di `AGENTS.md` sesuai folder repo. Periksa target tautan, namespace, overload, dan lifecycle; jalankan build/test hanya saat project serta dependensi yang diperlukan tersedia dan laporkan cakupan verifikasinya.
