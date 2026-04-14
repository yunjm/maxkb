# 全场景AI日志分析平台 - 知识库与RAG核心功能重构集成 详细Spec

## 1. 背景与目标 (Why)
目前基于若依（Vue3 + Spring Boot 4）的全场景AI日志分析平台已经初步实现了简单的模型管理与智能体管理。为了进一步提升平台在复杂运维场景下的日志分析深度与准确性，需要引入RAG（检索增强生成）能力。
通过参考开源项目 `MaxKB4j`，重构并实现知识库管理、文档解析切分、向量化检索等核心功能。本详细设计文档涵盖了**数据库设计**、**后端架构**、**前端架构**及**核心业务流**，供开发团队直接参考落地。

---

## 2. 数据库详细设计 (Database)
在若依原有的 MySQL 数据库基础上，新增以下业务表（均需包含若依基础字段：`create_by`, `create_time`, `update_by`, `update_time`, `remark`）。同时需引入向量数据库（如 PostgreSQL + pgvector，或 Milvus）。

### 2.1 MySQL 关系型数据表

**表1：知识库主表 (`ai_knowledge_base`)**
| 字段名 | 类型 | 描述 |
| --- | --- | --- |
| `kb_id` | bigint | 主键，知识库ID |
| `kb_name` | varchar(100) | 知识库名称（如：系统报错排查手册） |
| `kb_desc` | varchar(500) | 知识库描述 |
| `embedding_model_id` | bigint | 关联的Embedding模型ID |
| `status` | char(1) | 状态（0正常 1停用） |

**表2：知识库文档表 (`ai_kb_document`)**
| 字段名 | 类型 | 描述 |
| --- | --- | --- |
| `doc_id` | bigint | 主键，文档ID |
| `kb_id` | bigint | 关联的知识库ID |
| `file_name` | varchar(255) | 文件名/日志集名称 |
| `file_type` | varchar(50) | 文件类型（TXT, PDF, MD, DOCX） |
| `file_size` | bigint | 文件大小（字节） |
| `status` | tinyint | 解析状态（0:待处理, 1:解析中, 2:成功, 3:失败） |
| `chunk_size` | int | 分段大小（Token数，如 500） |
| `chunk_overlap` | int | 分段重叠大小（Token数，如 50） |

**表3：文档分段元数据表 (`ai_kb_segment`)** - *可选，用于关系库管理和手动修改切片*
| 字段名 | 类型 | 描述 |
| --- | --- | --- |
| `segment_id` | bigint | 主键，分段ID |
| `doc_id` | bigint | 关联的文档ID |
| `content` | text | 切分后的具体文本内容 |
| `token_count` | int | 该段落的Token数量 |
| `vector_id` | varchar(64) | 对应向量数据库中的主键ID |
| `status` | tinyint | 状态（0正常 1禁用） |

**表4：智能体与知识库关联表 (`ai_agent_kb_relation`)**
| 字段名 | 类型 | 描述 |
| --- | --- | --- |
| `agent_id` | bigint | 联合主键，智能体ID |
| `kb_id` | bigint | 联合主键，知识库ID |
| `similarity_threshold`| float | 检索相似度阈值（如 0.75） |
| `top_k` | int | 最大引用分段数量（如 3） |

### 2.2 向量数据库设计 (Vector DB)
以 **Milvus** 或 **pgvector** 为例：
- **Collection Name**: `kb_vector_collection`
- **Fields**:
  - `id`: Varchar/UUID (主键)
  - `kb_id`: Int64 (用于标量过滤)
  - `doc_id`: Int64 (用于标量过滤)
  - `content`: Text (存储原始文本，可选)
  - `embedding`: FloatVector (维度取决于所选Embedding模型，如 1536, 1024, 768)

---

## 3. 后端详细设计 (Spring Boot 4)

后端建议引入 `LangChain4j` 框架来简化 RAG 流水线的构建。

### 3.1 核心依赖 (pom.xml)
```xml
<!-- LangChain4j 核心及文档加载器 -->
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j</artifactId>
    <version>0.31.0</version>
</dependency>
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-document-parser-apache-pdfbox</artifactId>
</dependency>
<!-- 向量数据库 SDK (以 Milvus 为例) -->
<dependency>
    <groupId>io.milvus</groupId>
    <artifactId>milvus-sdk-java</artifactId>
    <version>2.3.4</version>
</dependency>
```

### 3.2 核心服务接口与 RAG 流水线 (Services)
1. **`IKnowledgeBaseService` & `IKbDocumentService`**
   - 负责 MySQL 中知识库和文档元数据的 CRUD。
2. **`IDocumentPipelineService` (数据处理流水线)**
   - **Load**: 接收用户上传的文件流，调用 `DocumentLoader` 解析文本。
   - **Split**: 使用 `DocumentSplitter` (如 `RecursiveCharacterTextSplitter`)，根据 `chunk_size` 和 `chunk_overlap` 切分文本为 `TextSegment` 列表。
   - **Embed**: 调用大模型 Embedding API（如 OpenAI, 智谱, 阿里通义）将 `TextSegment` 转换为 `Embedding` 向量。
   - **Store**: 将向量数据和元数据（kb_id, doc_id）写入向量数据库；将切片文本写入 MySQL `ai_kb_segment` 表。
   - *注意*：此过程应为**异步任务** (`@Async` 或放入 MQ/Redis 队列)，并在完成后更新 `ai_kb_document` 的 `status`。
3. **`IRetrievalService` (检索服务)**
   - 接收用户的查询文本（Query）。
   - 转换 Query 为向量。
   - 在向量库中根据 `kb_id` 进行过滤，并通过余弦相似度（Cosine Similarity）召回 Top-K 的 `TextSegment`。
4. **`IChatService` (对话增强)**
   - 拦截原有的智能体对话请求。
   - 查询 `ai_agent_kb_relation` 获取绑定的知识库和检索参数。
   - 调用 `IRetrievalService` 获取上下文。
   - 构建 RAG Prompt（如：`"基于以下运维日志和手册信息回答问题：\n{context}\n\n问题：{query}"`）。
   - 调用 LLM 生成回复，并封装引用来源（Citations）返回给前端。

### 3.3 核心 API 路由 (Controllers)
- `POST /ai/kb/list` - 知识库分页列表
- `POST /ai/kb/document/upload` - 上传文档并触发异步解析
- `GET /ai/kb/document/status/{docId}` - 轮询解析状态
- `POST /ai/kb/hit-test` - 命中测试接口（入参：kbId, queryText, topK；出参：检索到的分段列表及相似度得分）

---

## 4. 前端详细设计 (Vue3 + Element Plus)

在 `ruoyi-ui/src/views/ai/` 目录下新增或修改以下视图组件。

### 4.1 知识库管理 (`views/ai/kb/index.vue`)
- **布局**：若依标准的左侧分类树（可选）+ 右侧数据表格。
- **功能**：新增、编辑、删除、状态切换。
- **操作列**：包含“数据集管理”、“设置”按钮。

### 4.2 数据集与文档管理 (`views/ai/kb/dataset.vue`)
- **布局**：进入特定知识库后的文档列表。
- **上传组件**：使用 `el-upload`，支持拖拽。上传弹窗中包含**分段规则配置**：
  - 分段标识符（如 `\n\n`）
  - 分段最大长度（Token）
  - 分段重叠长度（Token）
- **状态展示**：使用 `el-tag` 展示解析状态（排队中、解析中、成功、失败）。支持“重新解析”操作。

### 4.3 分段详情与手动修正 (`views/ai/kb/segment.vue`)
- **功能**：点击文档后，展示该文档切分出的所有文本块。
- **交互**：允许管理员直接在页面上编辑切片内容（如修正解析错误的日志格式），保存后后端需重新生成该切片的 Embedding 并更新向量库。

### 4.4 命中测试看板 (`views/ai/kb/hitTest.vue`)
- **布局**：左右分栏布局。
- **左侧 (测试输入)**：`el-input` (type="textarea") 供用户粘贴报错日志或提问。包含“相似度阈值”和“返回数量(Top-K)”的调节滑块 (`el-slider`)。
- **右侧 (测试结果)**：展示召回的文档切片卡片列表，每张卡片高亮显示匹配内容，并标明**相似度得分**和**所属文档名称**。

### 4.5 智能体配置扩展 (`views/ai/agent/index.vue`)
- **表单修改**：在原有的智能体表单中增加一栏【知识库配置】。
- **组件**：使用 `el-select` (multiple) 关联已创建的知识库。
- **参数**：增加检索策略配置项（检索阈值、最大召回数、空检索是否允许大模型自由回答等）。

### 4.6 聊天/分析对话界面 (`views/ai/chat/index.vue`)
- **UI 增强**：在 AI 回复的气泡下方，增加【参考来源】折叠面板 (`el-collapse`)。
- **数据绑定**：解析后端返回的 Citations 数据，展示引用的具体运维文档名称及匹配的段落摘要，点击可查看完整日志/手册片段。
