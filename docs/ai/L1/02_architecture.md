# 02 · Architecture

> Two co-located processes. The browser publishes both mic **and camera** into the RTC channel, then calls Next.js `/api/*`, which rewrites to the FastAPI agent backend. The backend builds a single OpenAI Realtime MLLM with `input_modalities=["text","image"]` so the model can also see the user.

## Topology

```
Browser (localhost:3000)
  │  publishes mic + camera via agora-rtc-react (useLocalCameraTrack, usePublish)
  │  fetch /api/*
  ▼
Next.js (web/)  ──rewrite──▶  Agent backend (server/, :8000)
                                 │  builds OpenAIRealtime MLLM via .with_mllm()
                                 │  input_modalities=["text","image"]
                                 ▼
                              Agora ConvoAI Cloud
                                 │  user speech → OpenAI Realtime (voice-to-voice, server_vad)
                                 │  user's published camera frames → image input
                                 │  agent speech → user's channel
                                 ▼
                              User hears realtime voice; RTM transcript + metrics → web UI
```

- **`web/`** — Next.js 16 / React 19 / TypeScript. Owns UI plus the RTC/RTM client lifecycle. Publishes mic (`useLocalMicrophoneTrack`) **and** camera (`useLocalCameraTrack`) via `usePublish([mic, camera])`. Shows a local camera preview. Calls only `/api/*`.
- **`server/`** — Python FastAPI (:8000). Owns Agora token generation and agent session lifecycle. SDK: `agora-agents>=2.3.0` (`import agora_agent`).
- No `llm/` service, no mock vendor, no public tunnel — the MLLM is a single cloud-hosted model.

## Request lifecycle

1. Browser `GET /api/get_config` → Next rewrites to backend `/get_config`; backend mints a Token007 from `AGORA_APP_ID` + `AGORA_APP_CERTIFICATE` and returns channel + UIDs.
2. Browser joins the RTC channel, publishes mic + camera, then logs in to RTM.
3. Browser `POST /api/startAgent`; backend validates `OPENAI_API_KEY`, builds the MLLM with `input_modalities=["text","image"]`, and starts an async agent session.
4. Agora routes user audio to OpenAI Realtime; the user's published camera frames arrive as image input. The model streams voice-to-voice back into the channel.
5. RTM delivers transcript + metrics to the web UI.
6. `POST /api/stopAgent { agentId }` ends the session. The client unpublishes both tracks and releases them.

## Why vision is always on

`input_modalities=["text","image"]` is hardcoded in `Agent.start()` — the recipe ships with vision always enabled. The model can reason over camera frames whenever the user publishes a camera track. If the user denies camera access, the browser falls back to mic-only, but the MLLM config still requests image input.

## Key abstractions

- **`Agent`** (`server/src/agent.py`) — async wrapper around `AgoraAgent`; owns the `AsyncAgora` client, env, and the in-memory `_sessions` map keyed by `agent_id`. Hardcodes `input_modalities=["text","image"]` in `start()`.
- **`build_realtime_mllm()`** (`server/src/realtime_config.py`) — constructs the `OpenAIRealtime` vendor with `turn_detection={"mode": "server_vad"}`, optional `greeting_message`, and optional `input_modalities`.
- **Rewrite proxy** (`web/next.config.ts`) — the only browser→backend boundary; no Next Route Handlers exist for agent/token logic.

## Tech decisions

- **Rewrites, not Route Handlers** — hides backend placement behind `/api/*` so the same client works locally and deployed (set `AGENT_BACKEND_URL`).
- **MLLM-owned turn detection** — `server_vad` lives on the vendor; no top-level `turn_detection` on `AgoraAgent(...)`.
- **Key validated at `start()`** — server boots without `OPENAI_API_KEY`; `/startAgent` returns 400 until it is set.
- **Camera release on unmount** — `ConversationComponent.tsx` calls `localCameraTrack?.stop(); localCameraTrack?.close()` in a `useEffect` cleanup and in `handleEndConversation` to turn the camera light off.

## Related Deep Dives

- [realtime_vision_mllm_config](L2/realtime_vision_mllm_config.md) — full `OpenAIRealtime` vendor build, vision modalities, and session options.
- [camera_lifecycle](L2/camera_lifecycle.md) — camera track acquire, publish, preview, and release in the web client.
- [session_lifecycle](L2/session_lifecycle.md) — browser orchestration of config + start/stop, RTC/RTM, transcript mapping.
