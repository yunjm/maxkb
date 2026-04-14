# MaxKB4j → RuoYi-Vue 迁移分析报告

## 0. 范围与证据来源

- 模块边界：`maxkb4j-service` 下拆分 `system/oss/model/tool/knowledge/application/chat/workflow/trigger`（见 [maxkb4j-service](file:///workspace/MaxKB4j/maxkb4j-service)）。
- 路由清单：以各模块 `*Controller.java` 的 Spring MVC 注解为准（示例：[AuthController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/controller/AuthController.java)、[KnowledgeController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-knowledge/src/main/java/com/maxkb4j/knowledge/controller/KnowledgeController.java)、[ChatApiController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-chat/src/main/java/com/maxkb4j/chat/controller/ChatApiController.java)）。
- 数据表：Flyway 脚本 `V1~V5`（见 [db/migration](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration)）。
- 权限模型：Sa-Token JWT Stateless + 自定义注解 `@SaCheckPerm` + `StpInterface` 动态权限加载（见 [SaTokenConfigure](file:///workspace/MaxKB4j/maxkb4j-start/src/main/java/com/maxkb4j/start/config/SaTokenConfigure.java)、[SaCheckPermAspect](file:///workspace/MaxKB4j/maxkb4j-common/src/main/java/com/maxkb4j/common/aspect/SaCheckPermAspect.java)、[StpInterfaceImpl](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/service/impl/StpInterfaceImpl.java)、[PermissionEnum](file:///workspace/MaxKB4j/maxkb4j-common/src/main/java/com/maxkb4j/common/enums/PermissionEnum.java)）。
- 部署形态：Postgres(pgvector)+Mongo+SpringBoot（见 [docker-compose.yml](file:///workspace/MaxKB4j/docker-compose.yml)、[application.yml](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/application.yml)）。
- 前端形态：`maxkb4j-start` 内置静态资源 admin/chat（见 [static/admin](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/static/admin)、[static/chat](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/static/chat)）。

## 1) MaxKB4j 功能如何实现（按功能域）

### 1.1 system（系统/用户/权限/资源引用）

- 功能概览：管理员登录、验证码/邮件找回、用户管理、系统设置、目录树、资源引用与资源授权。
- 主要路由：
  - 登录与登出：`POST admin/api/user/login`、`POST admin/api/user/logout`（见 [AuthController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/controller/AuthController.java#L50-L96)）
  - 验证码与邮件找回：`GET admin/api/user/captcha`、`POST admin/api/user/send_email`、`POST admin/api/user/rePassword`（同上）
  - 用户管理（管理员）：`GET admin/api/user_manage/{page}/{size}`、`POST admin/api/user_manage`、`PUT/DELETE admin/api/user_manage/{id}`（见 [UserController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/controller/UserController.java)）
  - 系统配置：`GET/POST/PUT admin/api/email_setting`、`GET/POST admin/api/display/*`（见 [SystemSettingController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/controller/SystemSettingController.java)）
  - 目录树：`GET admin/api/workspace/default/{source}/folder`（见 [FolderController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/controller/FolderController.java)）
  - 资源引用：`GET admin/api/workspace/default/resource_mapping/{resourceType}/{resourceId}/{current}/{size}`（见 [ResourceMappingController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/controller/ResourceMappingController.java)）
  - 资源授权：`PUT admin/api/workspace/default/user_resource_permission/user/{userId}/resource/{type}` 等（见 [UserResourcePermissionController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/controller/UserResourcePermissionController.java)）
- 关键实现机制：
  - Sa-Token 双登录体系（ADMIN/USER）与 token-name 隔离（见 [StpKit](file:///workspace/MaxKB4j/maxkb4j-common/src/main/java/com/maxkb4j/common/util/StpKit.java#L8-L33)）
  - 资源级权限：`PermissionEnum#getResourcePerm` 拼接对象级权限串；`@SaCheckPerm` 通过 AOP 自动从路径变量 `{id}` 推导 `targetId` 后校验（见 [SaCheckPermAspect](file:///workspace/MaxKB4j/maxkb4j-common/src/main/java/com/maxkb4j/common/aspect/SaCheckPermAspect.java#L20-L58)）
  - 权限来源：`StpInterfaceImpl#getPermissionList` 从 `user_resource_permission` 聚合，并叠加默认权限（见 [StpInterfaceImpl](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/service/impl/StpInterfaceImpl.java#L17-L67)）
- 关键数据表：
  - `user`、`user_resource_permission`、`system_setting`、`folder`（见 [V1__init_tables.sql](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql)）
  - `resource_mapping`（见 [V5__add_table.sql](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V5__add_table.sql#L1-L51)）

### 1.2 oss（文件上传/下载，Mongo 存储）

- 功能概览：上传文件（admin/chat 两套入口）、以 fileId 下载/访问。
- 主要路由：`POST /admin/api/oss/file`、`POST /chat/api/oss/file`、`GET /oss/file/{fileId}` 等（见 [FileController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-oss/src/main/java/com/maxkb4j/oss/controller/FileController.java#L20-L45)）
- 关键实现：`MongoFileService` 负责存取与流式输出（同上）。
- 数据：Flyway 中没有 oss 表；文件存储在 Mongo。

### 1.3 model（大模型配置/Provider）

- 功能概览：模型 CRUD、模型参数表单、Provider/模型类型列表。
- 主要路由：
  - 模型管理：`POST/GET/PUT/DELETE admin/api/workspace/default/model...`（见 [ModelController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-model/src/main/java/com/maxkb4j/model/controller/ModelController.java)）
  - Provider 列表与表单：`GET admin/api/provider/*`（见 [ProviderController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-model/src/main/java/com/maxkb4j/model/controller/ProviderController.java)）
- 权限：`MODEL_CREATE/READ/EDIT/DELETE`（见 [ModelController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-model/src/main/java/com/maxkb4j/model/controller/ModelController.java#L32-L59) 与 [PermissionEnum](file:///workspace/MaxKB4j/maxkb4j-common/src/main/java/com/maxkb4j/common/enums/PermissionEnum.java)）。
- 关键数据表：`model`（jsonb `meta`、`model_params_form`）（见 [V1 model](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L638-L669)）。

### 1.4 tool（工具库/调试/导入导出）

- 功能概览：工具 CRUD、分页与列表、内部工具模板、调试执行、导入导出、连接测试、上传技能文件。
- 主要路由：`admin/api/workspace/default/tool...`（见 [ToolController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-tool/src/main/java/com/maxkb4j/tool/controller/ToolController.java)）
- 关键实现：
  - 调试执行：HTTP 工具 `HttpRequestExecutor`、脚本工具 `GroovyScriptExecutor`（见 [ToolController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-tool/src/main/java/com/maxkb4j/tool/controller/ToolController.java#L100-L120)）
- 权限：`TOOL_*`（见 [PermissionEnum](file:///workspace/MaxKB4j/maxkb4j-common/src/main/java/com/maxkb4j/common/enums/PermissionEnum.java#L80-L92)）。
- 关键数据表：`tool`（jsonb 字段较多）（见 [V1 tool](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L592-L632)）。

### 1.5 knowledge（知识库/文档/分段/向量化/版本/动作）

- 功能概览：知识库（base/web/workflow）、文档导入（本地/网页/QA/表格）、分段、向量化、命中测试、问题生成与关联、导出、发布、版本与动作记录。
- 主要路由（节选）：
  - 知识库 CRUD：`GET/POST/PUT/DELETE admin/api/workspace/default/knowledge...`（见 [KnowledgeController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-knowledge/src/main/java/com/maxkb4j/knowledge/controller/KnowledgeController.java)）
  - 命中测试：`PUT /knowledge/{id}/hit_test`（同上）
  - 文档管理：`admin/api/workspace/default/knowledge/{id}/document...`（见 [DocumentController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-knowledge/src/main/java/com/maxkb4j/knowledge/controller/DocumentController.java)）
  - 段落与问题：`ParagraphController`、`ProblemController`（见 [ParagraphController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-knowledge/src/main/java/com/maxkb4j/knowledge/controller/ParagraphController.java)、[ProblemController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-knowledge/src/main/java/com/maxkb4j/knowledge/controller/ProblemController.java)）
- 权限：`KNOWLEDGE_*`、`KNOWLEDGE_DOCUMENT_*`、`KNOWLEDGE_PROBLEM_*` 等（见 [PermissionEnum](file:///workspace/MaxKB4j/maxkb4j-common/src/main/java/com/maxkb4j/common/enums/PermissionEnum.java#L42-L78)）。
- 关键数据表：
  - `knowledge/document/paragraph/problem/embedding/problem_paragraph_mapping`（见 [V1__init_tables.sql](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql)）
  - `embedding.embedding` 为 pgvector 类型（见 [V1 embedding](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L677-L710)）
  - `knowledge_workflow_version/knowledge_action`（见 [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L757-L819)）

### 1.6 application（应用编排/发布/导入导出/访问控制/统计/对话记录改进）

- 功能概览：应用创建与发布、导入导出、访问 token（公开访问/白名单）、API Key、版本、应用统计、对话日志与“加知识/改进”。
- 主要路由（节选）：
  - 应用 CRUD/发布/导入导出：`admin/api/workspace/default/application...`（见 [ApplicationController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationController.java)）
  - 平台接入与回调：`/application/{id}/platform/*` + `POST admin/api/chat/{key}/{id}`（见 [ApplicationAccessController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationAccessController.java)）
  - 会话与对话记录：`ApplicationChatController`、`ApplicationChatRecordController`（见 [ApplicationChatController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationChatController.java)、[ApplicationChatRecordController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationChatRecordController.java)）
  - API Key 管理：`ApplicationKeyController`（见 [ApplicationKeyController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationKeyController.java)）
  - 版本：`ApplicationVersionController`（见 [ApplicationVersionController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationVersionController.java)）
- 权限：`APPLICATION_*` + `APPLICATION_ACCESS_*` + `APPLICATION_CHAT_LOG_*`（见 [PermissionEnum](file:///workspace/MaxKB4j/maxkb4j-common/src/main/java/com/maxkb4j/common/enums/PermissionEnum.java#L18-L36)）。
- 关键数据表：
  - `application/application_access_token/application_api_key/application_version`（见 [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql)）
  - `application_access`（见 [V2__add_table.sql](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V2__add_table.sql#L1-L17)）
  - `application_chat/application_chat_record/application_chat_user_stats`（见 [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L237-L339)）
  - `application_chat_share_link`（见 [V5__add_table.sql](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V5__add_table.sql#L53-L72)）

### 1.7 chat（对外 Chat API / OpenAI 兼容 / Embed / Share / MCP）

- 功能概览：匿名授权、会话创建、SSE 流式对话、历史对话管理、语音相关（STT/TTS）、对话分享、嵌入脚本、MCP 接口、OpenAI 兼容接口。
- 主要路由：
  - Chat API：`chat/api/*`（见 [ChatApiController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-chat/src/main/java/com/maxkb4j/chat/controller/ChatApiController.java)）
  - OpenAI 兼容：`POST chat/api/{appId}/chat/completions`（见 [ChatOpenAiController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-chat/src/main/java/com/maxkb4j/chat/controller/ChatOpenAiController.java)）
- 关键实现：
  - SSE：`POST /chat/api/chat_message/{chatId}`，基于 Reactor `Sinks` 输出流（见 [ChatApiController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-chat/src/main/java/com/maxkb4j/chat/controller/ChatApiController.java#L93-L114)）
  - MCP：`POST /chat/api/mcp`，用 `ApplicationApiKeyEntity.secretKey` 做鉴权（见 [ChatApiController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-chat/src/main/java/com/maxkb4j/chat/controller/ChatApiController.java#L116-L128)）
- 数据依赖：主要复用 application 领域表（`application_*`）。

### 1.8 workflow（工作流引擎：节点、比较器、执行器）

- 功能概览：工作流执行引擎（节点注册、节点执行、比较器、异常体系），供 Application/Knowledge 的 `work_flow(jsonb)` 字段承载流程定义与快照。
- 关键实现入口：
  - 工作流执行器：`WorkFlowActuator`（见 [WorkFlowActuator](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-workflow/src/main/java/com/maxkb4j/workflow/service/WorkFlowActuator.java)）
  - 节点实现：`workflow/handler/node/impl/*`（见 [node/impl](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-workflow/src/main/java/com/maxkb4j/workflow/handler/node/impl)）
- 数据承载：
  - `application.work_flow`、`knowledge.work_flow`（jsonb）（见 [V1 application](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L85-L124)、[V1 knowledge](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L393-L429)）

### 1.9 trigger（事件触发器/任务/记录/Webhook）

- 功能概览：触发器 CRUD、按来源挂载、批量启停/删除、任务记录查询、Webhook 触发入口。
- 主要路由：
  - 触发器管理：`admin/api/workspace/default/trigger...`（见 [TriggerController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-trigger/src/main/java/com/maxkb4j/trigger/controller/TriggerController.java)）
  - 任务记录：`GET admin/api/workspace/default/trigger/{id}/task_record/{current}/{size}` 等（见 [TriggerTaskRecordController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-trigger/src/main/java/com/maxkb4j/trigger/controller/TriggerTaskRecordController.java)）
  - Webhook：`POST admin/api/trigger/v1/webhook/{id}`（见 [WebhookTriggerController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-trigger/src/main/java/com/maxkb4j/trigger/controller/WebhookTriggerController.java)）
- 关键数据表：`event_trigger/event_trigger_task/event_trigger_task_record`（见 [V3__add_trigger.sql](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V3__add_trigger.sql#L1-L70)）

## 2) 迁移到若依VUE：最佳方案与备选方案

### 2.1 最佳方案（推荐）：若依做统一门户 + MaxKB4j 保持独立服务（旁挂集成）

- 结论：优先推荐“旁挂集成”，即 MaxKB4j 维持现有服务/库表/向量检索能力，若依负责统一登录、菜单、审计与门户体验；通过网关/Nginx 做路由分发与单点登录互通。
- 关键理由：
  - MaxKB4j 数据层强依赖 Postgres 特性（pgvector、jsonb、数组列等），与若依常见 MySQL 栈差异极大（典型表：`embedding.embedding(public.vector)`，见 [V1 embedding](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L677-L710)）。
  - 工作流/触发器/聊天 SSE/MCP/文件 Mongo 等能力跨域耦合深，强行并入若依单体会引入大规模改造与回归风险。
- 鉴权与权限对齐（推荐实现）：
  - 若依登录后签发一个 MaxKB4j 可校验的 JWT（共享密钥或公私钥），MaxKB4j 继续使用 Sa-Token JWT Stateless（见 [SaTokenConfigure](file:///workspace/MaxKB4j/maxkb4j-start/src/main/java/com/maxkb4j/start/config/SaTokenConfigure.java#L9-L17)）。
  - 权限分两层：
    - 若依 perms（功能级）：用于菜单与按钮控制（例如 `maxkb:knowledge:edit`）
    - MaxKB4j 资源级权限（对象级）：继续由 `@SaCheckPerm` + `PermissionEnum#getResourcePerm` 执行（见 [SaCheckPermAspect](file:///workspace/MaxKB4j/maxkb4j-common/src/main/java/com/maxkb4j/common/aspect/SaCheckPermAspect.java#L20-L58)、[PermissionEnum#getResourcePerm](file:///workspace/MaxKB4j/maxkb4j-common/src/main/java/com/maxkb4j/common/enums/PermissionEnum.java#L106-L113)）
- 前端承载（推荐演进）：
  - P0：若依菜单以 iframe/外链方式接入 MaxKB4j admin（因为当前仅有编译产物 [static/admin](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/static/admin)）
  - P1：按接口重建关键页面为若依风格（从“高频管理”开始：知识库、应用、模型、工具）

### 2.2 备选方案 A：仅接管登录（最少改动），其余旁挂

- 思路：若依负责登录；MaxKB4j 内不再提供自有登录入口（或仅管理员保留），改为信任上游用户信息并换票进入 Sa-Token。
- 适用：必须单点登录，但不想维护双 token 体系。
- 代价：需要梳理用户来源、禁用/角色同步、退出登录一致性。

### 2.3 备选方案 B：深度融合（将 MaxKB4j 合并到若依后端/库表）

- 不推荐：会触发数据库栈迁移（MySQL → Postgres 或引入外置向量库）、权限与数据权限体系重做、SSE/MCP/工作流/触发器的完整回归测试。
- 仅在以下前提考虑：组织强制单体单库且可接受若依整体迁移到 Postgres，并愿意投入较长周期做全量回归。

## 3) 每个功能域迁移分析结果（统一模板）

> 模板字段：现状实现 / 若依承载 / 数据迁移 / 权限迁移 / 风险与复杂度 / 推荐落地顺序

### 3.1 system 域

- 现状实现：
  - 认证与登录：见 [AuthController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/controller/AuthController.java)
  - 用户管理：见 [UserController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/controller/UserController.java)
  - 系统设置：见 [SystemSettingController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/controller/SystemSettingController.java)
  - 对象级权限模型：见 [PermissionEnum](file:///workspace/MaxKB4j/maxkb4j-common/src/main/java/com/maxkb4j/common/enums/PermissionEnum.java)、[SaCheckPermAspect](file:///workspace/MaxKB4j/maxkb4j-common/src/main/java/com/maxkb4j/common/aspect/SaCheckPermAspect.java)、[StpInterfaceImpl](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/service/impl/StpInterfaceImpl.java)
- 若依承载：
  - 后端：新增 `maxkb-system` 业务模块（controller/service/mapper）或以网关转发保持 MaxKB4j 原后端
  - 前端：`MaxKB管理/系统管理/用户管理`、`系统设置`、`资源授权` 三个菜单（P0 iframe；P1 重建）
- 数据迁移：
  - 旁挂：保持 MaxKB4j `user/user_resource_permission` 表；若依用户与 MaxKB 用户建议建立映射（username/email/手机号 或单独映射表）
  - 深度融合：需把对象级权限迁到若依数据权限/行级权限体系，不建议直接把 `targetId` 塞入 perms
- 权限迁移：
  - 建议的若依权限字符串：`maxkb:system:user:list|add|edit|remove`、`maxkb:system:setting:email|display`、`maxkb:system:permission:grant`
- 风险与复杂度：
  - 风险：双用户体系导致角色/禁用状态不一致；邮件找回依赖 SMTP 配置
  - 复杂度：中（取决于是否统一用户源）
- 推荐落地顺序：
  - P0：若依菜单接入 + SSO 换票；P1：统一用户主数据源；P2：授权 UI 在若依重建但写回 MaxKB 表

### 3.2 oss 域

- 现状实现：见 [FileController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-oss/src/main/java/com/maxkb4j/oss/controller/FileController.java)
- 若依承载：
  - 后端：建议保持 MaxKB4j 原实现（MongoFileService）
  - 前端：一般无需独立菜单，仅作为知识库/应用模块的附件能力
- 数据迁移：保持 Mongo 存储；若依如需统一文件中心，建议仅做代理与审计，不迁移二进制
- 权限迁移：`maxkb:oss:file:upload|download`
- 风险与复杂度：中（跨域/iframe 场景下上传下载的鉴权与 Header 透传）
- 推荐落地顺序：P0 保持现状；P1 统一对象存储（可选）

### 3.3 model 域

- 现状实现：
  - 模型 CRUD：见 [ModelController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-model/src/main/java/com/maxkb4j/model/controller/ModelController.java)
  - Provider 列表/表单：见 [ProviderController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-model/src/main/java/com/maxkb4j/model/controller/ProviderController.java)
  - 数据表：`model`（见 [V1 model](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L638-L669)）
- 若依承载：
  - 菜单：`MaxKB管理/资源配置/模型管理`
  - 后端：旁挂优先；深度融合需要解决“密钥治理/脱敏/审计”与配置表单渲染
- 数据迁移：保留 Postgres jsonb 字段；避免向 MySQL 展开成大量列
- 权限迁移：`maxkb:model:list|add|edit|remove`
- 风险与复杂度：中（credential 的安全与审计）
- 推荐落地顺序：P0 复用原页；P1 若依侧增加密钥托管与审计（但写回 MaxKB）

### 3.4 tool 域

- 现状实现：见 [ToolController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-tool/src/main/java/com/maxkb4j/tool/controller/ToolController.java) 与调试执行入口（[ToolController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-tool/src/main/java/com/maxkb4j/tool/controller/ToolController.java#L100-L120)）
- 若依承载：`MaxKB管理/资源配置/工具管理`
- 数据迁移：表 `tool` 大量 jsonb 字段（见 [V1 tool](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L592-L632)），建议留在 Postgres
- 权限迁移：`maxkb:tool:list|add|edit|remove|import|export|debug`
- 风险与复杂度：高（脚本执行/Groovy、外部 HTTP 调用，需要治理与审计）
- 推荐落地顺序：P0 接入；P1 增加审批/只读共享/执行审计

### 3.5 knowledge 域

- 现状实现：
  - 知识库：见 [KnowledgeController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-knowledge/src/main/java/com/maxkb4j/knowledge/controller/KnowledgeController.java)
  - 文档：见 [DocumentController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-knowledge/src/main/java/com/maxkb4j/knowledge/controller/DocumentController.java)
  - 段落/问题：见 [ParagraphController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-knowledge/src/main/java/com/maxkb4j/knowledge/controller/ParagraphController.java)、[ProblemController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-knowledge/src/main/java/com/maxkb4j/knowledge/controller/ProblemController.java)
  - 向量表：`embedding.embedding(public.vector)`（见 [V1 embedding](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L677-L710)）
- 若依承载：
  - 菜单：`MaxKB管理/知识库/知识库管理`、`MaxKB管理/知识库/文档管理`
  - 后端：推荐旁挂；若深度融合则需重建向量检索与切分/embedding 流程
- 数据迁移：
  - 旁挂：保留原表结构
  - 深度融合：MySQL 不具备等价 pgvector 能力，通常需要外置向量库（Milvus/PGVector/ES向量）并重做检索链路
- 权限迁移：
  - `maxkb:knowledge:list|add|edit|remove|export|vector|hitTest`
  - `maxkb:knowledgeDocument:create|edit|remove|sync|vector|export|download|replace|migrate`
- 风险与复杂度：高（向量化重建耗时、召回差异、切分策略兼容）
- 推荐落地顺序：P0 保持 MaxKB 全链路；P1 在若依做审计/治理视图；P2（可选）再做空间/目录抽象

### 3.6 application 域

- 现状实现：
  - 应用管理：见 [ApplicationController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationController.java)
  - 访问控制：见 [ApplicationAccessController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationAccessController.java)
  - 对话与记录：见 [ApplicationChatController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationChatController.java)、[ApplicationChatRecordController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationChatRecordController.java)
  - API Key：见 [ApplicationKeyController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationKeyController.java)
- 若依承载：
  - 菜单：`MaxKB管理/应用/应用管理`、`访问控制`、`会话与标注`
  - SSE 与回调：优先通过网关/Nginx 透传，避免若依后端“中转流式响应”
- 数据迁移：表 `application_*` + `application_access` + `application_chat_share_link`（见 [db/migration](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration)）
- 权限迁移：
  - `maxkb:app:list|add|edit|remove|import|export|publish`
  - `maxkb:app:accessToken:read|edit`
  - `maxkb:app:chatlog:read|export|annotate|addKnowledge`
- 风险与复杂度：高（SSE 透传、网关 buffer、回调鉴权、APIKey 泄露治理）
- 推荐落地顺序：P0 旁挂；P1 统一域名与回调；P2 统一应用模板/商店体验（可选）

### 3.7 chat 域

- 现状实现：
  - Chat API：见 [ChatApiController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-chat/src/main/java/com/maxkb4j/chat/controller/ChatApiController.java)
  - OpenAI 兼容：见 [ChatOpenAiController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-chat/src/main/java/com/maxkb4j/chat/controller/ChatOpenAiController.java)
- 若依承载：
  - 菜单：`MaxKB管理/对外接口/Chat API`
  - 开放侧鉴权：不建议走若依 perms；建议继续用 accessToken/APIKey
- 数据迁移：主要复用 `application_*`；chat 前端为独立静态页（见 [static/chat](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/static/chat)）
- 权限迁移：管理侧 `maxkb:chat:share:create|read`
- 风险与复杂度：中（跨域与匿名 token 生命周期治理）
- 推荐落地顺序：P0 保持 MaxKB；P1 若依侧增加 APIKey 审计面板

### 3.8 workflow 域

- 现状实现：
  - 执行器与节点体系：见 [workflow](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-workflow/src/main/java/com/maxkb4j/workflow)
  - 数据承载：`application.work_flow` / `knowledge.work_flow`（jsonb）
- 若依承载：短期 iframe；长期重建可视化编辑器成本较高
- 数据迁移：jsonb 保持不动；版本表 `knowledge_workflow_version`（见 [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L757-L784)）
- 权限迁移：`maxkb:workflow:edit|run|version|publish`
- 风险与复杂度：高（跨域依赖多、回归面大）
- 推荐落地顺序：P0 保持 MaxKB 引擎；P1 若依做审计聚合；P2 再考虑 UI 重建

### 3.9 trigger 域

- 现状实现：
  - 触发器 CRUD：见 [TriggerController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-trigger/src/main/java/com/maxkb4j/trigger/controller/TriggerController.java)
  - Webhook：见 [WebhookTriggerController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-trigger/src/main/java/com/maxkb4j/trigger/controller/WebhookTriggerController.java)
  - 表：`event_trigger/*`（见 [V3__add_trigger.sql](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V3__add_trigger.sql#L1-L70)）
- 若依承载：
  - 菜单：`MaxKB管理/触发器/触发器管理`、`任务记录`
  - Webhook token：建议在网关层统一做签名校验，MaxKB 仅做来源可信校验
- 权限迁移：`maxkb:trigger:list|add|edit|remove|enable|record`
- 风险与复杂度：中（幂等、重复投递、调度一致性、可观测性）
- 推荐落地顺序：P0 旁挂；P1 webhook 收敛到网关并加签；P2 做统一任务中心（可选）

## 附录：MaxKB4j 权限模型摘要（用于若依对接）

- Token 模式：Sa-Token JWT Stateless（见 [SaTokenConfigure](file:///workspace/MaxKB4j/maxkb4j-start/src/main/java/com/maxkb4j/start/config/SaTokenConfigure.java#L9-L17)）
- 权限字符串格式：`{resource}:{operate}:/WORKSPACE/{workspaceId}/{resourceType}/{targetId}`（见 [PermissionEnum#getResourcePerm](file:///workspace/MaxKB4j/maxkb4j-common/src/main/java/com/maxkb4j/common/enums/PermissionEnum.java#L106-L113)）
- 注解鉴权：`@SaCheckPerm(PermissionEnum.X)` + AOP 拼接权限并校验（见 [SaCheckPerm](file:///workspace/MaxKB4j/maxkb4j-common/src/main/java/com/maxkb4j/common/annotation/SaCheckPerm.java#L11-L15)、[SaCheckPermAspect](file:///workspace/MaxKB4j/maxkb4j-common/src/main/java/com/maxkb4j/common/aspect/SaCheckPermAspect.java#L20-L58)）
- 权限加载：`StpInterfaceImpl#getPermissionList` 从 `user_resource_permission` 汇总，并叠加默认权限（见 [StpInterfaceImpl](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/service/impl/StpInterfaceImpl.java#L17-L67)）

## 附录：Controller 清单（覆盖性校验）

- system
  - [AuthController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/controller/AuthController.java)
  - [UserController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/controller/UserController.java)
  - [SystemSettingController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/controller/SystemSettingController.java)
  - [FolderController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/controller/FolderController.java)
  - [ResourceMappingController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/controller/ResourceMappingController.java)
  - [UserResourcePermissionController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-system/src/main/java/com/maxkb4j/system/controller/UserResourcePermissionController.java)
- oss
  - [FileController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-oss/src/main/java/com/maxkb4j/oss/controller/FileController.java)
- model
  - [ModelController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-model/src/main/java/com/maxkb4j/model/controller/ModelController.java)
  - [ProviderController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-model/src/main/java/com/maxkb4j/model/controller/ProviderController.java)
- tool
  - [ToolController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-tool/src/main/java/com/maxkb4j/tool/controller/ToolController.java)
- knowledge
  - [KnowledgeController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-knowledge/src/main/java/com/maxkb4j/knowledge/controller/KnowledgeController.java)
  - [DocumentController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-knowledge/src/main/java/com/maxkb4j/knowledge/controller/DocumentController.java)
  - [DocumentTemplateController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-knowledge/src/main/java/com/maxkb4j/knowledge/controller/DocumentTemplateController.java)
  - [ParagraphController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-knowledge/src/main/java/com/maxkb4j/knowledge/controller/ParagraphController.java)
  - [ProblemController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-knowledge/src/main/java/com/maxkb4j/knowledge/controller/ProblemController.java)
- application
  - [ApplicationController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationController.java)
  - [ApplicationAccessController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationAccessController.java)
  - [ApplicationChatController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationChatController.java)
  - [ApplicationChatRecordController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationChatRecordController.java)
  - [ApplicationKeyController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationKeyController.java)
  - [ApplicationStoreController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationStoreController.java)
  - [ApplicationVersionController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ApplicationVersionController.java)
  - [ChatMessageController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-application/src/main/java/com/maxkb4j/application/controller/ChatMessageController.java)
- chat
  - [ChatApiController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-chat/src/main/java/com/maxkb4j/chat/controller/ChatApiController.java)
  - [ChatOpenAiController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-chat/src/main/java/com/maxkb4j/chat/controller/ChatOpenAiController.java)
- trigger
  - [TriggerController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-trigger/src/main/java/com/maxkb4j/trigger/controller/TriggerController.java)
  - [TriggerTaskRecordController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-trigger/src/main/java/com/maxkb4j/trigger/controller/TriggerTaskRecordController.java)
  - [WebhookTriggerController](file:///workspace/MaxKB4j/maxkb4j-service/maxkb4j-trigger/src/main/java/com/maxkb4j/trigger/controller/WebhookTriggerController.java)

## 附录：关键 Web 配置（与若依集成相关）

- 异步线程池与 SSE：MVC async executor 由 [WebConfig](file:///workspace/MaxKB4j/maxkb4j-start/src/main/java/com/maxkb4j/start/config/WebConfig.java#L17-L27) 配置，网关转发 SSE 时需要注意不缓冲响应体。
- Chat API 的拦截器：`/chat/api/application/profile`、`/chat/api/open`、`/chat/api/chat_message/*` 由 [AuthInterceptor](file:///workspace/MaxKB4j/maxkb4j-core/src/main/java/com/maxkb4j/core/interceptor/AuthInterceptor.java) 拦截注册（见 [WebConfig](file:///workspace/MaxKB4j/maxkb4j-start/src/main/java/com/maxkb4j/start/config/WebConfig.java#L33-L40)）。
- 前端路由转发：`/admin/**` → `/admin/index.html`、`/chat/**` → `/chat/index.html`（见 [WebConfig](file:///workspace/MaxKB4j/maxkb4j-start/src/main/java/com/maxkb4j/start/config/WebConfig.java#L48-L60)），若依侧做反向代理时需保留此转发规则或由网关实现等价规则。
- CORS：全局 `allowedOriginPatterns("*")`（见 [WebConfig](file:///workspace/MaxKB4j/maxkb4j-start/src/main/java/com/maxkb4j/start/config/WebConfig.java#L68-L85)），生产环境建议在网关与服务端同时收敛允许域名范围。

## 附录：数据表清单（全量）

| 功能域 | 表名 | 关键字段/类型（节选） | 来源 |
|---|---|---|---|
| system | system_setting | PK `type`；`meta` jsonb | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L9-L21) |
| system | user | PK `id`；唯一键 `email/username` | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L25-L61) |
| system | user_resource_permission | PK `id`；`permission_list` varchar[] | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L65-L82) |
| system | folder | PK `id` | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L541-L585) |
| system | resource_mapping | PK `id` | [V5](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V5__add_table.sql#L1-L51) |
| model | model | PK `id`；`meta` jsonb；`model_params_form` jsonb | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L638-L669) |
| tool | tool | PK `id`；FK `user_id`；`input_field_list/init_field_list/init_params` jsonb | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L592-L632) |
| application | application | PK `id`；多个 jsonb；`tool_ids/application_ids/knowledge_ids` varchar[] | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L85-L145) |
| application | application_access_token | PK `application_id`；`white_list` varchar[] | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L150-L187) |
| application | application_api_key | PK `id`；`cross_domain_list` varchar[] | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L191-L232) |
| application/chat | application_chat | PK `id`；`asker/meta/source` jsonb | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L237-L272) / [V4](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V4__update_table.sql#L1-L3) |
| application/chat | application_chat_record | PK `id`；`details/source` jsonb；`improve_paragraph_id_list` varchar[] | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L277-L306) / [V4](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V4__update_table.sql#L5-L9) |
| application/chat | application_chat_user_stats | PK `id`；FK `application_id` | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L312-L339) |
| application | application_version | PK `id`；多个 jsonb；数组列同 application | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L344-L387) |
| application | application_access | PK `id`；`status/config` jsonb | [V2](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V2__add_table.sql#L4-L17) |
| application/chat | application_chat_share_link | PK `id`；`chat_record_ids` varchar[] | [V5](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V5__add_table.sql#L53-L72) |
| knowledge | knowledge | PK `id`；`meta/work_flow` jsonb | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L393-L429) |
| knowledge | document | PK `id`；`meta/status_meta` jsonb | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L436-L468) |
| knowledge | paragraph | PK `id`；`status_meta` jsonb | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L473-L507) |
| knowledge | problem | PK `id` | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L515-L534) |
| knowledge | embedding | PK `id`；`embedding` vector；`search_vector` tsvector | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L677-L710) |
| knowledge | problem_paragraph_mapping | PK `id`；FK `document_id` | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L716-L751) |
| knowledge/workflow | knowledge_workflow_version | PK `id`；`work_flow` jsonb | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L757-L784) |
| knowledge/workflow | knowledge_action | PK `id`；`details/meta` jsonb | [V1](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V1__init_tables.sql#L792-L819) |
| trigger | event_trigger | PK `id`；`trigger_setting/meta` jsonb | [V3](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V3__add_trigger.sql#L4-L23) |
| trigger | event_trigger_task | PK `id`；`parameter/meta` jsonb | [V3](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V3__add_trigger.sql#L29-L46) |
| trigger | event_trigger_task_record | PK `id`；`meta` jsonb | [V3](file:///workspace/MaxKB4j/maxkb4j-start/src/main/resources/db/migration/V3__add_trigger.sql#L51-L70) |
