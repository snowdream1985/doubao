# doubao —— 豆包图片 / 视频生成 Skill

通过本地 `doubao` CLI 调用火山方舟（Volcengine Ark）豆包视觉模型，在 Claude Code 中**生成图片**（Seedream 系列）与**生成视频 + 管理视频任务**（Seedance 系列）。生成内容默认面向手机/移动设备（竖屏 9:16）。

## 功能概览

| 能力 | 模型家族 | 模式 | 说明 |
|---|---|---|---|
| 图片生成 | Seedream | 同步 | 文生图、图生图、多图融合（≤14 张）、组图/连环图/四格漫画 |
| 视频生成 | Seedance | 异步任务（1–6 分钟） | 文生视频、图生视频、首尾帧、多模态参考（图+视频+音频）、视频延长/编辑 |
| 任务管理 | — | — | 查询 / 列表 / 下载 / 取消视频任务 |

- **图片**：默认 2K 竖屏 `1440x2560`，同步返回，结果为本地图片文件绝对路径。
- **视频**：默认 `720p` + `9:16`，异步任务；支持 `--wait`（阻塞等待并下载）或 `--no-wait`（立即拿 `task_id`，稍后轮询/下载）。

## 优势

- **开箱即用，零安装**：内置 Windows / Linux / macOS（Intel + Apple Silicon）四平台二进制，无需 pip/npm 装环境，找不到 PATH 上的 `doubao` 时自动回退到自带二进制。
- **图片视频一站式**：单个 skill 同时覆盖 Seedream 图片生成与 Seedance 视频生成 + 任务管理，无需在多个工具间切换。
- **移动端友好默认值**：图片默认 2K 竖屏 `1440x2560`，视频默认 `720p` + `9:16`，针对手机观看场景优化，不用每次手动指定。
- **自动模型降级**：用户未指定模型时按优先级依次尝试，遇模型未开通自动降级到下一个，免去查询账户开通了哪些模型的麻烦。
- **本地素材自动上传**：本地图片/视频/音频自动上传对象存储（OSS）再以 URL 传入，HTTP URL 直接透传，无需手动处理文件托管。
- **同步异步双模式**：短任务 `--wait` 阻塞等待并自动下载落地；长任务 `--no-wait` 立即返回 `task_id`，避免长时间阻塞对话。
- **Agent 友好输出**：`--json` 输出干净的机器可读 JSON（结果走 stdout，进度走 stderr），配合结构化退出码（0–5）让 Claude 能精准判断成功/失败并自动处理。
- **丰富的生成场景**：文生图、图生图、多图融合（≤14 张）、组图/连环画；文生视频、图生视频、首尾帧、多模态参考（图+视频+音频）、视频延长/编辑，覆盖绝大多数创作需求。

## 目录结构

```
doubao/
├── SKILL.md              # Skill 主文档（Claude 加载的指令）
├── README.md             # 本文件
├── bin/                  # 各平台自带的 doubao 可执行文件
│   ├── win-x64/doubao.exe
│   ├── linux-x64/doubao
│   ├── osx-arm64/doubao
│   └── osx-x64/doubao
└── references/
    ├── model-ids.md      # 图片模型 ID 与逐模型参数兼容矩阵
    └── model-matrix.md   # 视频模型分辨率×时长×参考媒体兼容矩阵
```

## 何时触发

当用户提到 **doubao / 豆包 / Seedream / Seedance** 并要求：

- **图片**：生成/画一张图、渲染、插画、图生图、改图、多图融合、组图/连环图/四格漫画/故事板
- **视频**：生成视频、让图片动起来、首尾帧过渡、视频延长/编辑/接续、多模态参考生成视频
- **任务**：查询/列表/下载/取消之前的视频任务

## 前置条件

1. **可执行文件**：优先用 PATH 上的 `doubao`；找不到时回退到 `bin/<平台>/` 下自带的二进制。
2. **API Key**：必须配置火山方舟 Ark API Key。
   ```bash
   doubao config show                 # 验证（脱敏显示）
   doubao config set api-key <KEY>    # 未设置时配置
   ```
   Key 来自 <https://console.volcengine.com/ark/region:ark+cn-beijing/apiKey>
3. **模型 ID 必须带日期后缀**，如 `doubao-seedream-5-0-260128`、`doubao-seedance-2-0-260128`。不确定时向用户确认，不要凭空猜后缀。

## 快速上手

```bash
# 文生图（2K 竖屏）
doubao image gen --model doubao-seedream-5-0-260128 \
  --prompt "身穿古风传统汉服的年轻女生高清写真" --size 1440x2560

# 图生图
doubao image gen --model doubao-seedream-5-0-260128 \
  --prompt "将服装改为红色" --image portrait.jpg

# 文生视频（720p / 9:16，默认阻塞等待并下载）
doubao video gen --model doubao-seedance-2-0-260128 --prompt "海浪拍打礁石"

# 长视频用异步：立即拿 task_id，稍后查询/下载
doubao video gen --model doubao-seedance-2-0-260128 --prompt "海浪拍打礁石" \
  --resolution 1080p --duration 10 --no-wait
doubao task get cgt-xxxxxxxx
doubao task download cgt-xxxxxxxx --out result.mp4
```

> Agent 调用时统一加 `--json`：stdout 为干净的机器可读 JSON，进度/调试信息走 stderr。

## 默认模型选择

用户未指定模型时，按优先级依次尝试，遇模型不存在（退出码 2，`InvalidEndpointOrModel.NotFound`）则降级到下一个：

- **图片**：`doubao-seedream-5-0-260128` → `4-5-251128` → `4-0-250828`
- **视频**：`doubao-seedance-2-0-260128` → `2-0-fast-260128` → `2-0-mini-260615` → `1-5-pro-251215`

## 退出码

| 码 | 含义 | 处理 |
|---|---|---|
| 0 | 成功 | 图片读 `.files[]`，视频读 `.video_path` |
| 1 | 参数错误 / 本地校验失败 | 读 `.error.hint`，多为模型×参数不匹配 |
| 2 | Ark API 业务错误 | 读 `.error.code`/`.message`；模型 404 或内容审核失败 |
| 3 | 网络 / 超时 | 等 10s 重试一次 |
| 4 | 文件系统错误 | 输出路径不可写 |
| 5 | API Key 缺失或无效 | `config show` 确认，必要时 `config set api-key` |

## 平台说明

- **Windows**：cmd.exe 下含中文/空格的 prompt 必须用双引号 `"..."`（不识别单引号）。
- **Linux/macOS**：bash/zsh 单双引号均可；自带二进制可能需 `chmod +x`，macOS 首次运行可能需去除 quarantine 属性（`xattr -d com.apple.quarantine`）。

更多细节见 [SKILL.md](./SKILL.md) 及 [references/](./references/) 下的完整参数与兼容矩阵。
