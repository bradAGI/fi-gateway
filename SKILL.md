---
name: fi
description: Drive the `fi` unified inference gateway — a Python CLI that aggregates 300+ free LLM models (Gemini, Groq, OpenRouter, NVIDIA NIM, Hugging Face, Cerebras, Mistral, Scaleway, Voyage, Jina, Pollinations, Cohere, Together, Hunyuan, Chutes, LLM7, Ollama Cloud) behind a single OpenAI- and Anthropic-compatible endpoint at http://localhost:4000. Trigger whenever the user asks to add API keys for any LLM provider; start/stop/check a local LLM proxy; list available models; test a model or group; add a new provider to `providers.toml`; refresh auto-discovered catalogs; debug routing, fallbacks, or upstream errors; or mentions "fi", "my free keys", "free models", "local proxy", "free inference", "gateway".
---

# fi — Free-Inference Gateway

This skill teaches your agent how to drive `fi`, a lightweight CLI at the skill root that turns the user's free API keys into one OpenAI- and Anthropic-compatible endpoint.

## When to use this skill

Trigger phrases:

- "add my **<provider>** key" / "store my **<provider>** key" / "I got a **<provider>** key"
- "start / boot / spin up the **proxy** / **gateway** / fi"
- "what free **models** do I have" / "list my models" / "models I can use"
- "test **<model>**" / "smoke test" / "ping gemini-2.5-flash"
- "show me **smart** / **code** / **vision** models" (filter by tag)
- "add provider **<x>**" / "add **<x>** to the catalog"
- "refresh / re-sync **OpenRouter** / NVIDIA / HF / models"
- "why is **<model>** failing" / "proxy isn't working" / "check the logs"
- "how many keys do I have" / "rotate keys"

If none of the above and the user is talking about LLM APIs generally, check first whether they already have `fi` installed (`./fi providers` works). If yes, offer to route their request through the gateway.

## Layout

The skill lives at the repo root — everything is co-located:

- `fi` — the CLI. Run as `./fi <command>`.
- `providers.toml` — committed catalog of all supported providers and their models.
- `docker-compose.yml` — defines the LiteLLM proxy service.
- `SKILL.md` — this file.

User data lives outside the skill in `~/.config/free-inference/`:

- `keys.env` — user's keys, chmod 600, never synced with the skill.
- `config.yaml` — auto-generated every `./fi start` / `./fi reload`.
- `.discovery-cache.json` — 24h cache for auto-discovered model lists.

## Commands cheatsheet

Drive the CLI directly with `./fi`:

```
./fi keys add <provider> <key>      Store a key (numbered slots auto-rotate)
./fi keys list                      Masked list of all keys
./fi keys remove <provider> [--index N]

./fi start                          Generate config + boot proxy
./fi stop                           docker compose down
./fi reload                         Regenerate + force-recreate
./fi restart                        Bounce without regenerating
./fi status                         Health + active providers
./fi logs [-f]                      Tail litellm logs

./fi providers                      Active vs inactive vs keyless
./fi models [-g GROUP]              Models the user can call
./fi sync                           Refresh auto-discovered catalogs
./fi config show                    Print generated config.yaml
./fi config path                    Print runtime paths

./fi test <model> [--prompt P]              OpenAI shape
./fi test-anthropic <model> [--prompt P]    Anthropic shape (/v1/messages)

./fi detect                                 Scan for installed client CLIs (pi, opencode, cc)
./fi wire <tool>                            Edit client config so it routes through fi (backs up to .fi-backup)
./fi unwire <tool>                          Restore the client config from the backup
```

The first `./fi start` prints a generated **master key** (`sk-fi-…`). That's what clients use, not the underlying provider keys.

## Capability tags

Each model in `providers.toml` carries capability tags (`groups = [...]`). These are **filter metadata, not callable model IDs**. Use them to browse the catalog:

```bash
./fi models -g smart       # show models tagged "smart"
./fi models -g code        # coder-tuned models
./fi models --working -g vision   # working vision-capable models
```

Known tags: `fast`, `smart`, `code`, `reasoning`, `vision`, `embed`, `rerank`, `free`.

**Always call a concrete alias** in a request (`gemini-2.5-flash`, `meta/llama-3.3-70b-instruct`, `qwen/qwen3-coder-480b-a35b-instruct`). The gateway does not expose `fast`/`smart`/etc. as callable model names — routing stays deterministic and logs are meaningful.

## Catalog schema — `providers.toml`

Adding a provider = appending one block, then `./fi reload`. No code change.

```toml
[[provider]]
name = "scaleway"
key_env = "SCALEWAY_API_KEY"
base_url = "https://api.scaleway.ai/v1"
litellm_prefix = "openai"           # or "gemini", "cohere", "nvidia_nim", "huggingface", …
optional_key = false                # true for keyless providers (Pollinations, LLM7)

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
| `pollinations` | `gen.pollinations.ai/v1/models` | none |

Results cache 24h at `~/.config/free-inference/.discovery-cache.json`. Users can run `./fi sync` to force-refresh. Discovery skips providers without a key (except keyless providers like Pollinations).

### LiteLLM prefix pitfalls

`litellm_prefix` determines how LiteLLM translates `/v1/messages` (Anthropic shape). The `openai` prefix routes `/v1/messages` through OpenAI's *Responses API* (`/v1/responses`), which most upstreams don't expose — requests 404. Use the dedicated prefix for providers that have one (`nvidia_nim`, `cohere`, `gemini`, `huggingface`, `voyage`). The `openai` prefix is still fine for `/v1/chat/completions`.

## Wiring coding-agent CLIs

`./fi detect` / `./fi wire <tool>` / `./fi unwire <tool>` auto-configure three client CLIs to route through the gateway. Each `wire` backs up the original config to `<path>.fi-backup`; `unwire` restores from it.

| Tool  | Binary    | Config file                                        | What gets written |
|-------|-----------|----------------------------------------------------|-------------------|
| `cc`  | `claude`  | `~/.claude/settings.json`                          | `env.ANTHROPIC_BASE_URL=http://localhost:4000` + master key. Settings chmod 600. |
| `opencode` | `opencode` | `~/.config/opencode/opencode.json`          | `provider.fi` block with `@ai-sdk/openai-compatible`, localhost:4000/v1, master key, and a curated default model list (Gemini Flash, Llama 3.3 70B, DeepSeek V3.2, Qwen3 Coder 480B, OpenRouter free router, LLM7 GPT-OSS). Edit the config to add more. |
| `pi`  | `pi`      | `~/.pi/agent/models.json`                          | `providers.fi` with `api: openai-completions`, localhost:4000/v1, same curated default model list. |

Trigger phrases: "wire Claude Code to fi", "point my CLI at fi", "set up fi for opencode", "detect my coding agents", "undo the fi wiring".

After wiring, tell the user to run `./fi start` if the proxy isn't up yet — the edited configs are harmless when localhost:4000 is unreachable (clients just error at request time).

## Workflows

### Onboarding (first-time user)

1. Ask which providers they have keys for. Common easy wins: Gemini (generous free tier), OpenRouter (aggregates many free models), Groq (fastest), NVIDIA NIM (135 models on free credits).
2. For each: `./fi keys add <provider> <key>`.
3. `./fi start` — note the printed master key.
4. `./fi test gemini-2.5-flash` (or any concrete alias from `./fi models`) — verify routing works end-to-end.
5. Show the OpenAI/Anthropic SDK snippet so they can point their client at `http://localhost:4000`.

### Adding a provider not in the catalog

1. Check their README or docs for base URL, auth scheme, and available models.
2. Append a `[[provider]]` block to `providers.toml` with the right `litellm_prefix`.
3. Add at least one `[[provider.model]]` block — or `discovery = "<kind>"` if the upstream has a catalog endpoint (`/v1/models`).
4. `./fi keys add <name> <key>` (unless `optional_key = true`).
5. `./fi reload`.
6. `./fi test <alias>` to verify.

### "No providers active" / 401 / 403

- `./fi keys list` — verify the key saved under the right `<PROVIDER>_API_KEY_N` slot.
- `./fi providers` — check whether the provider is marked active.
- `./fi config show` — inspect the generated LiteLLM config for that model.
- Confirm `key_env` in `providers.toml` matches the env var the upstream actually reads.

### "404 on Anthropic shape"

Almost always a `litellm_prefix` issue. Switch the provider from `openai` to its dedicated LiteLLM prefix and `./fi reload`.

### Adding a second key (rate-limit scaling)

`./fi keys add gemini <second-key>` — stored as `GEMINI_API_KEY_2`. The router round-robins across both deployments on the same model, doubling effective RPD.

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
1. `./fi keys add gemini AIza…`
2. `./fi keys add nvidia nvapi-…`
3. `./fi start`
4. Report the `sk-fi-…` master key + the OpenAI SDK snippet.

**Ask Claude to debug:**
> "why is `./fi test meta/function-xyz-v2` returning 404?"

Claude should:
1. `./fi logs | tail -40` to find the failing upstream.
2. If an NVIDIA alias, check whether the user's account has access to that specific NVIDIA function ID — many need per-account approval and will always 404.
3. Suggest `./fi probe && ./fi models --working` to narrow to aliases that actually respond for this account.

**Ask Claude to add a new provider:**
> "add SambaNova — key is samba_… and they have Llama 3.3 70B"

Claude should:
1. Look up SambaNova's base URL and LiteLLM prefix (check `providers.toml` for the pattern).
2. Append a `[[provider]]` block plus one `[[provider.model]]` for Llama 3.3 70B.
3. `./fi keys add sambanova samba_…`.
4. `./fi reload`.
5. `./fi test llama-3.3-70b-sambanova` to verify.
