# NovelFilm Studio 详细开发 TODO 与验收标准

本文基于 `docs/TwinFlim_PRD.md` 与 `docs/TwinFlim_tech.md` 整理，目标是让后续 goal 模型可以直接按清单开发、验证、修复，直到形成一个可运行的完整项目。

当前仓库基本为空壳，因此本 TODO 按“从 0 到可交付系统”的顺序组织。每个模块包含：开发任务、细分功能点、验收标准、建议测试方式。

## 0. 总体验收口径

### 0.1 必须交付的系统能力

- [ ] 私有化后台系统可本地一键启动。
- [ ] 管理员可登录并管理组织、部门、用户、角色、权限。
- [ ] 用户可创建项目，选择“上传小说”或“原创 IP”入口。
- [ ] 项目可自动加载节点化生产工作流。
- [ ] 节点画布可编辑、保存、运行、查看日志、处理失败节点。
- [ ] mock AI 网关可覆盖 LLM、文生图、文生视频、TTS、Embedding 等能力，保证无真实 API key 也能端到端运行。
- [ ] 原创 IP 流程可生成大纲、人设、章节正文、校验结果。
- [ ] Toonflow 编剧流程可生成分集规划、剧本、润色稿、分镜。
- [ ] MiroFish 图谱流程可抽取实体关系并展示知识图谱。
- [ ] 渲染流程可生成 mock 镜头视频，并用 FFmpeg 合成一个真实可播放 MP4。
- [ ] 协同审片可播放成片、添加时间帧批注、打回修改，打回任务能回流到画布。
- [ ] 积分成本从第一次 AI 调用开始记录，能按项目、厂商、能力类型查询。
- [ ] 所有关键写操作都有审计日志。
- [ ] Docker Compose 可启动完整依赖与应用。

### 0.2 全局 Definition of Done

每个功能点完成时必须满足：

- [ ] 后端 API 有明确请求/响应 schema。
- [ ] 前端页面无空白死链，loading、empty、error 状态完整。
- [ ] 权限校验同时存在于前端路由/菜单与后端 API。
- [ ] 数据变更有审计日志，任务型操作有运行日志。
- [ ] 异步任务状态可恢复，不依赖浏览器页面保持打开。
- [ ] mock 数据与真实数据库数据边界清晰，不能只做静态假页面。
- [ ] 至少有 smoke test 或手工验收步骤。
- [ ] README 或模块文档说明如何运行和验证。

### 0.3 推荐优先级

- P0：端到端闭环必须完成，否则项目不可交付。
- P1：核心产品体验必须完成，否则无法体现 PRD 价值。
- P2：企业级增强能力，可在 P0/P1 完成后补齐。

## 1. 工程初始化与开发环境

### 1.1 Monorepo 结构 P0

开发任务：

- [ ] 创建 `backend/`，承载 FastAPI、Celery、SQLAlchemy、LangGraph。
- [ ] 创建 `frontend/`，承载 React、TypeScript、Vite、Ant Design。
- [ ] 创建 `infra/`，承载 Docker Compose、Nginx、MinIO 初始化配置。
- [ ] 创建 `scripts/`，承载开发启动、数据初始化、测试脚本。
- [ ] 创建 `fixtures/`，承载 seed 数据、mock 小说、mock 视频片段。
- [ ] 创建 `docs/architecture.md`，说明实际落地架构。
- [ ] 创建 `.env.example`，列出全部环境变量。
- [ ] 创建项目根 README，包含快速启动、默认账号、服务端口、常见问题。

建议目录：

```text
backend/
  app/
    api/
    core/
    db/
    models/
    schemas/
    services/
    engines/
    workflows/
    workers/
    websocket/
  alembic/
  tests/
frontend/
  src/
    api/
    app/
    components/
    layouts/
    pages/
    routes/
    stores/
    styles/
    types/
  tests/
infra/
  docker-compose.yml
  docker-compose.prod.yml
  nginx/
scripts/
fixtures/
```

验收标准：

- [ ] `rg --files` 能看到上述主目录。
- [ ] README 中的启动命令真实可执行。
- [ ] `.env.example` 覆盖数据库、Redis、RabbitMQ、MinIO、Neo4j、JWT、AI provider、LangSmith 开关。

测试方式：

- [ ] 执行 `ls backend frontend infra scripts fixtures docs`。
- [ ] 按 README 从空环境启动一次。

### 1.2 后端脚手架 P0

开发任务：

- [ ] 初始化 Python 3.11+ 项目。
- [ ] 引入 FastAPI、Uvicorn、Pydantic v2、SQLAlchemy 2、asyncpg、Alembic。
- [ ] 引入 Redis、Celery、RabbitMQ 客户端配置。
- [ ] 引入 MinIO SDK、Neo4j async driver。
- [ ] 引入 LangChain、LangGraph。
- [ ] 建立 `app/main.py`。
- [ ] 建立统一配置 `app/core/config.py`，按环境变量加载。
- [ ] 建立统一响应结构：`code`、`message`、`data`、`request_id`。
- [ ] 建立统一异常处理：参数错误、鉴权失败、权限不足、资源不存在、业务错误、系统错误。
- [ ] 建立请求日志中间件，注入 request_id。
- [ ] 建立数据库 session dependency。
- [ ] 建立 Alembic 迁移。
- [ ] 建立 `/health`、`/ready`。
- [ ] 建立 `/api/v1` 路由前缀。

验收标准：

- [ ] `GET /health` 返回应用版本、环境、当前时间。
- [ ] `GET /ready` 能检查 PostgreSQL、Redis、RabbitMQ、MinIO、Neo4j 连接状态。
- [ ] API 报错返回统一结构，不泄露 Python traceback。
- [ ] OpenAPI 文档按模块 tag 分组。

测试方式：

- [ ] `curl http://localhost:8000/health`。
- [ ] 关闭 Redis 后访问 `/ready`，能看到 Redis unhealthy，服务不崩溃。

### 1.3 前端脚手架 P0

开发任务：

- [ ] 初始化 Vite + React + TypeScript。
- [ ] 引入 Ant Design 5、React Router、Zustand、@xyflow/react、@react-three/fiber、drei、socket.io-client、lucide-react。
- [ ] 建立 `AppLayout`：左侧导航、顶部工具栏、内容区。
- [ ] 建立路由守卫：未登录跳转登录页，权限不足显示 403。
- [ ] 建立 API client，自动携带 token，统一处理 401/403/500。
- [ ] 建立全局通知、错误提示、确认弹窗。
- [ ] 建立页面状态组件：Loading、Empty、Error、Forbidden。
- [ ] 建立基础主题：企业后台、紧凑密度、浅色优先。
- [ ] 左侧导航覆盖 8 大模块。

验收标准：

- [ ] `npm run dev` 可启动。
- [ ] 未登录访问任意业务页会跳转登录。
- [ ] 登录后左侧菜单与当前路由高亮一致。
- [ ] 刷新页面后登录态保持。
- [ ] API 401 会清理 token 并回到登录页。

测试方式：

- [ ] 浏览器访问 `/login`。
- [ ] 登录后访问 `/dashboard`、`/projects`、`/creative/canvas`。
- [ ] 手工删除 token 后刷新业务页。

### 1.4 Docker 开发环境 P0

开发任务：

- [ ] `infra/docker-compose.yml` 启动 PostgreSQL 15、Redis 7、RabbitMQ、MinIO、Neo4j 5。
- [ ] 增加 backend service。
- [ ] 增加 celery worker service。
- [ ] 增加 celery beat service，如暂不需要可保留 disabled profile。
- [ ] 增加 frontend service 或保留本地 dev 说明。
- [ ] 增加 MinIO bucket 初始化脚本。
- [ ] 增加 Neo4j 默认账号配置。
- [ ] 增加 PostgreSQL volume、Redis volume、MinIO volume、Neo4j volume。
- [ ] 创建 `scripts/dev-up.sh`、`scripts/dev-down.sh`、`scripts/seed.sh`。

验收标准：

- [ ] `docker compose -f infra/docker-compose.yml up -d` 后所有依赖 healthy。
- [ ] 后端容器能自动执行迁移或提供明确迁移命令。
- [ ] seed 后存在默认管理员账号。
- [ ] MinIO 中存在业务 bucket：assets、renders、outputs、reviews。

测试方式：

- [ ] `docker compose ps`。
- [ ] 使用默认管理员登录前端。
- [ ] 在 MinIO 控制台查看 bucket。

## 2. 统一数据模型与基础中台

### 2.1 组织、部门、用户、角色 P0

数据模型：

- [ ] `organization`：id、name、status、created_at、updated_at。
- [ ] `department`：id、org_id、parent_id、name、path、status。
- [ ] `user`：id、org_id、department_id、email、phone、username、password_hash、display_name、avatar_url、status、last_login_at。
- [ ] `role`：id、org_id、code、name、scope、is_system。
- [ ] `permission`：id、code、name、module、action。
- [ ] `user_role`：user_id、role_id、project_id nullable。
- [ ] `role_permission`：role_id、permission_id。
- [ ] `audit_log`：actor_id、action、resource_type、resource_id、before_json、after_json、ip、user_agent、created_at。

接口功能：

- [ ] 登录：账号密码登录。
- [ ] 刷新 token。
- [ ] 登出。
- [ ] 获取当前用户信息。
- [ ] 用户 CRUD、启用/禁用、重置密码。
- [ ] 部门 CRUD、树形查询。
- [ ] 角色 CRUD、分配权限。
- [ ] 给用户分配全局角色或项目角色。
- [ ] 权限列表查询。

前端功能：

- [ ] 登录页。
- [ ] 用户管理表格：搜索、筛选状态、创建、编辑、禁用、重置密码。
- [ ] 部门树管理。
- [ ] 角色权限配置页，按模块展示权限复选框。
- [ ] 当前用户个人菜单：退出登录。

验收标准：

- [ ] 超级管理员能创建部门、用户、角色。
- [ ] 制作人员不能访问系统管理 API，即使手工请求也返回 403。
- [ ] 被禁用用户无法登录。
- [ ] 重置密码后旧密码失效。
- [ ] 每次创建、编辑、禁用用户都有审计日志。

测试方式：

- [ ] API 测试覆盖登录成功、密码错误、禁用账号、权限不足。
- [ ] 手工用普通用户访问 `/system/users`。

### 2.2 项目中心 P0

数据模型：

- [ ] `project`：id、org_id、department_id、name、code、type、status、cover_url、owner_id、progress、source_type、description、created_at、updated_at、archived_at、deleted_at。
- [ ] `project_member`：project_id、user_id、role_in_project、permissions_json。
- [ ] `project_setting`：project_id、episode_count、episode_duration_sec、target_platform、orientation、style、resolution、language、quality_preset。
- [ ] `project_quota`：project_id、monthly_limit、used_amount、warning_threshold。
- [ ] `project_template`：id、name、category、config_json、workflow_template_id、share_scope、owner_id、is_official。
- [ ] `project_source`：project_id、source_type、novel_text_url、original_prompt、metadata_json。

接口功能：

- [ ] 项目列表，支持分页、关键字、状态、类型、部门、负责人、创建时间筛选。
- [ ] 项目卡片字段：封面、名称、类型、进度、状态、负责人、更新时间。
- [ ] 项目创建：上传小说。
- [ ] 项目创建：新建原创 IP。
- [ ] 项目基础信息编辑。
- [ ] 项目成员增删改。
- [ ] 项目额度调整。
- [ ] 项目归档、删除、恢复。
- [ ] 项目模板保存、复制、共享范围设置。
- [ ] 官方模板 seed。

前端功能：

- [ ] 项目列表支持卡片/列表切换。
- [ ] 项目筛选栏支持组合筛选。
- [ ] 新建项目向导分步表单：入口选择、基础信息、剧集参数、团队与额度、确认。
- [ ] 上传小说时支持 txt/md 文件，显示文件名、大小、字数估算。
- [ ] 原创 IP 创建时支持一句话选题、题材模板、生成模式。
- [ ] 项目详情含概览、成员、设置、成本、工作流入口。
- [ ] 回收站页面支持恢复和永久删除。

验收标准：

- [ ] 项目负责人能创建项目并自动成为项目成员。
- [ ] 上传小说项目能保存原文到对象存储，并在数据库记录 URL。
- [ ] 原创 IP 项目能保存原始创意 prompt。
- [ ] 非项目成员不能查看项目详情。
- [ ] 项目归档后不能继续运行工作流。
- [ ] 删除项目进入回收站，不立即物理删除。

测试方式：

- [ ] 创建 3 个不同状态项目，验证筛选。
- [ ] 用非成员账号直接请求项目详情 API，应 403 或 404。

### 2.3 资产中心 P0/P1

数据模型：

- [ ] `asset`：id、org_id、project_id nullable、type、name、description、cover_url、status、owner_id、share_scope、tags_json、metadata_json。
- [ ] `asset_version`：id、asset_id、version_no、file_url、preview_url、prompt、params_json、quality_score、created_by。
- [ ] `asset_usage`：asset_id、project_id、workflow_node_id、used_by、credit_refund、created_at。
- [ ] `asset_tag`：org_id、name、type。

资产类型细节：

- [ ] 角色资产：定妆图、外貌描述、性格、人设、提示词模板、绑定音色、多版本。
- [ ] 场景资产：场景图、场景描述、时代、风格、光影参数。
- [ ] 音频资产：配音音色、BGM、音效、时长、格式。
- [ ] 道具资产：关键道具图、描述、所属世界观。
- [ ] 工作流模板资产：节点组合、参数预设、适用题材。

接口功能：

- [ ] 资产列表分页、搜索、标签筛选、类型筛选、共享范围筛选。
- [ ] 创建资产。
- [ ] 上传资产文件到 MinIO。
- [ ] 添加资产版本。
- [ ] 编辑资产元数据。
- [ ] 批量导入、批量导出、批量打标签。
- [ ] 资产复用记录。
- [ ] 资产复用返还积分。

前端功能：

- [ ] 五类资产库分别有入口和筛选。
- [ ] 卡片/列表视图切换。
- [ ] 资产详情页展示元数据、版本历史、关联项目、使用次数。
- [ ] 角色资产详情能展示定妆图、多版本、提示词模板、绑定音色。
- [ ] 资产可从详情页“发送到画布”或在画布中拖拽使用。

验收标准：

- [ ] 上传图片资产后能在前端预览。
- [ ] 资产设置为“项目内”时，其他项目不可见。
- [ ] 使用已有资产创建工作流节点时会记录 asset_usage。
- [ ] 复用资产后生成一条积分返还或节约成本记录。

测试方式：

- [ ] 创建角色资产并生成两个版本。
- [ ] 在项目 A 复用资产，确认项目 B 无权限时不可见。

### 2.4 积分与成本管理 P0

数据模型：

- [ ] `credit_account`：owner_type、owner_id、balance、frozen、monthly_limit。
- [ ] `credit_transaction`：account_id、project_id、biz_type、biz_id、direction、amount、balance_after、reason、metadata_json。
- [ ] `provider`：name、type、base_url、api_key_encrypted、status、priority、quota_json、routing_tags_json。
- [ ] `provider_cost_rule`：provider_id、capability、model、unit、unit_price、credit_rate。
- [ ] `provider_call_log`：provider_id、project_id、capability、model、request_id、status、input_units、output_units、credits, cost_amount、latency_ms、error_message。
- [ ] `quota_alert`：scope_type、scope_id、threshold、current_value、status。

接口功能：

- [ ] 查询积分总览：总积分、已用、剩余、本月趋势。
- [ ] 部门/项目/用户额度分配。
- [ ] 额度预警阈值设置。
- [ ] 消耗明细分页查询。
- [ ] 按日、项目、能力类型、厂商筛选。
- [ ] 成本导出 CSV。
- [ ] 单集平均成本统计。
- [ ] 资产复用节约成本统计。

验收标准：

- [ ] 每次 AI provider 调用必须生成 provider_call_log。
- [ ] 成功调用必须扣减积分。
- [ ] 失败但已产生 provider 成本的调用也要记录成本状态。
- [ ] 余额不足时任务不应投递到 provider。
- [ ] 超过预警阈值时生成告警通知。

测试方式：

- [ ] 设置项目余额为 1，提交需要 10 积分任务，应被拒绝。
- [ ] 运行 3 次 mock LLM，查看消耗明细。

## 3. 多云 AI 算力网关 P0

### 3.1 能力抽象

统一能力：

- [ ] `llm.generate`：剧本、摘要、校验、提示词生成。
- [ ] `image.generate`：角色图、场景图、参考图。
- [ ] `image.transform`：图生图、风格迁移。
- [ ] `video.generate`：镜头视频。
- [ ] `tts.generate`：配音。
- [ ] `asr.transcribe`：预留。
- [ ] `embedding.create`：检索与图谱。

请求 schema：

- [ ] capability。
- [ ] project_id。
- [ ] prompt。
- [ ] negative_prompt。
- [ ] model。
- [ ] provider_id optional。
- [ ] routing_strategy。
- [ ] params_json。
- [ ] input_assets。
- [ ] expected_output。

响应 schema：

- [ ] status。
- [ ] text。
- [ ] artifacts。
- [ ] provider。
- [ ] model。
- [ ] usage。
- [ ] credits_charged。
- [ ] latency_ms。
- [ ] trace_id。
- [ ] error。

### 3.2 Mock Provider P0

开发任务：

- [ ] LLM mock 根据 prompt 类型返回结构化 JSON。
- [ ] 图片 mock 生成简单占位 PNG，包含标题、水印、角色/场景名称。
- [ ] 视频 mock 生成短 MP4 或返回 fixtures 中的视频片段。
- [ ] TTS mock 生成静音音频或 fixture 音频。
- [ ] Embedding mock 返回固定维度向量。
- [ ] 支持 `MOCK_PROVIDER_FAIL_RATE` 环境变量模拟失败。
- [ ] 支持 `MOCK_PROVIDER_LATENCY_MS` 模拟延迟。

验收标准：

- [ ] 无任何真实 API key 时，完整生产流程仍可跑通。
- [ ] mock 返回的 artifact 都真实存在于 MinIO 或本地对象存储映射中。
- [ ] fail_rate 设置为 1 时任务失败，并触发重试/熔断逻辑。

### 3.3 路由、限流、熔断 P0/P1

开发任务：

- [ ] provider registry。
- [ ] 路由策略：指定厂商、成本优先、质量优先、负载均衡。
- [ ] per-provider 并发限制。
- [ ] 请求超时控制。
- [ ] 失败重试，支持指数退避。
- [ ] 熔断：连续失败达到阈值后进入 open 状态。
- [ ] 半开恢复：过一段时间试探调用。
- [ ] 降级：首选 provider 不可用时切换备用 provider。

验收标准：

- [ ] 禁用 provider 后不会被路由选中。
- [ ] 连续失败达到阈值后 provider 状态变为熔断。
- [ ] 熔断期间任务路由到备用 provider。
- [ ] 所有路由决策写入调用日志 metadata。

测试方式：

- [ ] 配两个 mock provider，一个 fail_rate=1，一个正常，验证自动切换。

## 4. 粗粒度 DAG 工作流引擎 P0

### 4.1 数据模型

- [ ] `workflow_template`：name、category、share_scope、definition_json、preview_url。
- [ ] `workflow_instance`：project_id、template_id、name、status、layout_json、version、last_saved_at。
- [ ] `workflow_node`：workflow_id、node_key、type、subtype、title、status、position_x、position_y、config_json、input_schema_json、output_schema_json、version。
- [ ] `workflow_edge`：workflow_id、source_node_key、target_node_key、source_handle、target_handle、condition_json。
- [ ] `workflow_node_run`：node_id、run_id、status、started_at、finished_at、duration_ms、credits_used、error_code、error_message、input_json、output_json。
- [ ] `workflow_run_log`：workflow_id、node_id、run_id、level、message、metadata_json、created_at。

节点类型：

- [ ] 输入类：小说输入、创意输入、资产输入。
- [ ] 叙事类：原创 IP、图谱构建、剧情推演、剧本生成、一致性校验。
- [ ] 资产类：角色资产、场景资产、音频资产、道具资产。
- [ ] 制作类：分镜生成、3D 预演、镜头渲染、后期合成。
- [ ] 审核类：剧本审核、资产审核、成片审片、合规审核。
- [ ] 输出类：成片输出、打包下载、发布插件。
- [ ] 逻辑类：条件分支、并行分支、等待、人工确认。

### 4.2 调度能力

- [ ] 拓扑排序。
- [ ] 环检测。
- [ ] 依赖状态检查。
- [ ] 并行节点识别。
- [ ] 运行全流程。
- [ ] 运行选中节点。
- [ ] 运行选中分支。
- [ ] 暂停工作流。
- [ ] 取消工作流。
- [ ] 重试失败节点。
- [ ] 跳过节点。
- [ ] 禁用节点。
- [ ] 节点参数版本保存。
- [ ] 节点输入输出快照保存。

验收标准：

- [ ] 有环工作流不能保存为可运行状态，并给出具体成环节点。
- [ ] 上游失败时下游不能自动运行。
- [ ] 节点重试会创建新的 node_run，不覆盖历史记录。
- [ ] 刷新页面后画布布局和节点状态不丢失。
- [ ] 后端重启后 running/pending 任务能恢复或标记可重试。

测试方式：

- [ ] 构造 A->B->C 正常运行。
- [ ] 构造 A->B->A，应保存失败。
- [ ] 人为让 B 失败，验证 C 不运行。

### 4.3 默认工作流模板 P0

原创 IP 全链路模板：

- [ ] 创意输入。
- [ ] 原创 IP 生成。
- [ ] 知识图谱构建。
- [ ] 剧情推演。
- [ ] 剧本生成。
- [ ] 角色资产生成。
- [ ] 场景资产生成。
- [ ] 分镜生成。
- [ ] 镜头渲染。
- [ ] 后期合成。
- [ ] 合规审核。
- [ ] 成片输出。

上传小说双轨模板：

- [ ] 小说输入。
- [ ] 小说解析。
- [ ] 知识图谱构建。
- [ ] 资产轨：角色资产、场景资产、音频资产。
- [ ] 制作轨：分集规划、剧本、分镜、渲染、后期。
- [ ] 审片节点。
- [ ] 输出节点。

精品 3D 预演模板：

- [ ] 剧本输入。
- [ ] 分镜生成。
- [ ] 3D 导演台预演。
- [ ] 参考图输出。
- [ ] 对齐渲染。
- [ ] 审片。
- [ ] 输出。

验收标准：

- [ ] 新建项目时可选择模板。
- [ ] 模板加载后节点和连线完整。
- [ ] 模板中的节点都有默认参数，能直接运行 mock 流程。

## 5. LangGraph Agent 编排 P0/P1

### 5.1 LangGraph Runtime P0

开发任务：

- [ ] 建立 `backend/app/engines/langgraph_runtime/`。
- [ ] 封装 PostgresSaver。
- [ ] 新增 `langgraph_task`：biz_type、biz_id、thread_id、status、started_at、finished_at。
- [ ] 新增 `langgraph_step_log`：task_id、step_name、status、input_summary、output_summary、token_usage、duration_ms、error。
- [ ] 统一图启动 API。
- [ ] 统一图恢复 API。
- [ ] 统一图状态查询 API。
- [ ] 支持最大重试次数。
- [ ] 支持人工介入状态：waiting_human。
- [ ] LangSmith 配置为可选，未配置时不影响运行。

验收标准：

- [ ] 每次 LangGraph 执行都能通过 thread_id 查询。
- [ ] 服务重启后可恢复未完成图。
- [ ] 前端能看到步骤时间轴。
- [ ] 失败步骤有错误原因。

### 5.2 Toonflow 编剧状态机 P0

状态字段：

- [ ] project_id。
- [ ] novel_text。
- [ ] style_config。
- [ ] novel_analysis。
- [ ] episode_outline。
- [ ] episode_script。
- [ ] polished_script。
- [ ] storyboard。
- [ ] check_result。
- [ ] retry_count。
- [ ] max_retry。

Agent 节点：

- [ ] `NovelAnalyzerAgent`：提取世界观、主线、人物、冲突、章节结构。
- [ ] `EpisodePlannerAgent`：生成季/集规划、每集核心事件、钩子、反转。
- [ ] `ScriptWriterAgent`：按集生成标准影视剧本。
- [ ] `DialoguePolisherAgent`：优化台词口语化、节奏、角色差异。
- [ ] `StoryboardAgent`：拆分镜头，生成景别、机位、动作、时长。
- [ ] `ConsistencyCheckerAgent`：检查人设、时间线、伏笔、逻辑、合规。

图结构：

- [ ] analyze -> plan -> write -> polish -> storyboard -> check。
- [ ] check passed -> END。
- [ ] check failed and retry_count < max_retry -> write。
- [ ] check failed and retry_count >= max_retry -> waiting_human。

输出落库：

- [ ] `script_episode`。
- [ ] `script_scene`。
- [ ] `script_line`。
- [ ] `storyboard_shot`。
- [ ] `script_check_result`。

验收标准：

- [ ] 输入一段 mock 小说后能生成至少 3 集分集规划。
- [ ] 每集至少包含场景、人物、动作、对白。
- [ ] 每个分镜包含镜号、景别、画面描述、时长、角色、场景。
- [ ] 校验失败时会自动回退重写，最多不超过 max_retry。
- [ ] 超过 max_retry 后状态变为待人工处理。

测试方式：

- [ ] 让 mock checker 第一次失败第二次通过，验证循环。
- [ ] 让 mock checker 永远失败，验证 waiting_human。

### 5.3 MiroFish 图谱与推演 P1

图谱抽取：

- [ ] 文本分块。
- [ ] 实体抽取：人物、场景、事件、物品、势力。
- [ ] 关系抽取：认识、亲属、隶属、敌对、拥有、发生于、推动。
- [ ] 实体对齐：同名、别名、相似描述合并。
- [ ] Neo4j 批量写入。
- [ ] 图谱构建状态与错误记录。

GraphRAG：

- [ ] 按人物查询子图。
- [ ] 按事件查询上下文。
- [ ] 按章节/时间线查询关系变化。
- [ ] 输出 LLM 可用上下文片段。

多智能体推演：

- [ ] CharacterAgent：基于人设、目标、记忆生成行动。
- [ ] RefereeAgent：校验主线偏离、逻辑冲突、角色行为一致性。
- [ ] Memory：记录角色经历、关系变化、重要事件。
- [ ] LangGraph：角色行动 -> 裁判校验 -> 事件更新 -> 继续/结束。
- [ ] 支持 max_steps。
- [ ] 支持主线偏离时生成修正建议。

验收标准：

- [ ] 上传小说后知识图谱至少抽取 5 类实体中的 3 类。
- [ ] 图谱中人物关系可视化展示。
- [ ] 修改实体属性后，后续 GraphRAG 查询使用新值。
- [ ] 推演结果能写回事件历史。
- [ ] 裁判发现冲突时能输出具体冲突点。

### 5.4 原创 IP 生成引擎 P0/P1

生成模式：

- [ ] 极速模式：一句话选题 + 题材模板 -> 大纲、人设、章节正文。
- [ ] 定制模式：人物、节奏、文风、关键剧情可配置。
- [ ] 推演模式：调用 MiroFish 多智能体生成剧情演化。
- [ ] 衍生模式：基于已有项目图谱生成番外、前传、多结局。

参数：

- [ ] 题材。
- [ ] 目标字数。
- [ ] 时代背景。
- [ ] 核心风格。
- [ ] 目标受众。
- [ ] 主角/配角设定。
- [ ] 核心主线。
- [ ] 爽点密度。
- [ ] 伏笔数量。
- [ ] 影视化适配开关。

结果：

- [ ] 大纲。
- [ ] 人设表。
- [ ] 章节列表。
- [ ] 正文。
- [ ] 影视化改编建议。
- [ ] 知识图谱入口。
- [ ] 发起剧集生产入口。

验收标准：

- [ ] 极速模式只填一句话也能完成生成。
- [ ] 生成过程有阶段进度：大纲、章节、正文、校验。
- [ ] 用户可编辑正文并保存版本。
- [ ] 点击“发起剧集生产”会创建或加载工作流实例。
- [ ] 批量生成多个方向时，每个方向有改编潜力评分。

## 6. Moyin 渲染品控与 FFmpeg 后期 P0

### 6.1 渲染任务管理

数据模型：

- [ ] `render_task`：project_id、episode_id、shot_id、type、status、provider_id、credits_used、started_at、finished_at。
- [ ] `render_artifact`：task_id、artifact_type、url、metadata_json。
- [ ] `render_retry`：task_id、attempt_no、reason、error_message。
- [ ] `quality_check`：task_id、check_type、score、passed、detail_json。

功能：

- [ ] 从分镜批量创建镜头渲染任务。
- [ ] 单任务重试。
- [ ] 批量重试。
- [ ] 暂停、取消。
- [ ] 查看参数、输出预览、日志、错误。
- [ ] 记录 provider 与成本。

验收标准：

- [ ] 分镜列表能批量下发渲染。
- [ ] 每个镜头任务有唯一 ID 与状态。
- [ ] mock 视频 artifact 可播放。
- [ ] 失败任务能重试，重试记录保留。

### 6.2 角色一致性与质检 P1

功能：

- [ ] 提示词编译器：角色固定描述 + 场景 + 镜头 + 风格 + negative prompt。
- [ ] 参考图绑定。
- [ ] seed/style/camera 参数统一。
- [ ] 角色相似度 mock 评分。
- [ ] 构图还原度 mock 评分。
- [ ] 合规检查。
- [ ] 失败后自动重试或转人工。

验收标准：

- [ ] 同一角色跨镜头使用同一角色锚定信息。
- [ ] 质检失败时任务不会直接进入后期合成。
- [ ] 质检结果展示在渲染任务详情。

### 6.3 后期合成

功能：

- [ ] FFmpeg 命令封装。
- [ ] 镜头拼接。
- [ ] 字幕生成与叠加。
- [ ] 配音音轨合成。
- [ ] BGM 混音。
- [ ] 音效混音。
- [ ] 转码 MP4。
- [ ] 输出竖屏/横屏。
- [ ] 添加水印。
- [ ] 批量打包下载。

验收标准：

- [ ] 至少 3 个 mock 镜头能合成为 1 个 MP4。
- [ ] 输出文件能在浏览器播放器播放。
- [ ] 合成任务有耗时、日志、错误记录。
- [ ] 缺少某个镜头文件时合成失败并指出缺失文件。

测试方式：

- [ ] 使用 fixtures 中的 3 段视频跑合成。
- [ ] 删除其中一段后重跑，验证错误。

## 7. 前端核心页面详细 TODO

### 7.1 总控工作台 P0

功能点：

- [ ] 数据概览卡片：本月产出集数、在产项目数、剩余积分数、任务成功率。
- [ ] 时间维度切换：今日、本周、本月。
- [ ] 我的待办：待审核剧本、待审片、异常任务、待确认资产。
- [ ] 数字制片 Agent 日报置顶。
- [ ] 在产项目 Top5：封面、名称、进度、状态、负责人。
- [ ] 异常项目红色高亮。
- [ ] 快捷入口：新建原创项目、上传小说项目、打开节点画布、资产库、审片中心。
- [ ] 快捷入口可自定义排序。

验收标准：

- [ ] 卡片点击能跳转到对应明细页。
- [ ] 待办点击能定位到具体任务。
- [ ] 无数据时显示 empty 状态，不出现空白页。

### 7.2 节点化生产画布 P0/P1

顶部工具栏：

- [ ] 面包屑。
- [ ] 项目状态标签。
- [ ] 缩放滑块。
- [ ] 适配屏幕。
- [ ] 撤销/重做。
- [ ] 视图切换：双轨、单轨、列表。
- [ ] 保存。
- [ ] 预览。
- [ ] 运行下拉：全流程、选中节点、选中分支。
- [ ] 导出。

左侧节点库：

- [ ] 7 大类节点分组。
- [ ] 搜索节点。
- [ ] 折叠分组。
- [ ] 拖拽创建节点。
- [ ] 常用模板一键加载。

画布主区：

- [ ] 双轨分层，上方资产流水线，下方制作流水线。
- [ ] 节点固定尺寸 180x88。
- [ ] 状态色条：成功绿、运行蓝、等待灰、警告橙、失败红。
- [ ] 贝塞尔连线。
- [ ] 连线中点箭头。
- [ ] 滚轮缩放。
- [ ] 空格拖拽平移。
- [ ] 框选多选。
- [ ] 右键菜单：复制、粘贴、删除、禁用、运行、查看日志。
- [ ] 右下角缩略图。
- [ ] 异常节点灯泡提示。

右侧属性抽屉：

- [ ] 参数配置。
- [ ] 输入输出。
- [ ] 运行日志。
- [ ] 版本历史。
- [ ] LangGraph 内部步骤时间轴。

底部状态栏：

- [ ] 节点总数。
- [ ] 运行中/失败节点数。
- [ ] 累计消耗积分。
- [ ] 系统状态。
- [ ] 在线协作者数。
- [ ] 最后保存时间。

验收标准：

- [ ] 新增节点、移动节点、连线后刷新页面不丢失。
- [ ] 运行节点时状态实时变化。
- [ ] 失败节点显示错误摘要和重试按钮。
- [ ] 右键删除节点会同时删除相关连线。
- [ ] 资产库拖入角色资产时自动生成资产输入节点。

### 7.3 剧本创作工作台 P1

功能点：

- [ ] 左侧季-集树。
- [ ] 每集展示状态和批注数量。
- [ ] 中间剧本编辑器支持场景、人物、对话、动作、旁白类型。
- [ ] 富文本编辑。
- [ ] 快捷输入格式。
- [ ] 选中文字添加批注。
- [ ] 校验不通过内容红色波浪线。
- [ ] hover 显示问题和建议。
- [ ] 右侧智能助手：人设一致性、时间线、伏笔回收、逻辑自洽评分。
- [ ] 一键润色，展示多个版本。
- [ ] 人物关系快查。
- [ ] 场景信息快查。

验收标准：

- [ ] 编辑后保存会生成新版本或更新时间。
- [ ] 批注绑定到文本范围。
- [ ] 一键润色不会直接覆盖原文，必须用户选择应用。
- [ ] 校验结果能跳转定位到问题文本。

### 7.4 知识图谱可视化 P1

功能点：

- [ ] 左侧实体分类：人物、场景、事件、物品、势力。
- [ ] 分类可勾选显示/隐藏。
- [ ] 实体搜索。
- [ ] 中间力导向图。
- [ ] 节点拖拽、缩放、平移。
- [ ] 连线展示关系类型。
- [ ] 底部时间轴。
- [ ] 右侧实体详情抽屉。
- [ ] 编辑实体属性。
- [ ] 添加/删除关系。
- [ ] 导出图谱图片。
- [ ] 导出实体关系 CSV/Excel。

验收标准：

- [ ] 修改实体名称后图中节点立即更新并持久化。
- [ ] 隐藏某类实体后相关节点从图中消失。
- [ ] 导出文件可打开且包含当前筛选结果。

### 7.5 3D 导演台 P1

功能点：

- [ ] 左侧分镜列表：缩略图、镜号、景别、时长、状态。
- [ ] 分镜可拖拽排序。
- [ ] 中间 Three.js 低模场景。
- [ ] 人物、道具占位模型。
- [ ] 黄色取景框。
- [ ] 左键旋转、右键平移、滚轮缩放。
- [ ] 拖拽物体。
- [ ] 工具栏：选择、移动、旋转、缩放、网格开关。
- [ ] 运镜轨迹可视化。
- [ ] 关键帧调整。
- [ ] 右侧机位参数：景别、焦距、高度、俯仰、水平角度、运镜方式、比例。
- [ ] 灯光参数：主光、辅光、环境光、色温、预设。
- [ ] 人物调度：人物、站位、朝向、动作姿态。
- [ ] 输出设置：分辨率、质量、同步渲染。
- [ ] 底部分镜时间轴和播放头。
- [ ] 生成参考图并同步到渲染节点。

验收标准：

- [ ] Canvas 非空，有可见场景、人物、取景框。
- [ ] 修改机位参数后视窗变化。
- [ ] 生成参考图后对象存储中有图片 artifact。
- [ ] 渲染节点能读取该参考图 URL。

### 7.6 协同审片中心 P0/P1

功能点：

- [ ] 左侧季-集-镜头树。
- [ ] 镜头状态标识。
- [ ] 未处理批注角标。
- [ ] 中间播放器：播放、暂停、倍速、逐帧、音量、全屏。
- [ ] 批注工具：选择、画笔、箭头、矩形、文字。
- [ ] 颜色 4 种。
- [ ] 粗细 3 档。
- [ ] 批注绑定时间帧，精度 0.1 秒。
- [ ] 批注时间轴。
- [ ] 多人协同光标。
- [ ] 右侧批注列表：按时间排序、回复、标记状态、@人员。
- [ ] 基本信息：集数元数据、渲染参数、关联任务。
- [ ] 版本对比：两个版本并排同步播放。
- [ ] 审核通过。
- [ ] 打回修改。
- [ ] 打回后生成修改任务并回流画布。
- [ ] 快捷键：空格播放/暂停、方向键逐帧、工具切换。

验收标准：

- [ ] 批注刷新后仍存在。
- [ ] 两个浏览器同时打开，同一批注实时同步。
- [ ] 打回必须填写原因。
- [ ] 打回后对应画布节点状态变为 waiting_review 或 failed_review。
- [ ] 审核通过后视频版本不可继续编辑批注，只能查看历史。

## 8. 渲染生产中心、资产中心、成本、系统管理页面

### 8.1 渲染生产中心 P0

- [ ] 渲染任务表格：任务 ID、项目、集数、镜头号、类型、状态、耗时、积分、操作。
- [ ] 筛选：项目、集数、状态、类型、时间。
- [ ] 批量重试。
- [ ] 批量暂停。
- [ ] 批量取消。
- [ ] 导出任务。
- [ ] 详情面板：参数、输出预览、日志、错误、重试记录。
- [ ] 后期合成任务页：配音、配乐、字幕、拼接、转码。
- [ ] 成片输出管理：预览、下载、格式、分辨率、水印、打包。

验收标准：

- [ ] 表格状态与后端任务状态一致。
- [ ] 批量操作只影响选中任务。
- [ ] 详情日志按时间排序。

### 8.2 算力与成本页面 P0

- [ ] 积分总览。
- [ ] 本月消耗趋势图。
- [ ] 单集平均成本。
- [ ] 资产复用节约成本。
- [ ] 部门额度分配。
- [ ] 项目额度分配。
- [ ] 用户额度分配。
- [ ] 预警阈值设置。
- [ ] 消耗明细表。
- [ ] 按日/项目/类型/厂商筛选。
- [ ] 导出。
- [ ] 厂商管理：增删改查、API key、优先级、并发配额。
- [ ] 连通性测试。
- [ ] 路由策略配置。

验收标准：

- [ ] 消耗明细合计与总览一致。
- [ ] API key 不在前端明文回显。
- [ ] 连通性测试结果写入 provider 状态或日志。

### 8.3 系统管理页面 P1/P2

- [ ] 用户管理。
- [ ] 部门管理。
- [ ] 角色权限。
- [ ] SSO 配置占位。
- [ ] 密码策略。
- [ ] 操作审计日志检索。
- [ ] 敏感词库。
- [ ] 违规规则。
- [ ] 三级审核配置。
- [ ] 服务状态监控。
- [ ] 任务成功率/失败率。
- [ ] 告警中心。
- [ ] 系统日志检索。
- [ ] 存储配置。
- [ ] 消息通知配置。
- [ ] 全局字典。
- [ ] 标签体系。
- [ ] 备份恢复设置。

验收标准：

- [ ] 审计日志可按用户、动作、资源、时间筛选。
- [ ] 敏感词新增后立即影响合规检查。
- [ ] 普通用户看不到系统管理菜单。

## 9. WebSocket 与实时协同 P1

开发任务：

- [ ] 建立 WebSocket 鉴权。
- [ ] 任务状态频道：project:{id}:tasks。
- [ ] 画布协同频道：workflow:{id}:presence。
- [ ] 审片频道：review:{id}:annotations。
- [ ] 通知频道：user:{id}:notifications。
- [ ] 在线用户心跳。
- [ ] 断线重连。
- [ ] 事件去重。

事件类型：

- [ ] node.status.changed。
- [ ] workflow.saved。
- [ ] collaborator.joined。
- [ ] collaborator.left。
- [ ] review.annotation.created。
- [ ] review.annotation.updated。
- [ ] review.cursor.moved。
- [ ] notification.created。

验收标准：

- [ ] 节点运行状态不刷新页面可更新。
- [ ] 多人打开同一画布能看到在线人数。
- [ ] 多人审片批注实时出现。
- [ ] WebSocket 断开后重连不会重复创建批注。

## 10. 端到端流程验收

### 10.1 原创 IP 全自动闭环 P0

步骤：

- [ ] 用户登录。
- [ ] 在总控工作台点击“新建原创项目”。
- [ ] 输入一句话创意。
- [ ] 选择极速模式、全托管。
- [ ] 系统创建项目。
- [ ] 跳转节点画布。
- [ ] 自动加载原创 IP 全链路模板。
- [ ] 运行全流程。
- [ ] 原创 IP 节点输出大纲、人设、正文。
- [ ] 图谱节点输出实体关系。
- [ ] 编剧节点输出分集剧本。
- [ ] 分镜节点输出镜头列表。
- [ ] 资产节点输出角色/场景资产。
- [ ] 渲染节点输出 mock 镜头视频。
- [ ] 后期节点输出 MP4 成片。
- [ ] 合规节点通过。
- [ ] 输出节点归档成片。
- [ ] 用户可预览和下载。

通过标准：

- [ ] 全流程无需真实 AI key。
- [ ] 所有节点最终成功或明确进入待人工状态。
- [ ] 每个节点都有运行日志。
- [ ] 成片 MP4 可播放。
- [ ] 成本明细中能看到本流程产生的调用记录。

### 10.2 双轨团队协产流程 P1

步骤：

- [ ] 上传小说创建项目。
- [ ] 加载双轨生产模板。
- [ ] 资产轨生成角色和场景。
- [ ] 资产审核入库。
- [ ] 制作轨自动解锁分镜和渲染。
- [ ] 编剧修改剧本。
- [ ] 修改触发下游节点需重新生成标记。
- [ ] 导演进入 3D 预演。
- [ ] 生成参考图。
- [ ] 渲染师批量下发渲染。
- [ ] 审片人员逐帧批注。
- [ ] 打回修改。
- [ ] 修改任务回流画布。
- [ ] 重新提交审片。
- [ ] 通过后归档。

通过标准：

- [ ] 资产轨和制作轨状态互相联动。
- [ ] 打回意见不会丢失。
- [ ] 二审能看到上一版和新版对比。

### 10.3 精品 3D 预演流程 P1

步骤：

- [ ] 从剧本分镜进入 3D 导演台。
- [ ] 自动加载分镜对应人物和场景。
- [ ] 调整机位、灯光、站位。
- [ ] 生成参考图。
- [ ] 参考图绑定渲染任务。
- [ ] 渲染任务根据参考图生成输出。

通过标准：

- [ ] 参考图 artifact 存在。
- [ ] 渲染详情能看到参考图。
- [ ] 参数调整能保存并再次打开。

## 11. 测试清单

### 11.1 后端自动化测试 P0

- [ ] 鉴权：登录、刷新、登出、禁用用户、权限不足。
- [ ] 数据权限：项目成员可见、非成员不可见。
- [ ] 项目：创建、编辑、归档、删除、恢复。
- [ ] 资产：上传、版本、共享范围、复用记录。
- [ ] 积分：扣减、返还、余额不足、预警。
- [ ] AI 网关：mock success、mock fail、熔断、降级。
- [ ] DAG：拓扑排序、环检测、依赖失败、节点重试。
- [ ] LangGraph：通过、重试、max_retry、恢复。
- [ ] 渲染：任务创建、失败重试、artifact 记录。
- [ ] 审片：批注、回复、打回、通过。

### 11.2 前端 E2E P0/P1

- [ ] 登录成功。
- [ ] 新建原创项目。
- [ ] 打开节点画布。
- [ ] 运行工作流。
- [ ] 查看节点日志。
- [ ] 查看剧本。
- [ ] 查看图谱。
- [ ] 查看渲染任务。
- [ ] 播放成片。
- [ ] 添加审片批注。
- [ ] 打回修改。
- [ ] 成本明细出现记录。

### 11.3 UI 视觉与交互验收 P1

- [ ] 1200px 宽度下布局不重叠。
- [ ] 1440px 宽度下信息密度合理。
- [ ] 2560px 宽度下内容不无限拉伸。
- [ ] 按钮文字不溢出。
- [ ] 表格长文本有 tooltip 或省略。
- [ ] 画布节点 hover/selected/disabled/failed 状态清晰。
- [ ] 3D canvas 非空。
- [ ] 审片播放器控件不遮挡批注列表。

### 11.4 性能与稳定性 P1/P2

- [ ] 100 个 mock 镜头任务并发提交，任务状态最终一致。
- [ ] 后端重启后 pending/running 任务可恢复或可重试。
- [ ] Celery worker 重启后未确认任务不丢。
- [ ] Redis 短暂不可用时系统给出可理解错误。
- [ ] FFmpeg 合成 1 分钟 mock 视频耗时记录在任务日志。
- [ ] WebSocket 50 个连接 smoke test。

## 12. 部署与文档

交付文档：

- [ ] 架构说明书。
- [ ] 本地开发手册。
- [ ] 私有化部署手册。
- [ ] 环境变量说明。
- [ ] API 使用说明。
- [ ] 管理员操作手册。
- [ ] 制作人员操作手册。
- [ ] 审片人员操作手册。
- [ ] 量产最佳实践。
- [ ] 验收用例集。
- [ ] 故障排查手册。

部署能力：

- [ ] backend Dockerfile。
- [ ] frontend Dockerfile。
- [ ] worker Dockerfile 或复用 backend image。
- [ ] production compose 样例。
- [ ] 数据库迁移命令。
- [ ] seed 命令。
- [ ] 数据库备份脚本。
- [ ] 数据库恢复脚本。
- [ ] MinIO 备份说明。

验收标准：

- [ ] 新机器按部署手册可启动系统。
- [ ] 默认管理员可登录。
- [ ] seed demo 项目可运行。
- [ ] 文档中的命令与实际一致。

## 13. 推荐开发顺序

按下面顺序执行，完成一个阶段后再进入下一阶段。

1. [ ] 工程脚手架、Docker Compose、后端健康检查、前端登录页。
2. [ ] 组织用户权限、JWT、RBAC、审计日志。
3. [ ] 项目中心、项目成员、项目设置、项目模板。
4. [ ] 资产中心基础能力、对象存储上传预览。
5. [ ] 积分成本模型、provider 调用日志。
6. [ ] mock AI 网关、路由、失败模拟。
7. [ ] DAG 工作流模型、调度、Celery、节点日志。
8. [ ] 节点画布最小可用版。
9. [ ] mock 渲染与 FFmpeg 合成真实 MP4。
10. [ ] Toonflow LangGraph 编剧流水线。
11. [ ] 原创 IP 生成页面与流程接入。
12. [ ] MiroFish 图谱构建与可视化。
13. [ ] 渲染生产中心与成片输出管理。
14. [ ] 协同审片、批注、打回回流。
15. [ ] 数字制片 Agent 异常处理。
16. [ ] 3D 导演台与参考图同步。
17. [ ] 系统管理、合规、监控、告警。
18. [ ] E2E、压测、部署文档、验收用例。

## 14. MVP 截断线

时间不足时，以下内容必须先完成：

- [ ] Docker Compose 能启动依赖。
- [ ] 管理员可登录。
- [ ] 项目可创建。
- [ ] 画布可加载模板并运行。
- [ ] mock AI 网关可用。
- [ ] 原创 IP 可生成 mock 大纲和正文。
- [ ] 编剧流程可生成剧本和分镜。
- [ ] 渲染流程可生成 mock 视频。
- [ ] FFmpeg 可合成 MP4。
- [ ] 审片可播放、批注、打回。
- [ ] 打回任务可回流画布。
- [ ] 成本流水可查询。
- [ ] 审计日志可查询。
- [ ] README 能指导启动和验收。

MVP 完成后，再补真实 provider、深度图谱、多智能体推演、复杂 3D、LangSmith、APISIX、高可用部署。

## 15. 不可妥协的风险控制

- [ ] 不能只做静态前端页面，核心数据必须落库。
- [ ] 不能依赖真实 AI key 才能演示，mock provider 是 P0。
- [ ] 不能让 LangGraph 替代项目级 DAG，二者职责必须分清。
- [ ] 不能把权限只做在前端，后端 API 必须校验。
- [ ] 不能后补成本，provider 调用从第一天记录成本。
- [ ] 不能覆盖任务历史，重试必须保留 run 记录。
- [ ] 不能让审片批注只存在内存，必须持久化。
- [ ] 不能让上传文件散落本地，统一走对象存储抽象。
- [ ] 不能让任务失败无原因，必须记录 error_code 和 error_message。
- [ ] 不能让文档命令失效，交付前必须按文档重跑。

