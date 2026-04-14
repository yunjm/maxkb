# Tasks
 - [x] Task 1: 基础设施与依赖准备
   - [x] SubTask 1.1: 在后端（Spring Boot 4）引入向量数据库（如 pgvector/Milvus）依赖及相关配置。
   - [x] SubTask 1.2: 引入文本解析与分段处理库（参考MaxKB4j依赖，如Apache PDFBox, POI等）。
   - [x] SubTask 1.3: 引入 Embedding 模型的 API 客户端或本地调用逻辑。
 - [x] Task 2: 知识库管理模块 (后端实现)
   - [x] SubTask 2.1: 设计并创建知识库（Knowledge Base）、文档（Document）、分段（Segment）的数据库表结构及若依实体类。
   - [x] SubTask 2.2: 实现知识库的 CRUD 接口，并集成至若依权限体系。
   - [x] SubTask 2.3: 实现文件上传、解析、切分（Chunking）的异步处理逻辑与状态流转。
   - [x] SubTask 2.4: 实现文本向量化（Embedding）并存入向量数据库的持久化逻辑。
   - [x] SubTask 2.5: 实现针对特定知识库的命中测试（Hit Testing）检索接口。
 - [x] Task 3: 知识库管理模块 (前端实现 - ruoyi Vue3)
   - [x] SubTask 3.1: 在前端新增“知识库管理”菜单及列表视图。
   - [x] SubTask 3.2: 实现知识库创建、配置详情及基本设置页面。
   - [x] SubTask 3.3: 实现文档数据集上传、解析状态查看与重试页面。
   - [x] SubTask 3.4: 实现分段内容查看、手动修改与命中测试页面。
 - [x] Task 4: 智能体管理功能重构与扩展
   - [x] SubTask 4.1: 修改智能体数据表，增加与知识库的多对多关联表（Agent-Knowledge_Base）。
   - [x] SubTask 4.2: 修改后端智能体 CRUD 接口，支持保存知识库关联关系及检索参数（相似度阈值、Top-K等）。
   - [x] SubTask 4.3: 调整前端“智能体管理”表单，增加知识库下拉多选组件与RAG参数配置面板。
 - [x] Task 5: 核心 RAG 日志分析链路对接
   - [x] SubTask 5.1: 封装基于用户输入和所选知识库的联合向量检索服务。
   - [x] SubTask 5.2: 重构现有的模型对话接口，注入检索到的运维日志上下文并构建 RAG Prompt。
   - [x] SubTask 5.3: 在前端分析聊天界面，增加“引用来源（Citations）”展示功能，方便用户回溯原始日志或手册。

# Task Dependencies
- [Task 2] depends on [Task 1]
- [Task 3] depends on [Task 2]
- [Task 4] depends on [Task 2]
- [Task 5] depends on [Task 4]
