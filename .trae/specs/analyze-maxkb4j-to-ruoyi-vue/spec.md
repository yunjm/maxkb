# MaxKB4j 功能梳理与迁移到若依VUE方案 Spec

## Why
当前项目 MaxKB4j 功能覆盖 RAG 知识库、LLM 工作流、应用发布与对话接口等多个域。为了将其迁移/整合到若依VUE，需要先形成“功能—实现—依赖—接口—数据”的可追溯清单，再给出面向若依体系的最佳迁移落地方案。

## What Changes
- 产出一份“MaxKB4j 全功能实现梳理”报告：按功能域列出后端实现位置（模块/Controller/Service/关键模型）、关键数据表/文档结构、对外 API、权限点与关键配置。
- 产出一份“迁移到若依VUE的最佳方案”报告：给出推荐集成形态（单体/独立子系统/微服务），并对每个功能域提供迁移分析结果与落地步骤。
- **BREAKING**：无（本变更为分析与迁移方案产出，不修改业务功能）。

## Impact
- Affected specs: 架构梳理、API 清单、数据模型、鉴权与权限模型、前端路由/页面映射、迁移分期策略
- Affected code: 仅进行代码扫描与引用，不修改既有业务代码

## ADDED Requirements

### Requirement: 功能实现清单
系统 SHALL 生成覆盖 MaxKB4j 全部功能域的实现清单，并具备从功能到实现位置的可追溯性。

#### Scenario: 成功输出
- **WHEN** 以模块维度扫描 `maxkb4j-service/*` 与 `maxkb4j-start` 入口工程
- **THEN** 输出每个功能域的：
  - 模块归属（如 system/model/tool/chat/knowledge/oss/application/workflow/trigger）
  - 入口接口（Controller 路由、关键 DTO/VO）
  - 关键业务服务（Service 接口/实现类与核心方法）
  - 数据依赖（Flyway SQL 表、MongoDB collection（如存在）、外部服务/模型）
  - 权限点（`PermissionEnum`、自定义注解如 `@SaCheckPerm` 的使用方式）
  - 与 UI 的对应关系（如可从静态资源或接口命名推断的页面/菜单能力；若无源码则明确说明）

### Requirement: 若依VUE迁移方案（最佳路径）
系统 SHALL 提供一个推荐的“迁移到若依VUE”的最佳方案，并同时给出可选方案与适用条件。

#### Scenario: 迁移方案对比
- **WHEN** 评估若依VUE常见形态（单体 RuoYi-Vue / 增强版 / Cloud 微服务）与 MaxKB4j 的模块边界
- **THEN** 输出：
  - 推荐方案（默认基线：RuoYi-Vue 单体版；若用户后续指定版本，则按指定版本调整）
  - 备选方案（独立子系统 + SSO；或若依Cloud微服务化）
  - 关键决策点：鉴权/权限如何对齐、菜单与路由如何承载、数据迁移与表结构整合策略、发布与运维方式

### Requirement: 每功能域迁移分析结果
系统 SHALL 对每个功能域分别生成迁移分析结果，且同一结构模板输出，便于对比与落地。

#### Scenario: 单个功能域的迁移分析（模板）
- **WHEN** 选择任一功能域（例如：知识库管理）
- **THEN** 输出至少包含：
  - 现状实现：模块、关键类/接口、关键表/字段、关键流程
  - 若依侧承载：后端（controller/service/mapper/权限标识）、前端（菜单/路由/页面形态）
  - 数据迁移：表/字段映射、主键策略、增量迁移与回滚策略
  - 权限迁移：从 `PermissionEnum` 到 若依权限字符串（如 `system:user:list`）的映射建议
  - 风险与复杂度：第三方依赖、性能/并发、兼容性、测试与验收要点
  - 推荐落地顺序：依赖关系与分期（P0/P1/P2）

## MODIFIED Requirements
无。

## REMOVED Requirements
无。

## Assumptions & Constraints
- 若用户未指定，默认以 RuoYi-Vue（经典单体版）为迁移基线，并在报告中注明迁移到 RuoYi-Cloud/Plus 时的差异点。
- 代码仓库中前端为 `maxkb4j-start/src/main/resources/static/admin` 的编译产物；若缺少可编辑的前端源码，则迁移前端页面将采用“按接口重建页面/复刻交互”的方式，或以 iframe/反向代理短期过渡。
- 迁移分析优先覆盖后端能力与数据结构；UI 映射以“菜单/页面能力级别”进行归纳，不强行反编译静态产物。
