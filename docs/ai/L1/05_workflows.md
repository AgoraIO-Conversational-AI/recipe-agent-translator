# 05 · Workflows

> Step-by-step guides for the common changes in this recipe. Each ends with the narrowest verify command to run.

## Add or change a browser-facing route

1. Add the FastAPI handler in `server/src/server.py` (return the `{ code, msg, data }` envelope).
2. Add the `/api/<name>` → `/<name>` mapping in `web/next.config.ts` `rewrites()`.
3. Add a client helper in `web/src/services/api.ts`.
4. Extend `web/scripts/verify-api-contracts.ts` with the new path + envelope assertions.
5. Verify: `bun run verify:web` (and `bun run verify:local:fastapi` if it should go through the real backend).

## Change the language pair (SOURCE_LANG / TARGET_LANG / TTS_VOICE)

1. Set `SOURCE_LANG` to a valid Deepgram BCP-47 code (e.g. `fr` for French).
2. Set `TARGET_LANG` to the target language name as it should appear in the translation prompt (e.g. `French`).
3. Set `TTS_VOICE` to a MiniMax voice for that target language (e.g. `French_expressivefemale`).
4. Verify: `bun run verify:backend` (compile) + `cd server && pytest tests -v`.

> All three must match. See [07_gotchas](07_gotchas.md) for the coupling rules.

## Change the translation prompt or model

1. Prompt: edit `build_translation_system_messages()` in `server/src/translation_config.py`.
2. Model: set `OPENAI_MODEL` (default `gpt-4o-mini`).
3. Temperature: edit `temperature=0.3` in `Agent.start()` in `server/src/agent.py`.
4. Verify: `cd server && pytest tests -v`.

## Change the greeting

1. Set `AGENT_GREETING` in `server/.env.local`, or edit the default in `server/src/agent.py`.
2. Verify: `bun run verify:backend`.

## Adjust session parameters (codec, scenario)

1. Edit the `parameters` dict in `Agent.start()` (`audio_scenario`, `data_channel`, `enable_metrics`, etc.). `output_audio_codec` is also accepted per-request via `parameters` on `POST /startAgent`.
2. Verify: `bun run verify:local:fastapi`.

## Run / debug locally

```bash
bun run dev              # both processes
bun run doctor:local     # check creds + .env.local before a live call
```

## Verify before finishing

| Change touches…              | Run                                                                 |
| ---------------------------- | ------------------------------------------------------------------- |
| Web only                     | `bun run verify:web`                                                |
| Backend logic / vendor config| `bun run verify:backend` + `cd server && pytest tests -v`           |
| Route/proxy boundary         | `bun run verify:web:proxy` and/or `bun run verify:local:fastapi`    |
| Anything end-to-end (local)  | `bun run verify:local`                                              |

## Deploy

1. Deploy `web/` as a Next.js app.
2. Deploy `server/` (or any reachable FastAPI host); the published backend-only image is `ghcr.io/AgoraIO-Conversational-AI/recipe-agent-translator` on `v*` tags.
3. Set `AGENT_BACKEND_URL` in the web deployment so rewrites reach the backend.

## Related Deep Dives

- [translation_config.md](L2/translation_config.md) — vendor chain and session options.
- [session_lifecycle.md](L2/session_lifecycle.md) — client-side join/renewal/teardown.
