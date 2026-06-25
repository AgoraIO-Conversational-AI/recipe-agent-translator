# recipe-agent-translator — Repo Card

> Next.js web client + Python FastAPI backend for an Agora Conversational AI real-time speech translator. Cascading pipeline: DeepgramSTT → managed OpenAI (translate) → MiniMaxTTS. Zero-key by default — OpenAI is Agora-managed.

## Identity

| Field          | Value                                                                          |
| -------------- | ------------------------------------------------------------------------------ |
| Repo           | `AgoraIO-Conversational-AI/recipe-agent-translator`                            |
| Type           | `distributed-system` (single repo, two co-located processes)                   |
| Language       | Python 3.10+ (FastAPI + uvicorn) backend + Next.js 16 / React 19 web           |
| Deploy Target  | `web/` as Next.js app, `server/` as a reachable FastAPI service                |
| Owner          | Agora Conversational AI DevEx                                                  |
| Last Reviewed  | 2026-06-25                                                                     |
| Recipe Role    | `base`                                                                         |
| Recipe Version | `1.0.0`                                                                        |
| Recipe Status  | `experimental`                                                                 |

## L1 — Summaries

The Audience column helps agents prioritise: **Use** = consuming the recipe's behavior, **Maintain** = modifying internals.

| File                                     | Purpose                                                                           | Audience       |
| ---------------------------------------- | --------------------------------------------------------------------------------- | -------------- |
| [01_setup](L1/01_setup.md)               | bun + venv + pip setup, env vars (zero-key by default), commands                  | Use & Maintain |
| [02_architecture](L1/02_architecture.md) | Two-process topology, `/api/*` rewrite proxy, cascading pipeline lifecycle        | Maintain       |
| [03_code_map](L1/03_code_map.md)         | `web/` and `server/` trees with key file responsibilities                         | Maintain       |
| [04_conventions](L1/04_conventions.md)   | Python async + FastAPI patterns, Biome, JSON envelope, cascading-vendor VAD       | Maintain       |
| [05_workflows](L1/05_workflows.md)       | Add a route, change language pair, change LLM config, verify, deploy              | Use            |
| [06_interfaces](L1/06_interfaces.md)     | FastAPI route contracts, rewrites, env vars, vendor config                        | Use & Maintain |
| [07_gotchas](L1/07_gotchas.md)           | Language pair coupling, keyless default, no MLLM, `PORT` in env                  | Maintain       |
| [08_security](L1/08_security.md)         | Token007, App Certificate server-only, CORS, Agora-managed OpenAI key             | Maintain       |

## Recipe Profile

This repo declares `Recipe Role: base`. See [RECIPE.md](RECIPE.md) for extension points, invariants, and stable contracts before changing reusable surfaces.
