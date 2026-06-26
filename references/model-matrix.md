# 豆包 Seedance 视频模型矩阵

`doubao` 中文合并 skill 的视频二级参考。需要完整逐模型兼容信息时阅读。

## 模型家族

| 家族前缀 | 示例 ID | 说明 |
|---|---|---|
| `doubao-seedance-2-0-*` | `doubao-seedance-2-0-260128` | 完整 2.0，最高画质，支持 4k 与多模态参考 |
| `doubao-seedance-2-0-mini-*` | `doubao-seedance-2-0-mini-260615` | 最小最快最便宜，720p 上限，适合迭代/草稿 |
| `doubao-seedance-2-0-fast-*` | `doubao-seedance-2-0-fast-260128` | 比完整 2.0 快，720p 上限 |
| `doubao-seedance-1-5-pro-*` | `doubao-seedance-1-5-pro-251215` | 较旧但支持 `seed`/`camera-fixed`，无多模态参考 |
| `doubao-seedance-1-0-pro-*` | （随版本变化） | 旧版 |
| `doubao-seedance-1-0-pro-fast-*` | （随版本变化） | 旧版，仅 `first_frame`，无 `last_frame` |

**绝不凭空编日期后缀**，向用户确认带日期的完整 ID。

**skill 移动端默认**：用完整 `doubao-seedance-2-0-*` 家族 + `720p` + `--ratio 9:16`。mini/fast 仅用于草稿，或用户明确优先速度/成本而非画质时。

## 分辨率支持

| 分辨率 | 2.0 | 2.0-mini | 2.0-fast | 1.5-pro | 1.0-pro | 1.0-pro-fast |
|---|---|---|---|---|---|---|
| 480p | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 720p | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 1080p | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ |
| 4k | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

4k 输出是 10-bit H.265 编码——多数浏览器和 Windows Media Player 播不了，建议 VLC 或 MPV。

## 时长

| 模型 | 时长范围 | 支持 `-1`（自动） |
|---|---|---|
| 2.0 / 2.0-mini / 2.0-fast | [4, 15] s | ✅ |
| 1.5-pro | [4, 12] s | ✅ |
| 1.0-pro / 1.0-pro-fast | [2, 12] s | ❌（用确切值） |

`--frames` 是 `--duration` 的替代，仅 **1.0-pro / 1.0-pro-fast** 支持。范围 `[29, 289]`，须满足 `(frames - 25) % 4 == 0`。

## 逐参数兼容

|  | 2.0 系列 | 1.5-pro | 1.0-pro | 1.0-pro-fast |
|---|---|---|---|---|
| `--image first_frame` | ✅ | ✅ | ✅ | ✅ |
| `--image last_frame`（即 2 张图） | ✅ | ✅ | ✅ | ❌ |
| `--image reference_image`（1–9） | ✅ | ❌ | ❌ | ❌ |
| `--video reference_video`（1–3） | ✅ | ❌ | ❌ | ❌ |
| `--audio reference_audio`（1–3） | ✅ | ❌ | ❌ | ❌ |
| `--seed` | ❌ | ✅ | ✅ | ✅ |
| `--camera-fixed` | ❌ | ✅ | ✅ | ✅ |
| `--frames` | ❌ | ❌ | ✅ | ✅ |
| `--service-tier flex`（离线半价） | ❌ | ✅ | ✅ | ✅ |
| `--priority 0-9` | ✅ | ❌ | ❌ | ❌ |
| `--web-search` | ✅ | ❌ | ❌ | ❌ |
| `--generate-audio` | ✅（默认 true） | ✅（默认 true） | ❌ | ❌ |

## 角色推断

CLI 根据图片数量 + 是否有其他媒体自动推断 `--image-role`：

| 输入 | 推断角色 |
|---|---|
| 1 张图，无视频/音频 | `first_frame` |
| 2 张图，无视频/音频，模型支持尾帧 | `first_frame`、`last_frame` |
| 任意图 + 有视频/音频 | 所有图都是 `reference_image` |
| 1.0-pro-fast + 2 张图 | 推断失败——CLI 拒绝（模型不支持尾帧） |

显式传 `--image-role` 可覆盖。角色数量须与 `--image` 个数匹配。

**单次请求内互斥的两组**：
- `first_frame` + `last_frame`（首尾帧视频）
- `reference_image` + `reference_video` + `reference_audio`（多模态参考）

这两组**不能**在同一请求中混用。

## 输入媒体约束

### 图片（作视频参考）
- 单张 ≤ 30 MB，宽高比 (0.4, 2.5)，尺寸 (300, 6000) px
- 格式：jpeg、png、webp、bmp、tiff、gif（1.5-pro 与 2.0 系列额外支持 heic/heif）
- 多模态参考最多 9 张，首尾帧 2 张，首帧 1 张

### 视频（作参考）
- mp4 / mov，H.264 或 H.265 视频，AAC 或 MP3 音频
- 480p / 720p / 1080p / 4k 输入
- 单个视频 [2, 15] s，≤ 3 个，总时长 ≤ 15s
- 单个 ≤ 200 MB，宽高比 (0.4, 2.5)，尺寸 (300, 6000) px，总像素 [640×640, 3326×2494]
- FPS [24, 60]
- **本地文件作 `--video` 会 base64 膨胀，极易突破 64MB 请求体——请用公网 URL 或 `asset://` ID。**

### 音频（作参考）
- wav / mp3
- 单段 [2, 15] s，≤ 3 段，总时长 ≤ 15s
- 单段 ≤ 15 MB
- 音频不能单独传——须至少配一张图或一段视频

## 素材库

Seedance 2.0 系列拒绝用户提供的含真实人脸素材。变通方案：

1. 用 Ark 预置素材库的 `asset://<ASSET_ID>` URL（人脸已预清洗）。
2. 用本账号近 30 天内由 Seedance 2.0 生成的视频/图片。

对 `--video https://...` 或 `--image asset://...`，CLI 直接透传 URL，不做 base64 转换（避开 64 MB 限制）。

## 服务层级

- `default`（在线实时）：更高并发上限，全价
- `flex`（离线）：更高吞吐上限（TPD），半价，等待更久——**Seedance 2.0 系列不支持 flex**

## 其他技术限制

- 请求体合计 ≤ 64 MB。CLI 累加所有 base64 大小，超限提前拒绝。
- 生成视频 URL（火山 TOS 签名）在任务成功后 **24 小时**失效。
- 任务记录在 Ark 仅保留 **7 天**——之后 `task list` / `task get` 查不到。
- `execution_expires_after`：Ark 杀掉卡住任务前的等待时长。范围 [3600, 259200] s，默认 172800（48h），用 `--execution-expires-after` 指定。
