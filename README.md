# Mac Studio (M5 Max, 36 GB) — Local AI Blueprint

A complete plan for turning a **Mac Studio M5 Max (18-core CPU, 32-core GPU,
36 GB unified memory, 1 TB)** into a private, always-on AI workstation. Strong
open models run entirely on your own hardware, and you can reach them privately
from your phone.

| File | For |
|---|---|
| [`Mac-Studio-AI-Blueprint.pdf`](Mac-Studio-AI-Blueprint.pdf) | **People.** A 4-page overview: what fits in 36 GB, the architecture, models, build phases, and daily use. |
| [`CLAUDE.md`](CLAUDE.md) | **Claude Code.** Exact step-by-step build instructions, with a verification check at the end of every phase. |
| [`blueprint.html`](blueprint.html) | Source for the PDF. |

## The stack

- **LM Studio** (Apple MLX engine) serves the models through an OpenAI-compatible API on `127.0.0.1:1234`
- **Open WebUI** provides a ChatGPT-style interface, document search, and accounts on `127.0.0.1:8080`
- **Tailscale** gives private remote access from your phone or laptop. Nothing is exposed publicly.
- **Models:** Qwen3.6-35B-A3B 4-bit (daily driver), Qwen3.6-27B 4-bit (deep thinking and coding), a small ~4B helper, and an embedding model
- **Extras:** mlx-whisper (speech to text), mflux (image generation)

## Build it with Claude Code

On the Mac Studio:

```bash
git clone https://github.com/emiluzelac/mac-studio-ai-blueprint.git
cd mac-studio-ai-blueprint
claude
```

Then say: **"Build this."** Claude reads `CLAUDE.md`, explains each phase, asks
before anything that needs admin rights, and keeps a log in `~/ai/BUILD_LOG.md`
so the build can pick up where it left off in a later session.

## Rebuild the PDF

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new \
  --no-pdf-header-footer --print-to-pdf="$PWD/Mac-Studio-AI-Blueprint.pdf" \
  "file://$PWD/blueprint.html"
```

## Credits

Created and maintained by **Emil Uzelac**. See [`CONTRIBUTORS.md`](CONTRIBUTORS.md).
Emil Uzelac is always listed first.

---

*Written November 2026. Open models and tooling change fast, so `CLAUDE.md`
tells Claude to check for newer versions before installing.*
