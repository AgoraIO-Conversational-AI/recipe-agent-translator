---
recipe_version: 1.0.0
recipe_status: experimental
extension_points:
  - id: api.routes
    name: Browser-facing API routes
  - id: agent.vendor-config
    name: Vendor chain (STT language, LLM model/prompt, TTS voice), VAD, greeting, and session parameters
  - id: web.conversation-ui
    name: Conversation UI panels and controls
  - id: verification.contracts
    name: Contract, proxy, and local FastAPI smoke verification
invariants:
  - id: api.rewrite-boundary
    summary: Browser calls stay on /api/* and Next rewrites to FastAPI; no Route Handlers for agent/token logic.
  - id: secrets.server-only
    summary: Agora App Certificate stays in the Python backend; OPENAI_API_KEY is optional and also server-only if set.
  - id: pipeline.cascading-vendors
    summary: The pipeline is DeepgramSTT → OpenAI → MiniMaxTTS via .with_stt().with_llm().with_tts(); no MLLM, no llm/ service.
  - id: vad.agent-owned
    summary: turn_detection (VAD config) is set on AgoraAgent(...), not on the OpenAI vendor.
  - id: token.uid-concrete
    summary: Backend resolves missing, zero, or negative UIDs before issuing an RTC+RTM token.
  - id: openai.keyless-default
    summary: OPENAI_API_KEY is optional; api_key=None signals Agora-managed (zero-key). The server boots without it.
stable_contracts:
  - id: env.required
    summary: AGORA_APP_ID and AGORA_APP_CERTIFICATE are required; AGENT_BACKEND_URL is required by deployed web rewrites.
  - id: api.core-routes
    summary: GET /api/get_config, POST /api/startAgent, and POST /api/stopAgent remain the browser-facing contract.
  - id: response.envelope
    summary: Successful backend responses use { code, msg, data }.
  - id: language-pair.triplet
    summary: SOURCE_LANG (Deepgram BCP-47), TARGET_LANG (prompt name), and TTS_VOICE (MiniMax voice) must be changed together.
---

# Recipe Contract

This base recipe defines the reusable surface for a Python-backed Agora Conversational AI **translator** quickstart: a cascading DeepgramSTT → OpenAI → MiniMaxTTS pipeline behind a Next.js web client.

## Recipe Role

- Role: `base` recipe (self-contained, clone-and-run; no `Extends` pin).
- Target audience: developers building a real-time speech translation agent with a Python FastAPI backend and Next.js web client.
- Reuse model: clone, bind project (Agora App ID + Certificate), run zero-key out of the box, then customize the language pair or pipeline behavior.

## Recipe Scope

- Python FastAPI token generation and managed agent lifecycle.
- A cascading `DeepgramSTT → OpenAI → MiniMaxTTS` pipeline via `agora_agent` `.with_stt().with_llm().with_tts()` (no MLLM, no `llm/` service, no tunnel).
- Agora-managed OpenAI (zero-key default); optional `OPENAI_API_KEY` for BYO accounts.
- Configurable language pair via `SOURCE_LANG` / `TARGET_LANG` / `TTS_VOICE`.
- Next.js browser UI with RTC audio, RTM transcript/metrics, connection status.
- Rewrite-only `/api/*` browser facade hiding backend placement.
- Contract, proxy, and local FastAPI smoke verification that need no live Agora calls.

## Baseline Implementation Guidance

Use this repo's source and progressive disclosure docs as the starting point, then customize. Do not recreate the Agora ConvoAI integration from memory — vendor schemas, SDK builder fields, token behavior, and RTM details drift. Copy verified patterns from this repo.

## Extension Points

| ID | Surface | How to extend | Required follow-up |
| -- | ------- | ------------- | ------------------ |
| `api.routes` | `server/src/server.py`, `web/next.config.ts`, `web/src/services/api.ts` | Add FastAPI route, add rewrite, add browser fetch helper. | Extend `web/scripts/verify-api-contracts.ts`; add proxy/fastapi coverage if it belongs in local verification. |
| `agent.vendor-config` | `server/src/agent.py`, `server/src/translation_config.py` | Change `SOURCE_LANG`, `TARGET_LANG`, `TTS_VOICE`, `OPENAI_MODEL`, `temperature`, `build_translation_system_messages`, `turn_detection`, or session `parameters`. | Run `verify:backend` + `pytest tests`; update `server/.env.example` if adding env vars (never add `PORT`). Change `TTS_VOICE` when changing `TARGET_LANG`. |
| `web.conversation-ui` | `web/src/components/*`, `web/src/lib/conversation.ts` | Customize pre-call, transcript, metrics, connection status, mic, or visualizer UI. | Preserve RTC/RTM lifecycle ownership and transcript UID normalization. |
| `verification.contracts` | `web/scripts/*.ts`, root `package.json` | Add checks for new browser/backend boundaries. | Keep checks runnable without live Agora credentials. |

## Invariants

- Browser code calls only `/api/get_config`, `/api/startAgent`, and `/api/stopAgent` for the default flow.
- Next.js owns `/api/*` through rewrites only; no `web/app/api/**/route.ts` for agent/token logic.
- FastAPI owns token generation, `AGORA_APP_CERTIFICATE`, and agent lifecycle.
- The cascading vendor chain (`DeepgramSTT → OpenAI → MiniMaxTTS`) is the full pipeline; do not substitute an MLLM or reintroduce `llm/`.
- `turn_detection` (VAD) is set on `AgoraAgent(...)`, not on the `OpenAI` vendor.
- The backend issues one RTC+RTM-capable token for a concrete non-zero UID.
- `OPENAI_API_KEY=None` is valid and means Agora-managed; the server starts without it.

## Stable Contracts

| Contract | Stable shape |
| -------- | ------------ |
| Required backend env | `AGORA_APP_ID`, `AGORA_APP_CERTIFICATE` |
| Optional backend env | `SOURCE_LANG`, `TARGET_LANG`, `TTS_VOICE`, `OPENAI_MODEL`, `OPENAI_API_KEY`, `AGENT_GREETING`, `PORT` (env only) |
| Required web deploy env | `AGENT_BACKEND_URL` |
| `GET /api/get_config` | Query `channel?`, `uid?`; returns `data.app_id`, `data.token`, `data.uid`, `data.channel_name`, `data.agent_uid`. |
| `POST /api/startAgent` | Body `{ channelName, rtcUid, userUid, parameters? }`; returns `data.agent_id`, `data.channel_name`, `data.status`. |
| `POST /api/stopAgent` | Body `{ agentId }`; returns `{ code: 0, msg: "success" }`. |
| Success envelope | `{ "code": 0, "msg": "success", "data": ... }` where the route has data. |
| Verification entry points | `bun run verify:web`, `bun run verify:backend`, `bun run verify:web:proxy`, `bun run verify:local:fastapi`, `bun run verify:local`. |

## Internal / Subject to Change

- Visual layout, component composition, Tailwind classes, and assets under `web/src/components/`.
- Exact model name, voice ID, temperature, and VAD timing, as long as they stay documented extension points.
- In-memory `Agent._sessions` details; the stable behavior is start by channel/user and stop by returned `agent_id`.
- Verification internals under `web/scripts/`; the stable surface is the root script names and what they assert.
- `agora-agents` SDK minor-version behavior; this recipe lower-bounds `>=2.3.0` but does not freeze every field.

## Related Progressive Disclosure Docs

- `L1/01_setup.md` — setup, env, and commands.
- `L1/02_architecture.md` — request flow and topology.
- `L1/05_workflows.md` — common modification workflows.
- `L1/06_interfaces.md` — route, rewrite, env, and vendor chain contracts.
- `L1/L2/translation_config.md` — full vendor chain, VAD, and prompt detail.
- `L1/L2/session_lifecycle.md` — RTC/RTM/session orchestration.
