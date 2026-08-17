# Agora Conversational AI — Realtime Vision Recipe (Python)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Python](https://img.shields.io/badge/python-%3E%3D3.10-blue)](https://www.python.org/)
[![Bun](https://img.shields.io/badge/bun-latest-black)](https://bun.sh/)

The **realtime vision** recipe in the Agora Conversational AI recipes family.
Voice-to-voice conversation using a single **OpenAI Realtime** MLLM — no separate
STT, LLM, or TTS — that can **also see your camera**. Speak; the agent responds in
natural speech with ultra-low latency, and because the web client publishes your
camera track with `input_modalities=["text","image"]`, you can also **ask it what
it sees**.

**NOT zero-key** — `OPENAI_API_KEY` with Realtime API access is required.

> **Not yet live-verified.** This recipe passes static verification (compile,
> unit tests, web build) but has not been run against a live Agora + OpenAI
> Realtime session in our setup. Realtime image input over Agora is wired the same
> way as the standalone vision recipe but is unconfirmed end-to-end here.

**Pipeline:** `OpenAIRealtime` MLLM via `.with_mllm()` (server_vad turn detection,
`input_modalities=["text","image"]`)

## Prerequisites

- [Python 3.10+](https://www.python.org/)
- [Bun](https://bun.sh/)
- [Agora CLI](https://github.com/AgoraIO/cli) — makes generating an App ID + App Certificate easy
- **OpenAI API key with Realtime API access** — set as `OPENAI_API_KEY` in `server/.env.local`
- A browser that grants camera access (allow when prompted)

The same commands work on macOS, Linux, and Windows. On macOS/Linux, setup uses
`python3`; on Windows, it uses the Python launcher (`py`) or `python`. WSL and
virtualenv activation are not required.

## Run It

```bash
# 1. Install web deps + create the Python venv
bun run setup

# 2. Add Agora credentials (CLI), or edit server/.env.local by hand
agora login
agora project use <your-project>          # select which project to use
agora project env write server/.env.local # writes App ID + Certificate

# 3. Add your OpenAI Realtime API key to server/.env.local
#    OPENAI_API_KEY=sk-...   (required — OpenAI Realtime access)
#    OPENAI_MODEL=gpt-4o-realtime-preview  (optional, this is the default)

# 4. Run backend + web
bun run dev
```

Open [http://localhost:3000](http://localhost:3000) → **Start Conversation** →
allow camera access → speak, then ask "what do you see?"

### Working from a clone

If you cloned this repo (rather than scaffolding via the Agora CLI), the steps
above are complete as written: `bun run setup` creates the Python venv and
installs web dependencies, then `bun run dev` brings up both services. You
still need Agora credentials and `OPENAI_API_KEY` in `server/.env.local` before
a conversation can connect.

Services:

- Frontend — http://localhost:3000
- Backend — http://localhost:8000
- Mock LLM — N/A (single OpenAI Realtime MLLM, no mock service)
- API docs — http://localhost:8000/docs

## Deploy

Deploy `web` (Next.js) and `server` (a reachable FastAPI backend). Set
`AGENT_BACKEND_URL` in the web deployment so the Next rewrites reach the backend.

A backend-only Docker image is published to
`ghcr.io/AgoraIO-Conversational-AI/recipe-agent-realtime-vision` on `v*` tags.
It exposes **BACKEND-ONLY** (:8000). No separate service is needed.

## Environment variables

Backend env file: [`server/.env.example`](server/.env.example).

| Variable | Required | Default | Notes |
| --- | :---: | :---: | --- |
| `AGORA_APP_ID` | ✅ | — | Agora Console → Project → App ID |
| `AGORA_APP_CERTIFICATE` | ✅ | — | Agora Console → Project → App Certificate |
| `OPENAI_API_KEY` | ✅ | — | BYO OpenAI key with Realtime API access. Validated at agent start. |
| `OPENAI_MODEL` | | `gpt-4o-realtime-preview` | OpenAI Realtime model name |
| `AGENT_GREETING` | | built-in | Optional opening line override |

## Commands

```bash
bun run setup            # install web deps + create server/ venv
bun run dev              # run backend (:8000) + web (:3000)

bun run doctor           # prerequisite check (no creds needed)
bun run doctor:local     # + .env.local + credentials checks

bun run verify           # web-only gate (no Agora creds needed)
bun run verify:local     # full local gate: backend compile + smoke tests + web build
bun run clean            # remove venvs and build artifacts
```

Tests run standalone (no Agora cloud needed): `pytest` in `server/`, plus
`bun run verify` in `web/`. CI runs them on Linux/macOS/Windows × Python 3.10 & 3.13.

## Architecture

```
Browser (localhost:3000)
  │  publishes mic + camera via agora-rtc-react
  │  fetch /api/*
  ▼
Next.js  ──rewrite──▶  Agent backend  (server/, localhost:8000)
                          │  starts agent session (OpenAIRealtime MLLM,
                          │  input_modalities=["text","image"])
                          ▼
                       Agora ConvoAI Cloud
                          │  OpenAI Realtime (voice-to-voice, server_vad)
                          │  + user's published camera frames as image input
                          ▼
                       User hears realtime voice response
```

No cascading STT/LLM/TTS vendors. No `llm/` service.
See [ARCHITECTURE.md](./ARCHITECTURE.md).

## What You Get

- A **Next.js** web client (:3000) that publishes both mic and camera using
  `agora-rtc-react` (`useLocalCameraTrack`, `usePublish([mic, camera])`) and
  shows a small local camera preview. It only ever calls `/api/*`.
- A **FastAPI** agent backend (:8000) that owns Agora token generation and the agent session lifecycle.
- The `/api/get_config` · `/api/startAgent` · `/api/stopAgent` contract between the web client and the backend (Next rewrites, no Route Handlers).
- **OpenAI Realtime MLLM** attached via `.with_mllm()` — replaces the cascading STT→LLM→TTS with a single voice-to-voice model.
- **Vision input** via `input_modalities=["text","image"]` — the user's published camera frames are forwarded to the realtime model.
- **Server-side VAD** (`server_vad`) turn detection — owned by the MLLM, no top-level cascading VAD config needed.
- **BYO key** — `OPENAI_API_KEY` is required; validated at agent start.

## How It Works

1. The browser publishes both mic audio and the camera track into an Agora RTC channel.
2. The browser calls `/api/get_config`, which Next rewrites to the backend; the
   backend mints an Agora token from `AGORA_APP_ID` + `AGORA_APP_CERTIFICATE`.
3. The browser joins the RTC channel, then calls `/api/startAgent`; the backend
   validates `OPENAI_API_KEY` and starts an agent session using `OpenAIRealtime`
   with `input_modalities=["text","image"]`.
4. The user speaks. Agora routes audio to the OpenAI Realtime endpoint, and also
   captures the user's **published camera track**, forwarding frames as image input.
5. OpenAI Realtime processes voice-to-voice (and can reason over the camera image)
   and streams the response audio back.
6. The agent's voice plays in the channel. RTM transcript + metrics arrive in the web UI.
7. `/api/stopAgent` ends the session.

On the web side, `useLocalCameraTrack` obtains the camera stream and
`usePublish([mic, camera])` sends both tracks into the channel. A small local
preview shows the user what the agent sees, and the camera track is released on
unmount.

## Repo Map

- `web/` — Next.js frontend (:3000); RTC/RTM lifecycle and UI, publishes mic + camera.
- `server/` — FastAPI agent backend (:8000); Agora tokens + agent lifecycle, OpenAI Realtime MLLM with vision input.
- `ARCHITECTURE.md` — system shape and component boundaries.
- `AGENTS.md` — guide for coding agents working in this repo.

## Troubleshooting

| Problem | Fix |
| --- | --- |
| `/startAgent` returns 400 | Check `OPENAI_API_KEY` is set and has OpenAI Realtime API access. |
| Agent starts but no audio | Ensure the model (`OPENAI_MODEL`) supports realtime voice. |
| No camera preview appears | Allow camera access in the browser when prompted. |
| Agent does not react to the camera | Confirm the camera track published and that the realtime model accepts image input. |
| Local calls fail under a global proxy (Clash, etc.) | Configure your proxy to send `127.0.0.1`, `localhost`, and RFC-1918 ranges DIRECT. |

## More Docs

- [ARCHITECTURE.md](./ARCHITECTURE.md)
- [AGENTS.md](./AGENTS.md)

## License

Released under the [MIT License](./LICENSE).
