# 豆包图片模型 ID 与参数兼容性

`doubao` 中文合并 skill 的图片二级参考。需要以下信息时阅读：
- 传给 `--model` 的确切模型 ID
- 某模型家族是否支持某参数
- 各家族的 `--size` 取值

## 模型家族

| 家族前缀 | 能力 |
|---|---|
| `doubao-seedream-5-0-lite-*` | 文生图、图生图（1–14 参考）、组图、web_search、png/jpeg 输出、流式 |
| `doubao-seedream-4-5-*` | 文生图、图生图（1–14 参考）、组图、仅 jpeg |
| `doubao-seedream-4-0-*` | 文生图、图生图（1–14 参考）、组图、仅 jpeg |
| `doubao-seedream-3-0-t2i-*` | 仅文生图（无图生图、无组图）、仅 jpeg |

实际模型 ID 随火山引擎发布新版而变化。2026 年 6 月示例：

- `doubao-seedream-5-0-lite-260128`
- `doubao-seedream-4-5-251128`

**绝不凭空编日期后缀**。不确定时让用户从[控制台](https://console.volcengine.com/ark/region:ark+cn-beijing/openManagement)复制。

`--model` 也接受 endpoint ID（`ep-xxxxx`）。

## 各家族 `--size`

### Seedream 5.0 lite
- 语义值：`2K`、`3K`、`4K`
- 显式：宽×高，总像素 `[2560×1440=3 686 400, 4096×4096=16 777 216]`，宽高比 `[1/16, 16]`
- **skill 移动端默认：`1440x2560`（2K 级 9:16 竖屏）**
- API 默认：`2048x2048`
- 常用：`1440x2560`、`2048x2048`、`2304x1728`、`1728x2304`、`2848x1600`、`1600x2848`

### Seedream 4.5
- 语义值：`2K`、`4K`（**无 3K**——常见错误，CLI 会拒绝并返回退出码 1）
- 像素/比例约束同 5.0 lite

### Seedream 4.0
- 语义值：`1K`、`2K`、`4K`
- 总像素：`[1280×720, 4096×4096]`
- 宽高比规则同上

### Seedream 3.0 t2i
- 像素：`[512×512, 2048×2048]`
- 无语义快捷值

## 逐模型参数支持

| 参数 | 5.0-lite | 4.5 | 4.0 | 3.0-t2i |
|---|---|---|---|---|
| `--image`（图生图） | ✅ 1-14 | ✅ 1-14 | ✅ 1-14 | ❌ |
| `--sequential auto` | ✅ | ✅ | ✅ | ❌ |
| `--web-search` | ✅ | ❌ | ❌ | ❌ |
| `--output-format png` | ✅ | ❌（仅 jpeg） | ❌ | ❌ |
| `--stream` | ✅ | ✅ | ✅ | ❌ |
| `guidance_scale` | ❌ | ❌ | ❌ | ✅（当前 CLI 未暴露） |

## 参考图约束

- 单张 ≤ 30 MB
- 宽、高各 > 14 px
- 总像素 ≤ 6000×6000 = 36 MP
- 宽高比 `[1/16, 16]`
- 所有参考图 + 提示词 + 其他字段 base64 合计 ≤ 64 MB 请求体
- 格式：jpeg、png。5.0-lite/4.5/4.0 额外支持 webp、bmp、tiff、gif、heic、heif。

合计超 64 MB 时 CLI 拒绝，并提示先把大图传到公网 URL。

## 组图约束

- `--sequential auto` 让模型决定张数（即使 `--max-images 4` 也可能只返回 1 张）。
- `--max-images` 范围 `[1, 15]`。
- 输入参考图 + 生成图 ≤ 15。

## 输出产物

- 默认响应 `url`（24h 签名的火山 TOS 链接），CLI 自动下载到 `--out`。
- 远端 URL 24 小时失效。**不要把 URL 当成果物给用户**——本地文件才是产物。
- 扩展名 `.jpg`，除非 `--output-format png`（仅 5.0-lite）。
