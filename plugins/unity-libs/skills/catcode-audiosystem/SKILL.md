---
name: catcode-audiosystem
description: API and rules of the CatCode AudioSystem Unity package (com.catcode.audiosystem, namespace CatCode.Audio) - AudioService, AudioHandle, AudioPlayOptions, PlayAsync, Pause/Mute of AudioMixerGroup through AudioMixerGroupMirror. Use whenever code plays, stops, pauses or mutes sounds or music in a Unity project that has this package, or mentions AudioService, AudioHandle, PlayOneShot, PlaybackResult or audio mixer group pause, even if the package is not named.
---

# CatCode AudioSystem

Describes package version 1.x. 2D audio for Unity. Namespace `CatCode.Audio`. Sources: `Library/PackageCache/com.catcode.audiosystem@*/` — read them only if something below is not enough.

**Setup.** One `AudioService` (MonoBehaviour) in the scene with every used `AudioMixer` listed in its inspector. Assembly references: `CatCode.Audio`; for `PlayAsync` also `CatCode.Audio.UniTask` and `UniTask`.

## Core idea

The result receiver (callback or task) is passed at `Play`. `AudioHandle` is a remote control only: it cannot report completion later. Never design code that obtains a handle and then waits on it.

## API

```csharp
// AudioService
AudioHandle Play(AudioClip clip, AudioMixerGroup group, AudioPlayOptions options = default);
AudioHandle Play(clip, group, options, Action<PlaybackResult> onFinished);
AudioHandle Play<T>(clip, group, options, T state, Action<T, PlaybackResult> onFinished) where T : class; // no closure
void PlayOneShot(AudioClip clip, AudioMixerGroup group, float volume = 1f); // no handle, no callback
void StopAll();
AudioMixerGroupMirror GetAudioMixerGroupMirror(AudioMixerGroup group);

// CatCode.Audio.UniTask
UniTask<PlaybackResult> PlayAsync(this AudioService s, clip, group, options, out AudioHandle handle, CancellationToken ct = default);

new AudioPlayOptions(volume: 1f, pitch: 1f, loop: false); // default == AudioPlayOptions.Default
enum PlaybackResult { Completed, Interrupted }

// AudioHandle (struct, copyable; AudioHandle.None == default)
bool IsValid, IsPlaying, IsPaused;
float Volume, Pitch; bool Loop, Mute; AudioMixerGroup OutputAudioMixerGroup; // setters only
bool TryGetVolume(out float) / TryGetPitch / TryGetLoop / TryGetMute / TryGetOutputAudioMixerGroup;
void Pause(); void Resume(); void Stop();

// AudioMixerGroupMirror.Pause / .Mute
bool LocalValue;   // set on this group
bool TotalValue;   // including parent groups
void SetValue(bool value);
ObservableSubscription Subscribe(Action<bool> onTotalChanged); // .Dispose() / .Unsubscribe()
```

## Behaviour to rely on

- After a sound ends, its handle is invalid and every call through it is a safe no-op. Keep the handle in a field and overwrite it on the next `Play`; no null checks needed.
- `Stop()`, cancellation of the token, `StopAll()` and destroying `AudioService` all finish with `Interrupted`. `PlayAsync` never throws `OperationCanceledException`; a token cancelled before the call returns `Interrupted` without playing and gives `AudioHandle.None`.
- The callback runs after the playback is released, so starting the next sound from it is fine.
- Group `Pause`/`Mute` applies to the whole subtree and to one-shots; a one-shot started in a paused group is dropped.
- Group volume and other parameters still go through `AudioMixer.SetFloat` on exposed parameters.
- A group from a mixer not listed in `AudioService` plays without Pause/Mute and logs a warning.
- Unity API is main-thread only; token cancellation from another thread is handled by the package.

## Patterns

```csharp
_music = _audio.Play(clip, musicGroup, new AudioPlayOptions(loop: true));

_audio.Play(clip, sfxGroup, default, this, static (self, r) => self.OnShotFinished(r));

var task = _audio.PlayAsync(clip, voiceGroup, default, out var line, this.GetCancellationTokenOnDestroy());
line.Volume = 0.8f;
if (await task == PlaybackResult.Completed) ShowNextLine();

_audio.GetAudioMixerGroupMirror(masterGroup).Pause.SetValue(true); // pause menu
```
