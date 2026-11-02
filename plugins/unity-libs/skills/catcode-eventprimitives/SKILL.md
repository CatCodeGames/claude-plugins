---
name: catcode-eventprimitives
description: API and rules of the CatCode EventPrimitives Unity package (com.catcode.eventprimitives, namespace CatCode.EventPrimitives) - EventSource, EventValue, ObservableSource, ObservableValue, ObservableSubscription, NotificationMode, AddListenerScoped, Subscribe, SetValueWithoutNotify, ForceNotify, WaitAsync on UniTask. Use whenever code declares or subscribes to events, reactive values or observable state in a Unity project that has this package, or mentions IReadOnlyEventValue, IReadOnlyObservableValue, Raised, Changed, Publish or Subscribe, even if the package is not named.
---

# CatCode EventPrimitives

Describes package version 2.x. Namespace `CatCode.EventPrimitives` (UniTask extensions too). Sources: `Library/PackageCache/com.catcode.eventprimitives@*/` — read them only if something below is not enough.

**Assemblies.** `CatCode.EventPrimitives`; `WaitAsync` lives in `CatCode.EventPrimitives.UniTask`, compiled only when `com.cysharp.unitask` is installed.

## Core idea

Two families with the same shape. **Event\*** wrap a C# `event` (`Raised`/`Changed`, unsubscribe by the same delegate). **Observable\*** have no event: `Subscribe` returns an `ObservableSubscription` token, the only way to unsubscribe. Expose the `IReadOnly*` interface, keep the concrete class private.

## API

```csharp
enum NotificationMode { Always, OnChange }

// Events
EventSource / EventSource<T>        : event Action[<T>] Raised; void Raise([T value]);
EventValue<T>(T initial, NotificationMode mode = OnChange, IEqualityComparer<T> comparer = null)
    T Value { get; set; }  event Action<T> Changed;
    void SetValueWithoutNotify(T); void SetValueAndNotify(T); void ForceNotify();

// Event extensions (also non-generic for IReadOnlyEventSource)
EventSourceSubscriptionToken<T> AddListenerScoped(this IReadOnlyEventSource<T>, Action<T>);
EventValueSubscriptionToken<T>  AddListenerScoped(this IReadOnlyEventValue<T>, Action<T>, bool invokeImmediately = false);
void AddListener(...same, no token...); void RemoveListener(..., handler);

// Observables
ObservableSource / ObservableSource<T>: ObservableSubscription Subscribe(Action[<T>]); void Publish([T value]);
ObservableValue<T>(T initial, NotificationMode mode = OnChange, IEqualityComparer<T> comparer = null)
    T Value { get; set; }  Subscribe(Action<T>);
    void SetValueWithoutNotify(T); void SetValueAndNotify(T); void ForceNotify();
readonly struct ObservableSubscription : IDisposable { bool IsValid, IsDisposed; void Unsubscribe(); void Dispose(); }

// UniTask (same overloads for IReadOnlyEventSource[<T>] / IReadOnlyObservableSource[<T>] without checkInitialState)
UniTask WaitAsync<T>(this IReadOnlyEventValue<T> / IReadOnlyObservableValue<T>,
    Func<T, bool> predicate, bool checkInitialState = true, CancellationToken ct = default);
UniTask WaitAsync<T>(this ..., Func<T, bool> predicate, CancellationToken ct);
UniTask WaitAsync<T>(this ..., CancellationToken ct = default); // next change, current value ignored
```

## Behaviour to rely on

- `OnChange` (default) skips the setter when the comparer says equal; `SetValueAndNotify`/`ForceNotify` always notify, `SetValueWithoutNotify` never does.
- Subscribers are not invoked on subscribe. Pass `invokeImmediately: true` (events) or read `Value` yourself (observables).
- Tokens are structs: dispose is idempotent, `default` token is safe to dispose. Store them in fields, not copies you forget.
- Observables: unsubscribing during `Publish` is safe (removed ones are skipped); subscribers added during `Publish` are called from the next one.
- `WaitAsync` with `checkInitialState: true` completes synchronously if the predicate already holds. Cancellation throws `OperationCanceledException`; a predicate exception faults the task. The returned `UniTask` is pooled — await it once.
- No thread safety: main thread only.

## Patterns

```csharp
private readonly ObservableValue<int> _hp = new(100);
public IReadOnlyObservableValue<int> Hp => _hp;

_sub = player.Hp.Subscribe(OnHpChanged);   // OnDestroy: _sub.Dispose();

_token = state.Score.AddListenerScoped(UpdateLabel, invokeImmediately: true);

await player.Hp.WaitAsync(hp => hp <= 0, this.GetCancellationTokenOnDestroy());
```
