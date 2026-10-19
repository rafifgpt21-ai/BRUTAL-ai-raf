# Basic types, memory, mathematics, and text

Read this guide before choosing interop types, retaining pointers/spans, or writing UI text, logs, and localized strings. Examples are C# usage fragments; adjust package references in the game project. `unsafe` blocks require unsafe compilation to be enabled. This snapshot does not include a build project for compiling the examples.

## Core and memory ownership

Module names do not always match namespaces. `Bool8`, `Bool32`, `ByteSize`, and `ElementCount` are in `Brutal`; pointers are in `Brutal.Pointers`; allocators are in `Brutal.Memory`.

| Type | Use when |
| --- | --- |
| [`Bool8`](../../Brutal.Core/Types/Bool8.cs), [`Bool32`](../../Brutal.Core/Types/Bool32.cs) | A native signature requires an 8-bit or 32-bit boolean. Both convert implicitly to/from `bool`; choose according to the signature, rather than replacing every game `bool`. |
| [`ByteSize`](../../Brutal.Core/Types/ByteSize.cs), [`ElementCount`](../../Brutal.Core/Types/ElementCount.cs) | Distinguishing byte sizes from element counts. `ByteSize.Of<T>(count)` computes the size of `count` elements of `T`. |
| [`Ptr`](../../Brutal.Core/Pointers/Ptr.cs), [`Ptr<T>`](../../Brutal.Core/Pointers/PtrOfT.cs) | Wrapping `void*` or `T*` (`T : unmanaged`). These are address views without ownership or bounds checks. `Ptr.Offset` takes bytes; `Ptr<T>.Offset` advances by elements according to its implementation. |
| [`PtrPtr`](../../Brutal.Core/Pointers/PtrPtr.cs), [`PtrPtr<T>`](../../Brutal.Core/Pointers/PtrPtrOfT.cs) | A native signature requires a pointer to a pointer. Follow the binding signature. |

[`Mem`](../../Brutal.Memory/Mem.cs) wraps `NativeMemory`: `Alloc`, `AllocZeroed`, `AllocAndCopy`, `Copy`, and `Free`. `Mem.Alloc<T>(count)` returns a `Span<T>` over native memory, not a GC-owned buffer; the caller must still release it with `Mem.Free(span)`. Use `AllocZeroed` when the initial contents must be zero.

[`PtrOwner<T>`](../../Brutal.Memory/PtrOwner.cs) owns a pointer, frees it through `Dispose`, and provides a fallback finalizer. Its value constructor allocates a copy; its `Ptr<T>` constructor takes ownership of that pointer. Supply only allocations compatible with `Mem.Free`, with a single owner. Borrowed pointers from native libraries still require those libraries' destruction functions. `Release()` transfers responsibility for freeing the memory to the caller. Inspect the implementation before using other ownership helpers: `Reset()` currently calls `Release()` and discards the old pointer without freeing it, despite its comment saying otherwise.

```csharp
using Brutal.Memory;

unsafe
{
    // Correct: the owner stays alive while the pointer is used.
    using var value = new PtrOwner<int>(42);
    value.Ptr.Value = 7;
}
```

```csharp
using Brutal.Memory;

unsafe
{
    // Incorrect: a Ptr copy does not keep the allocation alive.
    var owner = new PtrOwner<int>(42);
    var borrowed = owner.Ptr;
    owner.Dispose();
    borrowed.Value = 7; // use-after-free
}
```

Views returned by `AsSpan`, `Cast`, and `Offset` remain valid only while their backing memory is alive. Converting a managed span to `Ptr<T>` does not permanently pin it; keep managed addresses in a `fixed` block throughout native access. [`FixedAttribute`](../../Brutal.Core/Attributes.cs) marks a contract and does not replace a `fixed` block.

## Collections

Use [`NativeList<T>`](../../Brutal.Collections/NativeList/NativeListOfT.cs) (`Brutal.Collections`, `T : unmanaged`) for a contiguous list in native memory. It exposes `Count`, `Capacity`, a `ref` indexer, `Add`, `AddRange`, `AsSpan`, and `Ptr`; release it with `using`/`Dispose`. .NET `List<T>` remains suitable for managed data, including reference objects.

```csharp
using Brutal.Collections;
using Brutal.Numerics;

using var positions = new NativeList<float3>(capacity: 16);
positions.Add(new float3(1, 2, 3));
positions.AsSpan()[0] = float3.Zero;
// Correct: obtain a fresh view after an operation that may change capacity.
positions.EnsureCapacity(64);
var current = positions.AsSpan();
```

Incorrect: retaining `positions.Ptr`, `ref positions[0]`, or an old span and using it after capacity growth, `TrimExcess`, or `Dispose`. [`NativeList._Reallocate`](../../Brutal.Collections/NativeList/NativeList.cs) allocates a new buffer and frees the old one.

Available variants:

- [`NativeList`](../../Brutal.Collections/NativeList/NativeList.cs): an untyped buffer without a fixed stride; public size and capacity are measured in bytes.
- [`NativeStrideList`](../../Brutal.Collections/NativeList/NativeStrideList.cs): owns a buffer with a fixed byte stride and an element count; use it when the layout is determined at runtime.
- [`NativeStrideSpan`](../../Brutal.Collections/Span/NativeStrideSpan.cs): a non-owning view. `AsSpan<T>()` requires `T` to match the stride in size, unless `T` is `byte`.
- [`ScratchSpan<T>`](../../Brutal.Collections/Span/ScratchSpan.cs), in `Brutal.Collections.Experimental`: a scratch buffer that uses the supplied stack span when large enough, otherwise rents an array from a pool. Use `using var scratch = new ScratchSpan<int>(requiredLength, stackalloc int[256]);`, then access `scratch.Span` before `Dispose` returns the array.
- [`FreeListIndexPool`](../../Brutal.Collections/FreeList.cs): an index pool with reusable slots. Check its API and index lifetime before using indices as entity IDs.
- [`Name<T>`](../../Brutal.Collections/Name/Name.cs) uses [`Namespace<T>`](../../Brutal.Collections/Name/Namespace.cs) to pool string IDs. Equality between `Name<T>` values uses string reference identity; use the same pool for names that must compare equal.

## Numerics and Maths

Import `Brutal.Numerics`. Types are named by component type and dimension: `float2/3/4`, `double2/3/4`, `int2/3/4`, `uint2/3/4`, `bool2/3/4`, and other integer variants. [`float3`](../../Brutal.Numerics/Vector/float3/float3.gen.cs) has `X/Y/Z` fields, a `(float x, float y, float z)` constructor, an indexer, and span views. [`floatQuat`](../../Brutal.Numerics/Quaternion/floatQuat/floatQuat.gen.cs) is a quaternion; [`float3x2`](../../Brutal.Numerics/Matrix/float3x2/float3x2.gen.cs) and [`float4x4`](../../Brutal.Numerics/Matrix/float4x4/float4x4.gen.cs) are matrices; `double` variants are also available.

These are separate BRUTAL types, not aliases for `System.Numerics.Vector3`, `Matrix4x4`, or `Quaternion`. Match the types required by binding signatures and explicitly convert components when crossing into .NET APIs. Operations are available as static methods on the types, such as [`float3.Dot`, `Normalize`, `NormalizeOrZero`](../../Brutal.Numerics/Vector/float3/float3.math-metric.gen.cs) and [`float3.Transform`](../../Brutal.Numerics/Vector/float3/float3.math.gen.cs). `Normalize` throws `DivideByZeroException` for a zero vector; choose `NormalizeOrZero` when zero is a valid input.

```csharp
using Brutal.Numerics;

var position = new float3(1, 2, 3);
System.Numerics.Vector3 dotNet = new(position.X, position.Y, position.Z);
float3 direction = float3.NormalizeOrZero(position);
float3 unchanged = float3.Transform(position, float4x4.Identity);

// Incorrect: float3 is not a Vector3 alias; this assignment has no conversion.
// System.Numerics.Vector3 wrong = position;
```

`float4x4` stores **rows** `X/Y/Z/W`; `float3.Transform(position, matrix)` sums rows weighted by the components, then adds row `W` as translation. Match the layout, multiplication order, and shader uploads to the renderer's contract; the type name alone does not determine GPU buffer layout.

Colors use vectors: [`float4.Rgba(r,g,b,a)`](../../Brutal.Numerics/Vector/float4/float4.color.gen.cs) uses 0–1 components; [`byte4`](../../Brutal.Numerics/Vector/byte4/byte4.color.gen.cs) uses byte components. [`Color`](../../Brutal.Numerics/Color/Color.cs) is a static preset class, not a color struct with an RGBA constructor. [`Color.Preset`](../../Brutal.Numerics/Color/Color.Preset.cs) provides conversions to `float3/4`, `byte3/4`, and `ushort3/4`.

```csharp
using Brutal.Numerics;

float4 tint = Color.Red;
float4 translucent = float4.Rgba(1f, 0.5f, 0f, 0.75f);
// Incorrect: Color is a static class.
// var wrong = new Color(255, 128, 0, 255);
```

[`Brutal.Maths.RandomExtensions`](../../Brutal.Maths/RandomExtensions.cs) adds range overloads for `NextSingle/NextDouble` and point sampling to `Random`, such as `NextSinglePointInSphere(radius: 2)`. Import `Brutal.Maths` to enable these extensions; vector operations remain in `Brutal.Numerics`.

## String8, views, and builders

Import `Brutal.Strings`. [`String8`](../../Brutal.Strings/String8/String8.cs) owns a byte array with a null terminator; its constructor from `ReadOnlySpan<char>` encodes UTF-8, while its byte constructor copies without validating the encoding. `Length` counts **bytes**, not Unicode characters. `ToString16()` produces a .NET `string`. [`RefString8`](../../Brutal.Strings/String8/RefString8.cs) is a borrowed `readonly ref struct` over a null-terminated byte string; it searches for the terminator when reading `Chars`, so the source must be terminated and remain alive long enough.

```csharp
using Brutal.Strings;

var owned = new String8("Player");
RefString8 borrowed = owned; // The backing String8 provides a terminator.
RefString8 literal = "Player"u8;
string managed = owned.ToString16();

// Incorrect: the RefString8 constructor does not append a terminator.
// byte[] raw = [65, 66];
// RefString8 invalid = raw; // Reading Chars may go beyond the buffer.
```

The `Char8`, `String8View`, `String16`, and `StringBuilder8` aliases in [`GlobalUsings.cs`](../../Brutal.Strings/GlobalUsings.cs) apply to that module's compilation, not automatically to the game. In game code, use `byte`, `ReadOnlySpan<byte>`, `string`, and `Brutal.Strings.StringBuilder<byte>`, or declare your own aliases.

[`Brutal.Strings.StringBuilder<TChar>`](../../Brutal.Strings/StringBuilder/StringBuilder.cs) and [`ValueStringBuilder<TChar>`](../../Brutal.Strings/StringBuilder/ValueStringBuilder.cs) support byte/char text. A builder's `WrittenSpan` is a borrowed view that must be obtained again after mutation; copy/convert it to retain the contents longer. `StringBuilder` conflicts with `System.Text.StringBuilder`, so use an alias or the full name when both are imported.

## ImGui text

[`ImString`](../../Brutal.ImGui/ImString/ImString.cs) is in `Brutal.ImGuiApi`. Use `u8` literals for static labels or [`ImString` interpolation](../../Brutal.ImGui/ImString/ImString.Interpolation.cs) for dynamic text. UTF-16 conversion and interpolation write to a shared arena; views remain valid only until the arena is cleared. [`ArenaStringBuilder`](../../Brutal.ImGui/ImString/ArenaStringBuilder.cs) has a fixed capacity and throws when full. The game's frame integration must call `ImString.SharedStorage.Clear()` after all consumers of that frame's text have finished; retain persistent data as `string`/`String8`, then create an `ImString` each frame. Serialize shared arena access on the UI thread.

[`ImInputString`](../../Brutal.ImGui/ImString/ImInputString.cs) is a mutable heap byte buffer for input fields that persist between frames; its capacity includes the terminator. [`ImGui.InputText`](../../Brutal.ImGui/Manual/Functions/ImGui.cs) accepts this buffer and updates `Length` when its contents change.

```csharp
using Brutal.ImGuiApi;

// Keep this buffer in UI state instead of recreating it every frame.
var playerName = new ImInputString(128, "Player"u8);
// During an active ImGui frame, with the context/window already prepared:
unsafe { ImGui.InputText("Name"u8, playerName); }
ImString label = $"Name: {playerName.Value}";
ImGui.Text(label); // Use during this frame.

// Incorrect: clearing the arena ends the lifetime of earlier dynamic text.
// ImString.SharedStorage.Clear();
// ImGui.Text(label); // Later text may overwrite the label.
```

The `ImString(ReadOnlySpan<byte>)` overload borrows bytes without copying or adding a terminator. For native APIs that read C strings, use `u8` literals, arena output, or null-terminated buffers; span length alone does not bound native reads. [`ImGui.Text`](../../Brutal.ImGui/Manual/Functions/ImGui.Text.cs) itself forwards a pointer to `PInvoke.TextUnformatted` with `text_end = null`.

## Logging and localization

Initialize [`LogSystem.Initialize(Action<ILoggingBuilder>)`](../../Brutal.Logging/LogSystem.cs) with the project's logging providers before use. [`LogCategory.Make(name)`](../../Brutal.Logging/LogCategory.cs) creates/retrieves a category; its level methods are `Trace`, `Debug`, `Info`, `Warning`, `Error`, and `Critical`. [`DefaultCategory.Log`](../../Brutal.Logging/DefaultCategory.cs) is available when a specific category is unnecessary.

[`LogString<TLevel>`](../../Brutal.Logging/LogString.cs) is a compiler-facing interpolated-string handler. Pass interpolation directly to logging methods so value evaluation and text construction can be skipped when the level/category is disabled. Avoid constructing `LogString` yourself.

```csharp
using Brutal.Logging;
using Microsoft.Extensions.Logging;

// The application chooses providers; AddConsole requires the .NET console provider.
LogSystem.Initialize(builder => builder.AddConsole());
var log = LogCategory.Make("Gameplay");
log.Info($"Player joined: {42}"); // The handler checks enablement.

// Incorrect for a hot path: the string is built before the log filter runs.
// string eager = $"Player joined: {42}";
// log.Debug(eager);
```

[`LString`](../../Brutal.Localization/LString.cs), in `Brutal.Localization`, stores an ID, template, and localized text. [`LStrings`](../../Brutal.Localization/LStrings.cs) is a facade over a privately held shared [`LocalizedStrings`](../../Brutal.Localization/LocalizedStrings.cs) instance: `Create` rejects duplicate IDs; `GetOrCreate` retrieves an existing registration. Register text once, then format it when displayed. `SetLocalizedStringProvider` installs a translation lookup delegate and updates registered instances; text falls back to the template when no translation is available.

Templates use placeholders such as `{Name}` according to [`DefaultOptions.ReplacementPattern`](../../Brutal.Localization/DefaultOptions.cs). Translations must preserve placeholder names and counts; [`Format(...)`](../../Brutal.Localization/LString.Format.gen.cs) validates the argument count and maps arguments in template order, even when a translation changes the placeholder order.

```csharp
using Brutal.Localization;
using Brutal.ImGuiApi;

var greeting = LStrings.GetOrCreate("ui.greeting", "Hello {Name}");
greeting.Localize("Welcome {Name}");
ImString label = greeting.Format("Rafi"); // Format first, then convert for UTF-8 UI.

// Incorrect: implicit LString -> ImString conversion calls Format() without arguments.
// ImString invalid = greeting; // The replacement count does not match.
// greeting.Localize("Welcome {Player}"); // The placeholder name does not match.
```
