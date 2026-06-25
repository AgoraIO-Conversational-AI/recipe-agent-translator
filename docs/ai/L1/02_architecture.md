# 02 · Architecture

> Two co-located processes. The browser talks only to Next.js `/api/*`, which rewrites to the FastAPI agent backend. The backend owns Agora tokens and the agent session, attaching a cascading STT → LLM → TTS pipeline using Agora-managed vendors.

## Topology

```
Browser (localhost:3000)
  │  fetch /api/*
  ▼
Next.js (web/)  ──rewrite──▶  Agent backend (server/, :8000)
                                 │  builds AgoraAgent with DeepgramSTT + OpenAI + MiniMaxTTS
                                 ▼
                              Agora ConvoAI Cloud
                                 │  user speech → Deepgram STT (language=SOURCE_LANG)
                                 │  transcript → OpenAI (translate to TARGET_LANG, keyless)
                                 │  translation → MiniMax TTS (voice=TTS_VOICE)
                                 ▼
                              User hears translated speech; RTM transcript + metrics → web UI
```

- **`web/`** — Next.js 16 / React 19 / TypeScript. Owns UI plus the RTC/RTM client lifecycle. Calls only `/api/*`.
- **`server/`** — Python FastAPI (:8000). Owns Agora token generation and agent session lifecycle. SDK: `agora-agents>=2.3.0` (`import agora_agent`).
- No `llm/` service, no mock vendor, no public tunnel — OpenAI is Agora-managed and keyless.

## Request lifecycle

1. Browser `GET /api/get_config` → Next rewrites to backend `/get_config`; backend mints a Token007 from `AGORA_APP_ID` + `AGORA_APP_CERTIFICATE` and returns channel + UIDs.
2. Browser joins the RTC channel, then `POST /api/startAgent`; backend builds the cascading vendor chain and starts an async agent session.
3. Agora runs Deepgram STT on user audio (in `SOURCE_LANG`), passes the transcript to OpenAI for translation into `TARGET_LANG`, and plays back the translated speech via MiniMax TTS (voice `TTS_VOICE`).
4. RTM delivers transcript + metrics to the web UI.
5. `POST /api/stopAgent { agentId }` ends the session.

## Why no `llm/` service

The translator recipe uses the **managed OpenAI vendor** (`agora_agent.agentkit.vendors.OpenAI`). Agora holds the OpenAI API key on its cloud — no separate LLM service is needed and no API key is required by default. An optional `OPENAI_API_KEY` env var lets you bring your own account if required.

## Key abstractions

- **`Agent`** (`server/src/agent.py`) — async wrapper around `AgoraAgent`; owns the `AsyncAgora` client, env, in-memory `_sessions` map keyed by `agent_id`, and the vendor chain construction.
- **`build_translation_system_messages()`** (`server/src/translation_config.py`) — pure function that produces the translation system prompt injected into every OpenAI call.
- **Rewrite proxy** (`web/next.config.ts`) — the only browser→backend boundary; no Next Route Handlers for agent/token logic.

## Tech decisions

- **Rewrites, not Route Handlers** — hides backend placement behind `/api/*` so the same client works locally and deployed (set `AGENT_BACKEND_URL`).
- **Cascading vendors, not MLLM** — `DeepgramSTT.with_llm(OpenAI).with_tts(MiniMaxTTS)` gives controllable language codes and independent vendor selection. Turn detection is on `AgoraAgent(...)`, not MLLM-owned.
- **VAD on `AgoraAgent`** — `turn_detection` config with `speech_threshold`, `start_of_speech`, and `end_of_speech` is set directly on `AgoraAgent(...)`.
- **Zero-key by default** — `OPENAI_API_KEY` is optional; `None` is passed to `OpenAI(api_key=None)`, which signals Agora-managed.

## Related Deep Dives

- [translation_config.md](L2/translation_config.md) — full vendor chain, VAD config, system prompt, and session parameters.
- [session_lifecycle.md](L2/session_lifecycle.md) — browser orchestration of config + start/stop, RTC/RTM, transcript mapping.
