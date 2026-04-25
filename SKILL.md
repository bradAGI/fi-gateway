---
name: infer
description: Drive the `infer` CLI (the fi-gateway unified inference gateway) — a Python CLI that aggregates ~150 free LLM models (Gemini, Groq, OpenRouter, NVIDIA NIM, Cerebras, Mistral, Scaleway, Voyage, Jina, Cohere, Together, Hunyuan, Chutes, LLM7, Ollama Cloud) behind a single OpenAI- and Anthropic-compatible endpoint at http://localhost:4000. Trigger whenever the user asks to add API keys for any LLM provider; start/stop/check a local LLM proxy; list available models; test a model; add a new provider to `providers.toml`; refresh auto-discovered catalogs; debug routing or upstream errors; wire a coding agent CLI (Claude Code, opencode, pi-mono, openclaw, hermes); or mentions "fi", "infer", "my free keys", "free models", "local proxy", "free inference", "gateway".
---

# infer — Free-Inference Gateway

This skill teaches your agent how to drive `infer`, the CLI at the skill root that turns the user's free API keys into one OpenAI- and Anthropic-compatible endpoint.

## When to use this skill

Trigger phrases:

- "add my **<provider>** key" / "store my **<provider>** key" / "I got a **<provider>** key"
- "start / boot / spin up the **proxy** / **gateway** / fi / infer"
- "what free **models** do I have" / "list my models" / "models I can use"
- "test **<model>**" / "smoke test" / "ping gemini-2.5-flash"
- "show me **smart** / **code** / **vision** models" (filter by tag)
- "add provider **<x>**" / "add **<x>** to the catalog"
- "refresh / re-sync **OpenRouter** / NVIDIA / HF / models"
- "why is **<model>** failing" / "proxy isn't working" / "check the logs"
- "how many keys do I have" / "rotate keys"

If none of the above and the user is talking about LLM APIs generally, check first whether they already have it installed (`./infer providers` works). If yes, offer to route their request through the gateway.

## Layout

The skill lives at the repo root — everything is co-located:

- `infer` — the CLI. Run as `./infer <command>`.
- `providers.toml` — committed catalog of all supported providers and their models.
- `docker-compose.yml` — defines the LiteLLM proxy service.
- `SKILL.md` — this file.

User data lives outside the skill in `~/.config/free-inference/`:

- `keys.env` — user's keys, chmod 600, never synced with the skill.
- `config.yaml` — auto-generated every `./infer start` / `./infer reload`.
- `.discovery-cache.json` — 24h cache for auto-discovered model lists.

## Commands cheatsheet

Drive the CLI directly with `./infer`:

```
./infer keys add <provider> <key>      Store a key (numbered slots auto-rotate)
./infer keys list                      Masked list of all keys
./infer keys remove <provider> [--index N]

./infer start                          Generate config + boot proxy
./infer stop                           docker compose down
./infer reload                         Regenerate + force-recreate
./infer restart                        Bounce without regenerating
./infer status                         Health + active providers
./infer logs [-f]                      Tail litellm logs

./infer providers                      Active vs inactive vs keyless
./infer models [-g GROUP]              Models the user can call
./infer sync                           Refresh auto-discovered catalogs
./infer config show                    Print generated config.yaml
./infer config path                    Print runtime paths

./infer test <model> [--prompt P]              OpenAI shape
./infer test-anthropic <model> [--prompt P]    Anthropic shape (/v1/messages)

./infer detect                                 Scan for installed client CLIs (pi, opencode, cc, openclaw, hermes)
./infer wire <tool>                            Edit client config so it routes through fi (backs up to .fi-backup)
./infer unwire <tool>                          Restore the client config from the backup
```

The first `./infer start` prints a generated **master key** (`sk-fi-…`). That's what clients use, not the underlying provider keys.

## Capability tags

Each model in `providers.toml` carries capability tags (`groups = [...]`). These are **filter metadata, not callable model IDs**. Use them to browse the catalog:

```bash
./infer models -g smart       # show models tagged "smart"
./infer models -g code        # coder-tuned models
./infer models --working -g vision   # working vision-capable models
```

Known tags: `fast`, `smart`, `code`, `reasoning`, `vision`, `embed`, `rerank`, `free`.

**Always call a concrete alias** in a request (`gemini-2.5-flash`, `meta/llama-3.3-70b-instruct`, `qwen/qwen3-coder-480b-a35b-instruct`). The gateway does not expose `fast`/`smart`/etc. as callable model names — routing stays deterministic and logs are meaningful.

## Catalog schema — `providers.toml`

Adding a provider = appending one block, then `./infer reload`. No code change.

```toml
[[provider]]
name = "scaleway"
key_env = "SCALEWAY_API_KEY"
base_url = "https://api.scaleway.ai/v1"
litellm_prefix = "openai"           # or "gemini", "cohere", "nvidia_nim", "huggingface", …
optional_key = false                # true for keyless providers (LLM7)

[[provider.model]]
alias = "qwen3.5-397b-scaleway"     # caller asks for this
upstream = "qwen3.5-397b-a17b"      # upstream's real model id
groups = ["smart", "code"]
rpm = 60                            # optional, used by the rate-limit router
tpm = 100000                        # optional
```

### Auto-discovery

Instead of (or alongside) static `[[provider.model]]` blocks, declare `discovery = "<kind>"`:

| Kind | Source | Auth |
|------|--------|------|
| `openrouter_free` | `openrouter.ai/api/v1/models`, filtered to `pricing.prompt == 0` | optional |
| `nvidia_nim` | `integrate.api.nvidia.com/v1/models` | required (the key) |
| `huggingface_inference` | `router.huggingface.co/v1/models` (live providers, HF $0.10/mo credits) | required |

Results cache 24h at `~/.config/free-inference/.discovery-cache.json`. Users can run `./infer sync` to force-refresh. Discovery skips providers without a key (except keyless providers).

### LiteLLM prefix pitfalls

`litellm_prefix` determines how LiteLLM translates `/v1/messages` (Anthropic shape). The `openai` prefix routes `/v1/messages` through OpenAI's *Responses API* (`/v1/responses`), which most upstreams don't expose — requests 404. Use the dedicated prefix for providers that have one (`nvidia_nim`, `cohere`, `gemini`, `huggingface`, `voyage`). The `openai` prefix is still fine for `/v1/chat/completions`.

## Wiring coding-agent CLIs

`./infer detect` / `./infer wire <tool>` / `./infer unwire <tool>` auto-configure five client CLIs to route through the gateway. Each `wire` backs up the original config to `<path>.fi-backup`; `unwire` restores from it.

| Tool  | Binary    | Config file                                        | Mechanism | What gets written |
|-------|-----------|----------------------------------------------------|-----------|-------------------|
| `cc`  | `claude`  | `~/.claude/settings.json`                          | direct JSON edit | `env.ANTHROPIC_BASE_URL` + master key, settings chmod 600 |
| `opencode` | `opencode` | `~/.config/opencode/opencode.json`           | direct JSON edit | `provider.fi` block (`@ai-sdk/openai-compatible`, localhost:4000/v1, curated default models: Gemini Flash, Llama 3.3 70B, DeepSeek V3.2, Qwen3 Coder 480B, OR free, LLM7 GPT-OSS) |
| `pi`  | `pi`      | `~/.pi/agent/models.json`                          | direct JSON edit | `providers.fi` (`api: openai-completions`, localhost:4000/v1, same curated default model list) |
| `openclaw` | `openclaw` | `~/.openclaw/config.yaml`                     | shells out to `openclaw onboard --custom-base-url …` | OpenClaw's onboarding wizard registers fi as a custom OpenAI-compatible provider |
| `hermes` | `hermes` | `~/.hermes/config.yaml`                          | shells out to `hermes config set model.…` | Sets `model.provider=custom`, `base_url`, `api_key`, and `default` |

Trigger phrases: "wire Claude Code to fi", "wire openclaw / hermes / pi to fi", "point my CLI at fi", "set up fi for opencode", "detect my coding agents", "undo the fi wiring".

After wiring, tell the user to run `./infer start` if the proxy isn't up yet — the edited configs are harmless when localhost:4000 is unreachable (clients just error at request time).

## Workflows

### Onboarding (first-time user)

1. Ask which providers they have keys for. Common easy wins: Gemini (generous free tier), OpenRouter (aggregates many free models), Groq (fastest), NVIDIA NIM (135 models on free credits).
2. For each: `./infer keys add <provider> <key>`.
3. `./infer start` — note the printed master key.
4. `./infer test gemini-2.5-flash` (or any concrete alias from `./infer models`) — verify routing works end-to-end.
5. Show the OpenAI/Anthropic SDK snippet so they can point their client at `http://localhost:4000`.

### Adding a provider not in the catalog

1. Check their README or docs for base URL, auth scheme, and available models.
2. Append a `[[provider]]` block to `providers.toml` with the right `litellm_prefix`.
3. Add at least one `[[provider.model]]` block — or `discovery = "<kind>"` if the upstream has a catalog endpoint (`/v1/models`).
4. `./infer keys add <name> <key>` (unless `optional_key = true`).
5. `./infer reload`.
6. `./infer test <alias>` to verify.

### "No providers active" / 401 / 403

- `./infer keys list` — verify the key saved under the right `<PROVIDER>_API_KEY_N` slot.
- `./infer providers` — check whether the provider is marked active.
- `./infer config show` — inspect the generated LiteLLM config for that model.
- Confirm `key_env` in `providers.toml` matches the env var the upstream actually reads.

### "404 on Anthropic shape"

Almost always a `litellm_prefix` issue. Switch the provider from `openai` to its dedicated LiteLLM prefix and `./infer reload`.

### Adding a second key (rate-limit scaling)

`./infer keys add gemini <second-key>` — stored as `GEMINI_API_KEY_2`. The router round-robins across both deployments on the same model, doubling effective RPD.

## Hard rules

- **Never push to git** unless the user explicitly says so.
- **Never commit `~/.config/free-inference/*`** — those are user secrets.
- **Never add Co-Authored-By or Claude/Anthropic attribution** to commits, PRs, or any generated content.
- **Never mock** an upstream provider — if a key is missing, fail loudly rather than fake a response.
- **Never edit `fi` to add third-party dependencies** without asking — the CLI is stdlib-only on purpose.
- **Never store keys in `providers.toml`** — they belong in `keys.env` only.

## Quick patterns

**Ask Claude to set everything up:**
> "add my gemini key AIza…, my nvidia key nvapi-…, then start the proxy and tell me the master key"

Claude should:
1. `./infer keys add gemini AIza…`
2. `./infer keys add nvidia nvapi-…`
3. `./infer start`
4. Report the `sk-fi-…` master key + the OpenAI SDK snippet.

**Ask Claude to debug:**
> "why is `./infer test meta/function-xyz-v2` returning 404?"

Claude should:
1. `./infer logs | tail -40` to find the failing upstream.
2. If an NVIDIA alias, check whether the user's account has access to that specific NVIDIA function ID — many need per-account approval and will always 404.
3. Suggest `./infer probe && ./infer models --working` to narrow to aliases that actually respond for this account.

**Ask Claude to add a new provider:**
> "add SambaNova — key is samba_… and they have Llama 3.3 70B"

Claude should:
1. Look up SambaNova's base URL and LiteLLM prefix (check `providers.toml` for the pattern).
2. Append a `[[provider]]` block plus one `[[provider.model]]` for Llama 3.3 70B.
3. `./infer keys add sambanova samba_…`.
4. `./infer reload`.
5. `./infer test llama-3.3-70b-sambanova` to verify.
