# Panduan Agent BRUTAL

BRUTAL adalah framework C# untuk menyusun game atau engine sendiri: binding native dan utilitas memori, matematika, teks, serta job. Game menentukan loop, rendering, dan sistem gameplay-nya. Mulai dari [README](README.md); snapshot ini belum memiliki project/solution untuk build dari source.

## Aturan utama

- Baca implementasi dan overload sebelum memakai API; nama folder paket sering berbeda dari namespace C#.
- Pilih helper publik atau `.Abstractions` yang sesuai, lalu binding publik; pakai `PInvoke` hanya bila perlu mengelola detail ABI sendiri.
- Perluas kode tulisan tangan melalui partial/helper yang sesuai. Jangan mengedit `.gen.cs` secara manual; perubahan permanen harus melalui generator, yang belum disertakan dalam snapshot ini.
- Catat siapa pemilik setiap handle/buffer, kapan dipinjam, dan siapa yang melepasnya. Menyalin struct handle tidak menggandakan resource native.
- Gunakan tipe ABI dan teks UTF-8 yang diminta signature. Simpan `bool`, `string`, dan koleksi .NET bila API dan kebutuhan game memang cocok.
- Hindari alokasi per frame yang tidak perlu; periksa implementasi sebelum menganggap koleksi atau konversi teks bebas alokasi.
- Hormati thread dan lifecycle library: window/UI, callback, GPU, dan job memiliki aturan berbeda; lihat panduan terkait.
- Untuk kontribusi, ikuti [CONTRIBUTING.md](CONTRIBUTING.md), termasuk sign-off commit. Rujuk [LICENSE.txt](LICENSE.txt) dan [NOTICE.txt](NOTICE.txt) saat distribusi.

## Pilih panduan sesuai pekerjaan

| Saat mengerjakan | Baca sebelum menulis kode |
| --- | --- |
| Binding, partial, namespace, persiapan proyek | [Arsitektur dan batas snapshot](docs/agents/architecture.md) |
| Tipe dasar, pointer, memori, koleksi, matematika, teks, log, lokalisasi | [Fondasi](docs/agents/foundation.md) |
| Window/input, Vulkan/OpenGL, tekstur dan model glTF | [Grafis dan aset](docs/agents/graphics-assets.md) |
| Fisika, audio, console/command ImGui, job paralel | [Fisika, audio, UI debug, dan job](docs/agents/physics-audio-jobs.md) |

## Semua modul dalam snapshot

Tabel ini mencakup 48 folder `Brutal.*`. Tautan menuju source; ketersediaan source tidak menjamin seluruh tool build atau binary native ikut tersedia.

| Modul | Kegunaan / batas yang perlu diketahui |
| --- | --- |
| [Brutal.Core](Brutal.Core/), [Brutal.Interop](Brutal.Interop/) | Tipe ABI, pointer, atribut/platform; marshaller string dan handle native |
| [Brutal.Memory](Brutal.Memory/), [Brutal.Collections](Brutal.Collections/) | Allocator/memori native; koleksi native dan stride |
| [Brutal.Numerics](Brutal.Numerics/), [Brutal.Maths](Brutal.Maths/) | Vektor/matriks/warna; extension `Random` |
| [Brutal.Strings](Brutal.Strings/), [Brutal.Logging](Brutal.Logging/), [Brutal.Localization](Brutal.Localization/) | UTF-8/builder, log kategori, teks terjemahan |
| [Brutal.Glfw](Brutal.Glfw/), [Brutal.Sdl](Brutal.Sdl/) | Window, input, event, integrasi API grafis |
| [Brutal.Sdl.Image](Brutal.Sdl.Image/), [Brutal.Sdl.Mixer](Brutal.Sdl.Mixer/), [Brutal.Sdl.Ttf](Brutal.Sdl.Ttf/) | Add-on SDL untuk gambar, mixing audio, font |
| [Brutal.Vulkan](Brutal.Vulkan/), [Brutal.Vulkan.Abstractions](Brutal.Vulkan.Abstractions/), [Brutal.Vulkan.Vma](Brutal.Vulkan.Vma/) | Binding/handle Vulkan, helper resource/rendering, allocator VMA |
| [Brutal.OpenGl](Brutal.OpenGl/), [Brutal.ShaderC](Brutal.ShaderC/), [Brutal.Slang](Brutal.Slang/) | Binding OpenGL; kompilasi shader |
| [Brutal.Texture](Brutal.Texture/), [Brutal.Texture.Abstractions](Brutal.Texture.Abstractions/) | Data tekstur dan pembaca format melalui decoder |
| [Brutal.Gli](Brutal.Gli/), [Brutal.Ktx](Brutal.Ktx/), [Brutal.Stb](Brutal.Stb/) | Tekstur GLI/KTX dan decoder STB |
| [Brutal.Gltf](Brutal.Gltf/), [Brutal.MeshOptimizer](Brutal.MeshOptimizer/), [Brutal.Render.Mesh](Brutal.Render.Mesh/) | Data/akses glTF, optimasi mesh, storage dan mesh prosedural |
| [Brutal.ImGui](Brutal.ImGui/), [Brutal.ImGui.Extensions](Brutal.ImGui.Extensions/), [Brutal.ImGui.Abstractions](Brutal.ImGui.Abstractions/) | UI immediate, overload/helper, console dan terminal |
| [Brutal.ImGui.Backends](Brutal.ImGui.Backends/), [Brutal.ImGui.FreeType](Brutal.ImGui.FreeType/), [Brutal.ImPlot](Brutal.ImPlot/) | Backend platform/render, integrasi font FreeType, plotting |
| [Brutal.Jolt](Brutal.Jolt/), [Brutal.Box2D](Brutal.Box2D/) | Fisika 3D dan 2D |
| [Brutal.Fmod](Brutal.Fmod/), [Brutal.OpenAl](Brutal.OpenAl/) | Audio native |
| [Brutal.Concurrency](Brutal.Concurrency/), [Brutal.RakNet](Brutal.RakNet/) | Job/threading; networking |
| [Brutal.Monitor.Common](Brutal.Monitor.Common/), [Brutal.Monitor.Host](Brutal.Monitor.Host/) | Opsi/protokol monitor; integrasi host, log dan crash dump |
| [Brutal.RenderDoc](Brutal.RenderDoc/), [Brutal.RenderDoc.ImGui](Brutal.RenderDoc.ImGui/) | Capture RenderDoc dan jendela kontrol ImGui |
| [Brutal.NativeFileDialog](Brutal.NativeFileDialog/), [Brutal.NotCurses](Brutal.NotCurses/) | Dialog file native dan UI terminal |
| [Brutal.FreeType](Brutal.FreeType/), [Brutal.ZLib](Brutal.ZLib/) | Folder lisensi pihak ketiga saja pada snapshot ini; API C# belum ada |

## Agent skills

### Issue tracker

Spec dan tiket disimpan di GitHub Issues rafsidqi/BRUTAL-raf.
Sebelum membaca, membuat, atau menyelesaikan tiket, baca
[aturan issue tracker](docs/agents/issue-tracker.md).

### Domain docs

Layout single-context: GLOSSARY.md dan docs/adr/ di root.
Sebelum eksplorasi atau perubahan domain, baca
[aturan domain](docs/agents/domain.md).
