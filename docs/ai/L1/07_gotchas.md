# 07 · Gotchas

> Non-obvious pitfalls specific to the realtime vision recipe. Read before changing the agent, camera, env, or verify scripts.

## `OPENAI_API_KEY` is validated at agent start, not boot

The server boots **without** `OPENAI_API_KEY` (so `doctor`/contract checks work), but `POST /startAgent` returns **400** until the key is set. `Agent.__init__` raises only for missing `AGORA_APP_ID`/`AGORA_APP_CERTIFICATE`. Don't move the OpenAI check into `__init__`.

## Turn detection is MLLM-owned

`turn_detection={"mode": "server_vad"}` is set on the `OpenAIRealtime` vendor in `realtime_config.py`. **Do not** set a top-level `turn_detection` on `AgoraAgent(...)` when using `.with_mllm()` — it has no effect and is misleading.

## Vision is always on — `input_modalities` is hardcoded

`input_modalities=["text","image"]` is passed to `build_realtime_mllm()` directly in `Agent.start()`, not via env or request parameter. Vision cannot be toggled at runtime without code changes. If the user denies camera access, the MLLM still requests image input; Agora forwards whatever track is published (none if denied).

## Camera track must be released on unmount and end-conversation

`localCameraTrack?.stop()` and `localCameraTrack?.close()` must be called in both:
1. The `useEffect` cleanup in `ConversationComponent.tsx` (handles tab close / navigation / error unmount).
2. `handleEndConversation` (handles explicit end-call button).

Failing to call both leaves the camera indicator light on in the browser. Do not remove either release path.

## No tools

The `OpenAIRealtime` MLLM has no tool support in this SDK. Do not wire tool/function-calling here — use a cascading-vendor recipe if you need tools.

## No `llm/` service, no mock, no tunnel

Unlike custom-llm recipes, there is a single cloud MLLM. Do not reintroduce `llm/`, a mock LLM service, or a public tunnel.

## Do not put `PORT` in `server/.env.example`

`verify:local:fastapi` injects a random `PORT` and loads env with `load_dotenv(override=True)`. A `PORT` line in `.env.example` (copied to `.env.local`) would clobber the injected port and break the smoke test.

## Keep `/api/*` ownership in rewrites

Adding `web/app/api/**/route.ts` for agent/token logic breaks the boundary — `verify-api-contracts.ts` explicitly fails if a `route.ts` exists under `app/api`. Token logic belongs in `server/`.

## camelCase request fields

`StartAgentRequest` uses `channelName`, `rtcUid`, `userUid` (camelCase) to match the browser client. Renaming one side without the other breaks the contract tests.

## UID normalization in transcripts

`normalizeTranscript` maps `uid === '0'` to the local UID. Token issuance also rejects zero/negative UIDs and generates a concrete one. Preserve both — speaker mapping and tokens depend on concrete UIDs.

## Local calls under a global proxy

Global proxies (Clash, etc.) can break `localhost`/RFC-1918 traffic. Configure the proxy to send `127.0.0.1`, `localhost`, and private ranges DIRECT, or `socksio` (in `requirements.txt`) plus `all_proxy` to route the backend through SOCKS.

## Not yet live-verified end-to-end

This recipe passes static verification (compile, unit tests, web build) but has not been run against a live Agora + OpenAI Realtime session. Realtime image input over Agora is wired the same way as the standalone vision recipe but is unconfirmed end-to-end.

## Related Deep Dives

- [realtime_vision_mllm_config](L2/realtime_vision_mllm_config.md) — correct MLLM/VAD/vision wiring.
- [camera_lifecycle](L2/camera_lifecycle.md) — camera release paths in detail.
