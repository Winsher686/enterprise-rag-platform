# 企业知识库与智能业务服务平台（enterprise-rag-platform）

> 面向企业内部知识管理和业务咨询场景的一体化 RAG 平台。
> 提供文档接入、知识处理、混合检索、智能问答、权限控制、版本管理和效果评估能力。

## 项目定位

本系统不是简单的 PDF 问答机器人，而是一套具备**知识生命周期管理、检索质量控制、权限隔离、异步任务处理和运行可观测能力**的企业知识服务系统。

第一期聚焦三个核心场景：

| 场景 | 说明 | 典型问题 |
| --- | --- | --- |
| 企业内部知识问答 | 制度、报销流程、审批规范、员工手册 | 差旅报销标准是什么？年假如何申请？ |
| 产品售前咨询 | 型号、参数、功能、适用场景、差异 | A 型号和 B 型号的主要区别？ |
| 售后知识辅助 | 故障手册、维修流程、售后政策、历史案例 | E102 错误码如何排查？ |

## 技术栈

- **语言/框架**：Python · FastAPI · SQLAlchemy
- **数据库**：PostgreSQL（业务元数据）+ Milvus（向量检索）
- **缓存/任务**：Redis + Celery
- **模型**：BGE-M3（Embedding）· BGE Reranker（重排）· Qwen/DeepSeek（生成）
- **检索**：BM25 + Dense Retrieval + RRF 融合 + Reranker
- **文档解析**：PyMuPDF · python-docx · openpyxl · PaddleOCR
- **可观测/评估**：LangFuse · RAGAS
- **测试/压测**：Pytest · Locust
- **部署**：Docker Compose

## 目录结构

```
enterprise-rag-platform/
├── backend/
│   ├── app/
│   │   ├── main.py                 # FastAPI 入口
│   │   ├── core/                   # 配置、安全、异常、日志
│   │   ├── api/                    # HTTP 接口层
│   │   ├── models/                 # SQLAlchemy 数据模型
│   │   ├── schemas/                # Pydantic 请求/响应模型
│   │   ├── repositories/           # 数据库访问层
│   │   ├── services/               # 业务逻辑层
│   │   ├── rag/                    # RAG 核心逻辑（embeddings/parsers/chunking/retrieval/reranking/generation）
│   │   ├── tasks/                  # Celery 异步任务
│   │   ├── integrations/           # 外部组件适配（milvus/redis/llm）
│   │   └── utils/
│   ├── migrations/                 # Alembic 数据库迁移
│   ├── tests/                      # 自动化测试
│   └── requirements.txt
├── docs/                           # 设计文档（见下方清单）
├── docker/                         # Docker 相关配置
├── scripts/                        # 运维/辅助脚本
├── docker-compose.yml
├── .env.example
└── README.md
```

## 设计文档（第 1 章交付物）

| 文档 | 说明 |
| --- | --- |
| `docs/01-requirements-analysis.md` | 项目背景、业务需求、角色与权限、功能/非功能需求 |
| `docs/02-system-architecture.md` | 系统总体架构、分层职责、核心数据对象与实体关系 |
| `docs/03-business-flow.md` | 离线知识构建链路、在线问答链路、L3 可靠性设计 |
| `docs/04-technology-selection.md` | 技术选型与方案对比、最终技术栈、验收标准 |
| `docs/diagrams/` | 架构图、实体关系图、链路流程图（SVG + Mermaid 源文件） |

### 第 1 章交付清单完成状态

| 交付项 | 状态 |
| --- | --- |
| 项目背景与业务需求文档 | ✅ `docs/01-requirements-analysis.md` |
| 用户角色与权限需求文档 | ✅ 同 `01` 第 2 节 |
| 功能需求与非功能需求清单 | ✅ 同 `01` 第 3、4 节 |
| RAG 离线知识构建流程图 | ✅ `docs/diagrams/ingestion-flow.svg` |
| RAG 在线问答流程图 | ✅ `docs/diagrams/qa-flow.svg` |
| 系统总体架构图 | ✅ `docs/diagrams/system-architecture.svg` |
| 核心数据实体关系图 | ✅ `docs/diagrams/er-diagram.svg` |
| 技术选型与方案对比文档 | ✅ `docs/04-technology-selection.md` |
| L3 工程可靠性设计清单 | ✅ `docs/03-business-flow.md` 第 3 节 |
| 项目代码目录结构 | ✅ 本仓库目录树 |

## 快速开始

> 第 2 章将完成环境搭建与代码落地，本章仅完成设计。

```bash
# 1. 复制环境变量
cp .env.example .env

# 2. 启动基础设施（PostgreSQL / Redis / Milvus）
docker compose up -d

# 3. 安装依赖
cd backend && pip install -r requirements.txt

# 4. 启动服务（后续章节实现）
uvicorn app.main:app --reload
```

## 里程碑

1. 业务需求与架构设计（本章）
2. 工程基础与数据模型
3. 文档知识构建
4. RAG 检索与问答
5. 高级知识检索
6. 可靠性与性能
7. 评估、安全与部署
