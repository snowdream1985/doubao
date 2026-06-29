---
name: doubao
description: 通过本地 doubao CLI 调用火山方舟（Volcengine Ark）豆包模型，生成图片（Seedream 系列）和视频（Seedance 系列）。当用户要生成/绘制/渲染图片、改图、多图融合、组图/连环图/四格漫画，或要生成/制作视频、让图片动起来、首尾帧、视频延长/编辑、多模态参考生成视频，或管理视频任务（查询/列表/下载/取消），并提到 doubao / 豆包 / Seedream / Seedance 时使用本 skill。生成内容默认面向手机/移动设备（竖屏 9:16）。图片为同步，视频为异步任务（每个 1-6 分钟）。
allowed-tools: Bash, Read, Write
---

# 豆包图片 / 视频生成

通过本地 `doubao` CLI 调用火山方舟豆包视觉模型。本 skill 同时覆盖**图片生成**（Seedream 系列）与**视频生成 + 任务管理**（Seedance 系列）。

## 何时使用

图片相关：
- 生成 / 画一张图 / 渲染 / 插画
- 图生图 / 把照片改成… / 参考人脸...
- 多图融合 / 把几张图合在一起
- 组图 / 连环图 / 四格漫画 / 故事板

视频相关：
- 生成视频 / 做一个视频
- 让这张图动起来 / 图生视频
- 首帧 + 尾帧过渡
- 视频延长 / 编辑 / 接续
- 多模态参考（图片 + 参考视频 + 参考音频，仅 Seedance 2.0 系列） / 用这张图 + 这段视频生成
- 查询 / 取消 / 下载之前的视频任务

**默认面向手机/移动设备**：图片默认 2K 竖屏 `1440x2560`，视频默认 `720p` + `9:16`。

## 前置条件

1. 需要可用的 `doubao` 可执行文件。优先用 PATH 上的 `doubao`；找不到时回退到本 skill 自带的二进制（路径相对于 **skill 根目录**，即本 `SKILL.md` 所在目录）：
   - Windows x64：`bin/win-x64/doubao.exe`
   - Linux x64：`bin/linux-x64/doubao`
   - macOS Apple Silicon：`bin/osx-arm64/doubao`
   - macOS Intel：`bin/osx-x64/doubao`
2. **每次 Bash 调用开头都要重新解析可执行文件**，不要假设 shell 变量跨工具调用保留。
3. 必须配置 Ark API Key。用 `"$DOUBAO" config show` 验证；若显示 `(未设置)`，运行 `"$DOUBAO" config set api-key <KEY>`（Key 来自 https://console.volcengine.com/ark/region:ark+cn-beijing/apiKey）。
4. **模型 ID 必须带日期后缀**（如 `doubao-seedream-5-0-260128`、`doubao-seedance-2-0-260128`，而非 `doubao-seedream-5-0`）。不确定就问用户，**不要凭空猜日期后缀**。

## 模型选择优先级

用户未指定模型时，按以下顺序依次尝试；若 API 返回模型不存在（退出码 2，`InvalidEndpointOrModel.NotFound`）则降级到下一个：

**图片**（优先 → 备选）：
1. `doubao-seedream-5-0-260128`
2. `doubao-seedream-4-5-251128`
3. `doubao-seedream-4-0-250828`

**视频**（优先 → 备选）：
1. `doubao-seedance-2-0-260128`
2. `doubao-seedance-2-0-fast-260128`
3. `doubao-seedance-2-0-mini-260615`
4. `doubao-seedance-1-5-pro-251215`

## 解析 CLI 可执行文件

每个 Bash 命令块开头都先跑这段解析器，再调用 `doubao`：

```bash
# SKILL_DIR：本 skill 的根目录（bin/ 所在目录）
# 用 Read 工具查出 SKILL.md 的绝对路径，去掉文件名得到目录，然后填入下方。
# 示例（Linux/macOS）：SKILL_DIR="/home/user/.codebuddy/skills/doubao"
# 示例（Windows Git Bash）：SKILL_DIR="/c/Users/user/.codebuddy/skills/doubao"
SKILL_DIR="<skill根目录绝对路径>"   # ← 调用前替换为实际路径

if command -v doubao >/dev/null 2>&1; then
  DOUBAO="doubao"
elif [ -x "$SKILL_DIR/bin/linux-x64/doubao" ]; then
  DOUBAO="$SKILL_DIR/bin/linux-x64/doubao"
elif [ -x "$SKILL_DIR/bin/osx-arm64/doubao" ]; then
  DOUBAO="$SKILL_DIR/bin/osx-arm64/doubao"
elif [ -x "$SKILL_DIR/bin/osx-x64/doubao" ]; then
  DOUBAO="$SKILL_DIR/bin/osx-x64/doubao"
elif [ -x "$SKILL_DIR/bin/win-x64/doubao.exe" ]; then
  DOUBAO="$SKILL_DIR/bin/win-x64/doubao.exe"
else
  echo "找不到 doubao CLI。请放到 PATH 上，或置于 <skill根目录>/bin/<rid>/ 下" >&2
  exit 127
fi
```

> **关于 `SKILL_DIR`**：每次 Bash 调用前，先用 `Read` 工具确认本 `SKILL.md` 的绝对路径，取其所在目录即为 `SKILL_DIR`。Windows 下 Git Bash 路径须转换为 `/c/...` 形式；若运行环境是原生 PowerShell，则需先确认 bash 可用，或直接把 doubao 放到 PATH 里。

之后统一用 `"$DOUBAO" image gen ...` / `"$DOUBAO" video gen ...` / `"$DOUBAO" task ...`，不要硬编码 `doubao`。

**所有从本 skill 发起的调用都加 `--json`**：stdout 是干净的机器可读 JSON，进度/调试走 stderr（可读但别当 JSON 解析）。

---

# 一、图片生成

## 核心流程

1. 用上面的解析器拿到 `DOUBAO`，再用一次 `"$DOUBAO" config show` 验证前置条件。
2. 从用户意图判断场景：文生图 / 图生图 / 多图融合 / 组图。
3. 选模型：新需求默认用 **Seedream 5.0**（生成内容主要在手机上看）；仅当用户明确要求或账户未开通 5.0 时才用 Seedream 4.5。
4. 默认输出形状：2K 竖屏。用显式像素 `1440x2560` 同时表达 2K 画质 + 手机竖屏比例。
5. 用 Bash 运行。退出码 0 = 成功，stdout 的 JSON 里 `.files[]` 是生成图片的绝对路径。
6. 把文件路径（数量、可选大小）回报给用户。

## 调用模式

### 文生图（最常见）

```bash
"$DOUBAO" image gen \
  --model <seedream-5.0-model-id> \
  --prompt "<用户提示词>" \
  --size 1440x2560 \
  --out <输出路径> \
  --json
```

- `--size`：默认 `1440x2560`（2K 级 9:16 竖屏）。也支持语义值 `2K` / `3K`（仅 5.0）/ `4K`，或显式像素 `2048x2048`。Seedream 4.5 只支持 `2K` / `4K`（无 `3K`）。
- `--out`：强烈建议用绝对路径。不给时落到当前目录 `{model}-{timestamp}.jpg`。

### 图生图 / 多图融合

重复传 `--image <路径或URL>`。本地文件会自动上传到对象存储（OSS）再以 URL 传入，HTTP URL 直接透传。

```bash
"$DOUBAO" image gen \
  --model <seedream-5.0-model-id> \
  --prompt "把背景换成樱花林，保持手机竖屏构图" \
  --image /abs/path/source.jpg \
  --size 1440x2560 \
  --out /abs/path/result.jpg \
  --json
```

最多 14 张参考图。本地文件自动上传 OSS，单图 ≤ 30MB（不再受请求体合计大小限制）。

### 组图（多图连续）

```bash
"$DOUBAO" image gen \
  --model <seedream-5.0-model-id> \
  --prompt "四格漫画：..." \
  --sequential auto \
  --max-images 4 \
  --size 1440x2560 \
  --out /abs/path/panels-dir \
  --json
```

`--sequential auto` 时模型自行决定张数（最多 `--max-images`），可能少于上限——这是正常行为。返回多张时 `--out` 当**目录**处理（自动生成 `image-00.jpg` ...）；返回 1 张时当文件名处理。

## 图片参数

| 选项 | 取值 | 说明 |
|---|---|---|
| `--model` | 带日期后缀的完整 ID | 必填。默认家族 Seedream 5.0，如 `doubao-seedream-5-0-*`；不确定就问用户 |
| `--prompt` | 字符串 | 必填（除非 `--prompt-stdin`） |
| `--prompt-stdin` | flag | 从 stdin 读提示词（提示词 > 30KB 时用，规避 Windows cmd 32KB 上限） |
| `--image` | 路径或 URL，可重复 | 参考图 1–14 张 |
| `--size` | `1440x2560` / `2K` / `3K` / `4K` / `WxH` | 默认 `1440x2560`；家族相关，详见 `references/model-ids.md` |
| `--sequential` | `disabled` / `auto` | `auto` 启用组图 |
| `--max-images` | 1–15 | 输入+输出合计 ≤ 15 |
| `--web-search` | flag | 仅 Seedream 5.0 |
| `--watermark` | `true`/`false` | 默认 false（传 `--watermark` 开启水印） |
| `--output-format` | `jpeg`/`png` | 仅 Seedream 5.0 |
| `--response-format` | `url`/`b64_json` | 默认 `url`；两种都会本地下载 |
| `--out` | 路径或目录 | 默认 `./{model}-{timestamp}.jpg` |
| `--json` | flag | agent 调用务必带 |
| `--quiet` | flag | stdout 仅绝对路径；与 `--json` 互斥 |

全量参数与逐模型兼容矩阵见 `references/model-ids.md`。

## 图片 JSON 输出（带 `--json`）

成功：
```json
{
  "model": "doubao-seedream-5-0-260128",
  "created": 1782295362,
  "files": ["/abs/path/result.jpg"],
  "items": [{"size": "1440x2560", "url": "https://ark-..."}],
  "usage": {"generated_images": 1, "output_tokens": 16384, "total_tokens": 16384}
}
```

---

# 二、视频生成 + 任务管理

视频任务是**异步**的，通常 1-6 分钟出片（2.0-mini @ 480p×4s 约 1.5 分钟，2.0 full @ 1080p×10s 可达 6 分钟+）。

## 两种工作流

### A. 同步（`--wait`，默认）——阻塞直到完成并下载

适用：用户在线等待；任务短（720p，≤5s）；可以让 Bash 调用挂几分钟。

```bash
"$DOUBAO" video gen \
  --model <seedance-2.0-model-id> \
  --prompt "<提示词>" \
  --resolution 720p \
  --ratio 9:16 \
  --duration 4 \
  --out /abs/path/result.mp4 \
  --json
```

任务完成后 Bash 返回，stdout 是最终 JSON。stderr 每 30s 打一行心跳（`[120s] status=running`）。

### B. 异步（`--no-wait` + `task get` / `task download`）——长任务首选

适用：任务长（1080p、4k、10s+），不想阻塞对话。

```bash
# 1. 提交，拿 task_id
"$DOUBAO" video gen \
  --model <seedance-2.0-model-id> \
  --prompt "<提示词>" \
  --resolution 720p \
  --ratio 9:16 \
  --duration 4 \
  --no-wait --json
# stdout: {"task_id":"cgt-...","model":"...","status":"queued"}

# 2. 稍后轮询
"$DOUBAO" task get cgt-... --json

# 3. 成功后下载
"$DOUBAO" task download cgt-... --out /abs/path/result.mp4 --json
```

任务用 `--return-last-frame` 创建时，`task download` 可加 `--last-frame` 取尾帧。

## 核心场景

场景由 flag 组合自动推断，常见情况无需手动传 `--image-role`。**默认 Seedance 2.0 full + 720p + 9:16**。

### 文生视频

```bash
"$DOUBAO" video gen --model <seedance-2.0-model-id> --prompt "..." --resolution 720p --ratio 9:16 --duration 4 --out <path> --json
```

### 图生视频（自动推断首帧）

```bash
"$DOUBAO" video gen --model <seedance-2.0-model-id> --prompt "..." --image <路径或URL> \
  --resolution 720p --ratio 9:16 --duration 4 --out <path> --json
```

CLI 自动推断 `role=first_frame`。

### 首尾帧

传**两张**图：

```bash
"$DOUBAO" video gen --model <seedance-2.0-model-id> --prompt "..." \
  --image <first.jpg> --image <last.jpg> \
  --resolution 720p --ratio 9:16 --duration 4 --out <path> --json
```

按顺序推断 `role=first_frame, last_frame`。除 `doubao-seedance-1-0-pro-fast` 外都支持。

### 多模态参考（仅 Seedance 2.0 系列）

混用 `--image`（1–9）、`--video`（≤3，总时长 ≤15s）、`--audio`（≤3，≤15s）——至少要有一张图或一段视频（不能只传音频）。

```bash
"$DOUBAO" video gen --model <seedance-2.0-model-id> \
  --prompt "动作同参考视频，主角换成参考图里的狐狸" \
  --image fox.jpg \
  --video https://example.com/dance.mp4 \
  --resolution 720p --ratio 9:16 --duration 4 --out <path> --json
```

混用图片与 `--video`/`--audio` 时，参考图必须显式 `--image-role reference_image`（否则 CLI 默认按 `first_frame`，Ark 会拒）。本地视频/音频会自动上传 OSS 后以 URL 传入（单视频 ≤ 200MB、单音频 ≤ 15MB）。

### 编辑 / 延长已有视频

同多模态参考——把已有视频作 `--video` 传入，在 `--prompt` 里描述修改/续写。Seedance 2.0 只信任本账号近 30 天内自己生成的视频里的人脸内容（或素材库 `asset://...` ID）。

### 任务管理

```bash
"$DOUBAO" task list --json                              # 近 7 天任务
"$DOUBAO" task list --status succeeded --page-size 10 --json
"$DOUBAO" task get cgt-... --json                       # 单任务状态
"$DOUBAO" task download cgt-... --out v.mp4 --json      # 成功后重新下载
"$DOUBAO" task cancel cgt-...                           # 取消排队中 / 删除已结束
```

`task list` 只返回近 7 天任务。生成视频 URL 在成功后 24h 失效——`task download` 仅在 URL 有效期内可用，过期文件即丢。

## 分辨率 × 时长 × 模型兼容

传错会返回退出码 1 + hint。完整矩阵见 `references/model-matrix.md`。**移动端默认**：Seedance 2.0 full + `720p` + `9:16`。

| 模型家族 | 最大分辨率 | 时长 | 参考媒体 | seed | camera_fixed |
|---|---|---|---|---|---|
| `doubao-seedance-2-0-*`（full） | 4k | 4–15s 或 -1 | ✅ 图+视频+音频 | ❌ | ❌ |
| `doubao-seedance-2-0-mini-*` | 720p | 4–15s 或 -1 | ✅ | ❌ | ❌ |
| `doubao-seedance-2-0-fast-*` | 720p | 4–15s 或 -1 | ✅ | ❌ | ❌ |
| `doubao-seedance-1-5-pro-*` | 1080p | 4–12s 或 -1 | ❌ | ✅ | ✅ |
| `doubao-seedance-1-0-pro-*` | 1080p | 2–12s | ❌ | ✅ | ✅ |
| `doubao-seedance-1-0-pro-fast-*` | 1080p | 2–12s | ❌（仅首帧） | ✅ | ✅ |

## 视频参数

| 选项 | 说明 |
|---|---|
| `--model` | 必填，带日期后缀。默认家族 Seedance 2.0 full，如 `doubao-seedance-2-0-*`；不确定就问用户 |
| `--prompt` | 纯文生视频时必填；有参考媒体（图/视频）时可选 |
| `--resolution` | `480p` / `720p` / `1080p` / `4k`；移动端默认 `720p` |
| `--ratio` | `16:9` / `4:3` / `1:1` / `3:4` / `9:16` / `21:9` / `adaptive`；移动端默认 `9:16` |
| `--duration` | 整数秒；传 `-1` 让模型自定（仅 2.0 / 1.5-pro） |
| `--image` | 可重复。本地路径自动上传 OSS，URL 透传 |
| `--image-role` | 覆盖自动推断：`first_frame` / `last_frame` / `reference_image` |
| `--video` | 参考视频，仅 Seedance 2.0 系列。本地文件自动上传 OSS（≤ 200MB），URL 透传 |
| `--audio` | 参考音频，仅 Seedance 2.0 系列 |
| `--generate-audio` | 2.0/1.5-pro 默认 true。传 `false` 出无声视频 |
| `--return-last-frame` | 额外返回尾帧 PNG，便于连续生成 |
| `--web-search` | 仅 2.0 系列 |
| `--priority` | 0–9，仅 2.0 系列 |
| `--service-tier` | `default`（在线实时，全价）/ `flex`（离线半价，**Seedance 2.0 系列不支持 flex**） |
| `--frames` | 整数，仅 **1.0-pro / 1.0-pro-fast**；范围 `[29, 289]`，须满足 `(frames-25) % 4 == 0`；与 `--duration` 二选一 |
| `--seed` | 非 2.0 系列 |
| `--camera-fixed` | 非 2.0 系列 |
| `--watermark` | 视频默认 false |
| `--wait` / `--no-wait` | 默认 `--wait`。长任务用 `--no-wait` |
| `--poll-interval` | 秒，默认 5（指数退避到 30） |
| `--timeout` | 秒，默认 1800（30min）。4k 调大 |
| `--out` | 默认 `./{model}-{timestamp}.mp4` |
| `--json` | agent 调用务必带 |

逐模型兼容、音频行为、优先级/service-tier 规则见 `references/model-matrix.md`。

## 视频 JSON 输出（`--wait` 成功）

```json
{
  "task_id": "cgt-...",
  "model": "doubao-seedance-2-0-260128",
  "status": "succeeded",
  "video_path": "/abs/path/result.mp4",
  "last_frame_path": "/abs/path/result.last-frame.png",
  "video_url": "https://ark-...",
  "resolution": "720p",
  "ratio": "9:16",
  "duration": 4,
  "framespersecond": 24,
  "generate_audio": true,
  "usage": {"completion_tokens": 40000, "total_tokens": 40000}
}
```

`--no-wait` 成功：
```json
{"task_id": "cgt-...", "model": "...", "status": "queued"}
```

---

# 三、通用：退出码与错误处理

错误 JSON：
```json
{"error": {"code": "invalid_argument", "message": "...", "hint": "..."}}
```

| 退出码 | 含义 | 处理 |
|---|---|---|
| 0 | 成功 | 图片读 `.files[]`，视频读 `.video_path` |
| 1 | 参数错误 / 本地校验失败 | 读 `.error.hint`，通常是模型 × 参数不匹配（如 Seedream 4.5 + 3K、2.0-fast + 1080p）。修正后重试 |
| 2 | Ark API 业务错误 | 读 `.error.code`/`.message`。`InvalidEndpointOrModel.NotFound` = 模型 ID 写错，问用户要带日期后缀的正确 ID。审核失败 = 敏感内容，请用户改提示词 |
| 3 | 网络 / 超时 | 等 10s 重试一次；视频可用 `task get` 看服务端真实状态 |
| 4 | 文件系统错误 | 输出路径不可写，问用户要可写目录 |
| 5 | API Key 缺失或无效 | `"$DOUBAO" config show` 确认；未设置则 `config set api-key <KEY>`。绝不回显 Key |

## 错误处理规则

- **退出码 2 不要盲目重试**——hint 通常已说明问题，修一次再升级给用户。
- **模型 404**（`InvalidEndpointOrModel.NotFound`）：若使用的是默认模型列表中的 ID，自动降级到下一个；若已降级至最后一个仍失败，或用户明确指定了某 ID，则告知用户该 ID 未开通，让其从控制台确认已开通的完整 ID。
- **敏感内容被拒**：告知用户、请其改写提示词，未经同意不要自动改写。
- **Seedance 2.0 真人脸审核失败**：2.0 系列拒绝含真实人脸的参考素材，除非来自本账号近 30 天 Seedance 2.0 生成内容或素材库 `asset://<ID>`。告知用户，不要重试。
- **OSS 上传失败**（退出码 3，`oss_upload_error`）：本地素材上传对象存储失败（网络/凭证/对象存储异常）。可重试一次；持续失败则改用公网 URL 传入。本地单文件超限（图 30MB / 视频 200MB / 音频 15MB）会在上传前被拒，提示用户压缩或改用 URL。
- **网络重试**：CLI 已对 5xx/瞬时错误内部重试，除非退出码 3 否则别在外层包重试。

## 输出约定

- 图片：始终告诉用户每个生成文件的绝对路径；组图列全部。
- 视频：报 `.video_path`；非默认时提一下 `.duration`/`.resolution`。
- 不要把 URL 当成果物——生成视频的 URL 是 24h 签名 URL，本地下载文件才是真正的产物。

# 四、Windows 平台使用示例

以下示例在 Windows（cmd.exe / PowerShell）下直接调用 `doubao`。前提：`doubao.exe` 在 PATH 上，或用本 skill 自带的 `bin/win-x64/doubao.exe`。`--model` 的日期后缀按实际开通的 ID 替换。

## 配置与信息

```
:: 配置 API Key
doubao config set api-key e8e32722-b5cf-47a2-8145-xxxxxxxxxxxx

:: 查看 API Key（脱敏）
doubao config show

:: 查看版本信息
doubao --version
```

## 图片生成

```
:: 文生图
doubao image gen --model doubao-seedream-5-0-260128 --prompt "身穿古风传统汉服的年轻女生高清写真"

:: 图生图 / 编辑图片
doubao image gen --model doubao-seedream-5-0-260128 --prompt "将服装改为红色" --image portrait.jpg

:: 指定图像大小：2K / 3K / 4K
doubao image gen --model doubao-seedream-5-0-260128 --prompt "身穿古风传统汉服的年轻女生高清写真" --size 2K

:: 指定图像大小：具体分辨率
doubao image gen --model doubao-seedream-5-0-260128 --prompt "身穿古风传统汉服的年轻女生高清写真" --size 1440x2560

:: 组图（自动判断张数）
doubao image gen --model doubao-seedream-5-0-260128 --prompt "制作乌鸦喝水的连环故事图片，生成4张图片" --sequential auto

:: 组图，限制最多生成数量
doubao image gen --model doubao-seedream-5-0-260128 --prompt "制作乌鸦喝水故事的连环故事图片" --sequential auto --max-images 6

:: 联网搜索
doubao image gen --model doubao-seedream-5-0-260128 --prompt "制作一张四川省乐山市未来5日的天气预报图" --web-search

:: 水印（默认不加，传 true 开启）
doubao image gen --model doubao-seedream-5-0-260128 --prompt "四川省乐山市未来一周天气新闻" --watermark true

:: 输出格式
doubao image gen --model doubao-seedream-5-0-260128 --prompt "最近一周科技行业热点新闻" --output-format png

:: 图像返回格式
doubao image gen --model doubao-seedream-5-0-260128 --prompt "最近一周科技行业热点新闻" --response-format url

:: 多参数随意组合
doubao image gen --model doubao-seedream-5-0-260128 --prompt "制作4张北京市最新天气预报图片，每日一张，再生成一张汇总图，汇总图片参考weather.jpg图片样式" --image weather.jpg --size 1440x2560 --sequential auto --max-images 6 --web-search --output-format png --response-format url --watermark true
```

## 视频生成

```
:: 文生视频
doubao video gen --model doubao-seedance-2-0-260128 --prompt "海浪拍打礁石"

:: 图生视频（首帧）
doubao video gen --model doubao-seedance-2-0-260128 --prompt "让她微笑" --image women.jpg

:: 图生视频（首尾帧）
doubao video gen --model doubao-seedance-2-0-260128 --prompt "乌鸦将石子放入瓶中，终于喝到水了" --image first.jpg --image last.jpg

:: 多模态参考：三张参考图（≥3 张图自动按 reference_image 处理）
doubao video gen --model doubao-seedance-2-0-260128 --prompt "乌鸦将石子放入瓶中，终于喝到水了" --image image-00.jpg --image image-01.jpg --image image-02.jpg

:: 多模态参考：视频参考
doubao video gen --model doubao-seedance-2-0-260128 --prompt "将乌鸦换成鹦鹉" --video v.mp4

:: 指定分辨率（默认 720p）
doubao video gen --model doubao-seedance-2-0-260128 --prompt "年轻女孩在跳舞，身穿汉服" --resolution 1080p

:: 指定比例（默认 9:16）
doubao video gen --model doubao-seedance-2-0-260128 --prompt "年轻女孩在跳舞，身穿汉服" --ratio 16:9

:: 指定时长（默认 5 秒）
doubao video gen --model doubao-seedance-2-0-260128 --prompt "年轻女孩在跳舞，身穿汉服" --duration 10

:: 不生成音频（默认生成）
doubao video gen --model doubao-seedance-2-0-260128 --prompt "年轻女孩在跳舞，身穿汉服" --generate-audio false

:: 添加水印（默认不加）
doubao video gen --model doubao-seedance-2-0-260128 --prompt "年轻女孩在跳舞，身穿汉服" --watermark true

:: 联网搜索
doubao video gen --model doubao-seedance-2-0-260128 --prompt "制作四川省乐山市未来5日的天气预报视频" --web-search

:: 返回视频尾帧图像
doubao video gen --model doubao-seedance-2-0-260128 --prompt "年轻女孩在跳舞，身穿汉服" --return-last-frame

:: 多参数随意组合
doubao video gen --model doubao-seedance-2-0-260128 --prompt "将乌鸦换成鹦鹉" --video v.mp4 --resolution 1080p --duration 10 --generate-audio true --watermark true --return-last-frame
```

## 视频异步工作流（长任务首选，不阻塞）

视频默认会阻塞等待并下载。长任务（1080p/4k、10s+）可用 `--no-wait` 立即拿 `task_id`，之后再查询/下载。

```
:: 1. 提交任务，立即返回 task_id（不等待、不下载）
doubao video gen --model doubao-seedance-2-0-260128 --prompt "海浪拍打礁石" --resolution 1080p --duration 10 --no-wait

:: 2. 查询单个任务状态（succeeded / running / failed ...）
doubao task get cgt-xxxxxxxx

:: 3. 列出近 7 天任务（可按状态/模型/分页过滤）
doubao task list
doubao task list --status succeeded --page-size 20

:: 4. 成功后下载视频（指定输出路径）
doubao task download cgt-xxxxxxxx --out result.mp4

:: 4b. 若任务创建时带了 --return-last-frame，可一并下载尾帧
doubao task download cgt-xxxxxxxx --out result.mp4 --last-frame

:: 5. 取消排队中或删除已结束的任务
doubao task cancel cgt-xxxxxxxx
```

> 生成视频的在线 URL 24 小时失效，务必用 `task download` 落地到本地；任务记录仅保留 7 天。

## 中文 prompt 与引号注意事项（重要）

不同 shell 对引号处理不同，**含中文/空格的 prompt 一定要正确加引号**：

- **cmd.exe**：用**双引号** `"..."` 包 prompt。**不要用单引号** `'...'`——cmd 不识别单引号，会把引号当成内容的一部分。
  ```
  doubao image gen --model doubao-seedream-5-0-260128 --prompt "身穿汉服的女孩"
  ```
- **PowerShell**：单双引号都可（`'...'` 为字面、`"..."` 支持变量展开）。中文无需转义。
- **Git Bash**：单双引号都可，按 POSIX 规则。
- prompt 特别长（超过 cmd 约 32KB 行长上限）时，改用 `--prompt-stdin` 从标准输入读入，规避命令行长度限制。

# 五、Linux / macOS 使用注意事项

Linux/macOS 下用 bash/zsh 直接调用 `doubao`（在 PATH 上）或本 skill 自带的二进制。命令写法与上面的 `bash` 示例一致，把 `doubao` 换成对应可执行文件即可。

## 选对架构的二进制

- **Linux x64** → `bin/linux-x64/doubao`
- **macOS Apple Silicon（M 系列）** → `bin/osx-arm64/doubao`
- **macOS Intel** → `bin/osx-x64/doubao`
- 用 `uname -m` 判断：`arm64` → osx-arm64，`x86_64` → osx-x64 / linux-x64。

## 执行权限（最常见的坑）

从 zip/git 解出的自带二进制可能丢失执行位，运行报 `Permission denied`：

```bash
chmod +x "$SKILL_DIR/bin/linux-x64/doubao"     # 或 osx-arm64 / osx-x64
```

## macOS 安全拦截（Gatekeeper / quarantine）

下载来的二进制首次运行可能被拦（"无法验证开发者""已损坏"）。去掉隔离属性即可：

```bash
xattr -d com.apple.quarantine "$SKILL_DIR/bin/osx-arm64/doubao" 2>/dev/null || true
```

或在「系统设置 → 隐私与安全性」里点「仍要打开」。

## 引号与路径

- bash/zsh 单双引号都可，中文 prompt 直接用双引号即可，无需转义：
  ```bash
  doubao image gen --model doubao-seedream-5-0-260128 --prompt "身穿汉服的女孩"
  ```
- 路径含空格务必加引号；推荐用绝对路径（默认输出落当前工作目录）。
- macOS 默认 shell 是 zsh，Linux 多为 bash，本 skill 的解析器脚本两者通用。

## 配置与 API Key

- 配置文件位置：`$XDG_CONFIG_HOME/doubao/config.json`，未设则 `~/.config/doubao/config.json`（写入时权限自动设为 `0600`，仅本人可读写）。
- 除 `doubao config set api-key` 外，也可用环境变量（优先级：`--api-key` > 配置文件 > `ARK_API_KEY`）：
  ```bash
  export ARK_API_KEY=e8e32722-b5cf-47a2-8145-xxxxxxxxxxxx
  ```

## 其它

- 多行命令用反斜杠 `\` 续行（如上文 bash 示例）。
- 退出码语义全平台一致：`0` 成功 / `1` 参数错 / `2` API 业务错 / `3` 网络或超时 / `4` 文件系统 / `5` 鉴权。



