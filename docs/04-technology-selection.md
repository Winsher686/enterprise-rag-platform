# 04 · 技术选型与方案对比

> 所属项目：项目一 —— 企业知识库与智能业务服务平台
> 文档定位：第 1 章交付物之一，明确「选什么技术」「为什么选」「最终技术栈」。

---

## 1. 选型原则

本项目不为堆技术栈而选型，而是**根据业务需求选择技术**。核心取舍依据：Python 技术栈、异步能力、规模化向量检索、中文企业知识库场景。

---

## 2. 后端框架

| 方案 | 优点 | 本项目选择 |
| --- | --- | --- |
| Flask | 简单、灵活 | 不选 |
| Django | 自带 ORM、Admin | 不选 |
| FastAPI | 异步支持、类型校验、API 文档 | **采用** |

选择 FastAPI 的原因：Python 技术栈、支持异步接口、与 Pydantic 集成、方便构建模型服务 API、自动生成 OpenAPI 文档。

---

## 3. 向量数据库

| 方案 | 特点 | 本项目选择 |
| --- | --- | --- |
| FAISS | 本地向量检索库 | 不作为主数据库 |
| Chroma | 开发和原型方便 | 不选 |
| Milvus | 独立向量数据库，支持规模化向量检索 | **采用** |
| pgvector | 与 PostgreSQL 集成简单 | 可作为替代方案 |

采用 Milvus 是为了让向量检索模块独立于业务数据库，便于后续调整向量索引和检索参数。

---

## 4. Embedding 模型

采用 **BGE-M3**。理由：
- 支持多语言文本表示；
- 支持稠密向量检索；
- 支持稀疏检索相关能力；
- 适配中文企业知识库场景。

> 注意：BGE-M3 有稀疏检索能力，不意味着必须同时用它和 Elasticsearch。第一版采用 **BGE-M3 Dense + BM25** 即可。

---

## 5. Reranker

采用 **BGE Reranker**。Embedding 负责快速召回候选，Reranker 负责对候选与问题做更精细的相关性判断。

检索流程：

```
用户问题 → Embedding/BM25 → 召回 Top 20~50 → RRF 融合 → Reranker → 选 Top 3~8 → 构建 Prompt
```

> 实际 Top-K 数值需通过评估集确定，不是固定标准。

---

## 6. 关键词检索

第一版采用轻量 **BM25**，暂不引入 Elasticsearch。确需独立关键词检索服务时再引入，避免一开始增加部署成本。

---

## 7. 最终技术栈

| 分类 | 技术 |
| --- | --- |
| 开发语言 | Python |
| Web 框架 | FastAPI |
| ORM | SQLAlchemy |
| 数据库 | PostgreSQL |
| 缓存与任务队列 | Redis |
| 异步任务 | Celery |
| 向量数据库 | Milvus |
| Embedding | BGE-M3 |
| Reranker | BGE Reranker |
| 关键词检索 | BM25 |
| LLM | Qwen / DeepSeek |
| 文档解析 | PyMuPDF、python-docx、openpyxl |
| OCR | PaddleOCR |
| 知识图谱 | Neo4j（后续扩展） |
| 评估 | RAGAS |
| 监控追踪 | LangFuse |
| 测试 | Pytest |
| 性能测试 | Locust |
| 部署 | Docker Compose |

---

## 8. L3 工程设计要点（提前考虑）

| 问题 | 设计对策 |
| --- | --- |
| 文档处理失败 | 任务状态机（PENDING→PROCESSING→SUCCESS/FAILED→RETRYING）、错误记录、可重试 |
| 数据不一致 | Chunk 索引状态、批次写入记录、失败补偿、索引完整性校验 |
| 权限越权 | 后端 RBAC + 检索阶段过滤，配套自动化越权测试 |
| 模型不可用 | 统一封装：超时、有限重试、指数退避、错误分类、并发限制 |
| 可观测性 | 请求 ID 全链路追踪（LangFuse） |

---

## 9. 验收标准

本章以「能否独立解释和完成设计」为验收标准，需能回答：

1. 为什么企业知识库需要 RAG，而不是直接使用大模型？
2. 为什么文档处理和用户问答要分成两条链路？
3. 为什么同时使用 PostgreSQL 和 Milvus？
4. 为什么文档解析要放到 Celery 异步任务中？
5. 为什么需要 BM25 + 向量检索 + RRF + Reranker？
6. 为什么用户权限必须在检索阶段执行？
7. PostgreSQL 与 Milvus 数据不一致时，如何恢复？
8. 为什么本项目选择模块化单体，而不是微服务？
9. 如何证明系统检索质量达到了预期？
10. 一个 RAG 系统从 Demo 升级到 L3，最重要的工程变化是什么？

---

## 10. 选型小结

| 决策项 | 结论 |
| --- | --- |
| 框架 | FastAPI（异步 + 类型校验 + 自动文档） |
| 元数据库 | PostgreSQL（唯一业务数据源） |
| 向量库 | Milvus（独立、可规模化） |
| 检索 | BM25 + Dense + RRF + Reranker 混合检索 |
| 模型 | BGE-M3 + BGE Reranker + Qwen/DeepSeek |
| 异步 | Redis + Celery |
| 质量保障 | RAGAS 评估 + LangFuse 追踪 + Pytest/Locust 测试 |
