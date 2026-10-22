<div align="center">

# 🧬 weclone-cloud

**把聊天记录变成会说话的你** · *Turn your chat logs into an AI twin*

不用显卡 · 纯 API · 能接微信 · 有完整运维手册

[![License: AGPL-3.0](https://img.shields.io/badge/license-AGPL--3.0-3da639.svg)](https://www.gnu.org/licenses/agpl-3.0)
[![Python](https://img.shields.io/badge/python-3.12-3776AB.svg?logo=python&logoColor=white)](https://www.python.org/)
[![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20Linux-0078D4.svg?logo=windows&logoColor=white)](#)
[![GPU](https://img.shields.io/badge/GPU-not%20required-brightgreen.svg)](#)
[![WeChat](https://img.shields.io/badge/WeChat-LangBot%20%2B%20WeChatPad-07C160.svg)](#)
[![Based on WeClone](https://img.shields.io/badge/based%20on-WeClone-8A2BE2.svg)](https://github.com/xming521/WeClone)

**中文** · [**English** →](README.en.md)

</div>

<!-- 有截图后取消注释，并把图片放到 docs/images/demo.png
<p align="center"><img src="docs/images/demo.png" width="780" alt="微信里的数字分身"></p>
-->

> [!TIP]
> **这是什么？** 从你的微信 / Telegram 聊天记录里学说话方式，然后以你的口吻在微信或网页里回消息。
> 全流程走 API（默认 DeepSeek），**不需要 NVIDIA 显卡**。

> **在 WeClone 仓库内使用**：父目录就是 WeClone，直接 `uv sync`，
> 不需要运行 `setup.cmd`（那会再克隆一份），下文路径里的 `app\` 换成 `..\`。

---

## ✨ 为什么用它

| 传统方案 | weclone-cloud |
|---|---|
| 本地 vLLM 打分，只支持 Linux + NVIDIA | 打分走 API，Windows 笔记本也能跑完 |
| 必须微调 7B 模型，显存 24GB 起 | 人设提示词先跑起来，想训练再导出 JSONL |
| 推理要本地加载权重 | 本机只跑一个轻量代理，模型在云端 |
| 只管训练，不管怎么用 | 附 LangBot + 微信协议端方案，含封号规避 |

## 🎯 它能做什么

| 能力 | 说明 |
|---|---|
| 🗂️ **数据加工** | 复用 WeClone 的 `make-dataset`：时间窗合并、问答配对、PII 脱敏 |
| ⭐ **智能打分** | 1–5 分评估对话质量，并发、断点续跑、结果缓存，先 `--dry-run` 看花费 |
| 🎭 **人设生成** | 从高分样本提炼「说话风格规则 + 真实对话示例」，无需训练 |
| 🔌 **OpenAI 兼容** | 一个 `/v1/chat/completions`，接 LangBot / AstrBot / 任何 SDK |
| 💬 **微信落地** | Docker 跑 LangBot + WeChatPadPro，含适配器补丁与拟人延迟 |
| 🧾 **可复现** | 导出带 `manifest.json`：条数、丢弃原因、token、文件 sha256 |

## 🚀 三步开始

```powershell
git clone https://github.com/lhongjie70-sudo/weclone-cloud.git
cd weclone-cloud

.\setup.cmd            # 拉取 WeClone、建 .venv、装依赖
                       # 网络不好：setup.cmd --pypi  或  setup.cmd --aliyun

notepad secrets.local.cmd   # 填 WECLONE_API_KEY=sk-...

.\start-chat.cmd       # 自动打开 http://127.0.0.1:8010/
```

接微信（需要 Docker Desktop）：

```powershell
.\start-wechat.cmd     # 分身 + 微信协议端 + LangBot 一起拉起
```

详细步骤见 **[安装教程](docs/install.md)**，微信扫码见 **[运维手册](docs/runbook.md)** 第三节。

## 🧭 数据怎么走

```
微信 / Telegram 导出
        │  weclone-cli make-dataset      （WeClone：合并、配对、脱敏）
        ▼
dataset/res_csv/sft/sft-my.json
        │  weclone-cloud score           （API 打分，可续跑）
        ▼
sft-my.scored.json
        │  weclone-cloud persona         （风格规则 + 示例）
        ▼
persona_system.md
        │  weclone-cloud serve           （鉴权代理：人设 / 关思考 / 延迟）
        ▼
http://127.0.0.1:8010/v1  ──►  LangBot  ──►  WeChatPad  ──►  微信
```

## 🛠️ 命令

| 命令 | 作用 | 常用参数 |
|---|---|---|
| `doctor` | 自检配置 / 密钥 / 连通性 | — |
| `score` | API 打分并写回 | `--dry-run --limit --workers --rescore` |
| `export` | 转 JSONL、去重、切验证集 | `--min-score --val-ratio --max-tokens` |
| `persona` | 生成 `persona_system.md` | `--examples --min-score` |
| `serve` | 启动分身代理 + 网页 | `--host --port --persona` |
| `finetune` / `status` | 云端微调任务（generic 只导出） | `--train --val --no-watch` |
| `pipeline` | score + export 串联 | `--input --out-dir` |

> 用 `python -m weclone_cloud.cli <命令>`，或安装后的 `weclone-cloud <命令>`。

## 📚 文档

| | |
|---|---|
| 📥 [install.md](docs/install.md) | 从零安装：环境、数据、分身、Docker、微信 |
| 🔧 [runbook.md](docs/runbook.md) | 日常启动、断电恢复、微信重登、故障速查 |
| 🧱 [architecture.md](docs/architecture.md) | 与 WeClone 的映射、设计取舍 |
| 🧩 [providers.md](docs/providers.md) | OpenAI / 百炼 / 本地模型配置 |
| 🛰️ [maintenance.md](docs/maintenance.md) | **长期维护**：版本矩阵、升级复验、发布流程 |

## ✅ 测试过的版本

| 组件 | 版本 | 固定方式 |
|---|---|---|
| WeClone | `master@d4af5ba` | 子目录跟随父仓库 |
| LangBot | **v4.11.1** | `start-langbot-docker.cmd` |
| WeChatPad-Docker | `c6d1b7a` | `prepare_wechatpad.ps1 -Ref`（已默认钉住） |
| Python | 3.12.x | `.venv` |

升级任何一项后，按 [maintenance.md](docs/maintenance.md) 第三节复验。

<details>
<summary><b>📁 目录结构</b>（点击展开）</summary>

```
weclone-cloud/
├─ weclone_cloud/           # 核心包（不修改 WeClone 源码）
│  ├─ config.py             # 配置、密钥来源、计价
│  ├─ api.py                # OpenAI 兼容客户端（重试/缓存/关思考/JSON 降级）
│  ├─ prompts.py            # 打分 rubric 与 schema
│  ├─ scoring.py            # 并发打分、断点续跑、报告
│  ├─ dataset.py            # JSONL 导出、去重、切分、manifest
│  ├─ finetune.py           # /files + /fine_tuning/jobs
│  ├─ serve.py              # 鉴权代理：人设注入、剥离 reasoning、拟人延迟
│  └─ web/index.html        # 自带聊天页
├─ tools/                   # 数据转换、启动器、补丁
├─ tests/                   # 无网络单测（httpx.MockTransport）
├─ docs/                    # 安装 / 运维 / 维护 / 架构 / 平台
├─ setup.cmd  start-chat.cmd  start-wechat.cmd
├─ bootstrap.ps1            # 复制 WeClone + 装依赖
└─ cloud.example.jsonc      # 配置模板
```

</details>

<details>
<summary><b>❓ 常见问题</b>（点击展开）</summary>

**Q：没有显卡真的能跑吗？**
能。打分、人设、对话全走 API，本地只跑 FastAPI 代理。唯一需要显卡的是"本机 LoRA 训练"，可选。

**Q：DeepSeek 能微调吗？**
不能，官方没有 `/fine_tuning/jobs`。本项目用 `provider=generic` 导出 JSONL，需要训练时交给支持微调的平台。

**Q：不训练，效果够用吗？**
能对齐句长、语气词、口头禅和标点习惯；稳定性不如微调。建议先用 `persona` 验证数据质量。

**Q：会封微信吗？**
个人号协议端有风险。用**小号**、低频、别群发；项目里已加 2–4 秒随机延迟、单条回复、去重。

**Q：和 WeClone 什么关系？**
本仓库是它的子项目，复用数据解析、替换后段链路，已作为 [PR #253](https://github.com/xming521/WeClone/pull/253) 合并进上游。

</details>

## ⚠️ 已知限制

- 只支持文本；带图片的样本在导出时丢弃
- `persona` 不改权重，纯提示词模仿
- 微信部分依赖 Docker，exe 无法内置容器
- 未内置限流，对外暴露请自行加网关

## 📜 声明

- 仅供个人学习与自用；使用他人聊天记录前**必须取得同意**
- 微信自动化可能违反平台协议，风险自负
- AI 生成内容应保留身份标识，不得用于冒充他人

## License

AGPL-3.0，与上游 [WeClone](https://github.com/xming521/WeClone) 一致。

---

<div align="center">

[**↑ 回到顶部**](#-weclone-cloud) · [**English** →](README.en.md)

</div>
