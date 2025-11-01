# Unity Essentials

This module is part of the Unity Essentials ecosystem and follows the same lightweight, editor-first approach.
Unity Essentials is a lightweight, modular set of editor utilities and helpers that streamline Unity development. It focuses on clean, dependency-free tools that work well together.

All utilities are under the `UnityEssentials` namespace.

```csharp
using UnityEssentials;
```

## Installation

Install the Unity Essentials entry package via Unity's Package Manager, then install modules from the Tools menu.

- Add the entry package (via Git URL)
    - Window → Package Manager
    - "+" → "Add package from git URL…"
    - Paste: `https://github.com/CanTalat-Yakan/UnityEssentials.git`

- Install or update Unity Essentials packages
    - Tools → Install & Update UnityEssentials
    - Install all or select individual modules; run again anytime to update

---

# Timing

> Quick overview: High‑performance coroutine scheduler using IEnumerator<float> with segment control (Update, FixedUpdate, LateUpdate, LazyUpdate). Start, pause, resume, kill by handle or group. Zero‑alloc pools and batched LazyUpdate.

A compact runtime system for fine‑grained coroutine control. Instead of Unity’s yield instructions, coroutines yield a float number of seconds to wait. The scheduler runs them in one of four segments, supports pause/resume/kill via a tiny handle, and includes a batched LazyUpdate lane for deferrable work.

![screenshot](Documentation/Screenshot.png)

## Features
- Segmented execution
  - `Segment.Update`, `Segment.FixedUpdate`, `Segment.LateUpdate`, and `Segment.LazyUpdate`
  - Each segment ticks in the corresponding Unity loop; LazyUpdate processes up to 100 items per frame
- Simple API with lightweight handles
  - `ushort RunCoroutine(IEnumerator<float> routine, Segment segment = Segment.Update, string handleGroup = "Default")`
  - `PauseCoroutine(handle)`, `ResumeCoroutine(handle)`, `KillCoroutine(handle)`
  - `KillAllCoroutines()`, `KillAllCoroutines(int segmentIndex)`, `KillAllCoroutines(string handleGroup)`
  - `IsCoroutineActive(handle)`
- Precise time control
  - Yield seconds via `yield return 0.25f` to wait 250 ms
  - Constants: `Timing.Continue` (0f), `Timing.WaitForOneFrame` (one frame), `Timing.WaitForFixedUpdate()` helper
  - Access `Timing.DeltaTime` and `Timing.LocalTime` per segment
- Grouping
  - Tag coroutines with `handleGroup` and stop them in bulk later
- Safety and performance
  - Exceptions are logged and the offending coroutine is stopped
  - Internals use pooled arrays for low garbage; tests include a zero‑alloc sanity check

## Requirements
- Unity 6000.0+
- Runtime module; no external dependencies

## Usage

### 1) Start a coroutine
```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEssentials;

public class Example : MonoBehaviour
{
    void Start()
    {
        // Run on Update segment with a group tag
        _handle = Timing.RunCoroutine(Work(), Segment.Update, "Gameplay");
    }

    private ushort _handle;

    private IEnumerator<float> Work()
    {
        Debug.Log("Step 1");
        yield return 0.1f;                 // wait 0.1 seconds

        Debug.Log("Step 2");
        yield return Timing.WaitForOneFrame; // exactly one frame

        // Wait until next FixedUpdate has occurred
        yield return Timing.WaitForFixedUpdate();

        Debug.Log("Done");
        yield break; // or just fall off the end
    }
}
```

### 2) Pause/Resume/Kill
```csharp
Timing.PauseCoroutine(_handle);
// ... later
Timing.ResumeCoroutine(_handle);
// ... or stop
Timing.KillCoroutine(_handle);
```

### 3) Run on FixedUpdate or LateUpdate
```csharp
Timing.RunCoroutine(PhysicsRoutine(), Segment.FixedUpdate);
Timing.RunCoroutine(EndOfFrameRoutine(), Segment.LateUpdate);
```

### 4) LazyUpdate batching
Use LazyUpdate for large queues of non‑urgent tasks. Up to 100 items are processed each frame, then the pointer wraps and continues next frame.
```csharp
for (int i = 0; i < 10_000; i++)
    Timing.RunCoroutine(DoOneHeavyThing(i), Segment.LazyUpdate, "HeavyQueue");

// Later, stop everything in this queue
Timing.KillAllCoroutines("HeavyQueue");
```

### 5) Conditional waits
```csharp
using System; // for Func<>

IEnumerator<float> WaitForCondition(Func<bool> predicate)
{
    // Keep yielding one frame while the predicate is false
    while (!predicate())
        yield return Timing.WaitForOneFrame;
}

// Or use helpers in a loop
while (!myReadyFlag)
    yield return Timing.WaitUntil(myReadyFlag);

while (isLoading)
    yield return Timing.WaitWhile(isLoading);
```

## How It Works
- Scheduler
  - Each segment keeps a pooled array of `ProcessData` entries; each holds the enumerator, wait time, pause flag, handle version, and group
  - Update/FixedUpdate/LateUpdate call `ProcessSegment`; Update also runs a batched `ProcessLazySegment`
- Stepping
  - A coroutine is advanced while `LocalTime >= WaitUntil`
  - `yield return x` sets `WaitUntil = LocalTime + x` (NaN becomes 0 for immediate)
  - `Timing.WaitForOneFrame` uses a sentinel that forces a one‑frame break
- Time
  - Update/LateUpdate use `Time.deltaTime`/`Time.time`; FixedUpdate uses `Time.fixedDeltaTime`/`Time.fixedTime`
  - `Timing.WaitForFixedUpdate()` returns Continue only after a FixedUpdate occurred since the last LateUpdate
- Cleanup and safety
  - On completion or exception the coroutine is stopped and its slot returned to the pool
  - Global cleanup utilities: kill by handle, by segment, by group, or all

## Notes and Limitations
- Yield type is `float` seconds only; Unity yield instructions (e.g., `new WaitForSeconds`) are not used here
- Single‑threaded main‑thread scheduler; do not call from background threads
- Ordering within a segment depends on the pooled storage; it is not strictly FIFO under all operations
- Handles are 16‑bit; reuse can occur over a long session - check `IsCoroutineActive(handle)` if you cache handles
- LazyUpdate processes at most 100 entries per frame; tune code accordingly for very large queues

## Files in This Package
- `Runtime/Timing.cs` – Core scheduler, segments, wait helpers, handles, group and bulk operations
- `Runtime/UnityEssentials.Timing.asmdef` – Runtime assembly definition
- `Tests/TimingTests.cs` – Unit tests for lifecycle, segments, and zero‑alloc check

## Tags
unity, coroutine, scheduler, timing, update, fixedupdate, lateupdate, lazyupdate, async, runtime
