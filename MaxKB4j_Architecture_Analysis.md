# MaxKB4j 架构与功能实现分析报告

MaxKB4j 是一个基于 Java 21 和 Spring Boot 3 构建的企业级智能问答系统，核心集成了 RAG（检索增强生成）与可视化 LLM 工作流引擎。本文档详细梳理了该项目核心模块的代码实现原理。

---

## 1. 知识库与 RAG 实现 (`maxkb4j-knowledge`)
该模块负责文档的解析、切分、向量化以及混合检索，是构建企业私有知识库的核心。

### 1.1 文档解析 (`DocumentParser`)
系统定义了统一的 `DocumentParser` 接口，通过策略模式根据文件扩展名动态匹配解析器：
- **常规文档**：支持 Word、Excel、TXT、HTML、Markdown、CSV、PPT 等格式的解析。
- **PDF 与 OCR**：`PdfParser` 基于 `PDFBox` 提取文本，并具备智能排版识别（合并连续且同字体字号的文本行）。针对扫描版 PDF，系统会自动进行文本密度检测，若判定为扫描件，则逐页渲染为图像并调用内置的 OCR 引擎提取文本。

### 1.2 文本切分 (`DocumentSplitService`)
为保证检索精度，长文本在向量化前会进行结构化切分（Chunking）：
- **层级切分**：默认按 Markdown 标题层级（`#` 到 `######`）进行递归切分。
- **降级切分**：若切分后的 Chunk 仍超过长度限制（默认 512 Token），则降级使用句子级（`SentenceSplitter`）或按行切分。
- **格式保护**：通过正则表达式识别 Markdown 表格等特殊结构，确保其作为一个完整的 Chunk 被保留，避免被强制截断而破坏语义。

### 1.3 向量化与双写存储 (`VectorStoreImpl` & `FullTextStoreImpl`)
系统采用了**双写存储架构 (Dual-Write)**，支持多种检索模式：
- **向量存储**：底层基于 PostgreSQL 的 `pgvector` 插件。利用 LangChain4j 的 `EmbeddingModel` 批量向量化文本，并包含批处理与自动重试机制。
- **全文存储**：底层基于 MongoDB。文本入库前经 `Tokenizer`（基于 Jieba 分词器）进行中文分词，存入 MongoDB 供倒排索引检索。
- **组合存储 (`CompositeStoreImpl`)**：作为门面服务，在数据的增删改查时同步调度上述两种存储，确保数据一致性。

### 1.4 多模式检索 (`DataRetriever` & `RetrieveService`)
支持三种检索模式：
- **向量检索**：将 Query 向量化后，通过 PostgreSQL 进行余弦相似度（ANN）查询。
- **全文检索**：对 Query 分词后，利用 MongoDB 的聚合管道与 `$meta: 'textScore'` 进行打分匹配。
- **混合检索 (Hybrid)**：并发查询 PostgreSQL 和 MongoDB。若同一段落被两者命中，取最高分作为最终得分，随后按分数降序截取 Top K。最终由 `RetrieveService` 回表查询完整段落文本，组装 Prompt。

---

## 2. 大模型集成与路由 (`maxkb4j-model`)
该模块作为模型工厂，深度依赖 LangChain4j 并做了丰富扩展，负责管理各类 AI 模型的生命周期与调用。

### 2.1 LangChain4j 深度集成与 Provider 架构
- **模板方法与策略模式**：所有模型提供商继承自基类 `AbsModelProvider`，该基类封装了通用的 HTTP Client 构建与参数解析逻辑。通过 `ModelProvider` 枚举注册了 OpenAI、阿里云百炼、Ollama、DeepSeek 等十余种国内外模型。
- **动态构建**：`ModelProviderServiceImpl` 根据配置动态构建 LangChain4j 的 `ChatModel`、`StreamingChatModel` 等实例。
- **优雅降级**：对于不支持的模型类型，系统默认返回 `DisabledChatModel`，避免空指针异常。

### 2.2 多模态模型支持
- **视觉理解**：通过给 `ChatModel` 传入特定配置启用多模态支持（如 `qwen-vl`）。
- **图像生成**：实现 LangChain4j 的 `ImageModel` 接口，采用适配器模式处理不同厂商 API（如通义万相、DashScope API）。
- **语音处理 (TTS & STT)**：自定义了 `STTModel` 和 `TTSModel` 接口，直接集成 OpenAI SDK 或阿里云原生 SDK 处理语音转文本与文本转语音请求。

---

## 3. 可视化工作流引擎 (`maxkb4j-workflow`)
工作流引擎用于编排复杂的 AI 任务，采用高度模块化和基于 DAG（有向无环图）的设计。

### 3.1 节点处理器 (NodeHandler) 注册
- **自动扫描与注册**：`NodeHandlerAutoRegistrar` 在 Spring 启动时扫描带有 `@NodeHandlerType` 注解的 Bean，并动态注册到 `NodeHandlerRegistry` 中。
- **统一生命周期**：所有节点处理器继承自 `AbsNodeHandler`，通过模板方法定义了严谨的执行生命周期（参数解析 -> 前置处理 -> 核心业务 `doExecute` -> 后置处理 -> 异常兜底）。涵盖了诸如大模型对话、知识库检索、条件分支、代码执行等几十种节点。

### 3.2 DAG 编排与执行 (`WorkFlowActuator`)
- **引擎驱动**：`AbsWorkflowHandler` 负责 DAG 的核心图遍历逻辑，从 `startNode` 递归推进，根据节点执行结果获取后续节点。
- **并发与依赖管理**：多分支执行时使用 `CompletableFuture` 和专用线程池并行处理，并提供节点级超时保护。执行前校验前置依赖，若不满足则标记为 `SKIP` 状态向下传递。
- **执行门面**：`WorkFlowActuator` 作为统一入口，根据工作流类型（如 ChatWorkflow、KnowledgeWorkflow）动态分发给对应的 Handler，并通过责任链（`ExceptionResolverChain`）统一处理异常与 SSE 实时流推送。

---

## 4. 应用与智能体对话流 (`maxkb4j-application`)
该模块处理面向用户的对话逻辑，区分了简单问答应用与复杂工作流应用。

### 4.1 简单应用管道 (`PipelineManage`)
处理单轮/多轮简单问答（`AppType.SIMPLE`）：
- **链式执行**：`PipelineManage` 按顺序执行 `AbsStep` 列表（包含重写问题、检索数据集、生成 Prompt 和模型对话）。
- **拦截与直出 (`AbsChatStep`)**：若知识库命中率极高且配置了“直接返回”，可短路拦截大模型调用，直接组装文本返回。
- **流式推送 (`ChatStep`)**：利用 Reactor 的 `Sinks.Many` 处理 SSE 实时流，通过 LangChain4j 的 `TokenStream` 异步推送深度思考内容、工具执行结果与普通文本。

### 4.2 多智能体工作流 (`ChatFlowServiceImpl`)
处理复杂的多智能体与业务编排逻辑：
- **图解析与构建**：将前端可视化的 JSON 配置反序列化为 DAG 图。利用 `NodeBuilder` 动态实例化各个节点（Agent、工具、条件分支等）。
- **复杂流转**：将构建好的 Workflow 对象委托给 `WorkFlowActuator` 执行。大模型的输出可作为下一个 Agent 节点的输入，原生支持多智能体串联协作与路由分发。

---

## 5. 外部工具接入与系统基础 (`maxkb4j-tool` & `maxkb4j-system`)

### 5.1 MCP 协议与 Skills 工具集成
- **MCP 协议支持**：基于 `dev.langchain4j.mcp`，支持 SSE 与 HTTP 两种通信模式。`McpToolUtil` 可实时拉取远程 MCP Server 的工具列表，`ToolProviderService` 负责将其转换为系统内部的 `McpToolVO` 及 LangChain4j 的 `ToolSpecification`，供底层 Agent 调用。
- **Skills 技能引擎**：支持本地 Shell 或文件系统的独立技能包。提供安全的 ZIP 解压（防目录穿越），通过 Jsoup 解析配置，并将初始参数动态注入环境变量供本地大模型助手推理执行，执行完毕后隔离清理。

### 5.2 权限控制与安全 (`Sa-Token`)
- **认证与细粒度鉴权**：全面集成 Sa-Token。自定义 `StpInterfaceImpl` 从数据库加载角色与权限。
- **数据隔离**：除全局权限外，支持基于工作空间 (Workspace) 和目标资源 ID (TargetId) 的细粒度权限管控。
- **注解拦截**：在 Controller 层大量使用 `@SaCheckRole` 和 `@SaCheckPermission` 进行拦截校验，保证接口调用的安全性。