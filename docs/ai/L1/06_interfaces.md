# 06 · Interfaces

> Boundary contracts: backend routes, the `/api/*` rewrite map, env vars, the response envelope, and the vendor chain config.

## Backend routes (port 8000)

The browser calls these as `/api/<name>`; Next rewrites to the backend `/<name>`.

### `GET /get_config`

- Query (optional): `channel?: string`, `uid?: int` (≤ 0 or missing → backend generates one).
- Returns `data`: `{ app_id, token, uid (string), channel_name, agent_uid (string) }`.
- Token is a Token007 RTC+RTM token, expiry 3600s, for a concrete non-zero UID.

### `POST /startAgent`

- Body: `{ channelName: string, rtcUid: int, userUid: int, parameters?: object }`.
  - `parameters.output_audio_codec?: string` is the only honored parameter field.
- Returns `data`: `{ agent_id, channel_name, status: "started" }`.
- 400 if `channelName`/`rtcUid`/`userUid` invalid.

### `POST /stopAgent`

- Body: `{ agentId: string }`.
- Returns `{ code: 0, msg: "success" }` (no `data`).

## Response envelope

```json
{ "code": 0, "msg": "success", "data": { } }
```

`data` omitted when the route has no payload. Non-zero `code` or missing `data` = error on the client side.

## Rewrite map (`web/next.config.ts`)

| Browser path        | Backend destination |
| ------------------- | ------------------- |
| `/api/get_config`   | `/get_config`       |
| `/api/startAgent`   | `/startAgent`       |
| `/api/stopAgent`    | `/stopAgent`        |

`rewrites()` returns `[]` when `AGENT_BACKEND_URL` is unset. The contract is asserted by `verify-api-contracts.ts` and exercised by `verify-local-proxy.ts`.

## Browser API client (`web/src/services/api.ts`)

- `getConfig({ channel?, uid? }) → GetConfigResponse`
- `startAgent(channelName, rtcUid, userUid) → agent_id`
- `stopAgent(agentId) → void`

## Environment variables

| Variable                | Scope              | Required | Default                         |
| ----------------------- | ------------------ | :------: | ------------------------------- |
| `AGORA_APP_ID`          | backend            |    ✅    | —                               |
| `AGORA_APP_CERTIFICATE` | backend            |    ✅    | —                               |
| `SOURCE_LANG`           | backend            |          | `es`                            |
| `TARGET_LANG`           | backend            |          | `English`                       |
| `TTS_VOICE`             | backend            |          | `English_captivating_female1`   |
| `OPENAI_MODEL`          | backend            |          | `gpt-4o-mini`                   |
| `OPENAI_API_KEY`        | backend            |          | — (optional, Agora-managed)     |
| `AGENT_GREETING`        | backend            |          | built-in line                   |
| `AGENT_BACKEND_URL`     | web (deploy)       |  ✅\*    | `http://localhost:8000` (dev)   |
| `PORT`                  | backend (env only) |          | `8000` — do **not** put in `.env.example` |

\* Required wherever the web app is deployed; rewrites are empty without it.

## Vendor chain config (`agent.py`)

`Agent.start()` builds the pipeline inline:

| Vendor       | Class         | Key params                                                    |
| ------------ | ------------- | ------------------------------------------------------------- |
| STT          | `DeepgramSTT` | `model="nova-3"`, `language=SOURCE_LANG`                      |
| LLM          | `OpenAI`      | `api_key=OPENAI_API_KEY` (None = Agora-managed), `model=OPENAI_MODEL`, `system_messages=[translate to TARGET_LANG]`, `temperature=0.3` |
| TTS          | `MiniMaxTTS`  | `model="speech_2_6_turbo"`, `voice_id=TTS_VOICE`             |

VAD is set on `AgoraAgent(...)` via `turn_detection` — not on the LLM vendor.

## Related Deep Dives

- [translation_config.md](L2/translation_config.md) — full vendor chain, VAD config, session parameters, and system prompt detail.
