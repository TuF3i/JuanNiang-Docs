---
title: JuanNiang-Neo 部署与调试指南
---


本文档面向运维和开发者，覆盖部署模式、环境变量、构建流程、健康检查、日志排查和常见故障处理。

> 配置以**环境变量**为准，`config/config.yaml` 仅为参考文档（二进制不读它）。运行时模块配置（Provider/MCP/Prompt/Skill/ACL/ReplyStrategy/T2I/Sandbox/Webhook/CronJob）存 Postgres，通过 Web 面板热切换。

## 部署模式

| 模式 | 用途 | 命令 |
|------|------|------|
| **Dev** | Vite `:3000` 热更新 + Go `:8090` API | `make dev` |
| **本地裸跑** | 跑已构建 binary，单端口服务 `web/dist` | `make build && make run` |
| **Docker Compose** | postgres + redis + app 全栈 | `make docker-up` |
| **预构建镜像** | 直接 pull `ghcr.io/juanniangdev/juan` | 见 README |

## 环境变量

|.env 变量|默认|说明|
|----------|----|----|
| `WEB_DIR` | `web/dist` | 前端构建产物目录；容器内 `/app/web/dist`；空 → 引导提示页|
| `API_ADDR` | `:8090` | Hertz Web API + 仪表板监听地址|
| `JWT_SECRET` | `change-me-in-production` | JWT HMAC 密钥；**务必修改**|
| `OB_PORT` | `8081` | OneBot11 反向 WS 服务监听端口|
| `OB_TOKEN` | (空) | OneBot11 客户端访问令牌；空=不鉴权|
| `OB_ADMINS` | (空) | 逗号分隔的 admin QQ（env fallback；运行时实际从 DB `AdminQQNumbers` 读取）|
| `DB_HOST` | `postgres` (compose) / `localhost` | |
| `DB_PORT` | `5432` | |
| `DB_USER` | `postgres` | |
| `DB_PASSWORD` | `postgres` | |
| `DB_NAME` | `juan` | |
| `REDIS_ADDR` | `redis:6379` (compose) / `localhost:6379` | |
| `REDIS_PASSWORD` | `root` | |
| `REDIS_DB` | `0` | Redis 逻辑库索引|
| `REDIS_PREFIX` | `juan:` | ⚠ 未在 `.env.example` 但 `cache.NewCache` 实际读取|
| `IMG_DIR` | `data/imgs` | 图床图片存储目录（`imgstore`；元数据在 DB `image_assets`）|
| `T2I_BASE_URL` | (空/注释) | ⚠ 仅文档；运行时实际从 DB `t2i_configs` 读取|
| `SANDBOX_BASE_URL` | (空/注释) | ⚠ 同上，从 DB `sandbox_configs` 读取|
| `SANDBOX_API_KEY` | (空/注释) | ⚠ 同上|

> 注意：`T2I_BASE_URL` / `SANDBOX_API_KEY` 等是文档性 env，**真正生效**的配置在 DB（启动时 `loadT2IFromDB`/`loadSandboxFromDB` 读取并构建客户端）。首次启动 DB 无配置时自动 `InitConfig` 建默认行，前端可编辑。

## 开发环境配置（dev.yaml）

本地开发时可用 `dev.yaml` 配置基础设施连接端点，避免每次手动设置环境变量：

```bash
cp dev.yaml.example dev.yaml   # 复制并按需修改
make run                        # 自动读取 dev.yaml
make run-debug                  # 自动读取 dev.yaml + debug 模式
```

优先级：**环境变量 > dev.yaml > 内置默认值**。`dev.yaml` 不存在时程序正常启动（使用环境变量或内置默认值）。`make run` / `make run-debug` 通过 `-dev-config` 参数传入，二进制本身不硬编码该路径。

## 端口约定

| 端口 | 用途 |
|------|------|
| `8090` | Web API + 仪表板（前端 SPA 兜底同端口） |
| `8081` | OneBot11 反向 WebSocket 服务（QQ 机器人框架连接）|
| `8091` | Webhook HTTP 服务（独立端口，按 `WebhookConfig.Port`，默认关闭）|
| `3000` | Vite 开发服务器（仅 dev）；也可作 RAG-Service 默认监听端口 |

## 构建流程

### 本地构建

```bash
# 全量: 先构建前端再产二进制, 产物 bin/juan-niang-neo + web/dist
make build

# 仅构建 Go 后端 (依赖 web/dist 已存在)
make build-go

# 仅前端
make web-install        # 首次安装依赖
make web-build          # typecheck + vite build

# 开发: Vite(:3000) + Go(:8090) 并行
make dev

# 仅跑后端 go run, 自动读取 dev.yaml, 前端走 web/dist
make run

# Debug 模式：自动读取 dev.yaml + pprof (:6060) + Debug 级别日志
make run-debug

# 综合检查 (go vet + 前端 typecheck)
make lint
```

二进制构建参数：`CGO_ENABLED=0 -trimpath -ldflags "-s -w"`，无 `//go:embed web/dist`（前端磁盘服务，便于只换前端不重编 Go）。

### Docker 构建（仓库自带）

`deployments/Dockerfile` 三阶段：

1. **`web-builder`** (`node:20-alpine`)：`npm ci || npm install` → `npm run build` → `dist/`
2. **`go-builder`** (`golang:1.25-alpine`)：`go mod download` → `go build -trimpath -ldflags "-s -w" -o /juan-niang-neo ./cmd/server/`
3. **runtime** (`alpine:latest`)：装 `ca-certificates tzdata wget`，以 root 运行，复制 binary 与 `web/dist`，`WEB_DIR=/app/web/dist` `TZ=Asia/Shanghai`，`EXPOSE 8081 8090`，`HEALTHCHECK` `wget -qO- http://127.0.0.1:8090/health`

另有 `Dockerfile.cn` 使用国内镜像加速。

### 运行时数据持久化（Docker）

容器内 `/app/data` 整体由 compose bind-mount 到仓库 `data/` 目录，跨升级/重建保留：

| 路径 | 内容 | 丢失影响 |
|------|------|----------|
| `data/pluggins/` | Lua 插件（含启动时自动写入的 SDK 与 system 插件） | 插件丢失 |
| `data/imgs/` | 图床图片 | 图片丢失 |
| `data/plugin_store.json` | 插件商店配置（镜像源选择 / 自定义镜像 / 仓库地址） | 商店配置重置为默认 |

> ⚠️ 首次启动前 `mkdir -p data && chmod 777 data`（当前镜像以 root 运行；改为非 root 用户后需按用户赋权）。

## 可选服务：RAG-Service（向量检索）

RAG-Service 是**独立部署**的 Rust 服务（bge 模型进程内推理，零外部依赖），为知识库 / 长期记忆 / 群管理提供语义检索。**不部署也不影响主流程**——全部调用方自动降级为接入前行为。仓库：[JuanNiang-RAG-Service](https://github.com/JuanNiangDev/JuanNiang-RAG-Service)。

```bash
git clone https://github.com/JuanNiangDev/JuanNiang-RAG-Service && cd JuanNiang-RAG-Service
make download          # 下载 bge-small-zh-v1.5 GGUF 模型
cargo run --release    # 默认监听 127.0.0.1:3000
```

可用环境变量覆盖：`RAG_PORT` / `RAG_N_THREADS` / `RAG_MAX_CTX` 等。API 契约见其 `docs/API.md`（`PUT /tags/{tag}` upsert、`GET /tags/search` 检索、`DELETE /tags/{tag}`、`GET /health`、`GET /info`）。

部署后在 Web 面板「RAG 向量」页填 `base_url`（如 `http://localhost:3000`）并勾选启用；随后可在知识库 / 群管理 / 记忆页面点「同步向量库」做首次全量同步（新增/编辑/删除会自动双写双删，无需手动）。

## Prometheus 监控

`GET /metrics`（与 `/health` 同级，**无需 JWT**）暴露 Prometheus 文本格式指标（前缀 `juanniang_`），覆盖消息流 / Agent / 并发 / LLM / 群管理 / RAG / 插件 / HTTP / 库存 / 外部服务健康 + Go runtime。**注意：`/metrics` 无鉴权，公网暴露需在反代层限源 IP。**

### 采集配置（prometheus.yml）

```yaml
scrape_configs:
  - job_name: juanniang
    scrape_interval: 30s
    metrics_path: /metrics
    static_configs:
      - targets: ["127.0.0.1:8090"]
```

### 建议面板

- **总览**：事件吞吐（`rate(juanniang_events_total[5m])`）、Agent 循环结果（`by outcome`）、LLM 请求（`by provider`）、群管理处罚（`by action`）、外部服务健康（`juanniang_external_health`，0/1）
- **长尾**：`histogram_quantile(0.95, sum(rate(<xxx>_duration_seconds_bucket[5m])) by (le))` 看 LLM / Agent 循环 / HTTP / RAG 检索延迟
- **群管理调阈值**：`juanniang_groupmgr_rag_score` 分数分布，对照面板 `high_score` / `low_score`
- **降级监控**：`juanniang_rag_search_errors_total`、`juanniang_message_dropped_total{reason="irrelevant"}`

完整指标清单见 [Web API：适配器与会话](development/api/infra.md#9-metricsprometheus)。

## 健康检查

- `GET /health` 二级域名/api 均可：`{"status":"ok"}`，无需鉴权
- Docker `HEALTHCHECK` 调用的就是它（30s 间隔）
- `GET /api/v1/overview` 含 `t2i_active`/`t2i_healthy`/`sandbox_active`/`sandbox_healthy`/`rag_active`/`rag_healthy`（需 token，前端仪表板调用）
- `GET /api/v1/t2i/health` / `GET /api/v1/sandbox/health` / `GET /api/v1/rag/health` 实时探活；`GET /api/v1/rag/info` 返回模型/内存/向量规模

## 日志排查

- 使用 `internal/logging` 自定义日志包（底层 `github.com/fatih/color`），支持彩色 stdout、JSON 格式化、WARN+ 调用栈
- 双写到 stdout 与 `logging.Default` Hub（环形 250 条 + SSE 实时订阅）；GORM SQL 语句也可通过 Hub 订阅
- 通过 `logging.NewModule("name")` 创建模块 logger，Web UI 可集中查看所有模块日志
- 前端查看：Web 面板"日志"页（`GET /api/v1/logs` 最近 250 + `GET /api/v1/logs/stream` SSE）
- 命令行查看：`docker logs -f juan-niang-neo` 或 `journalctl -u juan-niang-neo -f`（systemd）
- 插件日志带 `[plugin:<name>]` 前缀
- 启动日志会打印各模块就绪状态、adapter 监听地址、加载的插件数与 Adapter Admins 列表

## Debug 模式

启动时加 `-debug` 标志开启：

```bash
make run-debug
# 或
./bin/juan-niang-neo -debug
# 自定义 pprof 端口
./bin/juan-niang-neo -debug -pprof-addr :6061
```

Debug 模式下：

| 功能 | 说明 |
|------|------|
| 日志级别 | Debug，所有 Debug 级别日志可见（插件图片处理耗时、异步消息发送耗时、Eino tool call 详情等） |
| pprof | HTTP 服务 `:6060`，支持 CPU/heap/goroutine 等 profile |
| 启动详情 | 打印 Go 版本、CPU 核数、每个插件的 name/version/permissions |

pprof 常用命令：

```bash
# CPU 火焰图（采集 30s）
go tool pprof -http :8080 http://127.0.0.1:6060/debug/pprof/profile

# goroutine 快照
go tool pprof -http :8080 http://127.0.0.1:6060/debug/pprof/goroutine

# 内存分配
go tool pprof -http :8080 http://127.0.0.1:6060/debug/pprof/heap
```

## 优雅退出

`SIGINT`/`SIGTERM` → 反向关闭顺序（`cmd/server/main.go:287` `shutdown`）：

1. `hago.Stop()`（当前为占位，仅打日志；事件循环/CronJob 退出依赖外层 ctx 取消）
2. `WebhookAdapter.Stop`（3s graceful）
3. `Adapter.Stop`（5s；先停 adapter 再停 web，避免 web 请求持 adapter 锁死锁）
4. `webEngine.Shutdown`（5s）

外层 watchdog 15s 超时强退，避免任一 Stop 调用挂死拖垮整体。

## 常见故障

| 现象 | 原因与解决 |
|------|-----------|
| 前端访问 404 但 API 正常 | `WEB_DIR` 未构建或不正确；`cd web && npm install && npm run build` |
| 前端显示引导提示页 | `web/dist/index.html` 缺失，同上 |
| 启动报 "Postgres 连接失败" | DB_HOST/PORT/USER/PASSWORD/NAME 错；compose 用 `postgres` 主机名 |
| 启动报 "Redis 連接失败" | REDIS_ADDR/PASSWORD 错；compose 用 `redis:6379` |
| OneBot 客户端连不上 8081 | OB_TOKEN 不匹配；浏览器访问无 `Authorization: Bearer`；检查防火墙 |
| LLM 不回复消息 | 1) 没配置/激活 text_model Provider；2) 相关性判断不通过（relevance 策略只回 @/命令/提及名字或高相关消息）；3) ACL 拒绝；4) 群聊静默短语 |
| Agent 提示"未启用 T2I" | Web 面板 T2I 配置未启用 / `base_url` 不可达；`GET /t2i/health` 为 false |
| RAG 检索/写入失败 | RAG-Service 未部署或不可达；`GET /rag/health` 为 false。未配置时各调用方自动降级（知识库 SQL 匹配 / 记忆 pg_trgm / 群管理关键词），不报错 |
| 群管理不生效/重复处罚 | 1) Web 面板群管理未启用；2) 排除群/白名单命中；3) **旧 Lua 插件 `redrock_group_manager` 未停用**（会双重检测重复处罚） |
| CronJob 不触发 | 留意这是 6 字段（秒级）cron；`0 0 9 * * *` 才是每天 9:00 |
| 插件改了不生效 | 改 `pluggin.yaml` 必须 reload；改 Lua 文件也要 toggle 后才重新 DoFile |
| `__NO_REPLY__` 类静默 | Agent LLM 主动判定不回复，检查 system prompt 与回复策略 |
| Adapter 重启后事件丢失 | 不会丢——EventLoop 检测 channel 关闭后 sleep 1s 重新获取句柄 |

## 反向代理

生产可用 nginx 在 `8081` / `8090` / `8091` 前：

```nginx
# API + 仪表板
location / {
    proxy_pass http://127.0.0.1:8090;
    proxy_set_header Host $host;
}
# SSE 日志流
location /api/v1/logs/stream {
    proxy_pass http://127.0.0.1:8090;
    proxy_buffering off;
    proxy_read_timeout 24h;
}
# OneBot11 反向 WS
location /ws {
    proxy_pass http://127.0.0.1:8081;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

## 系统服务（示例 systemd unit）

```ini
[Unit]
Description=JuanNiang-Neo
After=network-online.target postgresql.service redis.service
Wants=network-online.target

[Service]
Type=simple
User=jn
WorkingDirectory=/opt/juan-niang-neo
EnvironmentFile=/opt/juan-niang-neo/.env
ExecStart=/opt/juan-niang-neo/juan-niang-neo
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## 首次启动

1. 准备 Postgres + Redis（或直接 `make docker-up` 用 compose 拉起它们）
2. 设置 `.env`（至少改 `JWT_SECRET`）
3. 启动进程；首次会 AutoMigrate 39 张表 + 创建 `admin / Admin123`
4. 立即登录 Web 面板，`POST /change-password` 改默认密码
5. 在"Providers"页配置 LLM Provider（OpenAI 兼容端点），激活
6. 在"Adapter"页配置 OB_TOKEN 与 admin QQ，启用
7. 让 OneBot11 实现（NapCat/Lagrange 等）反向 WS 连 `ws://host:8081/`，带 `Authorization: Bearer <OB_TOKEN>`
8. 在"回复策略"页配置群聊行为（仅 `relevance` 按相关性回复：@/命令/提及名字必回）
9. 可选：部署 RAG-Service 并在「RAG 向量」页启用（知识库/记忆/群管理自动优先语义检索，未配置自动降级）；启用群管理并停用旧 Lua 插件 `redrock_group_manager` 防止双重处罚

## FAQ

**Q: 为什么 LLM 拒绝调用某个工具？** A: ACL 规则把它拒绝了，或它在 MCP 但 MCP 断连；可在"ACL"页或"日志"流查看。

**Q: Agent 在群里不回我？** A: 检查回复策略 + `isAtSelf` 是否精确匹配 `[CQ:at,qq=<bot>]`；`relevance` 模式下不会回复相关性低的消息。相关性判断有批量合并/冷却缓存/刷屏降级等优化——判断失败时默认不回复，可在回复策略页把"判断失败策略"改为 `reply` 照常回复。

**Q: 想只换前端不重编 Go？** A: 可以——前端是磁盘文件，`WEB_DIR` 指向新 `web/dist` 即可；二进制不嵌入它。

**Q: `internal/core/handler/` 是什么？** A: 当前为空目录占位，天真以为有 handler 包会失望；核心逻辑在 `dao`/`acl`/`cache` 里。