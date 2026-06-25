# 03 · Code Map

> Where things live. Two top-level modules: `web/` (Next.js client) and `server/` (FastAPI backend). Orchestration is in the root `package.json`.

## Root

| Path                  | Responsibility                                                        |
| --------------------- | --------------------------------------------------------------------- |
| `package.json`        | Bun workspace; `setup`, `dev`, `doctor*`, `verify*`, `clean` scripts. |
| `README.md`           | Setup, run modes, env, how it works, troubleshooting.                 |
| `ARCHITECTURE.md`     | System shape, vision input, and component boundaries.                 |
| `AGENTS.md`           | Coding-agent handbook + How to Load / Git Conventions / Doc Commands. |
| `Dockerfile`          | Backend-only image (`:8000`).                                         |
| `.github/workflows/`  | `ci.yml` (backend pytest matrix + web verify), `docker.yml`, `nightly.yml`. |

## `server/` — FastAPI backend (:8000)

| Path                              | Responsibility                                                                |
| --------------------------------- | ----------------------------------------------------------------------------- |
| `src/server.py`                   | FastAPI app, CORS, route handlers, error mapping, uvicorn entrypoint.         |
| `src/agent.py`                    | `Agent` class: `AsyncAgora` client, `start()`/`stop()`, `_sessions`. Hardcodes `input_modalities=["text","image"]` in `start()`. |
| `src/realtime_config.py`          | `build_realtime_mllm()` — constructs the `OpenAIRealtime` vendor.             |
| `scripts/run_fake_server.py`      | Boots `server.app` with a `FakeAgent` for the local FastAPI smoke test.       |
| `tests/test_agent_construction.py`| Builds the real `AgoraAgent`, fakes the SDK session, asserts shape.           |
| `tests/test_realtime_config.py`   | Asserts `OpenAIRealtime` config including `input_modalities=["text","image"]`.|
| `tests/conftest.py`               | `fake_env` fixture + `FakeAgent`; no cloud, no real creds.                    |
| `.env.example`                    | Env template (do not add `PORT`).                                             |
| `requirements*.txt`               | Runtime + dev (pytest) deps.                                                  |

## `server/src/server.py` routes

- `GET /get_config` — token + channel/UID config.
- `POST /startAgent` — start the realtime vision agent session.
- `POST /stopAgent` — stop by `agent_id`.

## `web/` — Next.js client (:3000)

| Path                                      | Responsibility                                                                       |
| ----------------------------------------- | ------------------------------------------------------------------------------------ |
| `next.config.ts`                          | `/api/*` rewrites to `AGENT_BACKEND_URL`; strict mode; Turbopack root.               |
| `src/services/api.ts`                     | Browser API client: `getConfig`, `startAgent`, `stopAgent`.                          |
| `src/lib/conversation.ts`                 | Transcript normalization, timestamp/UID mapping, visualizer state.                   |
| `src/lib/agora.ts`                        | `DEFAULT_AGENT_UID` constant.                                                        |
| `src/components/LandingPage.tsx`          | Conversation entry: config fetch, agent start, RTM login, teardown.                  |
| `src/components/ConversationComponent.tsx`| RTC join, mic + camera publish, local camera preview, transcript/metrics/state listeners, camera release on unmount. |
| `src/components/Quickstart*.tsx`          | Pre-call, transcript, metrics, layout panels.                                        |
| `scripts/verify-api-contracts.ts`         | Asserts rewrites + client paths + response envelope (no network).                    |
| `scripts/verify-local-proxy.ts`           | Stub backend; proxies `/api/*` through the rewrite map.                              |
| `scripts/verify-local-fastapi.ts`         | Spawns real FastAPI with `FakeAgent`; proxies routes end-to-end.                     |
| `scripts/doctor.ts`                       | Web prerequisite check.                                                              |

## Related Deep Dives

- None. For runtime flow see [02_architecture](02_architecture.md); for contracts see [06_interfaces](06_interfaces.md).
