# Fisika, audio, konsol, dan job

Baca bagian yang sesuai sebelum memasang callback fisika, mengelola audio, membuat konsol debug, atau membagi pekerjaan ke worker. Contoh adalah fragmen penggunaan; library native dan inisialisasi aplikasi harus sudah tersedia. Contoh bukan game lengkap dan belum diuji lewat build pada snapshot ini.

## Jolt: handle native dan callback C#

Namespace-nya `Brutal.JoltApi`. Tipe biasa seperti [PhysicsSystem](../../Brutal.Jolt/Generated/Classes/PhysicsSystem.gen.cs) dan [BoxShape](../../Brutal.Jolt/Generated/Classes/BoxShape.gen.cs) merupakan `readonly struct` pembungkus handle native, meskipun berada di folder `Classes/`. Buat lewat `Create(...)`. Ikuti keterangan **owned** atau **borrowed** pada setiap hasil API: misalnya `PhysicsSystem.GetBodyInterface()` meminjam dari sistem, sehingga pemanggil tidak memiliki lifetime-nya. Menyalin struct tidak menggandakan objek native.

`Managed...` adalah **kelas abstrak untuk implementasi callback C#**, seperti [ManagedBodyActivationListener](../../Brutal.Jolt/Generated/Classes/ManagedBodyActivationListener.gen.cs), [ManagedContactListener](../../Brutal.Jolt/Generated/Classes/ManagedContactListener.gen.cs), dan [ManagedObjectLayerPairFilter](../../Brutal.Jolt/Generated/Classes/ManagedObjectLayerPairFilter.gen.cs). Turunkan kelas, lalu `override` virtual yang dibutuhkan. Proxy native meneruskan panggilan ke instance C#; konversi implisit menghasilkan handle dasar yang diterima API Jolt.

```csharp
using Brutal.JoltApi;
using System.Threading;

sealed class ActivationCounter : ManagedBodyActivationListener
{
    private int _activated;
    public int Activated => Volatile.Read(ref _activated);

    protected override void OnBodyActivated(uint bodyID, ulong bodyUserData)
        => Interlocked.Increment(ref _activated);
}
```

Benar: pasang instance pada `system` yang sudah dibuat/diinisialisasi; hentikan pekerjaan fisika sebelum melepasnya. Urutan lifetime berikut dijalankan saat fisika tidak sedang diperbarui:

```csharp
using var listener = new ActivationCounter();
system.SetBodyActivationListener(listener);
try
{
    // Jalankan pembaruan fisika aplikasi dan tunggu worker selesai.
    if (listener.TakeCallbackException() is { } error)
        throw error;
}
finally
{
    system.SetBodyActivationListener(default); // Lepas sebelum listener.Dispose().
}
```

Proxy menahan `GCHandle` kuat ke dirinya; GC tidak akan membersihkannya. Panggil `Dispose()` eksplisit sesudah sistem berhenti merujuknya, termasuk filter yang diteruskan ke `PhysicsSystem.Init(...)`. `Handle` milik proxy; lifetime-nya mengikuti instance managed. Callback dapat berjalan pada worker Jolt: gunakan sinkronisasi dan hormati batas callback; pada aktivasi body, body terkunci dan callback tidak boleh menulis/mengaktifkan/menonaktifkan body. Exception ditangkap proxy; periksa `LastCallbackException` atau ambil/reset melalui `TakeCallbackException()` setelah pembaruan selesai.

Salah: mendeklarasikan `protected new void OnBodyActivated(...)` menyembunyikan virtual, sehingga thunk tidak dipasang. Gunakan `override` dengan signature persis. Salah juga melepas listener ketika masih terpasang atau mengandalkan GC untuk membersihkannya. Implementasi dan aturan lifetime ada di [ManagedBodyActivationListener](../../Brutal.Jolt/Generated/Classes/ManagedBodyActivationListener.gen.cs).

## Box2D: world 2D

Namespace `Brutal.Box2DApi`. Mulai dari default native yang disediakan, bukan struct definisi berisi nol. [B2WorldDef.Default](../../Brutal.Box2D/Structs.cs) memanggil default native; [Box2D.CreateWorld](../../Brutal.Box2D/Box2D.gen.cs) menerima `in B2WorldDef`. API instance [B2World](../../Brutal.Box2D/Structs.gen.cs) menyediakan `Step(float timeStep, int subStepCount)` dan `Dispose()`.

```csharp
using Brutal.Box2DApi;

var definition = B2WorldDef.Default;
using var world = Box2D.CreateWorld(in definition);
world.Step(1f / 60f, 4); // Satu langkah; aplikasi mengatur akumulator fixed-step.
```

`B2World`, `B2Body`, dan `B2Shape` adalah ID opaque, bukan objek managed pemilik resource. `IsNull()` hanya memeriksa nilai ID; `world.IsValid()` melakukan validasi native. Hapus world sekali oleh pemiliknya, lalu hentikan penggunaan salinan ID dan objek turunannya. `GetBodyEvents()`, `GetSensorEvents()`, dan `GetContactEvents()` mengembalikan data sementara untuk langkah saat ini; proses/salin data yang diperlukan sebelum langkah berikutnya. Lihat [implementasi ID manual](../../Brutal.Box2D/Structs.cs) dan [wrapper generated](../../Brutal.Box2D/Structs.gen.cs).

Salah: `var definition = default(B2WorldDef);` sebagai pengganti konfigurasi awal; gunakan `B2WorldDef.Default` lalu ubah field yang diperlukan. Salah pula membuat dua pemilik `using` dari salinan `B2World` yang sama.

## Audio: FMOD, OpenAL, dan SDL Mixer

Pilih API sesuai backend audio aplikasi. Modul-modul ini mengekspos pengelolaan resource, bukan komponen audio gameplay otomatis.

### FMOD

Core memakai `Brutal.FmodApi`, Studio memakai `Brutal.FmodApi.Studio`. [Fmod.SystemCreate(uint)](../../Brutal.Fmod/FmodCoreApi/Generated/Libraries/Fmod.gen.cs) menghasilkan `FmodSystem`; extension [Init, Update, Release](../../Brutal.Fmod/FmodCoreApi/Generated/Libraries/Fmod.FmodSystem.gen.cs) mengelola sistem. Versi header tersedia sebagai [Fmod.VERSION](../../Brutal.Fmod/FmodCoreApi/Manual/Fmod.cs).

```csharp
using Brutal.FmodApi;

var audio = Fmod.SystemCreate((uint)Fmod.VERSION);
try
{
    audio.Init(64, InitFlags.Normal, 0);
    audio.Update(); // Panggil pada loop aplikasi selama sistem aktif.
}
finally
{
    audio.Release();
}
```

Untuk suara, tersedia `audio.CreateSound(RefString8, Mode, in CreateSoundExInfo)`, `CreateStream(...)`, dan `PlaySound(Sound, ChannelGroup, Bool32)`. Siapkan `CreateSoundExInfo.Default`, yang mengisi `CbSize`; `default(CreateSoundExInfo)` melewati constructor tersebut. Lepas sound yang dimiliki memakai `sound.Release()` setelah selesai digunakan. Signature berada di [FmodSystem extensions](../../Brutal.Fmod/FmodCoreApi/Generated/Libraries/Fmod.FmodSystem.gen.cs), [Sound extensions](../../Brutal.Fmod/FmodCoreApi/Generated/Libraries/Fmod.Sound.gen.cs), dan [CreateSoundExInfo](../../Brutal.Fmod/FmodCoreApi/Manual/Structs/CreateSoundExInfo.cs). Untuk bank/event Studio, baca [FmodStudioSystem](../../Brutal.Fmod/FmodStudioApi/Generated/Libraries/FmodStudio.FmodStudioSystem.gen.cs) dan [EventInstance](../../Brutal.Fmod/FmodStudioApi/Generated/Libraries/FmodStudio.EventInstance.gen.cs).

Benar: periksa hasil `Try...` yang mengembalikan `Result` dengan `.Check()` atau tangani nilainya sendiri. Wrapper tanpa `Try` sudah memanggil `.Check()`. Salah: mengabaikan `Result` lalu memakai handle keluaran seolah operasi berhasil. [ResultExtensions.Check](../../Brutal.Fmod/FmodCoreApi/Manual/ResultExtensions.cs) melempar `FmodResultException` saat hasil bukan `Result.Ok`.

### OpenAL

Namespace `Brutal.OpenAlApi`. Buka [Alc.OpenDevice()](../../Brutal.OpenAl/Manual/Alc.Wrapper.cs), buat [Device.CreateContext()](../../Brutal.OpenAl/Manual/Device.cs), lalu [Context.MakeCurrent()](../../Brutal.OpenAl/Manual/Context.cs) sebelum operasi `Al`. Lepas source/buffer saat context masih current; detach context sebelum membuang context, lalu device.

```csharp
using Brutal.OpenAlApi;

using var device = Alc.OpenDevice();
using var context = device.CreateContext();
context.MakeCurrent();
try
{
    var buffer = Al.GenBuffer();
    try
    {
        Al.ThrowIfError();
        // buffer.SetData(format, pcmBytes, sampleRate): data PCM yang sudah didekode.
    }
    finally
    {
        buffer.Delete(); // Buffer contoh ini belum diikat ke source.
    }
}
finally
{
    Context.MakeCurrent(null);
}
```

[AlBuffer.SetData](../../Brutal.OpenAl/Manual/AlBuffer.cs) menerima format, `ReadOnlySpan<byte>` dan sample rate; ini unggahan PCM, bukan pembaca file WAV/MP3. [Al.GenBuffer/GenSource](../../Brutal.OpenAl/Generated/Al/Al.ObjectNames.gen.cs) membuat nama native. [AlBuffer](../../Brutal.OpenAl/Generated/Al/AlBuffer.ObjectName.gen.cs) dan [AlSource](../../Brutal.OpenAl/Generated/Al/AlSource.ObjectName.gen.cs) adalah struct nama yang harus dihapus tepat sekali; `new AlBuffer(1)` hanya membungkus angka, bukan alokasi. Lepas ikatan/pemakaian buffer oleh source sebelum menghapus buffer. Untuk parameter berbasis enum, import namespace opt-in `Brutal.OpenAlApi.Params`.

Benar: gunakan [Al.ThrowIfError()](../../Brutal.OpenAl/Manual/Errors.cs) untuk memeriksa operasi AL; wrapper tidak otomatis memeriksa setiap panggilan AL. Salah: membuat operasi AL pada worker tanpa menyiapkan context yang berlaku. Jika memakai context per-thread, periksa dukungan `ALC_EXT_thread_local_context` sebelum memakai `Context.MakeThreadCurrent(...)`, dan detach lewat `Context.MakeThreadCurrent(null)` pada thread itu sebelum membuang context.

### SDL Mixer

Namespace `Brutal.SdlApi.Mixer` memakai API SDL_mixer 3. Mulai dengan `SdlMixer.Init()`, lalu `CreateMixerDevice(uint, SdlAudioSpec*)` untuk output perangkat atau `CreateMixer(SdlAudioSpec*)` untuk mixing tanpa perangkat. `LoadAudio(MixMixer, string, bool)` menghasilkan `MixAudio`; `CreateTrack(MixMixer)` menghasilkan track. Signature dan kontrak pelepasan ada di [wrapper SDL Mixer](../../Brutal.Sdl.Mixer/Generated/Libraries/SdlMixer.Ergonomic.gen.cs).

`DestroyMixer(mixer)` menghancurkan seluruh track/group miliknya; hentikan penggunaan semua salinannya dan jangan memanggil `DestroyTrack`/`DestroyGroup` lagi sesudah itu. `MixAudio` dapat dibagi antar-mixer dan tetap harus dilepas pemiliknya melalui `DestroyAudio(audio)`. Operasi ini mengurangi reference count; sesudahnya pemanggil menganggap handle tidak valid walaupun track masih menahan referensi internal. Untuk mixer perangkat, jalankan `DestroyMixer` pada main thread; mixer tanpa perangkat mengizinkannya dari thread mana pun.

[Callback managed](../../Brutal.Sdl.Mixer/Manual/Libraries/SdlMixer.Callbacks.cs) berjalan pada thread audio dan ditahan oleh `GCHandle`. Bersihkan registrasi dengan setter callback yang sama dan argumen `null` selagi track/group/mixer masih hidup; penghancuran objek native tidak membersihkan registry managed. Batasi pemakaian `Span<float>` PCM ke callback tersebut. Panggil [ThrowPendingCallbackExceptions()](../../Brutal.Sdl.Mixer/SdlMixer.cs) secara berkala untuk meneruskan exception callback ke thread aplikasi. Setelah resource selesai dipakai, akhiri dengan [SdlMixer.Quit()](../../Brutal.Sdl.Mixer/Manual/Libraries/SdlMixer.Init.cs): wrapper menunggu native `Quit` kembali sebelum membebaskan sisa handle callback dan membersihkan exception tersimpan.

## ImGui: konsol dan command

Namespace yang benar ialah `Brutal.ImGuiApi.Abstractions`. [ConsoleWindow](../../Brutal.ImGui.Abstractions/ConsoleWindow/ConsoleWindow.cs) sudah membuat `TerminalInterface` pada constructor; `Show()` membuka konsol dan `Draw(float2 viewportPosition, float2 viewportSize, float unscaledDeltaTime, bool noFade = false)` menggambarnya. Panggil `Draw` dari frame ImGui aktif setelah aplikasi menyiapkan context, font, backend input, dan renderer. Hubungkan input aplikasi ke `HandleInputAction(ConsoleWindow.Actions.Up/Down/Submit/Toggle)`.

```csharp
using Brutal.ImGuiApi.Abstractions;

sealed class DebugCommands
{
    public int SpawnCount { get; private set; }

    [TerminalAction("spawn", "Atur jumlah spawn debug")]
    private void Spawn(int count = 1) => SpawnCount = count;
}
```

Fragmen setup, terpisah dari deklarasi kelas di atas:

```csharp
var console = new ConsoleWindow();
var commands = new DebugCommands();
console.Terminal.RegisterObject(commands);
console.Print("Konsol siap");
console.Show();
console.Terminal.Execute("spawn 3");
// Dalam frame ImGui: console.Draw(viewportPosition, viewportSize, deltaTime);
// Saat pemilik command dilepas:
console.Terminal.UnregisterObject(commands);
```

[TerminalInterface](../../Brutal.ImGui.Abstractions/ConsoleWindow/Terminal/TerminalInterface.cs) juga menerima `RegisterCommand(Delegate, object? inObject = null)`, tetapi method delegate tetap wajib memiliki [TerminalActionAttribute](../../Brutal.ImGui.Abstractions/ConsoleWindow/Terminal/TerminalCallbackAttribute.cs). Salah: `RegisterCommand((Action)(() => ...))` tanpa atribut; registrasi akan melempar exception. `RegisterObject` mencari method beratribut dan membentuk `Action`, sehingga pilih callback `void` dengan parameter yang didukung. Gunakan nama command unik agar tidak bertabrakan antarregistrasi.

[Parser command](../../Brutal.ImGui.Abstractions/ConsoleWindow/Terminal/TerminalCommand.cs) mendukung tipe primitif, `string`, dan `DateTime`, termasuk nilai parameter default. `ArgParseMode.Unparsed` membutuhkan tepat satu parameter `string`; vektor bukan tipe argumen yang didukung. Terminal menyediakan `help`, `clear`, dan pemisahan beberapa command dengan `;`. `Execute` menangani `TerminalInterfaceException`; exception gameplay dari callback bukan hasil boolean kegagalan yang dijamin tertangani.

`ConsoleWindow` juga merupakan `ILoggerProvider` lewat `CreateLogger(...)`. `Print`/`OnLog` menampung pesan dengan lock, lalu `Draw` memanggil `SwapBuffers` untuk menerapkannya pada UI. Gunakan jalur pesan ini dari worker; jalankan rendering dan registrasi/eksekusi command pada thread aplikasi yang mengelola UI. Lihat [ConsoleWindow dan logger](../../Brutal.ImGui.Abstractions/ConsoleWindow/ConsoleWindow.cs).

## Concurrency: batch job paralel

Implementasikan [IJob.Run()](../../Brutal.Concurrency/Jobs/IJob.cs) di namespace `Brutal.Concurrency.Jobs`. [JobScheduler](../../Brutal.Concurrency/Jobs/JobScheduler.cs) membagi job menurut estimasi biaya ke beberapa `JobRunner`, yang memakai .NET ThreadPool. Estimasi biaya adalah bobot relatif untuk distribusi, bukan deadline.

```csharp
using Brutal.Concurrency.Jobs;

sealed class SquareJob(int[] output, int index) : IJob
{
    public void Run() => output[index] = index * index;
}
```

Fragmen pemanggil:

```csharp
var output = new int[8];
var scheduler = new JobScheduler(2, "Precompute");
for (int i = 0; i < output.Length; ++i)
    scheduler.Queue(new SquareJob(output, i), timeCostEstimate: 1);
scheduler.ExecuteJobs();
scheduler.Wait();
// Hasil output sekarang dapat dikonsumsi pemanggil.
```

Queue seluruh batch saat runner idle, panggil `ExecuteJobs()`, lalu tunggu selesai sebelum memakai hasil atau mengantre batch berikutnya. `Queue`/`ExecuteJobs` menolak pekerjaan saat runner sibuk; kelola pemanggilan scheduler dari satu thread pengendali. Salah: memanggil `Queue` sesudah `ExecuteJobs()` dengan asumsi job selalu dapat ditambahkan sewaktu worker berjalan. Contoh membagi satu elemen output per job agar tidak ada penulisan bersama pada elemen yang sama.

[JobRunner.Wait()](../../Brutal.Concurrency/Jobs/JobRunner.cs) melakukan busy-wait, sehingga cocok hanya untuk sinkronisasi pendek yang memang direncanakan. `AbortJobs(waitForCompletion: true)` menunggu pembatalan batch, tetapi tidak menghentikan `Run()` yang sedang berjalan. `JobRunner` tidak menangkap exception `Run()` menjadi hasil seperti `Task`; tangani kegagalan yang diharapkan di dalam job dan simpan hasil/error untuk dibaca pemanggil. Jangan menggantungkan shutdown resource pada pembatalan paksa job.

Untuk metrik konsisten, pakai `JobScheduler.GetMetrics(Span<JobRunnerMetrics>)` dengan kapasitas minimal sebanyak runner. Varian `GetMetricsUnsafe` tidak menjamin snapshot atomik selama worker berjalan. Scheduler ini bukan dependency graph otomatis: jadwalkan tahap yang bergantung pada hasil setelah batch prasyarat selesai. Sumber: [JobScheduler](../../Brutal.Concurrency/Jobs/JobScheduler.cs) dan [JobRunner](../../Brutal.Concurrency/Jobs/JobRunner.cs).
