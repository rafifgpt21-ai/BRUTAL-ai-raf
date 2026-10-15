# Tipe dasar, memori, matematika, dan teks

Baca panduan ini sebelum memilih tipe data interop, menyimpan pointer/span, atau menulis teks UI, log, dan lokalisasi. Contoh adalah fragmen pemakaian C#; sesuaikan referensi paket pada proyek game. Blok `unsafe` memerlukan kompilasi dengan unsafe diaktifkan. Snapshot ini tidak menyertakan proyek build untuk mengompilasi contoh.

## Core dan kepemilikan memori

Nama modul tidak selalu sama dengan namespace. `Bool8`, `Bool32`, `ByteSize`, dan `ElementCount` berada di `Brutal`; pointer berada di `Brutal.Pointers`; allocator di `Brutal.Memory`.

| Tipe | Gunakan saat |
| --- | --- |
| [`Bool8`](../../Brutal.Core/Types/Bool8.cs), [`Bool32`](../../Brutal.Core/Types/Bool32.cs) | Signature native meminta boolean berukuran 8 atau 32 bit. Keduanya memiliki konversi implisit dari/ke `bool`; pilih sesuai signature, bukan sebagai pengganti semua `bool` game. |
| [`ByteSize`](../../Brutal.Core/Types/ByteSize.cs), [`ElementCount`](../../Brutal.Core/Types/ElementCount.cs) | Membedakan byte dan jumlah elemen. `ByteSize.Of<T>(count)` menghitung ukuran `count` elemen `T`. |
| [`Ptr`](../../Brutal.Core/Pointers/Ptr.cs), [`Ptr<T>`](../../Brutal.Core/Pointers/PtrOfT.cs) | Membungkus `void*` atau `T*` (`T : unmanaged`). Ini view alamat, tanpa ownership atau pemeriksaan batas. `Ptr.Offset` menerima byte; `Ptr<T>.Offset` menambah elemen menurut implementasinya. |
| [`PtrPtr`](../../Brutal.Core/Pointers/PtrPtr.cs), [`PtrPtr<T>`](../../Brutal.Core/Pointers/PtrPtrOfT.cs) | Signature native membutuhkan pointer ke pointer. Ikuti signature binding. |

[`Mem`](../../Brutal.Memory/Mem.cs) membungkus `NativeMemory`: `Alloc`, `AllocZeroed`, `AllocAndCopy`, `Copy`, dan `Free`. `Mem.Alloc<T>(count)` mengembalikan `Span<T>` di memori native, bukan buffer milik GC; pemanggil tetap harus membebaskannya dengan `Mem.Free(span)`. Gunakan `AllocZeroed` ketika isi awal harus nol.

[`PtrOwner<T>`](../../Brutal.Memory/PtrOwner.cs) memiliki pointer, membebaskannya lewat `Dispose`, serta menyediakan finalizer cadangan. Constructor dari nilai mengalokasikan salinan; constructor dari `Ptr<T>` mengambil ownership pointer tersebut. Berikan hanya alokasi yang kompatibel dengan `Mem.Free`, dengan satu pemilik. Pointer pinjaman dari library native tetap mengikuti fungsi penghancur library itu. `Release()` memindahkan tanggung jawab pembebasan ke pemanggil. Periksa implementasi sebelum memakai helper ownership lain: `Reset()` saat ini memanggil `Release()` dan membuang pointer lama tanpa membebaskannya, meskipun komentarnya menyebut pembebasan.

```csharp
using Brutal.Memory;

unsafe
{
    // Benar: pemilik hidup selama pointer dipakai.
    using var value = new PtrOwner<int>(42);
    value.Ptr.Value = 7;
}
```

```csharp
using Brutal.Memory;

unsafe
{
    // Salah: salinan Ptr tidak ikut mempertahankan alokasi.
    var owner = new PtrOwner<int>(42);
    var borrowed = owner.Ptr;
    owner.Dispose();
    borrowed.Value = 7; // use-after-free
}
```

View hasil `AsSpan`, `Cast`, dan `Offset` hanya valid selama backing memory hidup. Konversi span managed ke `Ptr<T>` tidak melakukan pinning permanen; untuk pemakaian alamat managed oleh native, pertahankan `fixed` selama seluruh akses. [`FixedAttribute`](../../Brutal.Core/Attributes.cs) adalah penanda kontrak, bukan pengganti blok `fixed`.

## Koleksi

Gunakan [`NativeList<T>`](../../Brutal.Collections/NativeList/NativeListOfT.cs) (`Brutal.Collections`, `T : unmanaged`) ketika memerlukan daftar kontigu di memori native. Ia memiliki `Count`, `Capacity`, indexer `ref`, `Add`, `AddRange`, `AsSpan`, dan `Ptr`; tutup dengan `using`/`Dispose`. `List<T>` .NET tetap cocok untuk data managed, termasuk objek referensi.

```csharp
using Brutal.Collections;
using Brutal.Numerics;

using var positions = new NativeList<float3>(capacity: 16);
positions.Add(new float3(1, 2, 3));
positions.AsSpan()[0] = float3.Zero;
// Benar: ambil ulang view setelah operasi yang mungkin mengubah kapasitas.
positions.EnsureCapacity(64);
var current = positions.AsSpan();
```

Salah: menyimpan `positions.Ptr`, `ref positions[0]`, atau span lama lalu memakainya setelah pertumbuhan kapasitas, `TrimExcess`, atau `Dispose`. [`NativeList._Reallocate`](../../Brutal.Collections/NativeList/NativeList.cs) mengalokasikan buffer baru dan membebaskan buffer lama.

Varian yang tersedia:

- [`NativeList`](../../Brutal.Collections/NativeList/NativeList.cs): buffer tanpa tipe/stride tetap; ukuran dan kapasitas publiknya berupa byte.
- [`NativeStrideList`](../../Brutal.Collections/NativeList/NativeStrideList.cs): pemilik buffer dengan stride tetap dalam byte dan jumlah elemen; gunakan jika layout ditentukan runtime.
- [`NativeStrideSpan`](../../Brutal.Collections/Span/NativeStrideSpan.cs): view tanpa ownership. `AsSpan<T>()` mensyaratkan ukuran `T` sama dengan stride, kecuali `T` adalah `byte`.
- [`ScratchSpan<T>`](../../Brutal.Collections/Span/ScratchSpan.cs), namespace `Brutal.Collections.Experimental`: scratch buffer berbasis span stack bila cukup, selain itu meminjam array dari pool. `using var scratch = new ScratchSpan<int>(requiredLength, stackalloc int[256]);` lalu gunakan `scratch.Span` sebelum `Dispose` mengembalikan array.
- [`FreeListIndexPool`](../../Brutal.Collections/FreeList.cs): pool indeks dengan slot yang dapat dipakai ulang. Periksa API dan masa berlaku indeks sebelum menjadikannya ID entitas.
- [`Name<T>`](../../Brutal.Collections/Name/Name.cs) memakai [`Namespace<T>`](../../Brutal.Collections/Name/Namespace.cs) untuk pooling string ID. Kesetaraan antar-`Name<T>` memakai identitas referensi string; gunakan pool yang sama untuk nama yang harus saling setara.

## Numerics dan Maths

Pakai `using Brutal.Numerics;`. Tipe bernama menurut komponen dan dimensinya: `float2/3/4`, `double2/3/4`, `int2/3/4`, `uint2/3/4`, `bool2/3/4`, dan varian integer lainnya. [`float3`](../../Brutal.Numerics/Vector/float3/float3.gen.cs) memiliki field `X/Y/Z`, constructor `(float x, float y, float z)`, indexer, serta view span. [`floatQuat`](../../Brutal.Numerics/Quaternion/floatQuat/floatQuat.gen.cs) adalah quaternion; [`float3x2`](../../Brutal.Numerics/Matrix/float3x2/float3x2.gen.cs) dan [`float4x4`](../../Brutal.Numerics/Matrix/float4x4/float4x4.gen.cs) adalah matriks; varian `double` juga tersedia.

Ini tipe BRUTAL tersendiri, bukan alias `System.Numerics.Vector3`, `Matrix4x4`, atau `Quaternion`. Cocokkan tipe pada signature binding dan konversikan komponen secara eksplisit ketika melintasi API .NET. Operasi tersedia sebagai method statis pada tipe, misalnya [`float3.Dot`, `Normalize`, `NormalizeOrZero`](../../Brutal.Numerics/Vector/float3/float3.math-metric.gen.cs) dan [`float3.Transform`](../../Brutal.Numerics/Vector/float3/float3.math.gen.cs). `Normalize` melempar `DivideByZeroException` untuk vektor nol; pilih `NormalizeOrZero` bila nol adalah input yang sah.

```csharp
using Brutal.Numerics;

var position = new float3(1, 2, 3);
System.Numerics.Vector3 dotNet = new(position.X, position.Y, position.Z);
float3 direction = float3.NormalizeOrZero(position);
float3 unchanged = float3.Transform(position, float4x4.Identity);

// Salah: float3 bukan alias Vector3; assignment ini tidak memiliki konversi.
// System.Numerics.Vector3 wrong = position;
```

`float4x4` menyimpan **baris** `X/Y/Z/W`; `float3.Transform(position, matrix)` menjumlahkan baris berbobot komponen lalu baris `W` sebagai translasi. Cocokkan layout, urutan perkalian, dan upload shader dengan kontrak renderer; nama tipe saja tidak menentukan layout buffer GPU.

Warna memakai vektor: [`float4.Rgba(r,g,b,a)`](../../Brutal.Numerics/Vector/float4/float4.color.gen.cs) menggunakan komponen 0–1; [`byte4`](../../Brutal.Numerics/Vector/byte4/byte4.color.gen.cs) menggunakan komponen byte. [`Color`](../../Brutal.Numerics/Color/Color.cs) adalah class statis preset, bukan struct warna dengan constructor RGBA. [`Color.Preset`](../../Brutal.Numerics/Color/Color.Preset.cs) menyediakan konversi ke `float3/4`, `byte3/4`, dan `ushort3/4`.

```csharp
using Brutal.Numerics;

float4 tint = Color.Red;
float4 translucent = float4.Rgba(1f, 0.5f, 0f, 0.75f);
// Salah: Color merupakan class statis.
// var wrong = new Color(255, 128, 0, 255);
```

[`Brutal.Maths.RandomExtensions`](../../Brutal.Maths/RandomExtensions.cs) menambahkan overload rentang `NextSingle/NextDouble` dan sampling titik pada `Random`, misalnya `NextSinglePointInSphere(radius: 2)`. Import `Brutal.Maths` untuk mengaktifkan extension; operasi vektor tetap berada di `Brutal.Numerics`.

## String8, view, dan builder

Gunakan `using Brutal.Strings;`. [`String8`](../../Brutal.Strings/String8/String8.cs) memiliki array byte tersendiri dengan terminator nol; constructor dari `ReadOnlySpan<char>` mengenkode UTF-8, constructor dari byte menyalin tanpa validasi encoding. `Length` adalah jumlah **byte**, bukan karakter Unicode. `ToString16()` menghasilkan `string` .NET. [`RefString8`](../../Brutal.Strings/String8/RefString8.cs) adalah `readonly ref struct` pinjaman ke string byte berteminator nol; ia mencari terminator ketika membaca `Chars`, sehingga sumber harus memiliki terminator dan masa hidup yang cukup.

```csharp
using Brutal.Strings;

var owned = new String8("Pemain");
RefString8 borrowed = owned; // backing String8 menyediakan terminator
RefString8 literal = "Pemain"u8;
string managed = owned.ToString16();

// Salah: constructor RefString8 tidak menambahkan terminator.
// byte[] raw = [65, 66];
// RefString8 invalid = raw; // pembacaan Chars dapat melewati buffer
```

Alias `Char8`, `String8View`, `String16`, dan `StringBuilder8` pada [`GlobalUsings.cs`](../../Brutal.Strings/GlobalUsings.cs) berlaku pada kompilasi modul itu, bukan otomatis pada game. Di kode game, gunakan `byte`, `ReadOnlySpan<byte>`, `string`, dan `Brutal.Strings.StringBuilder<byte>` atau deklarasikan alias sendiri.

[`Brutal.Strings.StringBuilder<TChar>`](../../Brutal.Strings/StringBuilder/StringBuilder.cs) dan [`ValueStringBuilder<TChar>`](../../Brutal.Strings/StringBuilder/ValueStringBuilder.cs) mendukung teks byte/char. `WrittenSpan` milik builder adalah view pinjaman yang harus diperoleh ulang setelah mutasi; salin/konversikan untuk menyimpannya lebih lama. Nama `StringBuilder` bertabrakan dengan `System.Text.StringBuilder`, jadi gunakan alias atau nama lengkap saat keduanya diimpor.

## Teks ImGui

[`ImString`](../../Brutal.ImGui/ImString/ImString.cs) berada di `Brutal.ImGuiApi`. Gunakan literal `u8` untuk label statis atau [`interpolasi ImString`](../../Brutal.ImGui/ImString/ImString.Interpolation.cs) untuk teks dinamis. Konversi UTF-16 dan interpolasi menulis ke arena bersama; view hanya berlaku sampai arena dibersihkan. [`ArenaStringBuilder`](../../Brutal.ImGui/ImString/ArenaStringBuilder.cs) berkapasitas tetap dan melempar ketika penuh. Integrasi frame game perlu memanggil `ImString.SharedStorage.Clear()` sesudah seluruh pemakai teks frame selesai; simpan data persisten sebagai `string`/`String8`, lalu bentuk `ImString` setiap frame. Akses arena bersama diserialisasi pada thread UI.

[`ImInputString`](../../Brutal.ImGui/ImString/ImInputString.cs) adalah buffer byte mutable di heap untuk field input yang bertahan antar-frame; kapasitas mencakup terminator. [`ImGui.InputText`](../../Brutal.ImGui/Manual/Functions/ImGui.cs) menerima buffer ini dan memperbarui `Length` ketika isinya berubah.

```csharp
using Brutal.ImGuiApi;

// Simpan buffer ini di state UI, bukan buat ulang setiap frame.
var playerName = new ImInputString(128, "Pemain"u8);
// Pada frame ImGui aktif, dengan konteks/jendela yang sudah disiapkan:
unsafe { ImGui.InputText("Nama"u8, playerName); }
ImString label = $"Nama: {playerName.Value}";
ImGui.Text(label); // gunakan pada frame ini

// Salah: arena Clear mengakhiri masa berlaku teks dinamis sebelumnya.
// ImString.SharedStorage.Clear();
// ImGui.Text(label); // label dapat tertimpa teks berikutnya
```

Overload `ImString(ReadOnlySpan<byte>)` meminjam byte tanpa menyalin/menambahkan terminator. Untuk API native yang membaca C string, gunakan literal `u8`, hasil arena, atau buffer berteminator nol; panjang span saja tidak membatasi pembacaan native. [`ImGui.Text`](../../Brutal.ImGui/Manual/Functions/ImGui.Text.cs) sendiri meneruskan pointer ke `PInvoke.TextUnformatted` dengan `text_end = null`.

## Logging dan lokalisasi

Inisialisasikan [`LogSystem.Initialize(Action<ILoggingBuilder>)`](../../Brutal.Logging/LogSystem.cs) dengan provider logging proyek sebelum pemakaian. [`LogCategory.Make(name)`](../../Brutal.Logging/LogCategory.cs) membuat/mengambil kategori; method levelnya `Trace`, `Debug`, `Info`, `Warning`, `Error`, dan `Critical`. [`DefaultCategory.Log`](../../Brutal.Logging/DefaultCategory.cs) tersedia jika kategori khusus tidak diperlukan.

[`LogString<TLevel>`](../../Brutal.Logging/LogString.cs) adalah interpolated-string handler internal untuk compiler. Berikan interpolasi langsung ke method log supaya evaluasi nilai dan pembangunan teks dilewati ketika level/kategori nonaktif. Hindari membangun `LogString` sendiri.

```csharp
using Brutal.Logging;
using Microsoft.Extensions.Logging;

// Provider dipilih aplikasi; AddConsole memerlukan provider console .NET.
LogSystem.Initialize(builder => builder.AddConsole());
var log = LogCategory.Make("Gameplay");
log.Info($"Pemain masuk: {42}"); // handler mengecek enablement

// Salah untuk hot path: string sudah dibentuk sebelum filter log bekerja.
// string eager = $"Pemain masuk: {42}";
// log.Debug(eager);
```

[`LString`](../../Brutal.Localization/LString.cs), namespace `Brutal.Localization`, menyimpan ID, template, dan teks lokal. [`LStrings`](../../Brutal.Localization/LStrings.cs) adalah facade atas instance [`LocalizedStrings`](../../Brutal.Localization/LocalizedStrings.cs) bersama yang disimpan secara privat: `Create` menolak ID duplikat; `GetOrCreate` mengambil yang sudah terdaftar. Daftarkan teks sekali, lalu format saat ditampilkan. `SetLocalizedStringProvider` memasang delegate pencari terjemahan dan memperbarui instance terdaftar; tanpa terjemahan teks kembali ke template.

Template memakai placeholder seperti `{Name}` sesuai [`DefaultOptions.ReplacementPattern`](../../Brutal.Localization/DefaultOptions.cs). Terjemahan harus mempertahankan nama dan jumlah placeholder; [`Format(...)`](../../Brutal.Localization/LString.Format.gen.cs) memvalidasi jumlah argumen dan memetakan urutan argumen dari template, walau terjemahan mengubah urutan placeholder.

```csharp
using Brutal.Localization;
using Brutal.ImGuiApi;

var greeting = LStrings.GetOrCreate("ui.greeting", "Halo {Name}");
greeting.Localize("Selamat datang {Name}");
ImString label = greeting.Format("Rafi"); // format dahulu, lalu UTF-8 UI

// Salah: konversi implisit LString -> ImString memanggil Format() tanpa argumen.
// ImString invalid = greeting; // jumlah replacement tidak cocok
// greeting.Localize("Welcome {Player}"); // nama placeholder tidak cocok
```
