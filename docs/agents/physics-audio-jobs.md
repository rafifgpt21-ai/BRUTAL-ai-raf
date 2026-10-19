# Physics, audio, consoles, and jobs

Read the relevant section before adding physics callbacks, managing audio, creating a debug console, or distributing work across workers. Examples are usage fragments; native libraries and application initialization must already be available. They are not complete games and have not been built in this snapshot.

## Jolt: native handles and C# callbacks

The namespace is `Brutal.JoltApi`. Regular types such as [PhysicsSystem](../../Brutal.Jolt/Generated/Classes/PhysicsSystem.gen.cs) and [BoxShape](../../Brutal.Jolt/Generated/Classes/BoxShape.gen.cs) are `readonly struct` wrappers around native handles, despite residing in `Classes/`. Create them through `Create(...)`. Follow the **owned** or **borrowed** annotations on each API result: for example, `PhysicsSystem.GetBodyInterface()` borrows from the system, so the caller does not own its lifetime. Copying a struct does not duplicate the native object.

`Managed...` types are **abstract classes for implementing callbacks in C#**, such as [ManagedBodyActivationListener](../../Brutal.Jolt/Generated/Classes/ManagedBodyActivationListener.gen.cs), [ManagedContactListener](../../Brutal.Jolt/Generated/Classes/ManagedContactListener.gen.cs), and [ManagedObjectLayerPairFilter](../../Brutal.Jolt/Generated/Classes/ManagedObjectLayerPairFilter.gen.cs). Derive a class and `override` the required virtual methods. A native proxy forwards calls to the C# instance; implicit conversion provides the base handle accepted by Jolt APIs.

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

Correct: attach the instance to an already created/initialized `system`; stop physics work before releasing it. Run this lifetime sequence while physics is not updating:

```csharp
using var listener = new ActivationCounter();
system.SetBodyActivationListener(listener);
try
{
    // Update application physics and wait for workers to finish.
    if (listener.TakeCallbackException() is { } error)
        throw error;
}
finally
{
    system.SetBodyActivationListener(default); // Detach before listener.Dispose().
}
```

The proxy holds a strong `GCHandle` to itself; the GC will not clean it up. Explicitly call `Dispose()` after the system stops referring to it, including filters passed to `PhysicsSystem.Init(...)`. The proxy owns `Handle`; its lifetime follows the managed instance. Callbacks may run on Jolt workers: synchronize shared data and respect callback restrictions; during body activation, the body is locked and the callback must not write to, activate, or deactivate bodies. The proxy catches exceptions; inspect `LastCallbackException` or retrieve/reset it through `TakeCallbackException()` after the update finishes.

Incorrect: declaring `protected new void OnBodyActivated(...)` hides the virtual method, so the thunk is not installed. Use `override` with the exact signature. Releasing a listener while still attached or relying on the GC for cleanup is also incorrect. See [ManagedBodyActivationListener](../../Brutal.Jolt/Generated/Classes/ManagedBodyActivationListener.gen.cs) for implementation and lifetime rules.

## Box2D: a 2D world

Use `Brutal.Box2DApi`. Start with the supplied native defaults, rather than a zeroed definition struct. [B2WorldDef.Default](../../Brutal.Box2D/Structs.cs) calls the native default function; [Box2D.CreateWorld](../../Brutal.Box2D/Box2D.gen.cs) accepts `in B2WorldDef`. The [B2World](../../Brutal.Box2D/Structs.gen.cs) instance API provides `Step(float timeStep, int subStepCount)` and `Dispose()`.

```csharp
using Brutal.Box2DApi;

var definition = B2WorldDef.Default;
using var world = Box2D.CreateWorld(in definition);
world.Step(1f / 60f, 4); // One step; the application manages a fixed-step accumulator.
```

`B2World`, `B2Body`, and `B2Shape` are opaque IDs, not managed resource-owning objects. `IsNull()` only checks the ID value; `world.IsValid()` performs native validation. The owner destroys the world once, then stops using copies of the ID and its child objects. `GetBodyEvents()`, `GetSensorEvents()`, and `GetContactEvents()` return temporary data for the current step; process/copy the necessary data before the next step. See the [manual ID implementation](../../Brutal.Box2D/Structs.cs) and [generated wrapper](../../Brutal.Box2D/Structs.gen.cs).

Incorrect: using `var definition = default(B2WorldDef);` as the initial configuration; use `B2WorldDef.Default`, then change the required fields. Creating two `using` owners from copies of the same `B2World` is also incorrect.

## Audio: FMOD, OpenAL, and SDL Mixer

Choose an API for the application's audio backend. These modules expose resource management rather than automatic gameplay audio components.

### FMOD

Core uses `Brutal.FmodApi`; Studio uses `Brutal.FmodApi.Studio`. [Fmod.SystemCreate(uint)](../../Brutal.Fmod/FmodCoreApi/Generated/Libraries/Fmod.gen.cs) returns a `FmodSystem`; the [Init, Update, Release](../../Brutal.Fmod/FmodCoreApi/Generated/Libraries/Fmod.FmodSystem.gen.cs) extensions manage the system. The header version is available as [Fmod.VERSION](../../Brutal.Fmod/FmodCoreApi/Manual/Fmod.cs).

```csharp
using Brutal.FmodApi;

var audio = Fmod.SystemCreate((uint)Fmod.VERSION);
try
{
    audio.Init(64, InitFlags.Normal, 0);
    audio.Update(); // Call in the application loop while the system is active.
}
finally
{
    audio.Release();
}
```

For sounds, use `audio.CreateSound(RefString8, Mode, in CreateSoundExInfo)`, `CreateStream(...)`, and `PlaySound(Sound, ChannelGroup, Bool32)`. Prepare `CreateSoundExInfo.Default`, which populates `CbSize`; `default(CreateSoundExInfo)` bypasses that constructor. Release owned sounds with `sound.Release()` when finished. Signatures are in the [FmodSystem extensions](../../Brutal.Fmod/FmodCoreApi/Generated/Libraries/Fmod.FmodSystem.gen.cs), [Sound extensions](../../Brutal.Fmod/FmodCoreApi/Generated/Libraries/Fmod.Sound.gen.cs), and [CreateSoundExInfo](../../Brutal.Fmod/FmodCoreApi/Manual/Structs/CreateSoundExInfo.cs). For Studio banks/events, read [FmodStudioSystem](../../Brutal.Fmod/FmodStudioApi/Generated/Libraries/FmodStudio.FmodStudioSystem.gen.cs) and [EventInstance](../../Brutal.Fmod/FmodStudioApi/Generated/Libraries/FmodStudio.EventInstance.gen.cs).

Correct: check `Try...` calls returning `Result` with `.Check()`, or handle the value yourself. Wrappers without `Try` already call `.Check()`. Incorrect: ignoring `Result` and using the output handle as though the operation succeeded. [ResultExtensions.Check](../../Brutal.Fmod/FmodCoreApi/Manual/ResultExtensions.cs) throws `FmodResultException` when the result is not `Result.Ok`.

### OpenAL

Use `Brutal.OpenAlApi`. Open [Alc.OpenDevice()](../../Brutal.OpenAl/Manual/Alc.Wrapper.cs), create a context with [Device.CreateContext()](../../Brutal.OpenAl/Manual/Device.cs), then call [Context.MakeCurrent()](../../Brutal.OpenAl/Manual/Context.cs) before `Al` operations. Release sources/buffers while the context is still current; detach the context before disposing it, then dispose the device.

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
        // buffer.SetData(format, pcmBytes, sampleRate): already decoded PCM data.
    }
    finally
    {
        buffer.Delete(); // This example buffer has not been attached to a source.
    }
}
finally
{
    Context.MakeCurrent(null);
}
```

[AlBuffer.SetData](../../Brutal.OpenAl/Manual/AlBuffer.cs) accepts a format, `ReadOnlySpan<byte>`, and a sample rate; it uploads PCM rather than reading WAV/MP3 files. [Al.GenBuffer/GenSource](../../Brutal.OpenAl/Generated/Al/Al.ObjectNames.gen.cs) creates native names. [AlBuffer](../../Brutal.OpenAl/Generated/Al/AlBuffer.ObjectName.gen.cs) and [AlSource](../../Brutal.OpenAl/Generated/Al/AlSource.ObjectName.gen.cs) are name structs that must be deleted exactly once; `new AlBuffer(1)` only wraps a number, without allocating. Detach a buffer from sources and stop their use of it before deleting the buffer. For enum-based parameters, import the opt-in `Brutal.OpenAlApi.Params` namespace.

Correct: use [Al.ThrowIfError()](../../Brutal.OpenAl/Manual/Errors.cs) to check AL operations; wrappers do not automatically check every AL call. Incorrect: performing AL operations on a worker without setting up an applicable context. When using per-thread contexts, check for `ALC_EXT_thread_local_context` support before using `Context.MakeThreadCurrent(...)`, and detach with `Context.MakeThreadCurrent(null)` on that thread before disposing the context.

### SDL Mixer

The `Brutal.SdlApi.Mixer` namespace uses the SDL_mixer 3 API. Start with `SdlMixer.Init()`, then use `CreateMixerDevice(uint, SdlAudioSpec*)` for device output or `CreateMixer(SdlAudioSpec*)` for mixing without a device. `LoadAudio(MixMixer, string, bool)` returns `MixAudio`; `CreateTrack(MixMixer)` returns a track. Signatures and release contracts are in the [SDL Mixer wrapper](../../Brutal.Sdl.Mixer/Generated/Libraries/SdlMixer.Ergonomic.gen.cs).

`DestroyMixer(mixer)` destroys all its tracks/groups; stop using all copies and do not call `DestroyTrack`/`DestroyGroup` on them afterward. `MixAudio` can be shared across mixers and must still be released by its owner through `DestroyAudio(audio)`. This operation decrements its reference count; afterward, the caller treats the handle as invalid even if a track still holds an internal reference. For device mixers, call `DestroyMixer` on the main thread; mixers without a device allow it from any thread.

[Managed callbacks](../../Brutal.Sdl.Mixer/Manual/Libraries/SdlMixer.Callbacks.cs) run on the audio thread and are retained by `GCHandle`. Clear registrations through the same callback setter with a `null` argument while the track/group/mixer is still alive; native object destruction does not clear the managed registry. Limit PCM `Span<float>` use to that callback. Call [ThrowPendingCallbackExceptions()](../../Brutal.Sdl.Mixer/SdlMixer.cs) periodically to rethrow callback exceptions on the application thread. After resources are no longer in use, finish with [SdlMixer.Quit()](../../Brutal.Sdl.Mixer/Manual/Libraries/SdlMixer.Init.cs): the wrapper waits for native `Quit` to return before freeing remaining callback handles and clearing stored exceptions.

## ImGui: consoles and commands

The correct namespace is `Brutal.ImGuiApi.Abstractions`. [ConsoleWindow](../../Brutal.ImGui.Abstractions/ConsoleWindow/ConsoleWindow.cs) creates a `TerminalInterface` in its constructor; `Show()` opens the console, and `Draw(float2 viewportPosition, float2 viewportSize, float unscaledDeltaTime, bool noFade = false)` draws it. Call `Draw` during an active ImGui frame after the application has set up the context, fonts, input backend, and renderer. Connect application input to `HandleInputAction(ConsoleWindow.Actions.Up/Down/Submit/Toggle)`.

```csharp
using Brutal.ImGuiApi.Abstractions;

sealed class DebugCommands
{
    public int SpawnCount { get; private set; }

    [TerminalAction("spawn", "Set the debug spawn count")]
    private void Spawn(int count = 1) => SpawnCount = count;
}
```

Setup fragment, separate from the class declaration above:

```csharp
var console = new ConsoleWindow();
var commands = new DebugCommands();
console.Terminal.RegisterObject(commands);
console.Print("Console ready");
console.Show();
console.Terminal.Execute("spawn 3");
// During an ImGui frame: console.Draw(viewportPosition, viewportSize, deltaTime);
// When releasing the command owner:
console.Terminal.UnregisterObject(commands);
```

[TerminalInterface](../../Brutal.ImGui.Abstractions/ConsoleWindow/Terminal/TerminalInterface.cs) also accepts `RegisterCommand(Delegate, object? inObject = null)`, but the delegate's method must still have a [TerminalActionAttribute](../../Brutal.ImGui.Abstractions/ConsoleWindow/Terminal/TerminalCallbackAttribute.cs). Incorrect: `RegisterCommand((Action)(() => ...))` without an attribute; registration throws. `RegisterObject` finds attributed methods and creates `Action` delegates, so choose `void` callbacks with supported parameters. Use unique command names to avoid registration conflicts.

The [command parser](../../Brutal.ImGui.Abstractions/ConsoleWindow/Terminal/TerminalCommand.cs) supports primitive types, `string`, and `DateTime`, including default parameter values. `ArgParseMode.Unparsed` requires exactly one `string` parameter; vectors are not supported argument types. The terminal provides `help`, `clear`, and multiple commands separated by `;`. `Execute` handles `TerminalInterfaceException`; gameplay exceptions from callbacks are not guaranteed to be handled as a boolean failure result.

`ConsoleWindow` also implements `ILoggerProvider` through `CreateLogger(...)`. `Print`/`OnLog` buffers messages under a lock, then `Draw` calls `SwapBuffers` to apply them to the UI. Use this message path from workers; run rendering and command registration/execution on the application thread managing the UI. See [ConsoleWindow and logging](../../Brutal.ImGui.Abstractions/ConsoleWindow/ConsoleWindow.cs).

## Concurrency: parallel job batches

Implement [IJob.Run()](../../Brutal.Concurrency/Jobs/IJob.cs) in `Brutal.Concurrency.Jobs`. [JobScheduler](../../Brutal.Concurrency/Jobs/JobScheduler.cs) distributes jobs by estimated cost across `JobRunner` instances that use the .NET ThreadPool. Cost estimates are relative distribution weights, not deadlines.

```csharp
using Brutal.Concurrency.Jobs;

sealed class SquareJob(int[] output, int index) : IJob
{
    public void Run() => output[index] = index * index;
}
```

Caller fragment:

```csharp
var output = new int[8];
var scheduler = new JobScheduler(2, "Precompute");
for (int i = 0; i < output.Length; ++i)
    scheduler.Queue(new SquareJob(output, i), timeCostEstimate: 1);
scheduler.ExecuteJobs();
scheduler.Wait();
// The caller can now consume output.
```

Queue the entire batch while runners are idle, call `ExecuteJobs()`, then wait for completion before using the results or queuing the next batch. `Queue`/`ExecuteJobs` reject work while runners are busy; control the scheduler from a single thread. Incorrect: calling `Queue` after `ExecuteJobs()` and assuming jobs can always be added while workers run. The example assigns one output element per job to avoid concurrent writes to the same element.

[JobRunner.Wait()](../../Brutal.Concurrency/Jobs/JobRunner.cs) busy-waits, so reserve it for planned, short synchronization. `AbortJobs(waitForCompletion: true)` waits for batch cancellation but does not interrupt a currently running `Run()`. `JobRunner` does not capture `Run()` exceptions as a `Task`-like result; handle expected failures inside the job and store results/errors for the caller. Do not depend on forcibly stopping jobs to shut down resources.

For consistent metrics, use `JobScheduler.GetMetrics(Span<JobRunnerMetrics>)` with capacity for at least the number of runners. `GetMetricsUnsafe` does not guarantee an atomic snapshot while workers run. This scheduler is not an automatic dependency graph: schedule dependent stages after their prerequisite batches finish. Sources: [JobScheduler](../../Brutal.Concurrency/Jobs/JobScheduler.cs) and [JobRunner](../../Brutal.Concurrency/Jobs/JobRunner.cs).
