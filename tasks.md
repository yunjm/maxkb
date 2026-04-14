# 全场景AI日志分析平台 - RAG功能开发 Tasks

## Phase 1: 数据库与基础设施搭建
- [ ] Task 1.1: 执行 SQL 脚本，创建 MySQL 关系表 (`ai_knowledge_base`, `ai_kb_document`, `ai_kb_segment`, `ai_agent_kb_relation`)。
- [ ] Task 1.2: 部署并初始化向量数据库（推荐 Milvus 或 PostgreSQL+pgvector），创建对应的 Collection/Table 及向量索引。
- [ ] Task 1.3: 在 Spring Boot `pom.xml` 中引入 `langchain4j`, `langchain4j-document-parser-apache-pdfbox`, 及对应的向量库 SDK。
- [ ] Task 1.4: 在 `application.yml` 中配置向量数据库连接信息及 Embedding 模型的 API Key。

## Phase 2: 后端核心业务开发 (Spring Boot 4)
- [ ] Task 2.1: 使用若依代码生成器，生成 MySQL 四张表的基础 Entity, Mapper, Service, Controller 代码。
- [ ] Task 2.2: 实现 `IDocumentPipelineService`，编写文档读取、文本切分（RecursiveCharacterTextSplitter）逻辑。
- [ ] Task 2.3: 实现 Embedding 调用逻辑，对接向量数据库完成文本向量的入库操作。
- [ ] Task 2.4: 将文档解析、向量化过程封装为异步任务（或使用消息队列），并在任务节点更新 `ai_kb_document` 的状态。
- [ ] Task 2.5: 实现 `IRetrievalService`，编写根据 Query 生成向量并在向量库中召回 Top-K 的查询接口。
- [ ] Task 2.6: 实现命中测试 API `/ai/kb/hit-test`，返回召回的分段文本与相似度得分。
- [ ] Task 2.7: 重构现有的 `Agent` 增删改查接口，支持关联关系的级联保存与更新。
- [ ] Task 2.8: 重构大模型对话接口：在提问前拦截，调用 `IRetrievalService` 获取上下文，拼接 RAG Prompt 后请求大模型，并将引用数据随响应返回。

## Phase 3: 前端业务开发 (Vue3 + Element Plus)
- [ ] Task 3.1: 路由与菜单配置：在若依管理后台添加“知识库管理”及其子页面的路由和菜单。
- [ ] Task 3.2: 开发 `views/ai/kb/index.vue`（知识库列表页）。
- [ ] Task 3.3: 开发 `views/ai/kb/dataset.vue`（文档列表页），集成带进度条和切分参数配置的上传组件。
- [ ] Task 3.4: 开发 `views/ai/kb/segment.vue`（分段详情页），支持分段内容的查看与手动编辑更新。
- [ ] Task 3.5: 开发 `views/ai/kb/hitTest.vue`（命中测试页），实现输入文本与展示召回结果卡片的交互。
- [ ] Task 3.6: 修改 `views/ai/agent/index.vue`（智能体管理页），增加知识库多选下拉框及 RAG 参数（阈值、TopK）配置表单。
- [ ] Task 3.7: 修改 `views/ai/chat/index.vue`（对话分析控制台），在 AI 聊天气泡UI中渲染“引用来源”组件。

## Phase 4: 联调与测试验证
- [ ] Task 4.1: 全链路联调：上传测试运维文档 -> 解析成功 -> 命中测试校验准确度。
- [ ] Task 4.2: 智能体绑定知识库进行对话测试，验证日志排查问题是否优先基于知识库回答。
- [ ] Task 4.3: 性能优化：针对长文档上传解析过程增加超时控制及失败重试机制。