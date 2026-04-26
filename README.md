<p align="center">
  <img src="banner.png" alt="infer — free LLM gateway" width="100%">
</p>

<h1 align="center">fi-gateway</h1>

<p align="center">
  <strong>Wire your coding agent to free LLMs.</strong><br>
  One endpoint speaking OpenAI <em>and</em> Anthropic <em>and</em> embeddings shapes.<br>
  Claude Code, opencode, pi-mono — auto-configured.
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="License: MIT"></a>
  <img src="https://img.shields.io/badge/install-npx%20skills-informational" alt="npx skills">
  <img src="https://img.shields.io/badge/python-3.11%2B-informational" alt="Python 3.11+">
  <img src="https://img.shields.io/badge/docker-required-blue" alt="Docker required">
</p>

---

## Quickstart

**1.** Get the repo. Either install as a skill (Claude Code / opencode / pi / 41 other agents pick it up automatically) or clone directly:

```bash
# Option A — skill install (recommended for agent users)
npx skills add bradagi/fi-gateway

# Option B — direct clone (no Node, no skill registration; just the CLI)
git clone https://github.com/bradAGI/fi-gateway && cd fi-gateway
```

**2.** Add free keys, boot the proxy, verify what works:

```bash
./infer keys add gemini AIza...
./infer keys add nvidia nvapi-...
./infer keys add openrouter sk-or-...

./infer start                  # prints the master key (sk-infer-…)
./infer probe                  # parallel smoke test, caches working/broken
./infer reload                 # regenerate config, expose only verified models
```

**3.** Wire any coding agent CLIs you have installed:

```bash
./infer wire cc                # Claude Code   → ~/.claude/settings.json
./infer wire opencode          # opencode      → ~/.config/opencode/opencode.json
./infer wire pi                # pi-mono       → ~/.pi/agent/models.json
./infer wire openclaw          # openclaw      → ~/.openclaw/config.yaml      (via `openclaw onboard`)
./infer wire hermes            # hermes-agent  → ~/.hermes/config.yaml        (via `hermes config set`)
```

**4.** *(Optional)* Make the command run from anywhere:

```bash
./infer install            # symlinks ~/.local/bin/infer → this script
infer doctor               # works from any directory
```

Detects your shell (bash / zsh / fish on macOS and Linux) and prints the right `~/.zshrc` / `~/.bashrc` / `~/.config/fish/config.fish` line if `~/.local/bin` isn't on `$PATH`. `./infer uninstall` removes the symlink. Pass `--name <other>` to install under a different command name.

> **Permission denied?** If `./infer` errors with "Permission denied" right after clone or skill install, the executable bit didn't survive the transfer (some `npx`/`tar` pipelines strip it). Run `chmod +x infer` once and you're set.

That's it — your wired agent now routes through `localhost:4000` using your free keys. Every wired tool gets a clean `/v1/models` list of probe-verified callable aliases.

> **Custom port?** Export `INFER_PORT=8080` (or any free port) before any `./infer` invocation and the gateway, doctor output, wire URLs, and probe all use it. Container-internal port is always 4000; only the host-side mapping changes.

## Talk to your agent

The skill teaches your agent how to drive the gateway. Examples:

> *"Add my gemini key AIza…"* → `./infer keys add gemini …` + `./infer reload`
> *"Wire me up for Claude Code"* → `./infer wire cc` (auto-backs up settings.json)
> *"What free models do I have?"* → `./infer probe` + `./infer models --working`
> *"Show me a health check"* → `./infer doctor`
> *"Why is `<alias>` failing?"* → `./infer logs` + diagnosis

## Features

| | |
|---|---|
| **One endpoint, three shapes** | `/v1/chat/completions` (OpenAI), `/v1/messages` (Anthropic), `/v1/embeddings` |
| **~150-model catalog** | Gemini, Gemma, Groq, OpenRouter free, NVIDIA NIM, Cerebras, Mistral, Scaleway, Voyage, Jina, Mixedbread, Nomic, Cohere, Together, Hunyuan, Chutes, LLM7, Ollama Cloud |
| **Auto-discovery** | OpenRouter, NVIDIA, Gemini refresh from live `/v1/models` (24h cache); image/video/audio/parser endpoints filtered upstream |
| **Probe + auto-exclude** | `./infer probe` smoke-tests every alias; broken ones never reach `/v1/models` after `./infer reload` |
| **Embeddings as first class** | `embed`-tagged aliases probed via `/v1/embeddings`; vectors flow through the same master key |
| **Deterministic routing** | Capability tags (`fast`/`smart`/`vision`/`embed`/`code`/`reasoning`) are filter metadata only — every request names a concrete model, never a group |
| **Key rotation** | Add multiple keys per provider; router round-robins for 2× effective RPD |
| **Lightweight** | Single Python file, stdlib only. Only host requirement: `docker compose` |

## Use it from any SDK

```python
# Chat
from openai import OpenAI
client = OpenAI(base_url="http://localhost:4000/v1", api_key="sk-infer-…")
client.chat.completions.create(
    model="gemini-2.5-flash",       # any alias from ./infer models --working
    messages=[{"role": "user", "content": "Hello"}],
)

# Embeddings
client.embeddings.create(model="gemini-embedding-001", input="vectorize me")

# Anthropic shape — works against every model in your catalog, not just real Anthropic
from anthropic import Anthropic
client = Anthropic(base_url="http://localhost:4000", api_key="sk-infer-…")
client.messages.create(
    model="gemini-2.5-pro",
    max_tokens=256,
    messages=[{"role": "user", "content": "Hello"}],
)
```

## CLI

```
Lifecycle    ./infer start | stop | restart | reload | status | logs [-f]
Health       ./infer doctor                        proxy + providers + keys + probe age + drift + wired clients
Globalize    ./infer install [--name N] [--dir D] [--force]   symlink into a $PATH directory (default name 'infer')
             ./infer uninstall [--name N] [--dir D]            remove the symlink
Keys         ./infer keys add <provider> <key>
             ./infer keys list | remove <provider> [--index N]
Catalog      ./infer providers                     active vs inactive
             ./infer models [-g GROUP] [-w/--working | --broken]
             ./infer sync                          refresh auto-discovered catalogs
             ./infer config show | path
Smoke test   ./infer test <alias> [--prompt P]            OpenAI shape
             ./infer test-anthropic <alias> [--prompt P]  Anthropic shape
Probe        ./infer probe [-g GROUP] [-p PROVIDER] [-c N] [-t SEC] [--include-broken]
Wiring       ./infer detect                        scan installed agent CLIs
             ./infer wire cc | opencode | pi | openclaw | hermes
             ./infer unwire cc | opencode | pi | openclaw | hermes
```

## Probe + auto-exclude

The catalog tells you what *might* work; probe tells you what *does* work for your account.

```bash
./infer probe                      # parallel smoke test
./infer probe --group code         # narrow to a tag
./infer probe --provider gemini    # narrow to a provider
./infer probe --include-broken     # also retest previously-failed aliases
./infer models --working           # filter the catalog to verified hits
```

Each run writes `~/.config/infer/.probe-cache.json` with status, latency, and error class per alias. On the next `./infer reload`, broken aliases are excluded from `config.yaml` — the router never picks them, clients never see them in `/v1/models`. `./infer sync` prunes stale entries when upstreams rotate their catalogs.

## Auto-discovery

| Kind | Source | Cache |
|------|--------|-------|
| `openrouter_free` | `openrouter.ai/api/v1/models` filtered to `pricing.prompt == 0` | 24h |
| `nvidia_nim` | `integrate.api.nvidia.com/v1/models` | 24h |
| `gemini` | `generativelanguage.googleapis.com/v1beta/openai/models` | 24h |
| `opencode_zen_free` | `opencode.ai/zen/v1/models` filtered to `-free` suffix + stealth list | 24h |

Each handler classifies discovered ids into `chat / embed / rerank / drop`. Image / video / audio / TTS / document-parser / deprecated endpoints are dropped at discovery — they never enter the catalog. Embedding models stay in catalog with the `embed` tag and are probed via `/v1/embeddings`.

> Hugging Face routing is **not** in the default catalog — the $0.10/mo free credit on non-PRO accounts runs out fast. Add it manually if you have a PRO subscription.

## Adding a provider

Append one TOML block, then `./infer reload`:

```toml
[[provider]]
name = "your-provider"
key_env = "YOURPROVIDER_API_KEY"
base_url = "https://api.yours.example/v1"
litellm_prefix = "openai"              # or "gemini", "cohere", "nvidia_nim", …

[[provider.model]]
alias = "yours-flagship"
upstream = "their/model-id"
groups = ["smart", "vision"]            # filter tags only — not callable aliases
```

Use `discovery = "<kind>"` instead of static models if the provider exposes a `/v1/models` endpoint that should refresh on its own.

## Layout

```
infer-gateway/           # skill root == repo root
├── SKILL.md             # what your agent reads to learn how to drive infer
├── infer                # single-file Python CLI (stdlib only)
├── providers.toml       # catalog
├── docker-compose.yml   # one service: LiteLLM proxy
└── README.md
```

User state in `~/.config/infer/` (gitignored, 600):

| File | Lifecycle |
|------|-----------|
| `keys.env` | written by `./infer keys add` |
| `config.yaml` | regenerated each `./infer start` / `./infer reload`; auto-excludes probe-failed aliases |
| `.discovery-cache.json` | upstream catalog snapshots, 24h TTL |
| `.probe-cache.json` | per-alias verify results, 24h TTL |

## Caveats

- **Wiring locks your tool to gateway liveness** — kill the proxy and your wired client errors until restart. `./infer unwire <tool>` reverts via the `.infer-backup` file.
- **NVIDIA NIM gating** — many endpoints require separate per-account approval and silently 404 ("Function not found"). Probe catches this.
- **OpenRouter free-pool 429s** — aggressive shared-tier rate limits on popular free models (gemma, llama, qwen-coder). Re-run probe after the window resets.
- **Anthropic-shape translation** — the `openai` litellm prefix routes `/v1/messages` through OpenAI's Responses API which most upstreams don't expose. Use the dedicated prefix (`nvidia_nim`, `cohere`, `gemini`) when available.
- **Streaming tool-calls** — event order may differ subtly from native Anthropic for edge cases; standard streaming works fine.
- **`cache_control` blocks** are stripped on `/v1/messages` for non-Anthropic upstreams — free providers don't have prompt caching.

## Requirements

Python 3.11+ · `docker compose` · Linux / macOS / WSL2.

## License

MIT
