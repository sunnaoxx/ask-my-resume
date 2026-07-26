# AskMyResume 项目文档

**AskMyResume（与简历对话）**：求职者上传简历 → 解析为多维度个人档案 → 审阅修改后发布 → 生成专属链接 → 面试官免登录与档案进行 RAG 对话。这是一个以**学习 RAG 技术**为核心目的的全栈项目。

## 文档索引与建议阅读顺序

| 文档 | 内容 | 什么时候读 |
|---|---|---|
| [00-blueprint.md](00-blueprint.md) | **项目蓝图**：定位、核心流程、全部关键技术决策（总纲，其他文档以此为准） | 最先读 |
| [01-requirements.md](01-requirements.md) | 需求分析：用户故事、功能清单、非功能需求 | 开工前 |
| [02-tech-stack.md](02-tech-stack.md) | 技术选型：每项选择的理由与备选方案对比 | 开工前 |
| [03-architecture.md](03-architecture.md) | 系统架构：模块划分、核心流程时序、目录结构 | 开工前 |
| [04-data-model.md](04-data-model.md) | 数据模型：表结构、字段、索引、JSONB 内容格式 | M0-M1 |
| [05-rag-pipeline.md](05-rag-pipeline.md) | **RAG 管线设计**（核心学习文档）：解析→抽取→分块→向量化→检索→生成→评估 | M2-M5 全程 |
| [06-api-design.md](06-api-design.md) | API 设计：全部端点的请求/响应约定 | M1 起 |
| [07-frontend.md](07-frontend.md) | 前端设计：页面结构、组件、状态与 SSE 消费 | M2 起 |
| [08-security-privacy.md](08-security-privacy.md) | 安全与隐私：认证、限流、分享链接、prompt injection 防护 | M4 前必读 |
| [09-roadmap.md](09-roadmap.md) | **开发路线图**：M0-M6 里程碑、任务清单、每阶段学习目标 | 全程对照 |
| [10-testing-evaluation.md](10-testing-evaluation.md) | 测试与评估：单测/集成测试 + RAG 质量评估（ragas、golden set） | M3 起 |
| [11-deployment.md](11-deployment.md) | 部署与运维：Docker Compose、环境变量、上线清单 | M0 与 M6 |

## 快速开始（规划期）

1. 通读 `00-blueprint.md` 与 `09-roadmap.md`，了解全貌与节奏。
2. 按 `09-roadmap.md` 的 M0 任务清单初始化工程。
3. 开发中遇到具体问题，查对应专题文档。
