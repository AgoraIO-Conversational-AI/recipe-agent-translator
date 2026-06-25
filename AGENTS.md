# Agent Development Guide

For coding agents working in `recipe-agent-translator`. This repository is the
**translator** recipe in the Agora Conversational AI recipes family.

## How to Load

This repository uses progressive disclosure documentation. Docs live under
`docs/ai/` in three levels.

1. Read [docs/ai/L0_repo_card.md](docs/ai/L0_repo_card.md) to identify the repo.
2. This repo declares `Recipe Role: base`; read [docs/ai/RECIPE.md](docs/ai/RECIPE.md) before changing reusable recipe contracts.
3. Load ALL 8 files in [docs/ai/L1/](docs/ai/L1/). They are small — load all upfront.
4. Follow L2 deep-dive links only when L1 isn't detailed enough. The index is at [docs/ai/L1/L2/_index.md](docs/ai/L1/L2/_index.md).

The sections below remain the canonical contributor handbook for hands-on work;
the `docs/ai/` tree is the structured summary used by AI agents.

## System shape

- **`server/`** — Python FastAPI agent backend (:8000). Owns Agora token
  generation and agent session lifecycle. Uses the managed `OpenAI` vendor
  (Agora-managed, keyless) for translation. SDK: `agora-agents>=2.3.0`
  (`import agora_agent`).
- **`web/`** — Next.js 16 / React 19 / TypeScript frontend (:3000).
- Auth: Token007 from `AGORA_APP_ID` + `AGORA_APP_CERTIFICATE`.
- No `llm/` service — OpenAI is Agora-managed (zero-key by default).

## Pipeline

`DeepgramSTT(language=SOURCE_LANG)` → `OpenAI` (translates to `TARGET_LANG`) → `MiniMaxTTS(voice=TTS_VOICE)`

## Routing / ownership

- UI and RTC/RTM lifecycle live in `web/`.
- Browser-facing `/api/*` paths are Next rewrites (`web/next.config.ts`) to the
  agent backend; do not add `web/app/api/**/route.ts` for agent/token logic.
- Token generation and agent lifecycle live in `server/src/`.
- Translation prompt builder lives in `server/src/translation_config.py`.

## Supported modes

- **Local:** `bun run dev` starts `server` (:8000) and `web` (:3000).
  The web app calls `/api/*`; Next rewrites to
  `AGENT_BACKEND_URL=http://localhost:8000`.
- **Deploy:** deploy `web` (Next) + `server` (reachable FastAPI).
  Set `AGENT_BACKEND_URL` in the web deployment.

## Env vars

| Variable | Default | Notes |
|---|---|---|
| `AGORA_APP_ID` | — | required |
| `AGORA_APP_CERTIFICATE` | — | required |
| `SOURCE_LANG` | `es` | Deepgram STT language code |
| `TARGET_LANG` | `English` | Language name for translation prompt |
| `TTS_VOICE` | `English_captivating_female1` | MiniMax voice matching `TARGET_LANG` |
| `OPENAI_MODEL` | `gpt-4o-mini` | OpenAI model for translation |
| `OPENAI_API_KEY` | — | optional — BYO only if your account requires it |

## Patterns

- Keep the web client calling `/api/*`; hide backend placement behind Next rewrites.
- Keep token generation and the App Certificate in `server/`.
- `OPENAI_API_KEY` is optional: Agora manages the OpenAI key by default (keyless).
- Changing `TARGET_LANG` means also picking a matching target-language `TTS_VOICE`.

## Anti-patterns

- Do not reintroduce `llm/` or the `CustomLLM` vendor.
- Do not reintroduce Next Route Handlers for agent/token logic.
- Do not put `PORT` in `server/.env.example` (it would clobber the random port
  that `verify:local:fastapi` injects via `load_dotenv(override=True)`).

## Commands

```bash
bun run setup
bun run dev
bun run doctor
bun run doctor:local
bun run verify         # web-only, no creds
bun run verify:local   # full local gate
```

Narrower checks: `bun run verify:backend`, `bun run verify:local:fastapi`,
`bun run verify:web:proxy`. Backend tests: `cd server && pytest tests -v`.

## Done criteria

1. Run the narrowest relevant verification command.
2. Web-affecting changes: `bun run verify:web` passes.
3. Backend-affecting changes: `bun run verify:local` (or narrower
   `verify:local:fastapi` / `verify:backend`) passes.
4. If you change required env vars or setup steps, update the root README,
   the relevant module README, and `server/.env.example` together.
5. If the change touches workflows, interfaces, gotchas, or security details,
   update the matching file under [docs/ai/L1/](docs/ai/L1/) and bump
   `Last Reviewed` in [docs/ai/L0_repo_card.md](docs/ai/L0_repo_card.md).

## Git Conventions

### Commit messages — conventional commits

- **Format:** `type: description` or `type(scope): description`
- **Types:** `feat:` (new feature), `fix:` (bug fix), `chore:` (maintenance, version bumps), `test:` (test additions/changes), `docs:` (documentation)
- **Scoped variant:** `feat(scope):`, `fix(scope):` — e.g. `fix(server): validate channel name`
- **Lowercase after prefix** — `feat: add feature`, not `feat: Add feature`
- **Present tense** — "add feature", not "added feature"

### Branch names

- **Format:** `type/short-description` — lowercase, hyphen-separated
- **Types match commit types:** `feat/`, `fix/`, `chore/`, `test/`, `docs/`
- **Examples:** `feat/add-language-selector`, `fix/tts-voice-mismatch`, `docs/progressive-disclosure`

### General rules

- **Repo-local `AGENTS.md` is the authoritative source for repo conventions.**
- **No AI tool names** — never mention claude, cursor, copilot, cody, aider, gemini, codex, chatgpt, or gpt-3/4 in commit messages or PR descriptions.
- **No Co-Authored-By trailers** — omit AI attribution lines.
- **No `--no-verify`** — let git hooks run normally.
- **No git config changes** — do not modify `user.name` or `user.email`.

## Doc Commands

| Command       | When to use                                                                  |
| ------------- | ---------------------------------------------------------------------------- |
| generate docs | No `docs/ai/` directory exists yet                                           |
| update docs   | Code changed since the `Last Reviewed` date in L0                            |
| test docs     | Verify docs give agents the right context (writes `docs/ai/test-results.md`) |
| fix docs      | Close findings from a docs review or test run                                |

See the [progressive disclosure standard](https://github.com/AgoraIO-Community/ai-devkit/blob/main/docs/standard/progressive-disclosure-standard.md) and [workflows](https://github.com/AgoraIO-Community/ai-devkit/blob/main/docs/workflows/progressive-disclosure-docs.md) for the full specification.
