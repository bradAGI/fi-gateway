<p align="center">
  <img src="banner.png" alt="fi — free-inference gateway" width="100%">
</p>

<h1 align="center">fi</h1>

<p align="center">
  <strong>A Claude Code skill</strong> that gives your agent access to 300+ free LLM models<br>
  behind one endpoint that speaks OpenAI <em>and</em> Anthropic shapes.
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="License: MIT"></a>
  <img src="https://img.shields.io/badge/install-npx%20skills-informational" alt="npx skills">
  <img src="https://img.shields.io/badge/deps-stdlib%20only-brightgreen" alt="stdlib only">
  <img src="https://img.shields.io/badge/docker-required-blue" alt="Docker required">
</p>

---

## Install

```bash
npx skills add bradagi/fi-gateway
```

That's it. The skill installs into your agent's skills directory (`~/.claude/skills/fi-gateway/` for Claude Code, equivalent paths for [41+ other agents](https://github.com/vercel-labs/skills)) — `SKILL.md`, the CLI, provider catalog, and Docker compose file all arrive together.

## Use it

Talk to your agent the way you'd talk to a teammate. The skill wires up the common workflows:

```
you> add my gemini key AIza...
claude> [stores as GEMINI_API_KEY_1, runs ./fi reload]

you> what free models do I have?
claude> [runs ./fi models — shows your 300+ aliases across providers]

you> spin up the proxy
claude> [runs ./fi start — proxy live at http://localhost:4000, prints master key]

you> smoke test a reasoning model
claude> [picks an under-quota reasoning model, runs ./fi test]

you> add NVIDIA too — nvapi-...
claude> [stores the key, runs ./fi sync, discovers 135 NVIDIA models, ./fi reload]

you> why is the smart group 404'ing?
claude> [checks ./fi logs, diagnoses upstream access issue]
```

Under the hood, the skill drives a single-file Python CLI (`fi`) that runs a LiteLLM proxy in Docker. The skill is the *voice* — the CLI is the *mechanism*.

## What you get

- **One endpoint, two shapes** — `/v1/chat/completions` (OpenAI) *and* `/v1/messages` (Anthropic) from the same master key.
- **300+ models out of the box** — Gemini, Groq, OpenRouter, NVIDIA NIM, Hugging Face, Cerebras, Mistral, Scaleway, Voyage, Jina, Pollinations, and more.
- **Auto-discovery** — OpenRouter, NVIDIA, Hugging Face, Pollinations catalogs refresh from live APIs; no static lists to maintain.
- **Rate-limit-aware routing** — respects each provider's RPM/TPM, cools down on 429s, rotates across multiple keys per provider.
- **Group aliases** — `fast`, `smart`, `vision`, `embed`, `rerank`, `code`, `reasoning` pick the cheapest under-quota deployment and fall back on 429/5xx.

## Requirements

The skill runs commands on your machine, so you need:

- Python 3.11+ (stdlib only — no pip install)
- `docker` + `docker compose` plugin
- Linux, macOS, or WSL2

## Using clients directly

Once `./fi start` is running (your agent can do that for you), any OpenAI or Anthropic SDK points at one URL:

```python
# OpenAI SDK
from openai import OpenAI
client = OpenAI(base_url="http://localhost:4000/v1", api_key="sk-fi-…")
client.chat.completions.create(
    model="fast",                    # or "gemini-2.5-flash", "smart", "code", "embed", …
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

## Without the skill — direct CLI

If you want to drive it yourself instead of through your agent, the CLI lives at the repo root:

```bash
git clone https://github.com/bradAGI/fi-gateway
cd fi-gateway

./fi keys add gemini AIza...
./fi start
./fi test fast
```

Full command list:

```
./fi start           Generate config and boot the proxy
./fi stop            docker compose down
./fi reload          Regenerate config + force-recreate container
./fi restart         Bounce without regenerating
./fi status          Health + active providers
./fi logs [-f]       Tail litellm logs

./fi keys add <p> <key>
./fi keys list
./fi keys remove <p> [--index N]

./fi providers       Active vs inactive
./fi models [-g G]   Filter by group
./fi sync            Force-refresh auto-discovered catalogs
./fi config show     Print generated config.yaml
./fi config path     Runtime file paths

./fi test <model> [--prompt P]             OpenAI shape
./fi test-anthropic <model> [--prompt P]   Anthropic shape
```

## How it's laid out

```
fi-gateway/              # skill root — same as repo root
├── SKILL.md             # what your agent reads to learn how to drive fi
├── fi                   # single-file Python CLI (stdlib only)
├── providers.toml       # catalog — 20 providers, 4 with auto-discovery
├── docker-compose.yml   # one service: LiteLLM proxy
├── README.md            # this file
├── LICENSE
└── banner.png
```

User data lives in `~/.config/free-inference/`:

- `keys.env` — chmod 600, gitignored, never synced to the skill repo
- `config.yaml` — regenerated every `./fi start` / `./fi reload`
- `.discovery-cache.json` — 24h cache of auto-discovered model lists

## Auto-discovery

| Kind | Source | Cached |
|------|--------|--------|
| `openrouter_free` | `openrouter.ai/api/v1/models` (filtered to `pricing.prompt == 0`) | 24h |
| `nvidia_nim` | `integrate.api.nvidia.com/v1/models` (auth required) | 24h |
| `huggingface_inference` | `router.huggingface.co/v1/models` (live providers, $0.10/mo free credits) | 24h |
| `pollinations` | `gen.pollinations.ai/v1/models` (keyless) | 24h |

Discovery is skipped for providers without a configured key (except `optional_key` providers like Pollinations). Agents can run `./fi sync` to force a refresh.

## Adding a provider

Append one TOML block, then `./fi reload`:

```toml
[[provider]]
name = "your-provider"
key_env = "YOURPROVIDER_API_KEY"
base_url = "https://api.yours.example/v1"
litellm_prefix = "openai"           # or "gemini", "cohere", "nvidia_nim", …

[[provider.model]]
alias = "yours-flagship"
upstream = "their/model-id"
rpm = 60
groups = ["smart", "vision"]
```

No code change needed — the catalog is data.

## Caveats

- `/v1/messages` drops `cache_control` blocks for non-Anthropic upstreams (free providers don't have prompt caching).
- Streaming event order for tool-call edge cases may differ subtly from native Anthropic; standard streaming works fine.
- Some providers need a specific `litellm_prefix` for Anthropic-shape translation — `openai` routes `/v1/messages` through OpenAI's Responses API which most providers don't expose. Use the dedicated prefix (`nvidia_nim`, `cohere`, `gemini`) when available.
- Hugging Face models are "credit-gated free" — callable with HF's $0.10/mo free credit, not zero-cost per call. Filter with `./fi models -g free` to see models with at least one zero-priced provider.

## License

MIT
