---
recipe_version: 1.0.0
recipe_status: experimental
extension_points:
  - id: api.routes
    name: Browser-facing API routes
  - id: agent.mllm-config
    name: OpenAIRealtime MLLM model, VAD, greeting, modalities, and session parameters
  - id: web.conversation-ui
    name: Conversation UI panels, camera preview, and controls
  - id: web.camera-lifecycle
    name: Camera track acquisition, publish, preview, and release
  - id: verification.contracts
    name: Contract, proxy, and local FastAPI smoke verification
invariants:
  - id: api.rewrite-boundary
    summary: Browser calls stay on /api/* and Next rewrites to FastAPI; no Route Handlers for agent/token logic.
  - id: secrets.server-only
    summary: Agora App Certificate and OPENAI_API_KEY stay in the Python backend.
  - id: mllm.single-vendor
    summary: A single OpenAIRealtime MLLM via .with_mllm() replaces cascading STT/LLM/TTS; no llm/ service, no tools.
  - id: mllm.vision-always-on
    summary: input_modalities=["text","image"] is hardcoded in Agent.start(); vision is always enabled in this recipe.
  - id: vad.mllm-owned
    summary: turn_detection (server_vad) is set on the vendor, not on AgoraAgent(...).
  - id: token.uid-concrete
    summary: Backend resolves missing, zero, or negative UIDs before issuing an RTC+RTM token.
  - id: camera.client-owned
    summary: Camera track acquisition (useLocalCameraTrack) and release happen in the web client; the backend is unaware.
stable_contracts:
  - id: env.required
    summary: AGORA_APP_ID, AGORA_APP_CERTIFICATE, and OPENAI_API_KEY are required; AGENT_BACKEND_URL is required by deployed web rewrites.
  - id: api.core-routes
    summary: GET /api/get_config, POST /api/startAgent, and POST /api/stopAgent remain the browser-facing contract.
  - id: response.envelope
    summary: Successful backend responses use { code, msg, data }.
---

# Recipe Contract

This base recipe defines the reusable surface for a Python-backed Agora Conversational AI **realtime vision** quickstart: a single OpenAI Realtime MLLM (voice-to-voice) with vision input (`input_modalities=["text","image"]`) behind a Next.js web client that publishes both mic and camera.

## Recipe Role

- Role: `base` recipe (self-contained, clone-and-run; no `Extends` pin).
- Target audience: developers building an ultra-low-latency voice-to-voice agent that can also see the user's camera, backed by Python FastAPI and Next.js.
- Reuse model: clone, bind project, set `OPENAI_API_KEY`, run, then customize MLLM behavior, camera handling, or browser UI.

## Recipe Scope

- Python FastAPI token generation and managed agent lifecycle.
- A single `OpenAIRealtime` MLLM attached via `.with_mllm()` with `input_modalities=["text","image"]` always enabled (no cascading vendors, no `llm/` service, no mock, no tunnel).
- Next.js browser UI with RTC audio + camera publish (`useLocalCameraTrack`, `usePublish([mic, camera])`), local camera preview, RTM transcript/metrics, connection status.
- Rewrite-only `/api/*` browser facade hiding backend placement.
- Contract, proxy, and local FastAPI smoke verification that need no live Agora calls.
- Not yet live-verified end-to-end against a real Agora + OpenAI Realtime session.

## Baseline Implementation Guidance

Use this repo's source and progressive disclosure docs as the starting point, then customize. Do not recreate the Agora ConvoAI integration from memory — vendor schemas, SDK builder fields, token behavior, and RTM details drift. Copy verified patterns from this repo.

## Extension Points

| ID | Surface | How to extend | Required follow-up |
| -- | ------- | ------------- | ------------------ |
| `api.routes` | `server/src/server.py`, `web/next.config.ts`, `web/src/services/api.ts` | Add FastAPI route, add rewrite, add browser fetch helper. | Extend `web/scripts/verify-api-contracts.ts`; add proxy/fastapi coverage if it belongs in local verification. |
| `agent.mllm-config` | `server/src/realtime_config.py`, `server/src/agent.py` | Change `OPENAI_MODEL`, `turn_detection`, `greeting_message`, or `input_modalities`. | Run `verify:backend` + `pytest tests`; document new env in `server/.env.example` (never add `PORT`). |
| `web.conversation-ui` | `web/src/components/*`, `web/src/lib/conversation.ts` | Customize pre-call, transcript, metrics, connection status, mic, camera preview, or visualizer UI. | Preserve RTC/RTM lifecycle ownership, camera track release on unmount, and transcript UID normalization. |
| `web.camera-lifecycle` | `web/src/components/ConversationComponent.tsx` | Adjust how the camera track is acquired, previewed, published, or released. | Ensure `localCameraTrack?.stop(); localCameraTrack?.close()` is called on unmount and end-conversation. |
| `verification.contracts` | `web/scripts/*.ts`, root `package.json` | Add checks for new browser/backend boundaries. | Keep checks runnable without live Agora credentials. |

## Invariants

- Browser code calls only `/api/get_config`, `/api/startAgent`, and `/api/stopAgent` for the default flow.
- Next.js owns `/api/*` through rewrites only; no `web/app/api/**/route.ts` for agent/token logic.
- FastAPI owns token generation, `AGORA_APP_CERTIFICATE`, `OPENAI_API_KEY`, and agent lifecycle.
- A single `OpenAIRealtime` MLLM handles the full voice-to-voice pipeline; `turn_detection` (`server_vad`) is vendor-owned.
- `input_modalities=["text","image"]` is hardcoded in `Agent.start()` — vision is always enabled in this recipe.
- The MLLM has no tools in this SDK; do not add tool calling here.
- The backend issues one RTC+RTM-capable token for a concrete non-zero UID.
- Camera track acquisition and release are entirely client-side (`useLocalCameraTrack`, `localCameraTrack.stop()/close()` on unmount and end-conversation).

## Stable Contracts

| Contract | Stable shape |
| -------- | ------------ |
| Required backend env | `AGORA_APP_ID`, `AGORA_APP_CERTIFICATE`, `OPENAI_API_KEY` |
| Optional backend env | `OPENAI_MODEL`, `AGENT_GREETING`, `PORT` (env only) |
| Required web deploy env | `AGENT_BACKEND_URL` |
| `GET /api/get_config` | Query `channel?`, `uid?`; returns `data.app_id`, `data.token`, `data.uid`, `data.channel_name`, `data.agent_uid`. |
| `POST /api/startAgent` | Body `{ channelName, rtcUid, userUid, parameters? }`; returns `data.agent_id`, `data.channel_name`, `data.status`. |
| `POST /api/stopAgent` | Body `{ agentId }`; returns `{ code: 0, msg: "success" }`. |
| Success envelope | `{ "code": 0, "msg": "success", "data": ... }` where the route has data. |
| Verification entry points | `bun run verify:web`, `bun run verify:backend`, `bun run verify:web:proxy`, `bun run verify:local:fastapi`, `bun run verify:local`. |

## Internal / Subject to Change

- Visual layout, component composition, Tailwind classes, and assets under `web/src/components/`.
- Camera preview placement and styling within `ConversationComponent.tsx`.
- Exact model name, VAD timing, voice, and greeting text, as long as they stay documented extension points.
- In-memory `Agent._sessions` details; the stable behavior is start by channel/user and stop by returned `agent_id`.
- Verification internals under `web/scripts/`; the stable surface is the root script names and what they assert.
- `agora-agents` SDK minor-version behavior; this recipe lower-bounds `>=2.3.0` but does not freeze every field.

## Related Progressive Disclosure Docs

- `L1/01_setup.md` — setup, env, and commands.
- `L1/02_architecture.md` — request flow, camera publish, and topology.
- `L1/05_workflows.md` — common modification workflows.
- `L1/06_interfaces.md` — route, rewrite, env, and MLLM contracts.
- `L1/L2/realtime_vision_mllm_config.md` — full MLLM config detail including vision modalities.
- `L1/L2/camera_lifecycle.md` — camera track acquire, publish, preview, and release.
- `L1/L2/session_lifecycle.md` — RTC/RTM/session orchestration.
