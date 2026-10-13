# AI PPT Generator

一套以 **AI 工程化 + 文档渲染 + 异步任务编排** 为核心的全栈项目，基于 FastAPI + LangChain + LangGraph + python-pptx + React 开发。

用户只需要输入一句主题、粘贴一段长文本，或者上传 PDF、Word、Markdown、TXT 文档，就能先得到一份可修改的大纲，再并发生成完整页面，最终导出**原生可编辑的 PPTX 文件**。

## 项目特点

### 1）多种输入生成大纲

支持从主题、长文本和文档三种入口创建 PPT。用户可以控制页数、排版模式、文字量、语气和目标受众。

模型不会直接越过用户生成最终页面，而是先产出结构化大纲，标题、页面目标和要点都能修改，也可以增删页面、调整顺序，再决定是否开始生成。

### 2）异步并发生成

一份 10 页 PPT 如果串行调用模型，很容易让请求等待一分钟以上。项目把每一页拆成独立任务，通过 ARQ Worker 并发执行，用 Redis 保存任务状态并传递事件，前端通过 SSE 实时接收进度。

生成过程中，已经完成的页面可以立即预览，失败页面可以单独重试，用户也可以软取消剩余任务，不需要整份重来。

### 3）在线编辑

生成后的内容会回到结构化模型里编辑，编辑器左侧是页面缩略图，中间是 16:9 画布，右侧提供 AI 修改、主题、版式和图片面板。

文字可以直接编辑；灵活排版页面支持拖动分隔线、调整内容块尺寸、跨容器移动和更换排布；主题切换后，Web 预览和 PPTX 导出会读取同一份主题数据。

### 4）原生 PPTX 导出

AI 输出的文字长度不可预测，直接导出很容易出现溢出、缺页或整页被渲染成图片的问题。项目在导出前检查结构、文字占用、数字来源和页面边界，把问题分成 error 和 warning：error 阻断导出，warning 提醒用户核对但允许继续。

通过检查后，后端用 python-pptx 逐块构造文本框、表格、图表和图片，再回读生成的文件进行验证。这样得到的 PPTX 不是一张铺满全页的截图，而是能继续修改的原生对象。

## 功能模块

项目涵盖用户项目管理、输入解析、大纲生成、页面生成、在线编辑、主题布局、图片管线、导出质量检查 8 大模块。

## 系统架构

本项目采用前后端分离 + 异步 Worker 架构。前端是 React + TypeScript SPA，后端是 FastAPI + ARQ Worker 服务，通过 REST API + SSE 通信。

后端内部按照 API 层、领域层、工作流层、渲染层分层，耗时的 AI 任务通过 ARQ 异步队列和 Redis pub/sub 来管理。内容、布局、主题三种数据通过 `shared/` 目录下的 JSON 文件前后端物理共享。

核心业务流程：注册登录 → 创建项目 → 输入主题或材料 → 生成并确认大纲 → 并发生成页面 → 在线编辑 → 质量检查 → 导出 PPTX。

## 技术栈

### 后端（Python）

| 方向 | 技术 |
| --- | --- |
| Web 框架 | FastAPI + Uvicorn |
| 数据校验 | Pydantic v2 + pydantic-settings |
| 数据库 | PostgreSQL + SQLAlchemy 2.x（async）+ Alembic |
| 异步任务 | ARQ + Redis pub/sub |
| AI 编排 | LangChain Core / langchain-openai / LangGraph |
| 文档解析 | pdfplumber、python-docx、markdown-it-py |
| 文档渲染 | python-pptx |
| 文字度量 | FontTools |
| 认证 | PyJWT + pwdlib[argon2] |
| 对象存储 | 本地存储 / 腾讯云 COS |

### 前端

| 方向 | 技术 |
| --- | --- |
| 框架 | React 19 + TypeScript |
| 构建 | Vite |
| 路由 | React Router |
| 数据请求 | TanStack Query |
| 状态管理 | Zustand |
| 样式 | Tailwind CSS |
| 图表 | Recharts |
| 接口类型 | openapi-typescript（由后端 OpenAPI 生成） |

### Java 后端（可选）

`java_backend/` 是用 Spring Boot 复刻 Python 后端 API 与 Worker 的实现，共用同一套数据库、Redis 与 SSE 事件契约，便于按自身技术栈选择运行其中一套。

| 方向 | 技术 |
| --- | --- |
| Web 框架 | Spring Boot 3.5 + Java 21 虚拟线程 |
| ORM | MyBatis-Plus |
| AI 编排 | LangChain4j + LangGraph4j |
| 文档解析 | Apache PDFBox + Apache POI |
| 认证 | JJWT + argon2-jvm |

## 目录结构

```
.
├── backend/            # Python 后端（FastAPI + ARQ Worker）
│   ├── app/
│   │   ├── api/        # API 层：路由、SSE、依赖注入
│   │   ├── core/       # 配置、数据库、Redis、安全
│   │   ├── domain/     # 领域层：内容模型、布局、几何、flex 求解、校验
│   │   ├── images/     # 图片管线：生图 / 图库 / 校验
│   │   ├── ingest/     # 文档解析：PDF / DOCX / Markdown / 纯文本
│   │   ├── llm/        # 模型调用、提示词、改稿工具
│   │   ├── models/     # ORM 模型
│   │   ├── render/     # PPTX 渲染与回读验证
│   │   ├── schemas/    # 请求 / 响应模型
│   │   ├── services/   # 业务服务
│   │   ├── worker/     # ARQ 任务与重试策略
│   │   └── workflows/  # LangGraph 工作流
│   ├── alembic/        # 数据库迁移
│   ├── regression/     # 固定语料回归集
│   └── tests/          # 测试
├── frontend/           # React SPA
│   └── src/
│       ├── api/        # 请求出口与 OpenAPI 生成的类型
│       ├── features/   # 按领域组织：deck / outline / projects / auth
│       ├── render/     # Web 侧渲染器（与 PPTX 渲染同源）
│       └── pages/      # 页面
├── java_backend/       # Spring Boot 后端（可选替代实现）
└── shared/             # 前后端共享数据：主题 / 布局 / flex 预设
```

## 快速开始

### 环境要求

- Python >= 3.12
- Node.js >= 18
- Docker（用于启动 PostgreSQL 与 Redis）
- [uv](https://github.com/astral-sh/uv)（Python 包管理）

### 1. 启动基础设施

```bash
docker compose up -d
```

统一使用 39xxx 端口段，避开本机已装的 PostgreSQL / Redis 默认端口：

- PostgreSQL：`39432`
- Redis：`39379`

### 2. 安装依赖

```bash
make install
```

或分别执行：

```bash
cd backend && uv sync
cd frontend && npm install
```

### 3. 下载度量字体

文字溢出检测需要度量字体（本地下载，不进仓库）：

```bash
make fonts
```

### 4. 配置环境变量

复制 `backend/.env.example` 为 `backend/.env`，至少填写模型凭证：

```bash
LLM_API_KEY=你的模型 API Key
LLM_BASE_URL=https://api.deepseek.com
LLM_MODEL=deepseek-v4-flash
```

图片能力可选配，不配置时会降级使用占位图：

```bash
IMAGE_PROVIDER=openai
IMAGE_API_KEY=
UNSPLASH_ACCESS_KEY=
```

### 5. 执行数据库迁移

```bash
make migrate
```

### 6. 启动服务

需要同时启动 API 与 Worker：

```bash
make dev-api      # 后端 API，监听 127.0.0.1:39800
make dev-worker   # ARQ Worker
make dev-web      # 前端开发服务器
```

## 常用命令

| 命令 | 说明 |
| --- | --- |
| `make up` / `make down` | 启动 / 停止基础设施容器 |
| `make install` | 安装前后端依赖 |
| `make fonts` | 下载文字度量字体 |
| `make migrate` | 执行数据库迁移 |
| `make migration m="描述"` | 生成新的迁移脚本 |
| `make dev-api` | 启动后端 API |
| `make dev-worker` | 启动 ARQ Worker |
| `make dev-web` | 启动前端开发服务器 |
| `make gen-api` | 由后端 OpenAPI 生成前端 TS 类型 |
| `make test` | 运行后端测试 |
| `make regression` | 运行固定语料回归集 |
| `make lint` | 代码检查 |

## 关键设计

### 结构化输出与硬约束

单页生成的槽位与容量上限直接写进提示词：布局是硬约束，与其事后裁剪模型写超的内容，不如一开始就把可用空间告诉它。模型输出后再校验槽位合法性、类型匹配与必填完整性，避免非法数据入库后靠渲染兜底。

### 自纠正工作流

单页生成使用 LangGraph 构建 `prepare → generate → check →（必要时）repair` 的回路。校验放进图里而不是调用方，是为了让修复轮次能拿到结构问题，形成自纠正回路而不是简单重试。只修复一轮：结构错误或内容过瘦才触发，溢出与容量告警不触发砍块重写。

### 页级并发与软取消

每一页拆成独立任务并发执行，用 `asyncio.Semaphore` 控制模型调用频率。取消标记只在每页开始前生效——正在执行的那一页让它跑完，中途丢弃既浪费已花费的调用，也会留下半截状态。

跨页的多样性（配图、版式骨架、callout 配额）必须提前分配，因为各页并发生成时互相看不见。

### 双端同源渲染

内容、布局、主题三种数据通过 `shared/` 目录下的 JSON 文件前后端物理共享，Web 预览与 PPTX 导出读取同一份主题与布局数据，两端不会因各自演进导致排版分叉。

### 导出质量门禁

导出前检查结构、文字占用、数字来源与页面边界，问题分为 error 与 warning：error 阻断导出，warning 提醒但允许继续。渲染完成后回读生成的文件进行验证，检查页数、画布尺寸、原生对象数量与内容完整性——包括检测是否存在覆盖整页的图片（疑似用截图承载正文）。验证未通过则不返回文件。

### 编辑并发控制

同一页的内容、样式、灵活布局结构保存必须串行：每次成功 revision +1，并发必然冲突。前端用按页串行的保存队列配合 revision 乐观锁处理编辑冲突，并维护本页的撤销/重做栈。

### 接口类型同源

前端 TypeScript 类型由后端 OpenAPI 自动生成，消灭前后端接口漂移。

## 测试与回归

```bash
make test         # 单元测试与接口测试
make regression   # 固定回归集：语料 × 3 主题
make lint         # ruff 代码检查
```

回归集门禁指标：可编辑性 100% 且溢出槽位比例 < 5%。

## 许可证

本项目仅供学习与技术交流使用。
