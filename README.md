# 基于 FastAPI + Streamlit 的【RAG 知识问答】系统

- **状态：** 功能冻结，只接受文档与保活修复；已实现功能见下一节
- **Live：** [chatter-5y96.onrender.com](https://chatter-5y96.onrender.com)
- **记录：** [笔记](docs/NOTES.md)
- **License：** [MIT](LICENSE.md)

## 已实现功能

- 打开页面即可提问，无需登录
- 侧边栏上传 PDF / Markdown / TXT，写入向量库
- 用 SQLAlchemy 连接串 + 单条 `SELECT` 从数据库同步文本
- 用公网 `http/https` 网址导入网页正文
- 入库前可改分块大小和重叠；只作用于之后导入的内容，不改写已有分块
- 查看文档与分块预览，可按文档删除
- 同一来源再导入会替换旧分块，不会堆出多份
- 基于知识库流式问答；回答后展示参考来源，区分已引用 / 检索到但未引用
- 「开启新聊天」清空当前会话的展示历史和喂给模型的短期上下文

未实现：登录与多用户隔离、对话持久化、多个独立知识库、混合检索 / rerank、计费与限流。

## 技术栈

| 技术 | 在本项目中的应用 |
| --- | --- |
| Python 3.12 | 基础编程语言 |
| [FastAPI](https://fastapi.tiangolo.com/) | 后端 API：入库、检索、SSE 流式问答 |
| [Streamlit](https://streamlit.io/) | 前端页面：侧边栏知识库 + 主区聊天 |
| [LlamaIndex](https://www.llamaindex.ai/) | 句子分块、Qdrant 向量索引 |
| [Qdrant](https://qdrant.tech/) | 向量库；生产用 Cloud |
| [LiteLLM](https://www.litellm.ai/) | 流式调用 DashScope（通义千问） |
| DashScope Embedding | 文本向量化，维度与 Qdrant collection 对齐 |
| [uv](https://docs.astral.sh/uv/) | `backend/`、`frontend/` 各自的依赖与运行 |
| Ruff、Pytest | 静态检查与单元测试 |
| [GitHub Actions](https://github.com/features/actions) | CI：两边分别 lint + 测试 |
| [Render](https://render.com/) | 线上：前端 Web Service + 后端 Web Service |

## 项目结构

```
rag-chatter/
├── .github/workflows/ci.yml   # CI：backend / frontend 分开跑
├── docs/
│   └── NOTES.md               # 学习笔记
├── backend/
│   ├── src/
│   │   ├── api/               # 路由、schema、异常处理
│   │   ├── app/services/      # 入库、删除、流式问答
│   │   ├── domain/            # 模型与业务异常
│   │   ├── infra/             # LLM、Qdrant、分块、进程内会话
│   │   └── main.py            # FastAPI 入口
│   ├── tests/
│   └── pyproject.toml
├── frontend/
│   ├── src/main.py            # Streamlit 入口
│   ├── tests/
│   └── pyproject.toml
├── LICENSE.md
└── README.md
```

## 快速开始

需要 **Python 3.12+** 和 [uv](https://docs.astral.sh/uv/)。

### 1. 克隆仓库并准备环境变量

```bash
git clone https://github.com/cacohe/rag-chatter.git
cd rag-chatter
cp backend/.env.example backend/.env.local
cp frontend/.env.example frontend/.env.local
```

Windows PowerShell 可用 `Copy-Item backend/.env.example backend/.env.local`。

后端至少配置：

1. [DashScope](https://dashscope.console.aliyun.com/)：`DASHSCOPE_API_KEY`、`DEFAULT_LLM`（须是代码里已注册的短名，默认 `qwen3.7-max`）
2. [Qdrant Cloud](https://cloud.qdrant.io/)：`QDRANT_MODE=cloud`、`QDRANT_URL`、`QDRANT_API_KEY`

前端配置 `BACKEND_API_URL`（本地默认 `http://localhost:8000/api/v1.0`）。

### 2. 安装依赖

```bash
cd backend && uv sync --all-groups
cd ../frontend && uv sync --all-groups
```

### 3. 启动

两个终端分别执行：

```bash
# 后端（在 backend/ 目录）
PYTHONPATH=src uv run python -m main

# 前端（在 frontend/ 目录）
PYTHONPATH=src uv run streamlit run src/main.py
```

打开 Streamlit 提示的本地地址。先上传一份文档入库，再提问，确认回答下方出现参考来源。

## 常用命令

在各自目录、且已 `uv sync --all-groups` 之后：

```bash
# backend/
PYTHONPATH=src uv run python -m main
uv run ruff check src tests --config pyproject.toml
uv run ruff format --check src tests --config pyproject.toml
uv run pytest tests/unittest/test_backend -q

# frontend/
PYTHONPATH=src uv run streamlit run src/main.py
uv run ruff check src tests --config pyproject.toml
uv run ruff format --check src tests --config pyproject.toml
uv run pytest tests/unittest/test_frontend -q
```

## CI（GitHub Actions）

配置文件：`.github/workflows/ci.yml`

**触发条件**：向 `master` / `main` 分支 push 或发起 Pull Request。

| 作业 | 步骤 | 说明 |
| --- | --- | --- |
| backend | `uv sync --frozen --all-groups`，Ruff，Pytest | 使用占位 `DASHSCOPE_API_KEY` |
| frontend | 同上 | `BACKEND_API_URL` 指向本地占位地址 |

CI **不会**连接 Qdrant Cloud，也**不会**调用真实模型。

## 部署（Render）

仓库里不保留 Docker / `render.yaml`。线上按控制台手动配置两个 Web Service：

1. **后端**
   - Build：`cd backend && uv sync`
   - Start：`cd backend && PYTHONPATH=src uv run python -m main`
   - 环境变量见 `backend/.env.example`。生产保持 `RELOAD=false`；云平台 `PORT` 优先生效。探活：`GET /health`

2. **前端**
   - Build：`cd frontend && uv sync`
   - Start：`cd frontend && PYTHONPATH=src uv run streamlit run src/main.py`
   - `BACKEND_API_URL` 填后端源站的 `/api/v1.0`（不要末尾 `/`）

知识向量在 Qdrant Cloud，不随 Render 实例重启丢失。对话历史在后端进程内存里，重启后清空。

## 范围与限制

- 无登录：知识库是整站共用的一份 collection，任何人打开页面都能看、能改。
- 对话只存在当前后端进程内存；刷新页面或换实例不会从磁盘恢复。
- 网页导入只允许公网 `http/https`，拒绝本机/私网且不跟随跳转。
- 数据库同步只执行单条 `SELECT`，并限制行数与单行长度。
- CORS 允许任意来源，适合 Demo，不适合直接当生产系统。

## 前后端 API

| 接口 | 方法 | 说明 |
| --- | --- | --- |
| /api/v1.0/knowledge/upload | POST | 上传文件入库 |
| /api/v1.0/knowledge/ingest/web | POST | 导入网页 |
| /api/v1.0/knowledge/sync/database | POST | 数据库 SELECT 同步 |
| /api/v1.0/knowledge/summary | GET | 文档与分块摘要 |
| /api/v1.0/knowledge/chunks | GET | 分块预览 |
| /api/v1.0/knowledge/documents | DELETE | 按文档名删除 |
| /api/v1.0/chat/stream | POST | SSE 流式问答 |
| /api/v1.0/chat/messages | GET | 当前会话展示历史 |
| /api/v1.0/chat/new | POST | 清空当前会话 |
| /health | GET | 进程存活 |

## License

[MIT](LICENSE.md)
