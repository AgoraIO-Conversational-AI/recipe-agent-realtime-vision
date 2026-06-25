# 08 · Security

> Trust boundaries, secret handling, and auth for the realtime vision recipe.

## Trust boundaries

| Hop                          | Auth                                                                 |
| ---------------------------- | -------------------------------------------------------------------- |
| Browser → agent backend      | None in local dev (the `/api/*` rewrite is same-origin).             |
| Agent backend → Agora cloud  | Token007, generated from `AGORA_APP_ID` + `AGORA_APP_CERTIFICATE`.   |
| Agora cloud → OpenAI Realtime| `OPENAI_API_KEY` (BYO), passed when the agent session starts.        |

## Secret handling

- **Server-only secrets:** `AGORA_APP_CERTIFICATE` and `OPENAI_API_KEY` live only in `server/.env.local` and never reach the browser. The browser receives a short-lived token, never the certificate or the OpenAI key.
- `server/.env.local` is gitignored; `server/.env.example` ships placeholders only.
- Tokens (`generate_convo_ai_token`) expire after 3600s and are minted per `get_config` call for a concrete non-zero UID.

## Camera access

- Camera access is requested client-side by `useLocalCameraTrack` in the browser. The browser's permission prompt controls whether camera frames are captured.
- If the user denies camera access, no camera track is published; the MLLM config still sets `input_modalities=["text","image"]` but Agora has no image frames to forward.
- The camera track is released (`stop()` + `close()`) on component unmount and explicit end-conversation to turn the camera indicator light off.
- No camera frames are stored by this recipe — they are forwarded live through Agora into the OpenAI Realtime session.

## CORS

The backend sets `CORSMiddleware` with `allow_origins=["*"]` — open by design for a local/dev recipe. **Lock this down to known origins before any production deployment.**

## Validation

- `Agent.start()` rejects empty `channel_name` and non-positive `agent_uid`/`user_uid` before issuing tokens or starting a session.
- `OPENAI_API_KEY` presence is enforced at `start()` (400 if missing).
- Route errors are sanitized: `_log_route_error` logs only non-`None` context; SDK exceptions map to 400/500 without leaking internals to the client beyond the message.

## Deployment notes

- Set `AGENT_BACKEND_URL` only to a backend you control; the rewrite forwards browser requests there verbatim.
- The published Docker image is **backend-only** (`:8000`); it does not bundle secrets.

## Related Deep Dives

- None.
