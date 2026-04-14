# Tasks
- [x] Task 1: 汇总模块与功能域边界
  - [x] 扫描 `maxkb4j-service` 的模块列表与每个模块的 Controller/Service/Entity 分布
  - [x] 汇总 `maxkb4j-start` 入口、配置与数据库迁移脚本（Flyway）
  - [x] 形成“功能域 → 模块/包路径 → 关键类”索引

- [x] Task 2: 生成“功能实现梳理”报告
  - [x] 按功能域输出：能力描述、关键接口路由、关键 DTO/VO、关键服务方法、关键数据表/字段、权限点
  - [x] 对外开放接口单独成章（如 Chat API / OpenAI 兼容接口 / MCP 接口 / embed）

- [x] Task 3: 设计“迁移到若依VUE”的最佳方案与备选方案
  - [x] 明确目标若依形态（默认 RuoYi-Vue 单体；如后续用户指定则调整）
  - [x] 给出推荐架构：集成模式（独立子系统 vs 合并到若依后端）、部署形态、网关/反向代理策略
  - [x] 设计鉴权/权限对齐方案（Sa-Token ↔ 若依 Spring Security/Token 体系的迁移路径）
  - [x] 设计数据整合策略（库表合并 vs 独立 schema；增量迁移与回滚）

- [x] Task 4: 为每个功能域生成迁移分析结果（逐域交付）
  - [x] system：登录/用户/权限/系统设置/目录与资源授权
  - [x] oss：文件上传与存储
  - [x] model：模型供应商/模型配置/参数与图标资源
  - [x] tool：工具管理（HTTP/脚本/MCP/技能等）
  - [x] knowledge：知识库/文档/段落/命中测试/向量化/发布/版本
  - [x] application：应用管理/发布/导入导出/访问控制/统计/对话记录
  - [x] chat：对话接口（SSE）、分享/嵌入、OpenAI兼容、MCP 接口
  - [x] workflow：工作流编排与执行（如存在：节点类型、运行记录、调试）
  - [x] trigger：定时任务/Webhook/触发记录

- [x] Task 5: 验证完整性与可落地性
  - [x] 覆盖全部 Controller（以 `@RestController` 扫描结果为准）
  - [x] 覆盖全部 Flyway 迁移脚本中的表
  - [x] 输出若依侧的权限字符串命名规范与菜单/路由生成建议
  - [x] 给出最小可行迁移分期（P0/P1/P2）与验收清单

# Task Dependencies
- Task 2 depends on Task 1
- Task 3 depends on Task 1
- Task 4 depends on Task 2 and Task 3
- Task 5 depends on Task 4
