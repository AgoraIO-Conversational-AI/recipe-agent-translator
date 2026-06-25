# 01 · Setup

> Install dependencies, configure env, and run the translator recipe locally. This recipe is **zero-key** by default — only `AGORA_APP_ID` and `AGORA_APP_CERTIFICATE` are required.

## Prerequisites

- Python 3.10+ (backend runs on 3.10 and 3.13 in CI)
- [Bun](https://bun.sh/) (runs the web app and orchestration scripts)
- [Agora CLI](https://github.com/AgoraIO/cli) (optional; easiest way to mint App ID + Certificate)

## Install

```bash
bun run setup            # installs web deps + creates server/ venv from requirements.txt
```

`setup` runs `setup:env` (copies `server/.env.example` → `server/.env.local` if missing), `setup:server` (recreates `server/venv`, installs `requirements.txt`), and `setup:web` (`bun install`).

## Configure env

Backend env file is `server/.env.local` (template: `server/.env.example`).

| Variable                | Required | Default                         | Notes                                                |
| ----------------------- | :------: | ------------------------------- | ---------------------------------------------------- |
| `AGORA_APP_ID`          |    ✅    | —                               | Agora Console → Project → App ID                     |
| `AGORA_APP_CERTIFICATE` |    ✅    | —                               | Agora Console → Project → App Certificate            |
| `SOURCE_LANG`           |          | `es`                            | Deepgram BCP-47 language code for the speaker        |
| `TARGET_LANG`           |          | `English`                       | Language name used in the translation system prompt  |
| `TTS_VOICE`             |          | `English_captivating_female1`   | MiniMax voice matching `TARGET_LANG`                 |
| `OPENAI_MODEL`          |          | `gpt-4o-mini`                   | OpenAI model for translation (Agora-managed)         |
| `OPENAI_API_KEY`        |          | —                               | BYO only — Agora manages the OpenAI key by default   |
| `AGENT_GREETING`        |          | built-in line                   | Optional opening utterance override                  |

Fill credentials via the Agora CLI or by hand:

```bash
agora login
agora project use <your-project>
agora project env write server/.env.local   # writes App ID + Certificate
# then optionally set SOURCE_LANG, TARGET_LANG, TTS_VOICE in server/.env.local
```

> When changing `TARGET_LANG`, also set `TTS_VOICE` to a MiniMax voice for that language. See [07_gotchas](07_gotchas.md).

> Do **not** add `PORT` to `server/.env.example` — see [07_gotchas](07_gotchas.md).

## Run

```bash
bun run dev              # backend (:8000) + web (:3000) via concurrently
```

Open <http://localhost:3000> → **Start Conversation** → speak in `SOURCE_LANG`. Backend API docs at <http://localhost:8000/docs>.

## Quick commands

```bash
bun run doctor           # shared prereqs (bun + node_modules); no creds needed
bun run doctor:local     # + .env.local + AGORA_APP_ID/CERTIFICATE present
bun run verify           # web-only gate (doctor + api contracts + web build)
bun run verify:local     # full local gate: backend compile + fastapi smoke + proxy + web build
bun run clean            # remove venvs and build artifacts
```

Backend unit tests run standalone (no cloud, no creds):

```bash
cd server && pytest tests -v
```

## Related Deep Dives

- None. For what each verify command asserts, see [05_workflows](05_workflows.md) and [06_interfaces](06_interfaces.md).
