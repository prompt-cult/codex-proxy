# codex-proxy

A Rust LLM **proxy stack** that holds the credentials for upstream LLM APIs so
that a tool or harness inside a jail can talk to the models **without ever
holding the model API keys**.

The harness talks to a loopback proxy that speaks the OpenAI Responses wire
API. The proxy holds the upstream credentials in `mlock(2)`-protected memory,
injects them into upstream requests, and translates the responses back. The
harness has no knowledge of the key — that is the entire point of the
privilege-separation model.

This repository is **not** the Codex CLI and does not contain the full
`codex-rs` workspace. It is a curated, slimmed workspace containing only the
proxy crates and their transitive first-party dependencies. The Rust code is
pinned to toolchain `1.93.0`; the workspace dependency set tracks the latest
resolved versions from upstream `openai/codex` (security updates included).

## Architecture

```
harness (jailed, keyless)
   │  OpenAI Responses API (127.0.0.1)
   ▼
codex-proxy-router   — loopback dispatcher (127.0.0.1:9090)
   │  routes by first path segment, boots children keyless
   ├── /opencode.ai/...  →  codex-opencode-proxy  (Zen paid / Go open models)
   └── /mistral.ai/...   →  codex-mistral-proxy   (Mistral models)
```

Each proxy is a standalone binary; the router is an optional dispatcher that
puts them behind one loopback address.

| Binary | Upstream | Wire contract |
| --- | --- | --- |
| `codex-opencode-proxy` | OpenCode Zen (`https://opencode.ai/zen/v1`) and Go (`/zen/go/v1`) | OpenAI Responses → passthrough / Anthropic Messages / Chat Completions translation |
| `codex-mistral-proxy` | Mistral (`https://api.mistral.ai`) | OpenAI Responses → Mistral Chat Completions translation |
| `codex-proxy-router` | — (dispatcher) | routes `/opencode.ai/*` and `/mistral.ai/*` to child proxies verbatim |

### codex-opencode-proxy

One binary, one key (`OPENCODE_API_KEY`), two upstream bases selected with
`--upstream-base`. Requests are routed by model family:

- `gpt-*`, `o1*`, `o3*`, `grok-*`, `muse-*`, … → `{base}/responses` (passthrough)
- `claude-*`, `minimax-*`, `qwen*`, … → `{base}/messages` (OAI Responses ↔ Anthropic Messages translation)
- `glm-*`, `kimi-*`, `deepseek-*`, `longcat-*`, `mimo-*`, `hy-*`, … → `{base}/chat/completions` (OAI Responses ↔ OpenAI-compatible Chat Completions translation)

Both bases share `GET /v1/models` discovery. The proxy is the client for the
`User-Agent` and `x-opencode-session` headers — client-supplied values are
never forwarded. Config (optional) lives in the codex config directory
(`CODEX_CONFIG_DIR` > `CODEX_HOME` > `~/.codex`) as
`proxy-opencode-zen.jsonc` / `proxy-opencode-go.jsonc` with keys
`upstream_base_url`, `model_exclude_globs`, `log_level`, `model_overrides`.

### codex-mistral-proxy

Mistral-only by design — no model-family routing, so any model Mistral exposes
works automatically. Exposes `POST /v1/responses`, `GET /v1/models`
(query string ignored, chat-capable models only), `GET /health`. Config:
`proxy-mistral-ai.jsonc` in the config directory.

### codex-proxy-router

Boots the provider proxies as children, one per upstream host, and relays
requests, headers, query strings, bodies, and SSE streams verbatim under
`/opencode.ai/<contract>` and `/mistral.ai/<contract>`. Serves `GET /health`
and (with `--http-shutdown`) `GET /shutdown`, which cleanly terminates child
proxies. Unknown routes return 404.

## Secret supply (Prompt Cult Proxy Protocol v1)

The shared contract lives in `codex-rs/proxy-protocol` and is documented in
`docs/proxy-protocol.md`. The router and provider proxies implement three
channels, selected with `--secret-mode`:

1. **`secret-push` (default, production).** Children are spawned keyless with
   a clean environment and closed stdin. The router generates a 256-bit boot
   token from the OS CSPRNG, writes it to a `0600` file in the child's temp
   directory, and passes only the file path via `--boot-token-file` — the
   token never appears on `argv`. The router pushes provider keys to each
   child's `POST /protocol/v1/secrets` using `X-Router-Boot-Token`; the child
   deletes the token file, ingests the key into `mlock(2)`-protected memory,
   zeroizes the request body, consumes the single-use token, and returns 204.
   Upstream requests are held (503) until key delivery completes. The
   protocol control plane is child-local — the router never forwards
   `/protocol/v1/secrets` from its own clients.
2. **`workload-identity` (reserved).** Zero secret material is passed by the
   router; children resolve credentials via platform mechanisms (KMS, IAM,
   Kubernetes Pod Identity, Vault). The first-party proxies do **not**
   implement this channel and fail negotiation loudly against it.
3. **`env-debug` (local testing only).** The router copies allow-listed env
   vars (`OPENCODE_API_KEY`, `MISTRAL_API_KEY`) into child environments and
   prints a prominent warning. Children unset the variable immediately after
   reading and zeroize raw buffers.

Standalone (not under the router), a proxy reads its key from the environment
variable (`MISTRAL_API_KEY` / `OPENCODE_API_KEY`), unsets it immediately, or
from stdin via raw `read(2)` when run interactively. Key bytes are zeroized
after use; the final `"Bearer <key>"` string is heap-allocated once and locked
with `mlock(2)` so it is never swapped to disk.

## Process hardening

`codex-rs/process-hardening` (`pre_main_hardening`, on at startup in every
binary) disables core dumps, prevents ptrace attach on Linux/macOS, removes
dangerous environment variables (`LD_*`, `DYLD_*`, and macOS malloc
stack-logging controls), and clears `RLIMIT_CORE` on BSDs.

## Building and running

```shell
cd codex-rs
cargo build --release
```

Standalone Mistral proxy:

```shell
MISTRAL_API_KEY=... ./target/release/codex-mistral-proxy --port 8901
# or pipe the key:  printenv MISTRAL_API_KEY | ./target/release/codex-mistral-proxy --port 8901
```

Router with all proxies (local testing; `env-debug` requires the env vars):

```shell
export MISTRAL_API_KEY=...
export OPENCODE_API_KEY=...
./target/release/codex-proxy-router --port 9090 --http-shutdown --secret-mode env-debug
```

Harness setup — point a provider at the loopback proxy and leave the key env
var empty:

```toml
[[providers]]
name = "opencode-zen"
api_base = "http://127.0.0.1:9099/v1"
api_key_env_var = ""          # the proxy holds the key; the harness must not
api_style = "openai-responses"
backend = "generic"

[[models]]
name = "gpt-5.4-mini"
provider = "opencode-zen"
```

**Do not set `api_key_env_var` to a real env var.** The harness must have no
knowledge of the key.

## Repository layout

```
codex-rs/
  mistral-proxy/     codex-mistral-proxy binary + e2e suite
  opencode-proxy/    codex-opencode-proxy binary + e2e suite
  codex-router/      codex-proxy-router binary + secret-flow suite
  proxy-protocol/    Prompt Cult Proxy Protocol v1 types and negotiation
  process-hardening/ pre_main_hardening lib
  utils/             absolute-path, home-dir (CODEX_CONFIG_DIR > CODEX_HOME)
docs/proxy-protocol.md   protocol spec
README.opencode-proxy.md / README.proxy-router.md / codex-rs/mistral-proxy/README.md
```

## Tests

`cargo test --workspace` covers unit tests plus three end-to-end suites:
`mistral-proxy` (13), `opencode-proxy` (16), and `codex-router` (5). The e2e
suites boot the real binaries against local mock upstreams (Mistral mock,
OpenCode fixture responses, protocol stubs) — no external network or real
credentials are required.
