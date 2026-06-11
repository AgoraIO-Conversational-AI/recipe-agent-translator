# Agora Conversational AI — Translator Recipe (Python)

The **translator** recipe in the Agora Conversational AI recipes family.
Real-time speech translation: speak in a source language, the agent translates and
speaks back in the target language. Fully **zero-key** — OpenAI is Agora-managed
(no `OPENAI_API_KEY` required unless you bring your own account).

**Pipeline:** `DeepgramSTT(language=SOURCE_LANG)` → `OpenAI` (translate) → `MiniMaxTTS(voice=TTS_VOICE)`

## Prerequisites

- [Python 3.8+](https://www.python.org/)
- [Bun](https://bun.sh/)
- Agora App ID + App Certificate (the [Agora CLI](https://github.com/AgoraIO/cli) makes this easy)

## Run it

```bash
# 1. Install web deps + create the Python venv
bun run setup

# 2. Add Agora credentials (CLI), or edit server/.env.local by hand
agora login
agora project use <your-project>          # select which project to use
agora project env write server/.env.local # writes App ID + Certificate

# 3. (Optional) customise language pair in server/.env.local
#    SOURCE_LANG=es       # Deepgram language code for the speaker
#    TARGET_LANG=English  # language name used in the translation prompt
#    TTS_VOICE=English_captivating_female1  # MiniMax voice matching the target

# 4. Run backend + web
bun run dev
```

Open [http://localhost:3000](http://localhost:3000) → **Start Conversation** → speak in `SOURCE_LANG`.

## Architecture

```
Browser (localhost:3000)
  │  fetch /api/*
  ▼
Next.js  ──rewrite──▶  Agent backend  (server/, localhost:8000)
                          │  starts agent session (managed OpenAI vendor)
                          ▼
                       Agora ConvoAI Cloud
                          │  Deepgram STT (managed, SOURCE_LANG)
                          │  OpenAI translation (Agora-managed, keyless)
                          │  MiniMax TTS (managed, TTS_VOICE)
                          ▼
                       User hears translated speech
```

No separate `llm/` service — OpenAI is Agora-managed and requires no API key.
See [ARCHITECTURE.md](./ARCHITECTURE.md).

## Project structure

```
recipe-agent-translator/
├── server/   # Agent backend (:8000) — tokens + agent lifecycle, managed OpenAI
│   ├── src/{server.py, agent.py, translation_config.py}
│   ├── tests/test_translation_config.py
│   └── requirements.txt
├── web/      # Next.js frontend (:3000)
└── package.json
```

## Environment variables

Backend env file: [`server/.env.example`](server/.env.example).

| Variable | Required | Default | Notes |
| --- | :---: | :---: | --- |
| `AGORA_APP_ID` | ✅ | — | Agora Console → Project → App ID |
| `AGORA_APP_CERTIFICATE` | ✅ | — | Agora Console → Project → App Certificate |
| `SOURCE_LANG` | | `es` | Deepgram STT language code (speaker's language) |
| `TARGET_LANG` | | `English` | Language name used in the translation prompt |
| `TTS_VOICE` | | `English_captivating_female1` | MiniMax voice matching `TARGET_LANG` |
| `OPENAI_MODEL` | | `gpt-4o-mini` | OpenAI model for translation |
| `OPENAI_API_KEY` | | — | Optional — Agora manages the OpenAI key by default (keyless). Set only if your account requires it. |
| `AGENT_GREETING` | | built-in | Optional opening line override |

> Note: when you change `TARGET_LANG`, also pick a matching `TTS_VOICE` for
> that target language.

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

## Troubleshooting

| Problem | Fix |
| --- | --- |
| Agent starts but does not translate | Check `SOURCE_LANG` is a valid Deepgram BCP-47 code. |
| Wrong TTS voice / language | Set `TTS_VOICE` to a MiniMax voice matching your `TARGET_LANG`. |
| Local calls fail under a global proxy (Clash, etc.) | Configure your proxy to send `127.0.0.1`, `localhost`, and RFC-1918 ranges DIRECT. |

## License

MIT
