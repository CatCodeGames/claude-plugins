---
name: catcode-uniloop
description: API and rules of the CatCode UniLoop Unity package (com.catcode.uniloop, namespace CatCode.PlayerLoops) - zero-allocation PlayerLoop scheduler: UniLoop.Schedule, ScheduleSlim, ScheduleOnce, UniLoop.Update/FixedUpdate/LateUpdate/PostUpdate, PlayerLoopTiming, PlayerLoopPhase, LoopHandle, SlimLoopHandle, TokenLoopHandle, WhileHandle, TokenWhileHandle, SetOnCompleted/SetOnCanceled. Use whenever code runs per-frame logic without MonoBehaviour.Update, schedules a delegate every frame, N times, once or while a predicate holds, or mentions UniLoop, even if the package is not named.
---

# CatCode UniLoop

Describes package version 2.x. Namespace `CatCode.PlayerLoops`, assembly `UniLoop` (auto-referenced). Sources: `Library/PackageCache/com.catcode.uniloop@*/` — read them only if something below is not enough.

## Core idea

A static scheduler that injects runners into the Unity PlayerLoop. Each task returns a struct handle; the handle type tells how the task ends: disposed by you, cancelled by a token, or completed by count/predicate.

## API

```csharp
enum PlayerLoopTiming { FixedUpdate, Update, LateUpdate }  // LateUpdate = PreLateUpdate
enum PlayerLoopPhase { Early, Late }                        // begin / end of that system

// static UniLoop — same overloads end with (timing, phase)
SlimLoopHandle   ScheduleSlim(Action action, timing, phase);
LoopHandle       Schedule(Action action, timing, phase);
TokenLoopHandle  Schedule(Action action, CancellationToken ct, timing, phase);
WhileHandle      Schedule(Action action, int count, timing, phase); // count > 0
WhileHandle      ScheduleOnce(Action action, timing, phase);
WhileHandle      Schedule(Func<bool> predicate, timing, phase);   // runs while true
TokenWhileHandle Schedule(Func<bool> predicate, CancellationToken ct, timing, phase);

ITimingScheduler GetLoop(timing, phase);
ITimingScheduler Update, FixedUpdate, LateUpdate;              // Early phase
ITimingScheduler PostUpdate, PostFixedUpdate, PostLateUpdate;  // Late phase
// ITimingScheduler.Loop : ScheduleSlim / Schedule(action[, ct | count]) / ScheduleOnce
// ITimingScheduler.While: Schedule(predicate[, ct])
RunnerMetrics GetMetrics(timing, phase, UniLoopRunnerType type); // Count, PendingAddCount, TotalCount

// Handles (readonly structs, default == inactive, every call on it is a no-op)
SlimLoopHandle   : IDisposable; bool IsDisposed;
LoopHandle       : IDisposable; bool IsDisposed; SetOnCanceled(...)
TokenLoopHandle  : bool IsActive; SetOnCanceled(...)                     // no Dispose
WhileHandle      : IDisposable; bool IsDisposed; SetOnCompleted(...), SetOnCanceled(...)
TokenWhileHandle : bool IsActive; SetOnCompleted(...), SetOnCanceled(...) // no Dispose
// SetOnX(Action) | SetOnX<T>(T state, Action<T>) | SetOnX<T>(Action<T>, T state), T : class; returns the handle
```

## Behaviour to rely on

- Play Mode and main thread only; in Editor/Development builds a violation throws `InvalidOperationException`. Initialized automatically; works with Domain Reload disabled.
- A task scheduled now first runs on the next pass of its runner (usually next frame), never inside the current iteration.
- `Dispose()` / token cancellation is applied when the runner next reaches the task: the action is skipped and `OnCanceled` fires. `IsDisposed`/`IsActive` change immediately.
- Callbacks run after the pass finishes, so scheduling new tasks from them is safe. Setting a callback on an inactive handle is ignored; set it right after `Schedule`.
- Count/once: `OnCompleted` fires after the last run. Predicate: returns `false` → completes; the predicate itself is the per-frame body.
- An exception in an action or predicate is logged and the task keeps running.
- Slim has no callbacks and is the cheapest; prefer it for permanent loops. A slim task disposed mid-frame may still run once this frame.
- Token handles cannot be disposed — cancel the token. Use `GetCancellationTokenOnDestroy()` or `destroyCancellationToken` to tie a task to a MonoBehaviour.
- On exiting Play Mode runners are removed without firing callbacks.
- Order within one phase: Slim, Loop, TokenLoop, Counted, While, TokenWhile.
- Editor monitor: `Window/UniLoop/Runner Monitor`.

## Patterns

```csharp
_tick = UniLoop.Update.Loop.ScheduleSlim(Tick);          // OnDisable: _tick.Dispose();

UniLoop.Update.Loop.Schedule(Move, destroyCancellationToken);

UniLoop.LateUpdate.Loop.ScheduleOnce(ApplyLayout);       // next frame

UniLoop.Update.While.Schedule(() => (_t += Time.deltaTime) < duration)
    .SetOnCompleted(this, static self => self.OnFadeDone());
```
