<p align="center">
  <img src="banner.png" alt="fi — free-inference gateway" width="100%">
</p>

<h1 align="center">fi-gateway</h1>

<p align="center">
  <strong>Wire your coding agent to free LLMs.</strong><br>
  One endpoint. OpenAI <em>and</em> Anthropic <em>and</em> embeddings shapes.<br>
  Claude Code, opencode, pi-mono — all three auto-configured.
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="License: MIT"></a>
  <img src="https://img.shields.io/badge/install-npx%20skills-informational" alt="npx skills">
  <img src="https://img.shields.io/badge/python-3.11%2B-informational" alt="Python 3.11+">
  <img src="https://img.shields.io/badge/docker-required-blue" alt="Docker required">
</p>

---

## 1. Install

```bash
npx skills add bradagi/fi-gateway
```

The skill installs into your agent's skills directory (`~/.claude/skills/fi-gateway/` for Claude Code, equivalent paths for [41+ other agents](https://github.com/vercel-labs/skills)) — `SKILL.md`, the CLI, provider catalog, and Docker compose file all arrive together.

## 2. Wire your coding agent

`fi-gateway` ships bindings for the three coding agent CLIs most people have installed. One command per tool:

```bash
./fi detect              # scans: Claude Code, opencode, pi-mono — installed? already wired?
./fi wire cc             # Claude Code → ~/.claude/settings.json (ANTHROPIC_BASE_URL + key)
./fi wire opencode       # opencode    → ~/.config/opencode/opencode.json (adds fi provider)
./fi wire pi             # pi-mono     → ~/.pi/agent/models.json (adds fi provider)
./fi unwire <tool>       # restore from .fi-backup
```

Each `wire` saves the original config to `<path>.fi-backup` before editing, so `./fi unwire` always reverts. For `cc`, the settings file is chmod'd to 600 since it now contains the gateway's master key.

After wiring, your coding agent's normal commands route through the fi gateway at `localhost:4000` instead of the vendor's API — every request uses *your* free keys (Gemini, NVIDIA, OpenRouter, …).

## 3. Add your free keys and start it

```bash
./fi keys add gemini AIza...
./fi keys add nvidia nvapi-...
./fi keys add openrouter sk-or-...

./fi start               # boots the proxy at http://localhost:4000 and prints the master key
./fi probe               # smoke-test every alias in parallel; cache which work for your account
```

`./fi start` prints a `sk-fi-…` master key — that's what wired tools use, not your raw provider keys. After `./fi probe` runs, `./fi reload` regenerates `config.yaml` to expose only verified-working aliases.

## Or just talk to your agent

The skill teaches your agent how to drive the gateway end-to-end:

```
you> add my gemini key AIza…
claude> [stores as GEMINI_API_KEY_1, runs ./fi reload]

you> wire me up for claude code
claude> [runs ./fi wire cc — sets ANTHROPIC_BASE_URL in settings.json, backs up original]

you> what free models do I have right now?
claude> [runs ./fi probe + ./fi models --working — shows your callable set by provider]

you> add nvidia too — nvapi-…
claude> [stores key, runs ./fi sync, discovers ~120 NVIDIA models, ./fi reload]

you> show me a health check
claude> [runs ./fi doctor — proxy state, providers, keys, probe age, drift, wired clients]
```

## What you get

- **One endpoint, three shapes** — `/v1/chat/completions` (OpenAI), `/v1/messages` (Anthropic), and `/v1/embeddings` from the same master key.
- **~150 models out of the box** — Gemini, Gemma, Groq, OpenRouter (free-priced only), NVIDIA NIM, Cerebras, Mistral, Scaleway, Voyage, Jina, Mixedbread, Nomic, Pollinations, Cohere, Together, Hunyuan, Chutes, LLM7, Ollama Cloud.
- **Auto-discovery** — OpenRouter, NVIDIA NIM, Gemini AI Studio, and Pollinations catalogs refresh from live APIs (24h cache); upstream-side image/video/audio/parser/deprecated endpoints are filtered out so the catalog stays clean.
- **Probe + auto-exclude** — `./fi probe` smoke-tests every alias and caches working/broken state. Broken aliases are automatically removed from `/v1/models` on the next `./fi reload`, so the router never picks a dead deployment and clients never see one in their model picker.
- **Embeddings, not just chat** — embedding models stay in the catalog tagged `embed`, get probed via `/v1/embeddings`, and serve real vectors through the same master key. RAG workflows route through `localhost:4000` without a separate provider integration.
- **Deterministic routing** — capability tags (`fast`, `smart`, `vision`, `embed`, `rerank`, `code`, `reasoning`) are filter metadata for `./fi models -g <tag>`. They are *not* callable aliases — you always name a concrete model in your request, so logs are honest and outputs aren't silently substituted.
- **Key rotation** — drop in multiple keys for the same provider; the router round-robins requests across them so two keys = 2× effective RPD.

## Requirements

- Python 3.11+ (stdlib only — no pip install)
- `docker` + `docker compose` plugin
- Linux, macOS, or WSL2

## Using the endpoint directly

Once `./fi start` is running, any OpenAI / Anthropic SDK or `curl` points at the same URL.

### curl

```bash
MASTER_KEY=$(./fi config show | grep -m1 LITELLM_MASTER_KEY | awk -F: '{print $2}' | xargs)
# or just: MASTER_KEY=sk-fi-...                                    # printed by ./fi start

# OpenAI shape — chat
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer $MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemini-2.5-flash",
    "messages": [{"role": "user", "content": "Reply with: READY"}],
    "max_tokens": 64
  }'

# OpenAI shape — embeddings
curl http://localhost:4000/v1/embeddings \
  -H "Authorization: Bearer $MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gemini-embedding-001", "input": "vectorize me"}'

# Anthropic shape — same gateway, /v1/messages
curl http://localhost:4000/v1/messages \
  -H "x-api-key: $MASTER_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemini-2.5-pro",
    "max_tokens": 256,
    "messages": [{"role": "user", "content": "Hello"}]
  }'

# Catalog
curl http://localhost:4000/v1/models -H "Authorization: Bearer $MASTER_KEY" | jq '.data[].id'
```

### SDKs

```python
# OpenAI shape — chat + embeddings
from openai import OpenAI
client = OpenAI(base_url="http://localhost:4000/v1", api_key="sk-fi-…")

client.chat.completions.create(
    model="gemini-2.5-flash",       # any concrete alias from `./fi models --working`
    messages=[{"role": "user", "content": "Hello"}],
)

client.embeddings.create(
    model="gemini-embedding-001",   # any embed-tagged alias
    input="vectorize me",
)

# Anthropic shape — same gateway, /v1/messages
from anthropic import Anthropic
client = Anthropic(base_url="http://localhost:4000", api_key="sk-fi-…")
client.messages.create(
    model="gemini-2.5-pro",
    max_tokens=256,
    messages=[{"role": "user", "content": "Hello"}],
)
```

LiteLLM translates Anthropic-format requests to whatever upstream the alias points at — so the Anthropic SDK works against every model in your catalog.

## CLI reference

All commands run from the repo root (or `~/.claude/skills/fi-gateway/` if installed via `npx skills add`):

```
# Lifecycle
./fi start / stop / restart / reload / status / logs [-f]

# One-shot health overview
./fi doctor                     Proxy + providers + keys + discovery + probe + drift + wired clients

# Keys
./fi keys add <provider> <key>
./fi keys list
./fi keys remove <provider> [--index N]

# Catalog
./fi providers                  Active vs inactive providers
./fi models [-g GROUP] [-w]     Filter aliases by group; -w = only probe-verified working
./fi sync                       Refresh auto-discovered catalogs (24h cache)
./fi config show / path

# Test routing
./fi test <alias> [--prompt P]              OpenAI shape
./fi test-anthropic <alias> [--prompt P]    Anthropic shape

# Probe (smoke-test every alias against the proxy)
./fi probe [-g GROUP] [-p PROVIDER] [-c N] [-t SEC] [--include-broken]

# Wire coding agent CLIs
./fi detect                     Status for pi / opencode / cc
./fi wire <tool>                cc | opencode | pi — edits config, saves .fi-backup
./fi unwire <tool>              Restore from the .fi-backup
```

## Repo layout

```
fi-gateway/              # skill root == repo root
├── SKILL.md             # instructions your agent reads to drive fi
├── fi                   # single-file Python CLI (stdlib only)
├── providers.toml       # catalog — 19 providers, 4 with auto-discovery
├── docker-compose.yml   # one service: LiteLLM proxy
├── banner.png
├── LICENSE
└── README.md
```

User data lives in `~/.config/free-inference/` (never synced to the skill repo):

- `keys.env` — chmod 600, gitignored
- `config.yaml` — regenerated every `./fi start` / `./fi reload`; auto-excludes probe-failed aliases
- `.discovery-cache.json` — 24h cache of auto-discovered model lists
- `.probe-cache.json` — 24h cache of `./fi probe` results

## Auto-discovery

| Kind | Source | Cached |
|------|--------|--------|
| `openrouter_free` | `openrouter.ai/api/v1/models` filtered to `pricing.prompt == 0` and text/embedding output modalities | 24h |
| `nvidia_nim` | `integrate.api.nvidia.com/v1/models` | 24h |
| `gemini` | `generativelanguage.googleapis.com/v1beta/openai/models` (chat + embed only) | 24h |
| `pollinations` | `gen.pollinations.ai/v1/models` (chat-completion-capable, keyless) | 24h |

Each handler classifies discovered ids into `chat / embed / rerank / drop`. Image, video, audio, TTS, document parsers, and deprecated endpoints are filtered upstream — they never enter the catalog and never waste probe time. Embedding models stay in the catalog with the `embed` tag and get probed via `/v1/embeddings`.

Discovery is skipped for providers without a configured key (except `optional_key` providers like Pollinations). Run `./fi sync` to force a refresh; sync also prunes probe-cache entries for aliases that have been removed from upstream.

> Hugging Face is not in the default catalog — the $0.10/month free HF credit is exhausted within a handful of calls on non-PRO accounts. Add it manually if you have a PRO subscription ($2/mo credit).

## Probe + auto-exclude

The catalog tells you what *might* work; probe tells you what *does* work for your account.

```bash
./fi probe                      # parallel smoke test against every alias
./fi probe --group code         # narrow to a tag
./fi probe --provider openrouter
./fi probe --include-broken     # also retest previously-failed aliases
./fi models --working           # filter the catalog to probe-verified hits
./fi models --broken            # the inverse, for debugging
```

Each probe writes `~/.config/free-inference/.probe-cache.json` with per-alias status, latency, and error class (auth / 404 / 429 / timeout / empty / etc.). On the next `./fi reload`, broken aliases are excluded from `config.yaml` — the router can't pick them, clients don't see them in `/v1/models`. Re-probe at any time to pick a model back up. `./fi sync` prunes stale probe entries when upstreams rotate their catalogs.

## Adding a provider

Append one TOML block, then `./fi reload`:

```toml
[[provider]]
name = "your-provider"
key_env = "YOURPROVIDER_API_KEY"
base_url = "https://api.yours.example/v1"
litellm_prefix = "openai"              # or "gemini", "cohere", "nvidia_nim", …

[[provider.model]]
alias = "yours-flagship"
upstream = "their/model-id"
rpm = 60
groups = ["smart", "vision"]            # filter tags only — not callable aliases
```

No code change needed — the catalog is data. Use `discovery = "<kind>"` instead of static models if the provider has a `/v1/models` endpoint that should refresh on its own.

## Caveats

- **Wiring locks your tool to the gateway's liveness** — if you `./fi wire cc` then kill the proxy, Claude Code will fail until you restart it. `./fi unwire cc` reverts.
- **`/v1/messages` drops `cache_control` blocks** for non-Anthropic upstreams (free providers don't have prompt caching).
- **Tool-call streaming** event order may differ subtly from native Anthropic for edge cases; standard streaming works fine.
- **Anthropic-shape translation** for non-`openai/`-prefix models — when wiring an alias for `/v1/messages` use the dedicated `litellm_prefix` (`nvidia_nim`, `cohere`, `gemini`) since the `openai` prefix routes `/v1/messages` through OpenAI's Responses API which most providers don't expose.
- **NVIDIA NIM gating** — many NVIDIA endpoints require separate per-account approval and silently 404 ("Function not found"). Probe catches this and the auto-exclude hides them from `/v1/models`.
- **OpenRouter free-pool 429s** — OR's free-tier rate limits are aggressive on popular models (gemma, llama, qwen-coder). Probe records them as broken; rerun later when the window resets.

## License

MIT
