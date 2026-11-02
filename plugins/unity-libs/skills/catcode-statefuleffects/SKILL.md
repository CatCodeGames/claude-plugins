---
name: catcode-statefuleffects
description: API and rules of the CatCode StatefulEffects Unity package (com.catcode.stateful-effects, namespace CatCode.StatefulEffects) - show/hide and one-shot effects as state machines - IShowHide, MonoShowHide, CallbackShowHide, ShowHidePolicy, IEffectAction, MonoEffectAction, CallbackEffectAction, AnimatorShowHide, DOTweenFloatShowHide, ShowAsync, HideAsync, PlayAsync. Use whenever code shows, hides or plays such effects, writes a new show/hide or effect component, or awaits their completion in a Unity project that has this package, even if the package is not named.
---

# CatCode StatefulEffects

Describes package version 3.x. Namespace `CatCode.StatefulEffects`. Sources: `Library/PackageCache/com.catcode.stateful-effects@*/` — read them only if something below is not enough.

**Assemblies:** `CatCode.StatefulEffects`; `CatCode.StatefulEffects.UniTask` (compiled only with UniTask); `CatCode.StatefulEffects.DOTween` (only with the `DOTWEEN` define or `com.demigiant.dotween`).

## Core idea

An effect is a state machine. A command changes `State` synchronously or is rejected; no queue, no return value. Completion comes later via `OnStateChanged`. `None` = not controlled (before the first command, after `Stop`), look undefined: the owner (spawner, pool, UI) sets the start state; components never play on start.

## API

```csharp
enum ShowHideState { None, Hidden, Showing, Shown, Hiding }
interface IShowHide {   // MonoShowHide : MonoBehaviour, IShowHide — use for inspector fields
    ShowHideState State; event Action<ShowHideState> OnStateChanged;
    void Show(); void Hide(); void SetShown(); void SetHidden(); void Stop(); }
struct ShowHidePolicy { TransitionPolicy Show, Hide; }   // default == Default
struct TransitionPolicy { InterruptResolution OnInterrupt /*Accept,Ignore*/; RepeatResolution OnRepeat /*Ignore,Restart*/; }

enum EffectActionState { None, Ready, Playing }
interface IEffectAction { State; OnStateChanged; void Play(); void Prepare(); void Stop(); }
struct EffectActionPolicy { InterruptResolution OnRepeat; }   // Play while Playing

// Bases: CallbackShowHide (MonoBehaviour, policy in inspector) or ShowHide (plain C#)
protected virtual void OnInitialize();   // CallbackShowHide, before the first command
protected abstract void OnShow(ShowHideState from, Action onCompleted); // + OnHide
protected abstract void OnSetShown(ShowHideState from);                 // + OnSetHidden, OnStop
// CallbackEffectAction / EffectAction: OnPlay(from, onCompleted), OnPrepare(from), OnStop(from)

// Ready-made: GameObjectShowHide, ParticleSystemShowHide, ParticleEffectAction, AnimatorShowHide,
new DOTweenFloatShowHide(Action<float> setValue, DOTweenFloatSettings show, hide, bool independentUpdate = false, policy = default);

// UniTask: ShowAsync / HideAsync(this IShowHide, [ShowHideWaitMode,] ct), PlayAsync(this IEffectAction, ct),
// WaitForStateAsync(state, ct) on both
enum ShowHideResult { AlreadyInState, Completed, Interrupted, Stopped, Rejected }
enum ShowHideWaitMode { FirstChange /*default*/, UntilReached }
enum EffectActionResult { AlreadyInState, Completed, Stopped }
```

## Behaviour to rely on

- `SetShown`/`SetHidden` are instant and always accepted. `Stop` releases control and leaves the look as is.
- Policy belongs to the running transition: `Show.OnInterrupt = Ignore` rejects `Hide` while `Showing`. Repeated `Show` while `Showing` is always ignored; from `Shown` with `Restart` it calls `OnSetHidden(Shown)`, then `OnShow(from: Hidden)`.
- Hook `from == None`: reset the view first; `from == Hiding`: reverse from the current look.
- `onCompleted` may be called synchronously; stale calls are ignored.
- `OnDisable` always calls `Stop`; a hook must never deactivate its own object or a parent.
- `AnimatorShowHide` needs the `ShowHide_AC` graph (clips via `AnimatorOverrideController`) and `AnimatorShowHideConfig`; repeat ignored, interrupt reverses.
- Particle components need `ParticleSystemStopListener` on each system. `DOTweenFloatShowHide` is not a component: call its `Stop()` on disable.
- Async: `FirstChange` returns `Interrupted` when another command takes over; `UntilReached` waits for the target or `None` (`Stopped`). `PlayAsync` completes on `Ready`. Cancellation throws `OperationCanceledException` and stops only the waiting.

## Patterns

```csharp
[SerializeField] MonoShowHide _panel;
_panel.SetHidden();   // owner sets the start state
if (await _panel.ShowAsync(this.GetCancellationTokenOnDestroy()) == ShowHideResult.Completed) EnableInput();

public sealed class ScaleShowHide : CallbackShowHide {   // OnHide, OnSetHidden mirrored
    [SerializeField] Transform _t; Tween _tween;
    protected override void OnShow(ShowHideState from, Action onCompleted) {
        if (from == ShowHideState.None) _t.localScale = Vector3.zero;
        _tween.Kill(); _tween = _t.DOScale(1f, 0.3f).OnComplete(() => onCompleted()); }
    protected override void OnSetShown(ShowHideState from) { _tween.Kill(); _t.localScale = Vector3.one; }
    protected override void OnStop(ShowHideState from) => _tween.Kill();
}

_hitFx.Prepare();   // on spawn from pool
await _hitFx.PlayAsync(ct); Despawn();
```
