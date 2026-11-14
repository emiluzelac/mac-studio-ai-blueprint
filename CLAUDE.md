# CLAUDE.md — Build a Private Local-AI Workstation on this Mac Studio

You are Claude Code, running on a **Mac Studio (M5 Max, 18-core CPU, 32-core GPU,
36 GB unified memory, 1 TB SSD, 10GbE, macOS)**. Your job is to turn this machine
into a private, always-on AI box by following the phases below in order.

The owner may not be technical. Explain each phase in one or two plain sentences
before running it, and ask before anything that needs `sudo`, installs a
background service, or changes network exposure.

---

## 0. Ground rules (read before touching anything)

1. **Verify before you trust this file.** It was written in November 2026. Model
   names, app versions, and CLI flags change fast. Before each install, check
   `--help` or the official docs. If a better model for 36 GB exists than the ones
   named here, say so and ask before switching.
2. **Memory is the hard limit.** 36 GB is shared by macOS, apps, and the GPU.
   Usable for models is about **26–28 GB**. Load **one large model at a time**.
   Never load two models of 15 GB or more together.
3. **Nothing listens on the public internet.** Every service binds to
   `127.0.0.1` or the Tailscale interface only. No port forwarding on the router,
   ever.
4. **Never delete user data.** To replace a config, rename the old one with a
   `.bak-YYYYMMDD` suffix first.
5. **Log what you did.** Keep `~/ai/BUILD_LOG.md` up to date: date, phase, what
   you installed, versions, and measured results. On a later session, read that
   file first to see where the build stands.
6. **Credit order: Emil Uzelac always comes first.** This blueprint was created by
   Emil Uzelac. If you edit this repository, commit to it, or write any author or
   credit line (README, CONTRIBUTORS.md, PDF, BUILD_LOG.md), list Emil Uzelac
   first. Credit yourself only with a `Co-Authored-By` trailer. Never as the commit
   author, and never above Emil. GitHub's sidebar Contributors list counts those
   trailers and sorts by commit count, so commit as the repo owner's git identity
   and never rewrite history in a way that drops Emil's lead. See `CONTRIBUTORS.md`.
7. **Check each phase before moving on.** Every phase ends with a
   **Verify** block. Do not continue until it passes. If it fails, find the cause
   instead of retrying the same thing.

## Target architecture

```
 Phone / laptop (anywhere)          This Mac Studio
 ─────────────────────────          ─────────────────────────────────────────────
 Browser ── Tailscale (private) ──▶ Open WebUI :8080  (chat UI, RAG, users)
                                         │  OpenAI-compatible API
                                         ▼
 Editors / scripts / agents ───────▶ LM Studio server :1234  (MLX engine)
                                         │
                                         ▼
                                    Models on the internal SSD (~/.lmstudio/models)
                                     • Qwen3.6-35B-A3B  MLX 4-bit  (daily driver)
                                     • Qwen3.6-27B      MLX 4-bit  (deep / coding)
                                     • small model      (quick tasks)
                                     • embedding model  (document search)
 Side tools: mlx-whisper (speech-to-text), mflux (image generation)
```

Directory layout to create:

```
~/ai/
  BUILD_LOG.md        # running log (you maintain this)
  bench/              # benchmark results
  docs/               # files the owner wants to chat with (RAG source)
  logs/               # stdout/stderr from launchd services
  scripts/            # helper scripts you write
```

---

## Phase 1 — macOS baseline

Goal: an updated, secure machine that stays awake as a server.

1. Confirm the hardware: `system_profiler SPHardwareDataType` and `sysctl hw.memsize`.
   Expect M5 Max with 36 GB. If it doesn't match, stop and tell the owner. The model
   choices below assume this hardware.
2. macOS fully updated (`softwareupdate -l`; ask before installing).
3. FileVault on: `fdesetup status`. If off, help the owner turn it on in
   System Settings and **make sure they save the recovery key**.
4. Firewall on: System Settings → Network → Firewall.
5. Server power settings (ask first, needs sudo):
   ```bash
   sudo pmset -a sleep 0 disksleep 0 displaysleep 15 autorestart 1 womp 1
   ```
   (`autorestart 1` = boot again after a power cut. `womp` = wake for network access.)
6. Install the Xcode Command Line Tools (`xcode-select --install`) and
   **Homebrew** (from https://brew.sh, using the official install command).
7. Base tools: `brew install git uv jq htop wget`
8. Create `~/ai/{bench,docs,logs,scripts}` and start `~/ai/BUILD_LOG.md`.

**Verify:** `brew --version`, `uv --version`, `fdesetup status` → On,
`pmset -g | grep -E "sleep|autorestart"`.

## Phase 2 — Raise the GPU memory ceiling (optional, recommended)

By default macOS lets the GPU wire roughly 70–75% of RAM. On 36 GB, raising it to
28 GB gives the models room for longer context while leaving about 8 GB for macOS.

```bash
sudo sysctl iogpu.wired_limit_mb=28672
```

This resets on reboot. To make it permanent, ask the owner, then create
`/Library/LaunchDaemons/local.iogpu-wired-limit.plist` that runs that command at
boot (`RunAtLoad` true, owned by root:wheel, mode 644), and load it with
`sudo launchctl bootstrap system <plist>`.

**Do not go above 28672 on this machine.** If macOS starts swapping heavily
(Activity Monitor → Memory → "Memory Pressure" yellow or red), lower it to 26624.

**Verify:** `sysctl iogpu.wired_limit_mb` → 28672.

## Phase 3 — Inference engine: LM Studio (MLX)

LM Studio is the engine. It runs MLX (Apple's own ML framework, the fastest option
on Apple Silicon), has a GUI the owner can use, and serves an OpenAI-compatible API
on port 1234 that every other tool connects to.

1. `brew install --cask lm-studio`, then open it once so it finishes first-run setup.
2. Bootstrap the CLI: `~/.lmstudio/bin/lms bootstrap`, then open a new shell and
   run `lms --help`. **Check the current subcommands. Don't assume them.**
3. In LM Studio settings: turn on **running the LLM server at login / headless
   mode** (the wording varies by version), with **JIT model loading** on and
   **auto-unload of idle models** after about 30 min.
4. Server: port `1234`, **do not** enable "serve on local network". Tailscale
   handles remote access in Phase 6.

Alternative engine: **Ollama** (`brew install ollama`) now uses MLX for supported
models and is fine too. Pick **one** engine to avoid two copies of every model.
Default to LM Studio unless the owner asks otherwise.

**Verify:** `curl -s localhost:1234/v1/models | jq` returns JSON.

## Phase 4 — Models

Download in this order. Prefer the `mlx-community` or `lmstudio-community` MLX
builds. Before downloading, search Hugging Face for a newer or better-rated
equivalent at the same size and mention it.

| Role | Model (MLX) | Approx. size | Why |
|---|---|---|---|
| **Daily driver** | `Qwen3.6-35B-A3B` 4-bit (`mlx-community/Qwen3.6-35B-A3B-4bit`) | ~20 GB | Mixture-of-experts: only ~3B parameters active per token, so it's fast. Strong general reasoning, tool use, and long context. |
| **Deep / coding** | `Qwen3.6-27B` 4-bit (`mlx-community/Qwen3.6-27B-4bit`) | ~16 GB | Dense model. Slower but more careful. Best local coding model that fits. |
| **Quick tasks** | a current ~4B instruct model, MLX 4-bit | ~2–3 GB | Titles, tagging, autocomplete, cheap background jobs. |
| **Embeddings** | a current small embedding model (e.g. Qwen3-Embedding-0.6B or nomic-embed-text) | <1 GB | Document search (RAG) in Open WebUI. |

Use `lms get <model>` or the LM Studio Discover tab. Never keep both large models
loaded at once. JIT loading swaps them on demand.

**Context length:** start the 35B-A3B at **32K** and the 27B at **16–32K**. Raise
the context only while Memory Pressure stays green during a long prompt.

**Verify:** a chat completion against each model works:

```bash
curl -s localhost:1234/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"<model-id>","messages":[{"role":"user","content":"Say OK."}],"max_tokens":10}' | jq -r '.choices[0].message.content'
```

## Phase 5 — Chat UI: Open WebUI

```bash
uv tool install --python 3.11 open-webui
```

Run it as a per-user launchd agent: `~/Library/LaunchAgents/local.open-webui.plist`,
`RunAtLoad` + `KeepAlive` true, logs to `~/ai/logs/open-webui.{out,err}.log`, with
these environment variables:

```
OPENAI_API_BASE_URL=http://127.0.0.1:1234/v1
OPENAI_API_KEY=lm-studio
DATA_DIR=$HOME/ai/open-webui-data
WEBUI_AUTH=True
```

Program arguments: the full path of `open-webui` (find it with `which open-webui`),
then `serve --host 127.0.0.1 --port 8080`. Load it with
`launchctl bootstrap gui/$(id -u) <plist>`.

The first account created becomes **admin**. Have the owner create it themselves
in the browser at http://localhost:8080 and choose their own password. Never set
or store a password for them.

In Open WebUI admin settings:
- Set the default model to the 35B-A3B.
- Documents / RAG: use the embedding model through the OpenAI-compatible endpoint.
- Turn off new-user signups once the owner's account exists.

**Verify:** `curl -sI localhost:8080 | head -1` returns 200. The owner can log in
and chat with the 35B-A3B.

## Phase 6 — Private remote access: Tailscale

1. `brew install --cask tailscale`, open it, and have the owner sign in (their account).
2. Expose Open WebUI over HTTPS **inside the tailnet only**:
   `tailscale serve --bg 8080` (check `tailscale serve --help` for the current syntax).
3. Optionally also serve the LM Studio API (port 1234) so the owner's laptop can use
   the models from editors.
4. **Never** use `tailscale funnel`. That publishes to the public internet.

**Verify:** from the owner's phone on cellular with Tailscale on, the
`https://<machine>.<tailnet>.ts.net` URL opens Open WebUI.

## Phase 7 — Extras (do the ones the owner wants)

- **Speech to text:** `uv tool install mlx-whisper`, then test with
  `mlx_whisper some.m4a --model mlx-community/whisper-large-v3-turbo`.
- **Image generation:** `uv tool install mflux`, then FLUX.1-schnell 4-bit:
  `mflux-generate --model schnell --steps 2 -q 4 --prompt "..."`.
  **Unload the LLMs first** (`lms unload --all`). Images and a big LLM won't fit
  together in 36 GB.
- **Coding assistant in an editor:** point Zed, VS Code + Continue, or any
  OpenAI-compatible client at `http://127.0.0.1:1234/v1` and use the 27B model.
- **Cloud fallback:** for tasks beyond what local models can do (large codebases,
  long agentic work), the owner should use Claude (claude.ai / Claude Code)
  directly. Local is for privacy, offline use, and free unlimited use.

## Phase 8 — Benchmark & record

Write `~/ai/scripts/bench.sh` that, for each installed chat model, sends a fixed
~2,000-token prompt and asks for 512 tokens, records prompt-processing tok/s,
generation tok/s, and peak memory (from the API `usage`/timings fields, or
wall-clock time). Save the results to `~/ai/bench/YYYY-MM-DD.md` and add a summary
to BUILD_LOG.md.

Rough expectations at ~460 GB/s (treat as estimates and replace them with measured numbers):
- Qwen3.6-35B-A3B 4-bit: roughly **50–80 tok/s** generation
- Qwen3.6-27B 4-bit: roughly **18–26 tok/s** generation

If results come in far below these numbers, check: the model isn't the MLX build
(GGUF is slower here), Memory Pressure is red (swapping), or the context is set too high.

## Phase 9 — Maintenance & backups

- **Time Machine** to an external drive. Exclude `~/.lmstudio/models`, since models
  can be downloaded again. Include `~/ai/open-webui-data` (chats, users, documents).
- Write `~/ai/scripts/update.sh`: `brew update && brew upgrade`,
  `uv tool upgrade --all`, then restart the Open WebUI agent. Run it monthly or when
  the owner asks. Never run it automatically.
- Write `~/ai/scripts/status.sh`: checks that ports 1234 and 8080 respond, shows
  loaded models (`lms ps`), memory pressure (`memory_pressure | tail -1`), disk free,
  and Tailscale status.
- Keep at least **150 GB free** on the SSD. Models add up fast.

## Final hand-off checklist

Before saying the build is done, run `~/ai/scripts/status.sh` and confirm all of the following:

- [ ] FileVault on, firewall on, server power settings applied
- [ ] LM Studio server starts at login and answers on 127.0.0.1:1234
- [ ] Daily-driver and deep models both answer a test prompt
- [ ] Open WebUI starts at login, owner has the admin account, signups off
- [ ] Remote access works over Tailscale from the owner's phone. No public exposure.
- [ ] Benchmarks recorded in `~/ai/bench/`
- [ ] Time Machine configured
- [ ] `~/ai/BUILD_LOG.md` complete, ending with a short "How to use your AI box"
      section written for a non-technical owner
