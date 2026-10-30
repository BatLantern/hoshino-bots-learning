# 排障手册

按「症状 → 先看哪里 → 怎么处理」组织。命令都在服务器上执行；`<...>` 是占位符。

## 1. 机器人完全不回消息

**先看 LLOneBot 是否在线、QQ 是否掉线：**

```bash
# 看容器是不是 Up 状态
docker ps --filter name=llonebot

# 看最近日志里有没有断开 / 重连 / 扫码提示
docker logs --tail 100 llonebot
```

**再看 AstrBot 有没有收到事件：**

```bash
# AstrBot 日志里应该能看到消息事件；没有就是 WS 没通
docker logs --tail 100 astrbot
```

**常见原因：**

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| LLOneBot 日志里连接被拒 | 反向 WS 地址写的是旧容器 IP | 改成 `ws://astrbot:6199/ws` |
| AstrBot 报 Token 不匹配 | 两边 Token 不一致 | 对齐 LLOneBot 的 `token` 与 AstrBot 平台配置 |
| LLOneBot 显示未登录 | QQ 掉线 / 需要重新扫码 | 经 SSH 隧道打开 `3080` 重新扫码 |

## 2. GsCore 的指令没反应，但普通聊天正常

说明 AstrBot 活着，问题在「AstrBot → GsCore」这一段。

**先确认两个容器在同一条网络：**

```bash
# 看 gsuid_core 是否在 bot-net 里；没有的话 AstrBot 用容器名找不到它
docker inspect gsuid_core --format '{{json .NetworkSettings.Networks}}'
```

**再从 AstrBot 容器里探测一下端口：**

```bash
# 在 astrbot 容器内访问 gsuid_core 的 8765；能通会打印 HTTP 状态码
docker exec astrbot sh -c 'curl -sS -o /dev/null -w "%{http_code}\n" --max-time 3 http://gsuid_core:8765/ || echo FAILED'
```

**常见原因：**

- `gsuid_core` 没接进 `bot-net`（本项目早期就是手动 `docker network connect` 补的，现在已写进 compose）。
- 适配器里的 `WS_TOKEN` 与 `data/config.json` 不一致。
- `GSCORE_ONLY_PREFIXES` 没包含你发的指令前缀。

## 3. 发消息后等很久，最后报超时

**先确认代理是否生效：**

```bash
# 从 astrbot 容器里测试出站；能返回状态码说明代理链路通
docker exec astrbot sh -c 'curl -sS -o /dev/null -w "%{http_code}\n" --max-time 10 https://api.deepseek.com'
```

**再看 `NO_PROXY`：**

```bash
# 确认容器名和 127.0.0.1 都在 NO_PROXY 里
docker inspect astrbot --format '{{json .Config.Env}}'
```

如果 `NO_PROXY` 漏了容器名，AstrBot 访问 GsCore 也会走代理，表现为「代理配好了，但适配器连不上」。

## 4. WebUI 打不开

**这是预期行为，不是故障。** 端口只绑了 `127.0.0.1`，公网访问不到。

```bash
# 在本机开隧道；保持这个窗口不关
ssh -L 6185:127.0.0.1:6185 root@<ECS_IP>
```

然后浏览器访问 `http://127.0.0.1:6185`。

> 不要把绑定改成 `0.0.0.0` 来「图方便」——等于把管理后台挂到公网。

## 5. 越来越卡，甚至被 OOM 杀掉

```bash
# 看内存和 swap；available 很低就要处理
free -h

# 看是谁在吃内存
docker stats --no-stream
```

4 GiB 机器跑四个容器的余量不大。可以：

- 给 AstrBot 限制日志量（见下）；
- 减少知识库一次检索的片段数；
- 确认 swap 已开启（本机是 4 GiB swap）。

## 6. 日志把磁盘吃满

```bash
# 看 Docker 日志占了多少
du -sh /var/lib/docker/containers/*/*-json.log 2>/dev/null | sort -h | tail

# 看整体磁盘
df -h /
```

**高风险操作提醒**：下面这条会删除所有未使用的镜像、容器、网络和构建缓存，可能误删你还要用的东西。**先 `docker system df` 看清楚再决定**，不要在不清楚影响时执行。

```bash
# 【高风险】清理未使用资源；执行前先确认没有需要保留的容器/镜像
# docker system prune -a
```

更稳妥的做法是给 compose 加日志上限（`logging.options.max-size`），从源头控制。

## 7. 时间对不上

```bash
# 看宿主机时间
date

# 看容器内时间；不一致会体现在日志和消息时间上
docker exec astrbot date
```

compose 里已经注入 `TZ=Asia/Shanghai`，AstrBot 还额外挂载了 `/etc/localtime`。如果还不对，检查宿主机时区。

## 8. 雀魂插件失效

这是另一个故事：官方换了登录流程、牌谱屋加了反爬，最后选择止损下线。完整过程见 [2026-10-16 复盘](retrospectives/2026-10-16-majsoul.md)。

结论：**不是所有失败都值得修**。先判断是「自己的配置问题」还是「上游环境变了」，后者要算投入产出。

## 9. 容器 Up 但机器人不回（静默失败）

**这是印象最深的一次，凌晨 1 点，从发现到恢复大约 5 分钟。**

### 时间线

| 步骤 | 做了什么 | 看到什么 |
| --- | --- | --- |
| 1 | 群友反馈机器人不回；先用管理权限“强唤醒”测试 | 强唤醒也没反应，排除「机器人睡觉了」 |
| 2 | 打开 AstrBot 日志面板，只看 error | 插件消息发送失败 → 紧接着保底机制也失败 |
| 3 | 查内存、磁盘、`docker ps` | 都正常，没有 OOM 记录 |
| 4 | 看 LLOneBot 的邮件告警 | 没有告警 |
| 5 | 重启 AstrBot | 仍然不回 |
| 6 | 观察心跳 | LLOneBot 的心跳包和 AstrBot 日志同时不走了 |
| 7 | 登录 LLOneBot WebUI | QQ 登录态已退出；容器还是 Up；日志报账号连接超时 |
| 8 | 手动重启 LLOneBot、重新登录、等 AstrBot 重连 | 恢复 |

### 根因

不是资源问题，也不是账号风控/顶号，而是 **LLOneBot 容器里的 QQ 主进程崩了，其他进程还活着**：

- 容器最外层看是 `Up`，WebUI 也还能打开；
- 但 QQ 账号已经掉线，LLOneBot 发不出消息，日志里是“账号连接超时”；
- 怀疑触发源是群友发的不兼容表情包/代码块，让 NTQQ 主进程卡死崩溃。具体机制没有完全复现，但现象能对上。

### 为什么 `restart: always` 没救回来

Docker 的自动重启策略只看**容器主进程（PID 1）有没有退出**。主进程活着、内部某个子进程死了，Docker 认为“容器正常”，不会重启。

用一句话记：**`docker ps` 只能证明“壳还在”，不能证明“服务能用”。** 这就是“静默失败”。

### 可复用的排查顺序

1. **业务层**：先看应用日志里的第一条错误（这里是“消息发送失败”），不要一上来就重启。
2. **资源层**：`free -h`、`df -h`、`docker stats --no-stream`，排除 OOM 和磁盘满。
3. **链路层**：按 QQ → LLOneBot → AstrBot → GsCore 分段，看哪一段的心跳/日志先停。
4. **登录态**：LLOneBot 的 WebUI 是否需要重新登录——这一步是本次的关键。
5. **恢复**：重启出问题的容器 + 重新登录 + 等上游重连，最后发一条消息验证。

### 预防

- 加一个**业务探针**：定时检查 LLOneBot 登录态 / WebSocket 是否还在，异常就发邮件；不要只盯进程。
- 保留“重启 + 扫码”的操作手册（本文就是）。
- 这个规模不需要 Prometheus + Grafana + node_exporter，也不值得上 K8s 探针；一条定时脚本 + 告警就够。
