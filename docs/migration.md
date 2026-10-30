# 业务迁移清单：ECS → 新服务器

> 背景：原来跑在阿里云免费 ECS 上，实例快到期，需要把整套机器人迁到新服务器。
> 原则：**先备份、再停机、后切换；旧机保留一段时间用于回滚。**

## 0. 先盘点要搬什么

| 类别 | 内容 | 体积 / 备注 |
| --- | --- | --- |
| 容器 | `llonebot`、`astrbot`、`gsuid_core`、`mihomo` | 4 个 |
| 镜像 | 4 个镜像 | 合计约 8.5 GB |
| 数据 | `AstrBot/data`、`gsuid_core`（含 plugins）、`llonebot-docker/{QQ,llonebot-data}`、`mihomo/config.yaml` | 约几百 MB，QQ 目录最大 |
| 网络 | 自定义 bridge 网络 `bot-net` | 需要在新机重建 |
| 配置 | Token、API Key、主人 QQ、代理订阅 | 只走手工拷贝，不进 Git |

## 1. 冻结（停机）

> ⚠️ **高风险·停机操作**：下面这条会让机器人立即下线。先选低峰时段，并在群里说一声。

```bash
# 【高风险·停机】停掉四个容器，保证备份时没有新的写入
docker stop llonebot astrbot gsuid_core mihomo
```

`docker stop` 之后，即使容器配置了 `restart: always`，也不会自己爬起来——这正好方便迁移。

## 2. 备份

```bash
# 建迁移目录，所有包先放这里
mkdir -p /root/migrate

# 打包 AstrBot 数据（-C 先切到父目录，避免把绝对路径打进包里）
tar -czf /root/migrate/astrbot-data.tar.gz -C /root/AstrBot data

# 打包 LLOneBot 的 QQ 登录态和配置
tar -czf /root/migrate/llonebot-data.tar.gz -C /root/llonebot-docker QQ llonebot-data

# 打包 GsCore 的源码、插件和数据（排除本地 .venv，容器用的是 /venv 卷）
tar -czf /root/migrate/gscore-data.tar.gz --exclude=.venv -C /root/gsuid_core .

# 代理配置单独拷（含订阅，注意保管）
cp /root/mihomo/config.yaml /root/migrate/mihomo-config.yaml

# 校验压缩包没坏：能列出内容就说明完整
tar -tzf /root/migrate/astrbot-data.tar.gz > /dev/null && echo OK
```

## 3. 镜像怎么搬

**方案 A（推荐）：新机重新获取。** 镜像本来就在上游，重新拉取/构建一次，比传 8.5 GB 干净。

**方案 B：导出导入。** 适合新机访问不到镜像源的情况：

```bash
# 先看四个镜像的实际体积，估算要传多少
docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}'

# 导出单个镜像（示例：astrbot）；文件名不要带冒号
docker save -o /root/migrate/astrbot.tar astrbot:latest

# 传到新机后导入
docker load -i /root/migrate/astrbot.tar
```

也可以走流式传输，不在旧机落盘：

```bash
# 直接通过 SSH 把镜像推给新机并加载（示例）；网络中断会失败，重试即可
docker save astrbot:latest | ssh root@<新机IP> 'docker load'
```

> 注意磁盘：旧机 40 GB 已用 63%，同时导出 4 个镜像会不够。要么一个个来、导完就删，要么直接流式传。

## 4. 新机准备

```bash
# 创建和旧机同名的网络；compose 里声明的是 external
docker network create bot-net

# 恢复数据到和旧机相同的路径（路径必须和 compose 里的挂载一致）
mkdir -p /root/AstrBot && tar -xzf astrbot-data.tar.gz -C /root/AstrBot

mkdir -p /root/llonebot-docker && tar -xzf llonebot-data.tar.gz -C /root/llonebot-docker

mkdir -p /root/gsuid_core && tar -xzf gscore-data.tar.gz -C /root/gsuid_core

# 恢复代理配置
mkdir -p /root/mihomo && cp mihomo-config.yaml /root/mihomo/config.yaml
```

把仓库里的 `deploy/` 四份 compose 拷到对应目录，或直接用 Git 拉取仓库。

## 5. 配置检查

- **路径**：数据目录和 compose 里的 `volumes` 必须对得上。
- **Token**：`WS_TOKEN`、LLOneBot 的 WS token、AstrBot 平台 token 三者要一致；迁移时顺便换一轮更安全。
- **地址**：容器之间坚持用服务名（`astrbot`、`gsuid_core`、`mihomo`），不要写死在旧机上记下来的容器 IP。
- **时区**：`TZ=Asia/Shanghai` + `/etc/localtime`。
- **安全组**：新机只放行 SSH（22）；WebUI 端口保持 `127.0.0.1`，走 SSH 隧道。
- **资源**：新机内存不要低于 4 GiB，AstrBot 做 RAG 时最吃内存。

## 6. 切换与验证

> 关键：**同一个 QQ 不能两边同时登录**，会互踢。一定要先停旧机，再起新机。

按顺序启动：`mihomo → llonebot → astrbot → gscore`，然后扫码登录。

验证清单：

- [ ] 私聊发一条，能回
- [ ] 群里 @ 一下，能回
- [ ] 发一个 GsCore 前缀指令（如 `core 帮助`），能回
- [ ] 问一个知识库里的问题，能检索到
- [ ] 主动消息 / 睡眠时段符合预期
- [ ] `docker ps` 四个容器都 Up
- [ ] `free -h`、`df -h` 余量正常
- [ ] 观察 24 小时，确认没有静默掉线

## 7. 回滚

- 旧机**先别释放**，保留 3–7 天。
- 新机异常时：停新机 → 起旧机 → 重新登录 QQ（旧登录态可能已失效，要准备扫码）。
- 记录切换时间点，方便对照日志。

## 8. 迁移后

- 把备份包从 `/root/migrate` 移走或删掉（里面有登录态和订阅）。
- 给新机加定时备份和日志轮转。
- 更新本仓库里“服务器”那一行描述。

## 风险清单

| 风险 | 规避 |
| --- | --- |
| QQ 双登互踢 | 先停旧机，再起新机 |
| 复制登录态后仍要验证 | 提前准备扫码，别在深夜做迁移 |
| 镜像 8.5 GB 撑爆磁盘 | 重新拉取，或流式传输 |
| 免费实例到期被回收 | 至少提前 3 天完成切换 |
| 新机安全组开太大 | 只开 22，WebUI 走隧道 |
| 路径/时区不一致 | 按本文路径恢复，启动后 `docker exec astrbot date` 核对 |
