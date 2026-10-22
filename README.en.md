<div align="center">

# weclone-cloud

**Turn your chat logs into an AI twin that talks like you**

No GPU · API-only · WeChat ready · Full runbook included

[![License: AGPL-3.0](https://img.shields.io/badge/license-AGPL--3.0-blue.svg)](https://www.gnu.org/licenses/agpl-3.0)
[![Python 3.12](https://img.shields.io/badge/python-3.12-3776AB.svg?logo=python&logoColor=white)](https://www.python.org/)
[![Platform](https://img.shields.io/badge/platform-Windows-0078D4.svg?logo=windows&logoColor=white)](#)
[![GPU](https://img.shields.io/badge/GPU-not%20required-brightgreen.svg)](#)
[![Based on WeClone](https://img.shields.io/badge/based%20on-WeClone-8A2BE2.svg)](https://github.com/xming521/WeClone)

[**← 中文**](README.md) · **English**

</div>

<!-- Drop a screenshot at docs/images/demo.png and uncomment:
![demo](docs/images/demo.png)
-->

---

## What it does

A subproject of [WeClone](https://github.com/xming521/WeClone). It keeps WeClone's
dataset parsing and replaces everything after it with OpenAI-compatible API calls:

```
WeChat / Telegram export
        │  weclone-cli make-dataset        (WeClone: merge, pair Q&A, scrub PII)
        ▼
dataset/res_csv/sft/sft-my.json
        │  weclone-cloud score             (API scoring 1-5, cached, resumable)
        ▼
sft-my.scored.json
        │  weclone-cloud persona           (style rules + real examples)
        ▼
persona_system.md
        │  weclone-cloud serve             (auth proxy: persona + no thinking + delay)
        ▼
http://127.0.0.1:8010/v1/chat/completions
        │
        └── LangBot ── WeChatPadPro ── personal WeChat
```

## Features

| | |
|---|---|
| **No GPU** | Scoring, persona and chat are API calls; locally you only run a small FastAPI proxy |
| **Corpus driven** | Style rules distilled from your highest-scored conversations, plus real excerpts |
| **Optional fine-tune** | Export OpenAI-format JSONL for platforms exposing `/fine_tuning/jobs` |
| **Drop-in endpoint** | OpenAI-compatible `/v1/chat/completions` — LangBot, AstrBot or any SDK |
| **WeChat ready** | Docker recipe for LangBot + WeChatPadPro, with two adapter patches |
| **Reproducible** | Content-addressed cache, resumable scoring, `manifest.json` with counts and sha256 |
| **Maintainable** | Pinned third-party versions, upgrade checklist, dual-repo release flow |

## Quick start (Windows, ~3 minutes)

```powershell
git clone https://github.com/lhongjie70-sudo/weclone-cloud.git
cd weclone-cloud

# 1) pull WeClone, create .venv, install deps (~300 MB)
.\setup.cmd

# 2) put your key into secrets.local.cmd (created by setup.cmd)
#    set "WECLONE_API_KEY=sk-..."

# 3) start and open the browser
.\start-chat.cmd
```

Then open <http://127.0.0.1:8010/>.

To connect WeChat: run `.\start-wechat.cmd` (needs Docker Desktop) and follow
[install.md](docs/install.md) sections 5–6.

## Commands

| Command | Purpose |
|---|---|
| `doctor` | Check config, keys, connectivity, job status |
| `score` | Score conversations 1–5 via API; writes `score` / `score_reason` |
| `export` | Convert to OpenAI JSONL, dedupe, split validation set |
| `persona` | Generate `persona_system.md` from high-score samples |
| `serve` | Authenticated OpenAI-compatible proxy + built-in web chat |
| `finetune` / `status` / `cancel` | Upload, create and poll a cloud fine-tune job |
| `pipeline` | `score` + `export` in one go |

Run as `python -m weclone_cloud.cli <command>` (or the installed `weclone-cloud`).

## Docs

| Doc | Contents |
|---|---|
| [docs/install.md](docs/install.md) | End-to-end Windows setup: data, scoring, persona, Docker, WeChat |
| [docs/runbook.md](docs/runbook.md) | Daily startup, recovery after reboot, WeChat re-login, troubleshooting |
| [docs/maintenance.md](docs/maintenance.md) | Version matrix, upstream rebase, third-party upgrades, release flow |
| [docs/architecture.md](docs/architecture.md) | Mapping to WeClone, design trade-offs |
| [docs/providers.md](docs/providers.md) | OpenAI / DashScope / local models |

## Tested versions

| Component | Verified | Pinning |
|---|---|---|
| WeClone | `master@d4af5ba` (2026-10-18) | follows the parent repo |
| LangBot | **v4.11.1** | `start-langbot-docker.cmd`; pin the image tag for production |
| WeChatPad-Docker | commit `c6d1b7a` | `tools/prepare_wechatpad.ps1 -Ref` (pinned by default) |
| Python | 3.12.x | `.venv` |

After upgrading anything, re-run the checklist in [maintenance.md](docs/maintenance.md) §3.

## Known limitations

- **DeepSeek has no fine-tuning API** — `finetune` only exports JSONL for it; use a
  platform with `/fine_tuning/jobs`, or WeClone's local LoRA (needs a GPU).
- `persona` imitates style with prompts; it does **not** change model weights.
- Text only; samples containing images are dropped on export.
- The personal-WeChat protocol adapter carries a ban risk — spare account, low volume, no mass sending.
- No built-in rate limiting; put it behind a gateway if you expose it.

## Disclaimer

- For personal learning and self-hosting. Get consent before using anyone's chat logs.
- Automating a personal WeChat account may violate platform terms; you accept the risk.
- Keep AI-generated content clearly attributed; do not impersonate people to deceive.

## License

AGPL-3.0, aligned with upstream WeClone.

---

<div align="center">

[**← 中文**](README.md) · [**↑ Back to top**](#weclone-cloud)

</div>
