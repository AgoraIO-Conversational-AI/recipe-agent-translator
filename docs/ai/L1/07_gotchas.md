# 07 · Gotchas

> Non-obvious pitfalls specific to the translator recipe. Read before changing language config, vendor chain, env, or verify scripts.

## Language pair must be changed as a unit

`SOURCE_LANG`, `TARGET_LANG`, and `TTS_VOICE` are coupled:

- `SOURCE_LANG` is a Deepgram BCP-47 code (e.g. `es`, `fr`). Setting an invalid code causes Deepgram to fail silently or use the wrong STT model.
- `TARGET_LANG` is a plain language name (e.g. `English`, `French`) injected into the translation system prompt in `translation_config.py`. It must be human-readable, not a BCP-47 code.
- `TTS_VOICE` must be a MiniMax voice for `TARGET_LANG`. A mismatched voice produces output in the wrong language or with broken accent.

Change all three together whenever you switch the translation target.

## `OPENAI_API_KEY` is optional (Agora-managed by default)

Unlike the realtime recipe, this recipe is **zero-key**. `OPENAI_API_KEY` is passed as `None` to `OpenAI(api_key=None)` when not set, signaling that Agora manages the key. The server boots and agents start without it. Set it only if your Agora account requires a BYO key.

## VAD is on `AgoraAgent`, not the LLM vendor

`turn_detection` (with `speech_threshold`, `start_of_speech`/`end_of_speech` VAD config) is set on `AgoraAgent(...)`. This is the **opposite** of the realtime recipe where VAD is MLLM-owned. Do not move `turn_detection` into the `OpenAI` vendor — it belongs on the agent.

## No MLLM — this is a cascading-vendor recipe

Do not substitute `.with_mllm()` for the cascading `.with_stt().with_llm().with_tts()` chain. The realtime MLLM (`OpenAIRealtime`) does not support separate language STT or TTS vendor selection.

## Do not reintroduce `llm/` or `CustomLLM`

The translator recipe is fully keyless with the managed `OpenAI` vendor. There is no `llm/` service, no mock LLM, and no tunnel needed.

## Do not put `PORT` in `server/.env.example`

`verify:local:fastapi` injects a random `PORT` and loads env with `load_dotenv(override=True)`. A `PORT` line in `.env.example` (copied to `.env.local`) would clobber the injected port and break the smoke test.

## Keep `/api/*` ownership in rewrites

Adding `web/app/api/**/route.ts` for agent/token logic breaks the boundary — `verify-api-contracts.ts` explicitly fails if a `route.ts` exists under `app/api`. Token logic belongs in `server/`.

## camelCase request fields

`StartAgentRequest` uses `channelName`, `rtcUid`, `userUid` (camelCase) to match the browser client. Renaming one side without the other breaks the contract tests.

## UID normalization in transcripts

`normalizeTranscript` maps `uid === '0'` to the local UID. Token issuance also rejects zero/negative UIDs and generates a concrete one. Preserve both — speaker mapping and tokens depend on concrete UIDs.

## Local calls under a global proxy

Global proxies (Clash, etc.) can break `localhost`/RFC-1918 traffic. Configure the proxy to send `127.0.0.1`, `localhost`, and private ranges DIRECT, or use `socksio` (in `requirements.txt`) with `all_proxy` to route through SOCKS.

## Related Deep Dives

- [translation_config.md](L2/translation_config.md) — correct vendor chain and VAD wiring.
