# 02 · 系统总体架构设计

> 所属项目：项目一 —— 企业知识库与智能业务服务平台
> 文档定位：第 1 章交付物之一，明确「系统长什么样」「分几层」「有哪些核心数据对象」。

---

## 1. 架构设计原则

本项目采用：**模块化单体架构 + 独立基础设施组件**。

**为什么不采用微服务？** 当前规模无需引入复杂微服务治理；模块化单体在保持代码边界清晰的同时，显著降低部署、调试和维护成本。

八条具体原则：

1. 业务模块之间通过明确的 Service 接口调用。
2. 数据库访问统一通过 Repository 层管理。
3. 文档处理与问答请求分离。
4. 耗时任务通过异步 Worker 执行。
5. 大模型、Embedding、Reranker 通过统一接口封装。
6. PostgreSQL 作为业务元数据的主要数据源。
7. Milvus 负责向量检索，不作为业务数据的唯一存储。
8. 权限校验由后端负责，不交给大模型。

---

## 2. 系统总体架构

> 分层架构图见 `docs/diagrams/system-architecture.svg`（另附 `system-architecture.mmd` Mermaid 源文件）。

```
                         ┌──────────────────────────────────────────────┐
                         │                 接入层 / 网关                 │
                         │              FastAPI API Gateway             │
                         └──────────────────────────────────────────────┘
                                               │
        ┌──────────────────┬───────────────────┼───────────────────┬──────────────────┐
        ▼                  ▼                   ▼                   ▼                  ▼
 ┌──────────────┐  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
 │ 认证与权限模块 │  │ 知识库管理模块 │   │  文档管理模块  │   │  问答服务模块  │   │ 异步任务调度  │
 │  JWT / RBAC  │  │              │   │              │   │              │   │  Celery Worker│
 └──────────────┘  └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
        │                  │                   │                   │                  │
        └──────────────────┴───────────────────┴───────────────────┴──────────────────┘
                                               │
          ┌────────────────────────────────────┼────────────────────────────────────┐
          │                                    │                                    │
          ▼                                    ▼                                    ▼
 ┌──────────────────┐              ┌──────────────────────────┐           ┌──────────────────┐
 │    检索层         │              │        模型层             │           │     数据层        │
 │ BM25 / Elasticsearch│            │  BGE-M3 / Qwen / DeepSeek│           │ PostgreSQL(元数据)│
 │  Milvus 向量检索  │              │  BGE Reranker            │           │  Milvus(向量)    │
 │  Reranker 重排    │              │                          │           │  Redis(缓存/队列)│
 └──────────────────┘              └──────────────────────────┘           └──────────────────┘
          │                                                                        │
          └─────────────────────────────┬──────────────────────────────────────────┘
                                        ▼
                          ┌──────────────────────────┐
                          │        可观测层           │
                          │  LangFuse / Python Logging│
                          └──────────────────────────┘
```

> 说明：Elasticsearch 为可选组件。第一版先实现轻量 BM25 检索，确有独立关键词检索需求时再引入，避免一开始就增加部署成本。

---

## 3. 各层职责

| 架构层 | 主要组件 | 职责 |
| --- | --- | --- |
| 接入层 | FastAPI | 接收 HTTP 请求、参数校验 |
| 认证层 | JWT、RBAC | 用户身份和数据权限验证 |
| 业务层 | Knowledge、Document、QA Service | 执行业务逻辑 |
| 任务层 | Celery、Redis | 文档异步处理 |
| 检索层 | BM25、Milvus、Reranker | 文档召回和排序 |
| 模型层 | BGE-M3、Qwen/DeepSeek | 向量化和答案生成 |
| 数据层 | PostgreSQL、Milvus | 业务元数据和向量数据 |
| 可观测层 | LangFuse、Python Logging | 调用追踪、日志和指标 |

---

## 4. 为什么同时使用 PostgreSQL 和 Milvus？

两者负责的数据类型不同：

**PostgreSQL 保存（业务元数据）**：
- 用户信息、知识库信息、文档信息、文档版本、Chunk 元数据、对话历史、任务状态、操作日志。

**Milvus 保存（向量数据）**：
- Chunk ID、Embedding 向量、检索必要元数据。

> Milvus 不负责管理整个业务生命周期。例如删除文档时，后端需同时处理 PostgreSQL 的文档记录与 Milvus 的向量记录；此过程可能失败，故必须设计索引一致性管理机制（详见 `03-business-flow.md`）。

---

## 5. 核心数据对象

### 5.1 核心实体

| 实体 | 说明 | 关键字段 |
| --- | --- | --- |
| User | 用户 | id、username、department_id |
| Department | 部门 | id、name |
| Role | 角色 | id、name |
| KnowledgeBase | 知识库 | id、name、owner_id |
| KBPermission | 知识库授权 | user_id、kb_id、permission |
| Document | 文档 | id、kb_id、name、status |
| DocumentVersion | 文档版本 | id、document_id、version |
| Chunk | 文档切片 | id、document_id、version、content |
| ProcessingTask | 处理任务 | id、document_id、status |
| Conversation | 对话 | id、user_id、title |
| Message | 消息 | id、conversation_id、role、content |
| RetrievalLog | 检索记录 | id、query、retrieval_result |
| OperationLog | 操作日志 | id、user_id、operation、created_at |

### 5.2 实体关系（ER 图）

实体关系图见 `docs/diagrams/er-diagram.svg`（另附 `er-diagram.mmd` Mermaid 源文件），关系要点：

- DEPARTMENT 包含（contains）USER
- USER 被分配（assigned）USER_ROLE，USER_ROLE 关联 ROLE
- USER 创建（creates）KNOWLEDGE_BASE
- KNOWLEDGE_BASE 授权（grants）KB_PERMISSION，KB_PERMISSION 关联 USER（receives）
- KNOWLEDGE_BASE 包含（contains）DOCUMENT
- DOCUMENT 拥有（has）DOCUMENT_VERSION，DOCUMENT_VERSION 产生（produces）CHUNK
- DOCUMENT 被处理（processes）生成 PROCESSING_TASK
- USER 拥有（owns）CONVERSATION，CONVERSATION 包含（contains）MESSAGE

### 5.3 文档版本与 Chunk 的关系

```
差旅费用管理办法.docx
    │
    ├── Version 1
    │      ├── Chunk 1
    │      └── Chunk 2
    │
    └── Version 2
           ├── Chunk 1
           ├── Chunk 2
           └── Chunk 3
```

文档更新后不能简单写入新 Chunk 就结束，必须明确：
- 哪个版本是当前有效版本？
- 旧版本是否保留？
- 检索时是否只允许使用当前版本？
- 新版本索引失败时，旧版本是否继续提供服务？
- 删除文档时，历史版本如何处理？

> 这些问题的落地实现见第 6 章。

---

## 6. 项目代码目录与职责

```
enterprise-rag-platform/
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── core/            # config.py / security.py / exceptions.py / logging.py
│   │   ├── api/             # deps.py + v1/(auth|knowledge_base|documents|chat).py
│   │   ├── models/          # SQLAlchemy 数据模型
│   │   ├── schemas/         # Pydantic 请求/响应模型
│   │   ├── repositories/    # 数据库访问层
│   │   ├── services/        # 业务逻辑层
│   │   ├── rag/             # RAG 核心（embeddings/parsers/chunking/retrieval/reranking/generation）
│   │   ├── tasks/           # Celery 异步任务
│   │   ├── integrations/    # 外部组件适配（milvus/redis/llm）
│   │   └── utils/
│   ├── migrations/
│   ├── tests/
│   └── requirements.txt
├── docs/
├── docker/
├── scripts/
├── docker-compose.yml
├── .env.example
└── README.md
```

**目录职责**：

| 目录 | 职责 |
| --- | --- |
| api | HTTP 接口层 |
| services | 业务逻辑层 |
| repositories | 数据库访问层 |
| models | SQLAlchemy 数据模型 |
| schemas | Pydantic 请求和响应模型 |
| rag | RAG 核心业务逻辑 |
| tasks | Celery 异步任务 |
| integrations | 外部组件适配层 |
| tests | 自动化测试 |

**为什么单独拆出 `rag`？** RAG 不应全部写在 FastAPI 路由函数里。问答流程应由 RAGService 负责，而不是在 `chat.py` 里直接完成向量检索、Reranker 调用和 LLM 生成 —— 这样才能独立测试和优化检索链路。

---

## 7. 架构小结

| 决策 | 结论 |
| --- | --- |
| 架构风格 | 模块化单体 + 独立基础设施 |
| 元数据存储 | PostgreSQL（唯一业务数据源） |
| 向量存储 | Milvus（仅向量检索） |
| 权限边界 | 后端 RBAC，检索阶段过滤 |
| 异步边界 | 耗时任务走 Celery，HTTP 快速返回 task_id |
| 代码分层 | api / services / repositories / rag 清晰分层 |
