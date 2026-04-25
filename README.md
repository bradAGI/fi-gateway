<p align="center">
  <img src="banner.png" alt="fi — free-inference gateway" width="100%">
</p>

<h1 align="center">fi-gateway</h1>

<p align="center">
  <strong>Wire your coding agent to 300+ free LLMs.</strong><br>
  One endpoint. OpenAI <em>and</em> Anthropic shapes. Claude Code, opencode, pi-mono — all three auto-configured.
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
./fi wire cc             # Claude Code → ~/.claude/settings.json  (ANTHROPIC_BASE_URL + key)
./fi wire opencode       # opencode    → ~/.config/opencode/opencode.json  (adds fi provider)
./fi wire pi             # pi-mono     → ~/.pi/agent/models.json           (adds fi provider)
./fi unwire <tool>       # restore from .fi-backup
```

Each `wire` saves the original config to `<path>.fi-backup` before editing, so `./fi unwire` is always safe. For `cc`, the settings file is chmod'd to 600 since it now contains the gateway's master key.

After wiring, your coding agent's normal commands route through the fi gateway at `localhost:4000` instead of the vendor's API — so every request uses *your* free keys (Gemini, NVIDIA, OpenRouter, HF, …) with rate-limit-aware fallbacks.

## 3. Add your free keys and start it

```bash
./fi keys add gemini AIza...
./fi keys add nvidia nvapi-...
./fi keys add openrouter sk-or-...

./fi start               # boots the proxy at http://localhost:4000 and prints the master key
```

Now your wired coding agents talk to the gateway. `./fi start` prints a `sk-fi-…` master key — that's what the wired tools use, not your raw provider keys.

## Or just talk to your agent

The skill teaches your agent how to drive the gateway end-to-end:

```
you> add my gemini key AIza…
claude> [stores as GEMINI_API_KEY_1, runs ./fi reload]

you> wire me up for claude code
claude> [runs ./fi wire cc — sets ANTHROPIC_BASE_URL in settings.json, backs up original]

you> what free models do I have?
claude> [runs ./fi models — shows your 300+ aliases across providers]

you> add nvidia too — nvapi-…
claude> [stores key, runs ./fi sync, discovers 135 NVIDIA models, ./fi reload]

you> why is the smart group 404'ing?
claude> [checks ./fi logs, diagnoses upstream access issue]
```

## What you get

- **One endpoint, two shapes** — `/v1/chat/completions` (OpenAI) *and* `/v1/messages` (Anthropic) from the same master key.
- **300+ models out of the box** — Gemini, Groq, OpenRouter, NVIDIA NIM, Hugging Face, Cerebras, Mistral, Scaleway, Voyage, Jina, Pollinations, Cohere, Together, Hunyuan, Chutes, LLM7, Ollama Cloud.
- **Auto-discovery** — OpenRouter, NVIDIA NIM, Hugging Face, Pollinations catalogs refresh from live APIs (24h cache); no static lists to maintain.
- **Rate-limit-aware routing** — respects each provider's RPM/TPM, cools down on 429s, round-robins across multiple keys per provider.
- **Group aliases** — `fast`, `smart`, `vision`, `embed`, `rerank`, `code`, `reasoning` pick the cheapest under-quota deployment and fail over on 429/5xx.

## Requirements

- Python 3.11+ (stdlib only — no pip install)
- `docker` + `docker compose` plugin
- Linux, macOS, or WSL2

## Using the endpoint directly

Once `./fi start` is running, any OpenAI or Anthropic SDK points at the same URL:

```python
# OpenAI SDK
from openai import OpenAI
client = OpenAI(base_url="http://localhost:4000/v1", api_key="sk-fi-…")
client.chat.completions.create(
    model="fast",                  # or "gemini-2.5-flash", "smart", "code", "embed", …
    messages=[{"role": "user", "content": "Hello"}],
)

# Anthropic SDK — same gateway, Anthropic shape
from anthropic import Anthropic
client = Anthropic(base_url="http://localhost:4000", api_key="sk-fi-…")
client.messages.create(
    model="gemini-2.5-pro",
    max_tokens=256,
    messages=[{"role": "user", "content": "Hello"}],
)
```

LiteLLM handles the translation under the hood — so the Anthropic SDK works against every model in your catalog, not just real Anthropic.

## CLI reference

All commands run from the repo root (or `~/.claude/skills/fi-gateway/` if installed via `npx skills add`):

```
# Proxy lifecycle
./fi start / stop / restart / reload / status / logs [-f]

# Keys
./fi keys add <provider> <key>
./fi keys list
./fi keys remove <provider> [--index N]

# Catalog inspection
./fi providers                   Active vs inactive providers
./fi models [-g GROUP]           Filter aliases by group
./fi sync                        Force-refresh auto-discovered catalogs
./fi config show / path

# Smoke tests
./fi test <alias> [--prompt P]           OpenAI shape
./fi test-anthropic <alias> [--prompt P] Anthropic shape

# Wire coding agent CLIs
./fi detect                      Status for pi / opencode / cc
./fi wire <tool>                 cc | opencode | pi — edits config, saves .fi-backup
./fi unwire <tool>               Restore from the .fi-backup
```

## Repo layout

```
fi-gateway/              # skill root == repo root
├── SKILL.md             # instructions your agent reads to drive fi
├── fi                   # single-file Python CLI (stdlib only)
├── providers.toml       # catalog — 20 providers, 4 with auto-discovery
├── docker-compose.yml   # one service: LiteLLM proxy
├── banner.png
├── LICENSE
└── README.md
```

User data lives in `~/.config/free-inference/` (never synced to the skill repo):

- `keys.env` — chmod 600, gitignored
- `config.yaml` — regenerated every `./fi start` / `./fi reload`
- `.discovery-cache.json` — 24h cache of auto-discovered model lists

## Auto-discovery

| Kind | Source | Cached |
|------|--------|--------|
| `openrouter_free` | `openrouter.ai/api/v1/models` (filtered to `pricing.prompt == 0`) | 24h |
| `nvidia_nim` | `integrate.api.nvidia.com/v1/models` (auth required) | 24h |
| `huggingface_inference` | `router.huggingface.co/v1/models` (live providers, $0.10/mo free credits) | 24h |
| `pollinations` | `gen.pollinations.ai/v1/models` (keyless) | 24h |

Discovery is skipped for providers without a configured key (except `optional_key` providers like Pollinations). Run `./fi sync` to force a refresh.

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
groups = ["smart", "vision"]
```

No code change needed — the catalog is data.

## Caveats

- **Wiring locks your tool to the gateway's liveness** — if you `./fi wire cc` then kill the proxy, Claude Code will fail until you restart it. `./fi unwire cc` anytime to revert.
- `/v1/messages` drops `cache_control` blocks for non-Anthropic upstreams (free providers don't have prompt caching).
- Streaming event order for tool-call edge cases may differ subtly from native Anthropic; standard streaming works fine.
- Some providers need a specific `litellm_prefix` for Anthropic-shape translation — `openai` routes `/v1/messages` through OpenAI's Responses API which most providers don't expose. Use the dedicated prefix (`nvidia_nim`, `cohere`, `gemini`) when available.
- Hugging Face models are "credit-gated free" — callable with HF's $0.10/mo free credit, not zero-cost per call. Filter with `./fi models -g free` to see models with at least one zero-priced provider.

## License

MIT
