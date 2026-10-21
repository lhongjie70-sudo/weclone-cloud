# 安装教程（Windows 全流程，已验证）

本文档记录的是**实际跑通**的步骤，不是理论流程。每一步都给出了"应该看到什么"，对不上就停下来排查。

> 目标：微信小号收到消息 → LangBot → 本机分身（你的人设）→ DeepSeek → 回微信。

---

## 0. 前置条件

| 依赖 | 版本/说明 | 检查命令 |
|---|---|---|
| Windows | 10/11（管理员权限用于装 Docker） | — |
| Python | **3.12**（3.13+ 部分依赖不兼容） | `py -3.12 --version` |
| uv | 依赖管理 | `uv --version` |
| Git | 拉取代码 | `git --version` |
| Docker Desktop | 接微信才需要，且必须先装 WSL2 | `docker info` |
| DeepSeek 密钥 | <https://platform.deepseek.com/api_keys> | — |

没有 Python 3.12 的话：

```powershell
winget install -e --id Python.Python.3.12
winget install -e --id astral-sh.uv
```

---

## 1. 获取代码与依赖

```powershell
git clone https://github.com/<your-name>/weclone-cloud.git
cd weclone-cloud

# 轻量模式：只装数据加工 + 代理所需（约 300MB）
.\bootstrap.ps1 -Install
```

`bootstrap.ps1` 会：

1. 把 WeClone 源码复制到 `app\`；
2. 在 `.venv\` 建一个 Python 3.12 虚拟环境；
3. 安装 WeClone 基础依赖 + `presidio`（脱敏）+ `zh_core_web_sm`（中文分词）；
4. 打一个补丁：WeClone 里 spaCy 写死 `n_process=24`，Windows 上会 `MemoryError`，改成 1。

> 想在本机跑 LoRA 训练（需要 NVIDIA 显卡，约 10GB）才追加 `-Full`。

验证：

```powershell
.\.venv\Scripts\python.exe -c "import weclone_cloud, fastapi; print('ok')"
.\.venv\Scripts\python.exe -m weclone_cloud.cli doctor
```

---

## 2. 准备数据

### 2.1 微信导出 → CSV

支持的形态：WeClone CSV / Telegram / Alpha 导出 / 微信 CSV / SQLite。先让工具认一下：

```powershell
.\.venv\Scripts\python.exe tools\inspect_data.py "D:\你的导出目录"
```

如果是 **Alpha 导出**（每个联系人一个文件夹，内含 `<名字>.csv`，列有 `type_name/is_sender/msg/CreateTime`）：

```powershell
.\.venv\Scripts\python.exe tools\from_alpha.py `
  --src "C:\Users\Administrator\Desktop\wx\Alpha导出-20261020-123117" `
  --dst "app\dataset\csv"
```

它会丢掉非文本行（表情包是 XML）并打印保留/丢弃统计，产物放到 `app\dataset\csv\<联系人>\<联系人>.csv`。

### 2.2 生成 SFT 数据集

```powershell
cd app
..\.venv\Scripts\python.exe -m weclone_cloud.cli --help   # 确认 CLI 可用后返回根目录
```

WeClone 的 `make-dataset` 用它自己的环境执行：

```powershell
cd app
# 确认 settings.jsonc 里 clean_dataset.enable_clean = false（打分交给下一步）
weclone-cli make-dataset
```

成功输出类似：

```
Chat record processing successful, obtained 2925 data entries
```

产物：`app\dataset\res_csv\sft\sft-my.json`。

> `length_cdf` 报 `No module named 'fire'`、`LTP` 相关失败都可以忽略——它们只影响统计与分词增强，不影响数据。

---

## 3. 打分与人设

```powershell
cd ..   # 回到仓库根目录

# 先看规模与预估 token（不花钱）
.\.venv\Scripts\python.exe -m weclone_cloud.cli score `
  --input app\dataset\res_csv\sft\sft-my.json --dry-run

# 正式打分（可 Ctrl+C，重跑会续跑，结果有缓存）
.\.venv\Scripts\python.exe -m weclone_cloud.cli score `
  --input app\dataset\res_csv\sft\sft-my.json
```

权重示例（2925 条 ≈ 0.03 元）。完成后生成 `sft-my.scored.json`。

生成人设提示词：

```powershell
.\.venv\Scripts\python.exe -m weclone_cloud.cli persona `
  --input app\dataset\res_csv\sft\sft-my.scored.json --examples 6
```

产物 `persona_system.md`：包含 5–8 条可执行的风格规则 + 6 段真实对话示例。

> 建议自己打开看一眼。语气、称呼、口头禅不准的话，直接手改，效果立竿见影。

---

## 4. 启动分身

**接微信必须监听 `0.0.0.0`**（容器才能访问）；只用浏览器则 `127.0.0.1` 就够。

把密钥写进 `secrets.local.cmd`（已在 `.gitignore`，不会进仓库）：

```bat
set "WECLONE_API_KEY=sk-你的key"
set "WECLONE_SERVE_KEY=local-key"
```

然后双击 `start-persona-docker.cmd`，或命令行：

```powershell
$env:WECLONE_API_KEY   = "sk-..."
$env:WECLONE_SERVE_KEY = "local-key"
.\.venv\Scripts\python.exe tools\launch.py --no-langbot --host 0.0.0.0
```

期望横幅：

```
persona API : http://0.0.0.0:8010/v1
serve key : local-key
```

自检：

```powershell
Invoke-RestMethod http://127.0.0.1:8010/healthz
# {"ok":true,"model":"deepseek-flash"}
```

对话自测（注意 PowerShell 5.1 会显示乱码，用下面的解码写法）：

```powershell
$h = @{ Authorization = "Bearer local-key" }
$b = @{ model = "deepseek-flash"; messages = @(@{ role = "user"; content = "在吗" }); stream = $false } | ConvertTo-Json -Depth 5
$r = Invoke-WebRequest http://127.0.0.1:8010/v1/chat/completions -Method Post -Headers $h -ContentType "application/json" -Body $b -UseBasicParsing
([Text.Encoding]::UTF8.GetString($r.RawContentStream.ToArray()) | ConvertFrom-Json).choices[0].message.content
```

---

## 5. Docker + LangBot

### 5.1 先装 WSL2（Docker Desktop 的前提）

管理员 PowerShell：

```powershell
wsl --install --no-distribution
```

**重启电脑**，然后启动 Docker Desktop，等 `docker info` 成功（首次约 1–2 分钟，重启后更久）。

### 5.2 拉取并启动 LangBot

```powershell
.\start-langbot-docker.cmd
```

脚本会：

1. `git clone` 官方仓库到 `langbot-src\`（没有 git 就下 zip）；
2. 修正 Windows 挂载路径（官方 compose 的 `${PWD}` 带盘符冒号，Docker 会报 `too many colons`）；
3. `docker compose up -d`（core + plugin runtime，不含 Box 沙箱）。

首次要拉几个 GB 镜像。看到两个容器 `(healthy)` 即可：<http://localhost:5300>。

> 拉镜像报 `EOF` / `auth.docker.io` 超时时，用 `start-langbot-docker.cmd --mirror` 走国内镜像源。

### 5.3 LangBot 里的模型配置

「模型」→ 新建供应商（类型 **OpenAI**）：

| 字段 | 值 |
|---|---|
| 请求 URL | `http://host.docker.internal:8010/v1` |
| API Key | 分身窗口的 `serve key` |
| 模型名 | `deepseek-flash` |
| 工具使用 | **关** |

**不要用 `127.0.0.1`**——那是容器自己。

容器到分身的连通性验证：

```powershell
docker exec langbot python -c "import urllib.request;print(urllib.request.urlopen('http://host.docker.internal:8010/healthz',timeout=5).read().decode())"
```

不通就在管理员 PowerShell 放行容器网段：

```powershell
New-NetFirewallRule -DisplayName "weclone-docker" -Direction Inbound -Protocol TCP `
  -LocalPort 8010 -RemoteAddress 172.16.0.0/12,10.0.0.0/8 -Action Allow
```

---

## 6. 微信协议端

### 6.1 启动

```powershell
.\start-wechatpad.cmd
```

等价于 clone `fdc310/WeChatPad-Docker` 后 `docker compose up -d`（MySQL + Redis + WeChatPad）。

三个容器都 `Up`、`9090` 可访问即成功：

```powershell
docker ps --filter name=wechatpad --filter name=mysql_wxpad --filter name=redis_wxpad --format "{{.Names}}  {{.Status}}"
(Invoke-WebRequest http://127.0.0.1:9090 -UseBasicParsing).StatusCode
```

### 6.2 取管理员密钥

这个包**不打印** key，写死在 `wechatpad-src\app\.env`：

```
ADMIN_KEY=fdc010310
```

### 6.3 生成授权码并扫码

管理员密钥只能调 `/admin/*`；业务接口要"授权码"。

```powershell
$r = Invoke-RestMethod "http://127.0.0.1:9090/admin/GenAuthKey1?key=fdc010310" `
  -Method Post -ContentType "application/json" -Body (@{ Count = 1; Days = 365 } | ConvertTo-Json)
$token = $r.Data.authKeys[0]
"TOKEN = $token"

$q = Invoke-RestMethod "http://127.0.0.1:9090/login/GetLoginQrCodeNew?key=$token" `
  -Method Post -ContentType "application/json" -Body (@{ Proxy = ""; Check = $false } | ConvertTo-Json)
Start-Process $q.Data.QrCodeUrl
```

用**小号**扫码，确认：

```powershell
Invoke-RestMethod "http://127.0.0.1:9090/login/GetLoginStatus?key=$token" | ConvertTo-Json -Depth 5
```

`"loginState": 1` 即成功。

### 6.4 绑定到 LangBot

「机器人」→ 新建 → 平台选 **WeChatPad**：

| 字段 | 值 |
|---|---|
| WeChatPad_ERL | `http://host.docker.internal:9090` |
| WeChatPad_Ws | `ws://host.docker.internal:9090/ws` |
| Admin_Key | `fdc010310` |
| Token | 6.3 生成的授权码 |
| wxid | `auto`（运行时会自动覆盖） |

保存 → 开开关 → 日志出现 `WeChatPad WebSocket connected`。

> 顺序很重要：**先起协议端，再起 LangBot**。否则 LangBot 启动时连不上 9090，会报
> `Max retries exceeded ... /login/GetLoginStatus`，需要去 UI 关开一次开关。

---

## 7. 两处必须知道的本机补丁

LangBot 4.11.1 的 WeChatPad 适配器有两个缺陷，本项目在
`langbot-src/src/langbot/pkg/platform/sources/wechatpad.py` 里修掉，并通过
`langbot-src/docker/docker-compose.yaml` 的 bind mount 注入容器：

| 问题 | 现象 | 修法 |
|---|---|---|
| `event` 未赋值 | 消息解析失败时 `UnboundLocalError`，整条回调崩溃 | 进 try 前 `event = None` |
| 无消息去重 | 协议端重连重放 → 一条消息回 3~6 遍 | 按 `new_msg_id` 去重（保留最近 500 条） |

**改完必须重建容器**（`docker restart` 不读新挂载）：

```powershell
docker compose -f langbot-src\docker\docker-compose.yaml up -d --force-recreate
docker exec langbot grep -c "_is_duplicate" /app/src/langbot/pkg/platform/sources/wechatpad.py
```

最后一条输出 `2` 才算生效。

---

## 8. 跑通验收

1. 微信里只有**一条**回复，2–4 秒后到达；
2. 回复是**短句**、符合你的说话习惯，**没有** `<think>` 英文块；
3. 分身窗口出现 `POST /v1/chat/completions 200`。

三项都对 → 完成。日常运维、重启恢复、故障速查见 [runbook.md](runbook.md)。

---

## 9. 常见错误

| 报错 | 原因 | 处理 |
|---|---|---|
| `failed to connect to the docker API ... dockerDesktopLinuxEngine` | Docker Desktop 没启动 | 开始菜单启动，`docker info` 成功再继续 |
| `docker compose ... too many colons` | 官方 compose 的 `${PWD}` 在 Windows 带盘符 | 跑 `tools/fix_langbot_compose.py`（脚本会自动做） |
| `failed to resolve reference ... EOF` | Docker Hub 鉴权被墙 | `start-langbot-docker.cmd --mirror` |
| 分身 `500 Internal Server Error` | 进程里没有 DeepSeek key | 检查 `secrets.local.cmd`，重启分身 |
| 回复带 `<think>` | 没走本机代理 | 流水线模型选 weclone 供应商那个 `deepseek-flash` |
| `-3 该链接已绑定微信号` | 授权码已绑旧会话 | 重新 `GenAuthKey1` |
| `Agent runner temporarily unavailable` | 并发运行被拒 | 流水线并发设 1，关掉错误提示推送 |
