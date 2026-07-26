# 11 - 部署与运维

> 本文档描述 AskMyResume 的运行方式：日常本地开发、完整容器化部署、环境变量清单、备份、轻量观测与上线检查。总体决策以 [00-blueprint.md](00-blueprint.md) 为准（部署形态：Docker Compose，postgres + backend + frontend 三服务）。安全细节见 [08-security-privacy.md](08-security-privacy.md)，各里程碑的部署任务见 [09-roadmap.md](09-roadmap.md)。

## 1. 本地开发跑法（日常姿势）

学习阶段绝大多数时间用这种方式：**只有 PostgreSQL 跑在容器里**，backend / frontend 直接跑在本机，改代码即时生效，调试器可用。

```bash
# 1. 只启动数据库（docker-compose.yml 中的 postgres 服务）
docker compose up -d postgres

# 2. 后端：热重载
cd backend
uv run uvicorn app.main:app --reload --port 8000

# 3. 前端：Next.js dev server
cd frontend
npm run dev   # http://localhost:3000
```

要点：

- postgres 用官方 pgvector 镜像 `pgvector/pgvector:pg16`，免去手动编译扩展；首次连接后仍需在迁移中 `CREATE EXTENSION IF NOT EXISTS vector`（见 [04-data-model.md](04-data-model.md)）。
- backend 本机跑意味着 bge-m3 模型下载到本机的 HuggingFace 缓存（`~/.cache/huggingface`），只下载一次。
- `.env` 放在项目根目录，backend 通过 pydantic-settings 读取（见 [03-architecture.md](03-architecture.md)）；本地开发 `DATABASE_URL` 指向 `localhost:5432`。

## 2. 完整容器化（三服务）

M6 部署阶段使用。拓扑：

```mermaid
flowchart LR
    U[浏览器] -->|"80/443"| C[Caddy 反代<br/>（仅公网部署时）]
    C --> F[frontend<br/>Next.js:3000]
    C --> B[backend<br/>FastAPI:8000]
    B --> P[(postgres<br/>pgvector/pgvector:pg16)]
    P --- V1[(volume: pgdata)]
    B --- V2[(volume: hf-cache<br/>HF_HOME)]
    B --- V3[(bind: data/uploads)]
```

### 2.1 docker-compose.yml 关键片段

```yaml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: askmyresume
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]

  backend:
    build: ./backend
    env_file: .env
    environment:
      HF_HOME: /hf-cache          # 模型缓存指到 volume
    volumes:
      - hf-cache:/hf-cache
      - ./data/uploads:/app/data/uploads
    depends_on:
      postgres: { condition: service_healthy }

  frontend:
    build: ./frontend
    ports: ["3000:3000"]

volumes:
  pgdata:
  hf-cache:
```

### 2.2 backend Dockerfile 要点

- 基础镜像 `python:3.12-slim`，用 `uv sync --frozen` 安装依赖（利用分层缓存：先拷 `pyproject.toml` + `uv.lock` 再拷代码）。
- **不要把 bge-m3 模型打进镜像**：模型约 2GB，会让镜像巨大且每次构建重新下载。正确做法是设 `HF_HOME=/hf-cache` 并挂 named volume（见上），模型在容器**首次启动时**由 `sentence-transformers` 下载到 volume，之后重建镜像 / 重启容器都直接命中缓存。
- **首次启动预热**：在 FastAPI lifespan 启动钩子中加载一次 `SentenceTransformer("BAAI/bge-m3")` 并 encode 一条空串，把「下载 + 加载到内存」的几分钟成本放到启动期，而不是第一个发布请求上。健康检查探针（`GET /api/v1/health`，见 [06-api-design.md](06-api-design.md)）应在预热完成后才返回就绪。

## 3. .env.example 全量清单

`.env.example` 提交进仓库，真实 `.env` 进 gitignore。每项一句说明：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `ANTHROPIC_API_KEY` | （无，必填） | Anthropic API 密钥；SDK 零参构造 `anthropic.AsyncAnthropic()` 时自动读取 |
| `DATABASE_URL` | `postgresql+asyncpg://postgres:postgres@localhost:5432/askmyresume` | SQLAlchemy async 连接串；容器化时 host 改为 `postgres` |
| `POSTGRES_PASSWORD` | `postgres` | compose 里 postgres 服务的密码，需与 `DATABASE_URL` 一致 |
| `JWT_SECRET` | `change-me` | JWT 签名密钥，上线**必须**换成 `secrets.token_urlsafe(32)` 生成值 |
| `UPLOAD_DIR` | `data/uploads` | 简历文件本地存储根目录（见蓝图 §3 Storage 抽象） |
| `EMBEDDING_PROVIDER` | `local` | `local`（bge-m3 本地）或 `voyage`（Voyage AI API） |
| `EMBEDDING_MODEL` | `BAAI/bge-m3` | provider=voyage 时改为 `voyage-3`；维度均为 1024，不可混用索引 |
| `EXTRACT_MODEL` | `claude-opus-5` | 简历结构化抽取模型 |
| `CHAT_MODEL` | `claude-opus-5` | 对话回答生成模型；预算敏感可换 `claude-sonnet-5` |
| `LIGHT_MODEL` | `claude-haiku-4-5` | 查询改写、会话标题等轻任务模型 |
| `CORS_ORIGINS` | `http://localhost:3000` | 逗号分隔的允许来源白名单，上线改为正式域名 |
| `FRONTEND_BASE_URL` | `http://localhost:3000` | 拼接分享链接完整 URL 的前端基址，上线改为正式域名 |
| `RATE_LIMIT_PUBLIC_CHAT` | `10/minute` | 公开对话端点 IP 级限流（slowapi 语法） |
| `RATE_LIMIT_AUTH` | `5/minute` | 登录/注册端点限流，防爆破 |
| `HF_HOME` | （本地开发不设） | HuggingFace 缓存目录；容器内设为 `/hf-cache` |
| `HF_ENDPOINT` | （可选） | 模型下载镜像，如 `https://hf-mirror.com`，国内网络下载慢时用 |

注：分享链接级的 `daily_question_limit` 是每条 `share_links` 记录上的字段（默认 30），不是环境变量，见 [04-data-model.md](04-data-model.md)。

## 4. 数据迁移与备份

### 4.1 alembic upgrade head 的执行时机

两种做法的取舍：

| 做法 | 优点 | 缺点 |
|---|---|---|
| 容器启动脚本里自动执行（`alembic upgrade head && uvicorn ...`） | 部署即最新 schema，单人项目零心智负担 | 迁移失败会卡启动；多副本会并发跑迁移 |
| 手动执行（`docker compose exec backend uv run alembic upgrade head`） | 可控、可先备份再迁 | 容易忘 |

**本项目选自动执行**：单副本、单人开发，"迁移失败卡启动"反而是好事——问题立刻暴露。养成习惯：改动数据的迁移（不只是加列）先在本地跑一遍，部署前先做一次 `pg_dump`。

### 4.2 备份

需要备份的只有两样：数据库、上传的简历文件。

```bash
# 数据库：逻辑备份（放 crontab，每天一次，保留 7 份即可）
docker compose exec -T postgres pg_dump -U postgres askmyresume \
  | gzip > backups/askmyresume-$(date +%F).sql.gz

# 上传文件：直接同步目录
rsync -a data/uploads/ backups/uploads/
```

恢复：`gunzip -c xxx.sql.gz | docker compose exec -T postgres psql -U postgres askmyresume`。`profile_chunks` 里的 embedding 会随 pg_dump 一起备份（vector 列可正常 dump/restore），无需重算；极端情况下丢了也能通过「重新发布档案」重建索引。

## 5. 日志与观测（轻量）

不上 Prometheus/Grafana。两层足够：

**结构化日志**：用标准库 `logging` + JSON formatter，每次 RAG 请求（公开对话端点）记录一条：

```json
{"event": "rag_request", "conversation_id": "...", "rewritten_query": "候选人在 X 公司负责什么",
 "retrieved_chunk_ids": ["...", "..."], "input_tokens": 1830, "output_tokens": 412,
 "latency_ms": {"rewrite": 380, "retrieve": 45, "generate": 2900}}
```

这条日志同时服务于调试（检索命中了什么、改写对不对）和 RAG 调优（配合 [10-testing-evaluation.md](10-testing-evaluation.md) 的评估）。

**数据库即观测面**：`messages` 表已存 `retrieved_chunk_ids`、`input_tokens`、`output_tokens`（见蓝图 §6），月度用量统计一条 SQL 搞定：

```sql
SELECT date_trunc('month', created_at) AS month,
       count(*) FILTER (WHERE role = 'assistant') AS answers,
       sum(input_tokens)  AS input_tokens,
       sum(output_tokens) AS output_tokens
FROM messages
GROUP BY 1 ORDER BY 1 DESC;
```

按 `claude-opus-5` 单价（$5/$25 每百万 token，蓝图 §4）手算月成本即可。

## 6. 成本控制的运维面

- **Anthropic Console** 是权威账单：定期看 Usage 页面，与上面的 SQL 对账；可在 Console 设置月度花费上限（spend limit）。
- **限流配置就是成本闸门**：公开端点是唯一不受你控制的流量入口，IP 限流（`RATE_LIMIT_PUBLIC_CHAT`）+ 链接级 `daily_question_limit` 共同封顶了最坏情况的日花费——上限 ≈ 活跃链接数 × 30 问 × 单问 token 成本。详见 [08-security-privacy.md](08-security-privacy.md)。
- prompt caching（system prompt 稳定部分加 `cache_control`，蓝图 §4）对多轮对话的输入 token 成本有实际削减，属于"写对代码就白拿"的优化。

## 7. 上线检查清单（部署到公网 VPS 时）

MVP 在本机跑通即可毕业；真要给面试官发链接，再过一遍这张表：

- [ ] **换掉所有默认密钥**：`JWT_SECRET`、`POSTGRES_PASSWORD` 重新生成；确认 `.env` 不在 git 里。
- [ ] **HTTPS**：加一个 Caddy 反代容器，Caddyfile 两行搞定自动证书：
  ```
  ask.example.com {
      handle /api/* { reverse_proxy backend:8000 }
      handle        { reverse_proxy frontend:3000 }
  }
  ```
- [ ] **防火墙**：只开 80/443（和你自己的 SSH 端口）；postgres 的 5432、backend 的 8000 一律不映射到宿主机公网。
- [ ] **CORS 收紧**：`CORS_ORIGINS` 只留正式域名，去掉 localhost。
- [ ] **备份任务**：§4.2 的 pg_dump + rsync 进 crontab，并真的做一次恢复演练。
- [ ] **限流生效验证**：curl 连打公开对话端点，确认返回 429。
- [ ] （可选）域名：一个便宜域名 + DNS A 记录指向 VPS 即可，Caddy 自动签证书。

## 8. 常见问题排查表

| 症状 | 原因 | 处理 |
|---|---|---|
| 迁移报错 `type "vector" does not exist` | pgvector 扩展未启用 | 确认镜像是 `pgvector/pgvector:pg16` 而非官方 `postgres`；首个迁移里执行 `CREATE EXTENSION IF NOT EXISTS vector` |
| backend 首次启动卡住几分钟 | bge-m3（~2GB）正在下载 | 正常现象；国内网络设 `HF_ENDPOINT=https://hf-mirror.com`；确认 volume 挂上了 `HF_HOME`，否则每次重建容器都重下 |
| SSE 回答不逐字出现，攒一大段才到 | 反向代理缓冲了响应 | Caddy 对 SSE 默认按流转发，若显式配置了缓冲需去掉；Nginx 需 `proxy_buffering off;`（或后端返回 `X-Accel-Buffering: no` 头）。SSE 事件格式见 [06-api-design.md](06-api-design.md) |
| backend 连不上数据库（容器化后） | `DATABASE_URL` 还写着 `localhost` | 容器内 host 应为服务名 `postgres` |
| 发布档案时 embedding 极慢 | CPU 跑 bge-m3 属正常（单档案几十个 chunk，秒级到十几秒） | 可接受；不能接受则 `EMBEDDING_PROVIDER=voyage` 切 API 方案 |
| 磁盘被撑满 | pgdata / hf-cache / uploads 都在长 | `docker system df` 查看；hf-cache 稳定在 ~2GB，主要盯 pg_dump 备份保留份数与 uploads |

---

至此，00-11 全部文档构成完整规划。开发从 [09-roadmap.md](09-roadmap.md) 的 M0 开始。
