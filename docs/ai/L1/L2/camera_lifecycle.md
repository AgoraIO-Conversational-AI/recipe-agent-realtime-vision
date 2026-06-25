# Deep Dive — Camera Lifecycle

> **When to Read This:** You are touching camera track acquisition, publish ordering, local preview rendering, or camera release paths. For the high-level picture, start at [02_architecture](../02_architecture.md).

The camera is entirely client-side. The backend builds the MLLM with `input_modalities=["text","image"]` but does not know about or control the camera. The web client acquires the camera, publishes it into the RTC channel, shows a local preview, and is responsible for releasing it correctly on unmount and on explicit end-conversation.

## Acquisition

`useLocalCameraTrack(isReady)` in `ConversationComponent.tsx` acquires the camera track. It is called with the same `isReady` flag as `useLocalMicrophoneTrack` — both tracks become available at the same time, after a one-tick delay to avoid StrictMode double-invocation issues.

```tsx
const { localMicrophoneTrack } = useLocalMicrophoneTrack(isReady);
const { localCameraTrack } = useLocalCameraTrack(isReady);
```

`isReady` is set to `true` in a `useEffect` with a `setTimeout(..., 0)` so the first render cycle completes before the SDK is asked to open devices.

## Publish

Both tracks are published together via `usePublish`:

```tsx
usePublish([localMicrophoneTrack, localCameraTrack]);
```

Agora forwards the published camera frames to the ConvoAI cloud, which provides them as image input to the `OpenAIRealtime` MLLM (because `input_modalities=["text","image"]` is set). If the user denies camera permission, `localCameraTrack` is `null` or an error state; `usePublish` handles a null gracefully — only the mic track is published.

## Local preview

A small preview is rendered in the conversation visualizer area:

```tsx
{localCameraTrack && (
  <div style={{ width: 240, height: 180, borderRadius: 8, overflow: "hidden" }}>
    <LocalVideoTrack track={localCameraTrack} play style={{ width: "100%", height: "100%" }} />
  </div>
)}
```

The preview only renders when the track is available. Adjust dimensions and placement here; the `LocalVideoTrack` component from `agora-rtc-react` mirrors what is being published.

## Release — two required paths

The camera must be released in both of these paths to avoid leaving the browser camera indicator light on:

### 1. Component unmount (`useEffect` cleanup)

```tsx
useEffect(() => {
  return () => {
    localCameraTrack?.stop();
    localCameraTrack?.close();
  };
}, [localCameraTrack]);
```

Handles tab close, browser navigation, error boundary unmount, or any other path where the component disappears without an explicit end-call.

### 2. Explicit end-conversation (`handleEndConversation`)

```tsx
if (localCameraTrack) {
  try { await client?.unpublish(localCameraTrack); } catch { ... }
  try { localCameraTrack.stop(); localCameraTrack.close(); } catch { ... }
}
```

Unpublishes the track from the RTC channel before releasing it so Agora stops forwarding frames before the device closes. Keep `stop()` + `close()` in both paths — do not rely on only one.

## What happens when camera is denied

- `useLocalCameraTrack` returns a null or failed track.
- `usePublish([mic, null])` publishes mic only.
- The local preview is hidden (`{localCameraTrack && ...}`).
- The MLLM is still started with `input_modalities=["text","image"]`; Agora simply has no image frames to forward. The model responds via audio only.
- No error is surfaced to the user for camera denial (treat as audio-only fallback).

## Related L1

- [02_architecture](../02_architecture.md) · [06_interfaces](../06_interfaces.md) · [07_gotchas](../07_gotchas.md)
