# AstrBot 配置清单

AstrBot 的配置大多在 WebUI 里点点点，不方便像 compose 那样存成模板。这里只记录**需要在 WebUI 里填什么**，不存任何真实值。

## 1. 消息平台（平台适配）

新建一个 **aiocqhttp（OneBot v11）** 平台：

| 项 | 填什么 | 说明 |
| --- | --- | --- |
| 连接方式 | 反向 WebSocket / 由 LLOneBot 主动连接 | 对应 LLOneBot 里的 `ws-reverse` |
| 端口 | `6199` | compose 已映射到 `127.0.0.1:6199` |
| Token | 自己生成一串随机值 | 必须与 LLOneBot 的 `token` 完全一致 |

## 2. 模型提供商

| 项 | 说明 |
| --- | --- |
| DeepSeek | 填 API Key；低谷时段使用 |
| 通义千问（阿里云百炼） | 填 API Key；高峰时段使用 |
| 嵌入模型 | 百炼的千问系列嵌入模型，知识库检索用 |
| 重排模型 | 百炼的重排模型，给召回结果重新排序 |

> 模型切换按 DeepSeek 的错峰计价安排：高峰走千问，低谷回 DeepSeek。

## 3. 知识库

5 个集合：`story`、`baike`、`character`、`emotion_happy`、`gal_anime`。
上传后按 **分块 1024 / 重叠 120** 切分；批处理 16、并发 1、失败重试 3。

## 4. 插件配置

- `astrbot_plugin_gscore_adapter`：IP 填 `gsuid_core`、PORT `8765`、BOT_ID `AstrBot`、WS_TOKEN 与 GsCore 一致、前缀列表按需填（`core` / `gs` / `sr` / `zzz` / `ww` / `蝙蝠` 等）。
- `antipromptinjector`：保持默认规则即可，先观察误伤。
- `proactive_chat`：注意发言频率，避免刷屏。
- `meme_manager`：图床凭据只填在 WebUI，不要写进仓库。

## 5. 不要做的事

- 不要把 `data/` 目录提交到 Git（里面有 API Key、聊天数据库、日志）。
- 不要把 WebUI 端口改成 `0.0.0.0` 直接暴露公网。
