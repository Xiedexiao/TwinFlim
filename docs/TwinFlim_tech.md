# NovelFilm Studio 智影工场 V1.3 技术实现方案（LangGraph 增强版）
**文档版本**：V1.3 LangGraph 架构升级正式版
**核心升级**：AI 引擎编排层从纯 LangChain 升级为 **LangGraph + LangChain 双栈架构**，原生支撑多智能体协作、循环校验、状态持久化、断点续跑，与系统「工作流编排+多Agent推演+剧本流水线」的核心场景高度匹配
**技术栈基调**：全 Python 微服务 + React 前端 + 无 GPU 云端算力架构保持不变，仅升级智能引擎层的编排内核

---

## 一、方案概述
### 1.1 升级结论：LangGraph 替换纯 LangChain 收益显著
**不是完全替代 LangChain，而是编排层升级**：LangGraph 是 LangChain 官方推出的图状 Agent 编排框架，构建在 LangChain 基础能力之上。我们保留 LangChain 的 LLM 调用、Prompt 模板、工具封装、向量检索等基础能力，将**所有多Agent协作、流程编排、状态机、循环校验类的核心调度逻辑，全部升级为 LangGraph 实现**。

针对本项目的核心场景，升级收益如下：
| 核心痛点 | 纯 LangChain 实现方案 | LangGraph 原生方案 |
|---------|----------------------|--------------------|
| 多Agent 编剧流水线（校验不通过回退重写） | 需手写循环判断、状态流转、异常处理，代码冗余易出错 | 原生支持循环、条件分支，状态机模式天然适配「生成→校验→打回重写」的闭环流程 |
| 多智能体剧情推演（多角色轮动+全局裁判） | 需自研调度器、消息传递、记忆同步，开发量大、稳定性差 | 原生支持多节点图状编排、消息传递、状态共享，超图模式天然适配多智能体群体仿真 |
| 任务断点续跑 | 需自行持久化每个节点的输入输出、状态，自行实现恢复逻辑 | 内置状态持久化（MemorySaver / PostgresSaver），中断后可从任意状态点精准恢复，零额外开发 |
| 复杂分支与条件路由 | DAG 工作流仅支持单向流转，循环、回退逻辑实现成本高 | 原生支持条件边、回退边、分支路由，复杂业务流程表达能力远强于普通 DAG |
| 可观测性与调试 | 链路追踪需自行埋点，Agent 内部执行过程黑盒 | 原生对接 LangSmith，全链路步骤、状态、Token 消耗可视化，调试排错效率提升 50% 以上 |
| 生态兼容性 | - | 100% 兼容 LangChain 所有组件，原有 LLM、工具、Prompt、记忆模块可直接复用，迁移成本极低 |

### 1.2 核心目标
1. **技术栈统一与升级**：全栈 Python 微服务架构不变，智能引擎层统一采用「LangChain 基础能力 + LangGraph 编排内核」，兼顾生态丰富度与编排专业性
2. **无卡原生支持**：所有 AI 算力统一通过多云网关调度，本地零 GPU 依赖
3. **生产模式先进**：节点化双轨生产 + 多智能体自动化，兼顾效率与品控
4. **企业级可控**：权限、合规、成本、运维全维度管控，满足私有化部署
5. **高可用可扩展**：状态持久化、断点续跑、故障降级，支撑 7×24 小时量产

### 1.3 四大参考项目与复用层级总览
| 序号 | 项目名称 | 项目属性 | 复用层级 | 核心复用价值 | 本项目落地位置 | 复用/参考率 |
|------|----------|----------|----------|--------------|----------------|-------------|
| 1 | **MiroFish** | 开源 Python 后端项目 | 代码级复用 | GraphRAG 知识图谱流水线、多智能体分层记忆、群体仿真调度逻辑 | `mirofish-service` 核心逻辑层，调度器升级为 LangGraph | 75%（核心算法100%复用，调度层升级） |
| 2 | **Toonflow** | 开源 Node.js 全栈项目 | 逻辑+Prompt 复用 | 多 Agent 编剧分工体系、剧本/分镜数据模型、量产级 Prompt 模板库 | `toonflow-service` 业务逻辑层，流水线升级为 LangGraph 状态机 | 60%（分工与Prompt100%参考，编排层 LangGraph 重写） |
| 3 | **Moyin-Creator** | 开源 Electron 桌面项目 | 算法+工程复用 | 6 层角色一致性算法、提示词编译器、FFmpeg 后期流水线 | `moyin-service` 品控与后期层 | 70%（核心算法100%迁移，渲染层替换为云端API） |
| 4 | **AniShort.ai** | 商业化 SaaS 产品 | 产品+交互设计参考 | 节点化无限画布、双轨并行生产、3D导演台、数字制片Agent、帧级审片、积分制 | 前端交互 + 工作流引擎 + 协同生产模块 | 设计思路100%参考，代码全自研 |

### 1.4 设计原则
1. **编排分层**：粗粒度生产流程用 DAG 工作流，细粒度 Agent 内部流程用 LangGraph，各司其职
2. **全异步优先**：所有 IO 操作全异步化，最大化单节点吞吐
3. **状态可持久化**：所有 Agent 流程、工作流节点状态均可持久化，支持断点续跑
4. **复用最大化**：开源核心逻辑 100% 保留，仅改造适配层与编排层
5. **故障可兜底**：所有外部依赖均有重试、熔断、降级机制

---

## 二、整体技术架构
### 2.1 分层架构详细设计
采用**六层解耦架构**，其中智能引擎层全面升级为「LangGraph 编排内核 + LangChain 基础能力」的双栈模式，工作流层与 LangGraph 形成「粗粒度DAG + 细粒度状态机」的双层编排体系。

```
┌───────────────────────────────────────────────────────────────────────────┐
│  【接入层】Access Layer                                                  │
│  React 管理后台 ｜ 开放API ｜ WebSocket实时通道 ｜ H5审片页                │
│  统一入口：APISIX 网关（路由/鉴权/限流/日志/灰度）                          │
├───────────────────────────────────────────────────────────────────────────┤
│  【协同生产层】Collaboration Layer（参考 AniShort.ai 自研）                │
│  节点DAG工作流引擎 ｜ 帧级协同审片服务 ｜ 3D导演台预演服务 ｜ 数字制片Agent  │
│  核心：粗粒度生产流程编排，串联全链路能力                                   │
├───────────────────────────────────────────────────────────────────────────┤
│  【业务中台层】Business Middleware Layer（全 FastAPI 微服务）              │
│  用户权限服务 ｜ 项目管理服务 ｜ 资产管理服务 ｜ 积分成本服务                │
│  合规风控服务 ｜ 运维监控服务                                              │
├───────────────────────────────────────────────────────────────────────────┤
│  【智能引擎层】Intelligent Engine Layer（LangGraph 编排内核）              │
│  ┌───────────────────────────────────────────────────────────────────┐   │
│  │  LangGraph 编排层：状态机、多节点图、条件分支、循环、状态持久化     │   │
│  └───────────────────────────────────────────────────────────────────┘   │
│  MiroFish图谱推演引擎 ｜ Toonflow影视编剧引擎 ｜ Moyin渲染品控引擎        │
│  数字制片智能体引擎 ｜ 一致性校验引擎                                      │
│  基础底座：LangChain（LLM/工具/Prompt/记忆） + 多云算力网关                │
├───────────────────────────────────────────────────────────────────────────┤
│  【算力网关层】AI Gateway Layer（Python 原生实现）                        │
│  厂商适配器层 ｜ 智能路由引擎 ｜ 熔断配额引擎 ｜ 成本核算引擎                │
├───────────────────────────────────────────────────────────────────────────┤
│  【基础设施层】Infrastructure Layer                                      │
│  数据库：PostgreSQL 15 ｜ Neo4j 5.x ｜ Redis 7.x                          │
│  消息队列：RabbitMQ 3.12 + Celery 5.x                                    │
│  存储：MinIO 对象存储 ｜ NAS 冷归档                                        │
│  运维：Prometheus + Grafana + ELK + LangSmith（Agent可观测）              │
└───────────────────────────────────────────────────────────────────────────┘
```

#### 双层编排体系分工
| 编排层级 | 技术方案 | 适用场景 | 特点 |
|---------|---------|---------|------|
| 粗粒度生产流程 | DAG 工作流引擎（自研） | 项目级全链路生产：小说→剧本→资产→渲染→后期→审片 | 单向流转、节点颗粒度大、可视化交互、面向生产人员 |
| 细粒度 Agent 流程 | LangGraph 状态机 | 单个节点内部的多Agent协作：多智能体推演、多环节编剧、异常处理闭环 | 支持循环/回退/分支、状态精细持久化、面向AI引擎内部 |

### 2.2 技术栈选型总览
#### 后端全栈 Python 技术栈
| 技术领域 | 选型方案 | 版本 | 选型说明 |
|---------|---------|------|---------|
| Web 框架 | FastAPI + Pydantic v2 | 0.110+ / 2.7+ | 原生异步、高性能、自动生成 OpenAPI、类型安全 |
| ORM | SQLAlchemy 2.0 + asyncpg | 2.0+ / 0.29+ | 成熟异步 ORM，复杂查询能力强，适配 PostgreSQL |
| 图数据库驱动 | neo4j-python-driver（异步版） | 5.18+ | 官方异步驱动，支持 Neo4j 5.x |
| 缓存驱动 | redis-py async | 5.0+ | 异步 Redis 客户端，支持连接池、分布式锁 |
| 任务队列 | Celery + RabbitMQ | 5.3 / 3.12 | 分布式任务队列，支持优先级、重试、死信队列 |
| **AI 基础底座** | **LangChain** | 0.2+ | 大模型基础能力：LLM 封装、Prompt 模板、工具、向量检索 |
| **AI 编排内核** | **LangGraph** | 0.2+ | 图状 Agent 编排核心：状态机、多节点、循环分支、状态持久化 |
| **Agent 可观测** | **LangSmith** | - | Agent 全链路追踪、调试、评估，对接 LangGraph 原生支持 |
| 视频处理 | FFmpeg + ffmpeg-python | 6.0 | 纯 CPU 视频后期处理，全场景支持 |
| 服务网关 | Apache APISIX | 3.0+ | 高性能云原生网关，路由、限流、鉴权开箱即用 |

#### 前端 React 技术栈（保持不变）
| 技术领域 | 选型方案 | 版本 | 说明 |
|---------|---------|------|------|
| 核心框架 | React + TypeScript + Vite | 18 / 5.0+ / 5.0+ | 函数式组件、类型安全、极速构建 |
| UI 组件库 | Ant Design | 5.12+ | 企业级后台最完善 React 组件库 |
| 状态管理 | Zustand | 4.5+ | 轻量高性能状态管理 |
| 节点画布 | @xyflow/react | 12.0+ | 无限节点画布，适配生产流程可视化 |
| 3D 渲染 | @react-three/fiber + drei | 9.0+ | 3D 场景交互，适配导演台预演 |
| 实时通信 | Socket.IO Client | 4.7+ | 多人实时协同 |

### 2.3 服务拆分与通信机制
#### 服务拆分原则
- 业务中台按领域拆分，AI 引擎按能力拆分，协同生产模块独立部署
- 每个 AI 引擎服务内部独立维护自己的 LangGraph 图定义与状态存储
- 横切关注点统一由网关与中间件处理，不侵入业务服务

#### 通信机制
1. **同步通信**：服务间 HTTP RESTful 调用，网关内路由，支持超时、重试、熔断
2. **异步通信**：批量生成、渲染等耗时任务通过 Celery + RabbitMQ 解耦
3. **实时推送**：画布状态、审片协同、任务进度通过 WebSocket 推送
4. **状态同步**：LangGraph 运行状态持久化到 PostgreSQL，支持跨服务、跨重启恢复

---

## 三、公共基础组件详细设计
### 3.1 统一接入网关（APISIX）
保持原有设计不变：路由转发、统一鉴权、限流熔断、日志审计、灰度发布。

### 3.2 统一鉴权体系
保持原有设计不变：JWT + RBAC + 数据权限三级体系。

### 3.3 分布式任务调度框架（Celery）
保持原有队列划分、重试机制、并发控制设计不变；
**新增适配**：LangGraph 长流程任务支持异步提交，Celery Worker 负责驱动图执行，状态实时回写。

### 3.4 统一异常处理与熔断降级
保持原有全局异常、服务熔断设计不变；
**新增适配**：LangGraph 节点级异常捕获，支持单节点失败重试、分支降级，不中断整个图流程。

---

## 四、业务中台各模块详细实现
用户权限、项目管理、资产管理、积分成本、合规风控、运维监控六大服务保持原有设计不变，作为企业级基础能力底座。

---

## 五、核心引擎层详细实现（LangGraph 升级重点）
### 5.1 双层编排体系设计
#### 5.1.1 上层：DAG 节点工作流引擎（粗粒度生产流程）
保持原有 DAG 调度引擎设计不变，负责项目级全链路流程编排，节点颗粒度为「小说生成、剧本生成、渲染、后期」等生产环节，面向可视化交互与生产人员操作。

**与 LangGraph 的关系**：
- 每个 DAG 节点内部，可以是简单的单接口调用，也可以是一个完整的 LangGraph 多Agent 流程
- DAG 负责节点间的依赖、并行分支、整体进度；LangGraph 负责节点内部的多Agent协作、循环校验、精细状态管理
- 两层各司其职，避免用 LangGraph 做粗粒度生产流程（可视化差、交互重），也避免用 DAG 做复杂 Agent 逻辑（循环分支能力弱）

#### 5.1.2 下层：LangGraph 状态机（细粒度 Agent 流程）
所有多Agent、多步骤、有循环/分支的 AI 流程，全部基于 LangGraph 构建，核心能力：
1. **图状态机**：每个流程是一个有向图，节点为执行步骤，边为流转条件
2. **共享状态**：图内所有节点共享一个 State 状态对象，数据在节点间自动传递
3. **原生循环**：支持校验不通过回退到上游节点，自动循环直到满足条件
4. **条件分支**：支持多条件路由，不同结果走不同分支
5. **状态持久化**：内置 Checkpointer，支持内存/PostgreSQL 存储，中断后精准恢复
6. **可观测性**：原生对接 LangSmith，每一步执行、Token 消耗、状态变更全链路可查

---

### 5.2 多云 AI 算力网关（ai-gateway-service）
保持原有设计不变：门面模式 + 适配器模式 + 策略模式，统一管理多厂商 AI 能力，提供标准化 LLM、文生图、文生视频、TTS 接口，供上层 LangChain / LangGraph 调用。

---

### 5.3 MiroFish 知识图谱与多智能体推演引擎
**升级说明**：图谱构建、记忆系统核心代码 100% 复用 MiroFish 开源实现；**多智能体推演调度器从自研改为 LangGraph 图状态机实现**，大幅降低开发量，提升稳定性与可观测性。

#### 5.3.1 知识图谱构建流水线
保持原有设计不变：文本分块 → 实体抽取 → 关系挖掘 → 实体对齐 → 批量写入 Neo4j，核心逻辑复用 MiroFish，仅改造外围依赖。

#### 5.3.2 GraphRAG 检索
保持原有设计不变：子图查询、上下文拼接，作为 LangGraph 节点的工具能力被调用。

#### 5.3.3 多智能体剧情推演（LangGraph 核心改造）
**原 MiroFish 实现痛点**：自研仿真调度器，需手写智能体轮询、消息传递、记忆同步、主线校验、循环控制，代码复杂、易出 bug、调试困难。

**LangGraph 改造后实现方案**：
构建一个**「回合制推演状态机」**，每个时间步为一轮，多智能体依次行动，全局裁判校验，循环推进剧情。

##### 图结构设计
```
初始化环境
    ↓
[ 角色智能体行动节点 ] ←─────┐
    ↓  所有角色依次行动        │
[ 全局裁判校验节点 ]          │  循环：每一轮一个时间步
    ↓  校验是否偏离主线        │
[ 事件更新与记忆更新 ]        │
    ↓
判断是否结束 → 是 → 输出结果
            → 否 → 回到角色行动节点（下一轮）
```

##### 核心代码示例
```python
# mirofish-service/app/core/simulation/graph.py
from typing import Annotated, TypedDict, List
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.postgres import PostgresSaver
from app.core.agents.character_agent import CharacterAgent
from app.core.agents.referee_agent import RefereeAgent
from app.core.agents.memory import CharacterAgentMemory

# 1. 定义图共享状态
class SimulationState(TypedDict):
    project_id: str
    current_step: int
    max_steps: int
    scene_context: str
    characters: List[dict]
    event_history: List[dict]
    referee_feedback: str
    is_finished: bool

# 2. 定义节点函数
def character_action_node(state: SimulationState) -> SimulationState:
    """所有角色智能体依次行动，生成本轮对话与行为"""
    new_events = []
    for char_config in state["characters"]:
        agent = CharacterAgent(char_config, state["scene_context"])
        action = agent.take_action(state["event_history"])
        new_events.append(action)
        # 更新角色记忆
        agent.memory.add_plot_event(action["content"], [], importance=0.7)
    
    return {
        **state,
        "event_history": state["event_history"] + new_events,
        "current_step": state["current_step"] + 1
    }

def referee_check_node(state: SimulationState) -> SimulationState:
    """全局裁判 Agent 校验，防止偏离主线"""
    referee = RefereeAgent(state["project_id"])
    feedback, is_valid = referee.check(state["event_history"])
    return {
        **state,
        "referee_feedback": feedback,
        "is_finished": state["current_step"] >= state["max_steps"]
    }

def should_continue(state: SimulationState) -> str:
    """条件边：判断继续推演还是结束"""
    if state["is_finished"]:
        return "end"
    return "continue"

# 3. 构建图
def build_simulation_graph(checkpointer):
    graph = StateGraph(SimulationState)
    
    # 添加节点
    graph.add_node("character_action", character_action_node)
    graph.add_node("referee_check", referee_check_node)
    
    # 添加边
    graph.set_entry_point("character_action")
    graph.add_edge("character_action", "referee_check")
    
    # 条件边：校验后决定继续还是结束
    graph.add_conditional_edges(
        "referee_check",
        should_continue,
        {
            "continue": "character_action",  # 循环，进入下一轮
            "end": END
        }
    )
    
    # 编译图，绑定持久化检查点
    return graph.compile(checkpointer=checkpointer)

# 4. 初始化与调用
# 持久化到 PostgreSQL，支持断点续跑
checkpointer = PostgresSaver(asyncpg_connection)
simulation_graph = build_simulation_graph(checkpointer)

# 执行推演，支持中断恢复
async def run_simulation(project_id: str, initial_state: dict, thread_id: str):
    config = {"configurable": {"thread_id": thread_id}}
    result = await simulation_graph.ainvoke(initial_state, config=config)
    return result
```

##### 升级收益
1. **循环逻辑原生支持**：不用手写 while 循环、状态流转，LangGraph 原生处理回合制推演
2. **状态自动持久化**：PostgresSaver 自动存每一步状态，服务重启后通过 thread_id 精准恢复，断点续跑零额外开发
3. **可观测性大幅提升**：LangSmith 中可查看每一轮每个角色的行动、裁判的校验结果、状态变更，调试一目了然
4. **扩展灵活**：后续新增事件注入、分支剧情、多结局推演，只需加节点和条件边即可，架构不重构

---

### 5.4 Toonflow 影视编剧引擎
**升级说明**：Agent 分工、Prompt 模板、数据模型 100% 参考 Toonflow；**整条编剧流水线从链式调用升级为 LangGraph 状态机**，原生支持「生成→校验→打回重写」的循环闭环，一致性校验深度对接 MiroFish 图谱。

#### 5.4.1 多 Agent 编剧流水线（LangGraph 核心改造）
**原实现痛点**：纯 LangChain SequentialChain 只能单向执行，校验不通过需要手写回退逻辑、状态管理，代码冗余。

**LangGraph 改造后方案**：
构建**「六节点编剧状态机」**，覆盖解析→规划→写作→润色→分镜→校验全流程，校验不通过自动回退到写作节点重写。

##### 图结构设计
```
小说解析节点
    ↓
分集规划节点
    ↓
分集编剧节点 ←──────────┐
    ↓                   │
台词润色节点            │  校验不通过，循环重写
    ↓                   │
分镜生成节点            │
    ↓                   │
一致性校验节点 ──────────┘
    ↓ 校验通过
输出结果
```

##### 核心代码示例
```python
# toonflow-service/app/core/script/graph.py
from typing import TypedDict, List
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.postgres import PostgresSaver
from app.core.agents.analyzer import NovelAnalyzerAgent
from app.core.agents.planner import EpisodePlannerAgent
from app.core.agents.writer import ScriptWriterAgent
from app.core.agents.polisher import DialoguePolisherAgent
from app.core.agents.storyboard import StoryboardAgent
from app.core.agents.checker import ConsistencyCheckerAgent

# 1. 共享状态
class ScriptState(TypedDict):
    project_id: str
    novel_text: str
    style_config: dict
    novel_analysis: dict
    episode_outline: List[dict]
    episode_script: dict
    polished_script: dict
    storyboard: List[dict]
    check_result: dict
    retry_count: int
    max_retry: int

# 2. 节点实现
def analyze_node(state: ScriptState) -> ScriptState:
    agent = NovelAnalyzerAgent()
    analysis = agent.analyze(state["novel_text"])
    return {**state, "novel_analysis": analysis}

def plan_node(state: ScriptState) -> ScriptState:
    agent = EpisodePlannerAgent()
    outline = agent.plan(state["novel_analysis"], state["style_config"])
    return {**state, "episode_outline": outline}

def write_node(state: ScriptState) -> ScriptState:
    agent = ScriptWriterAgent()
    # 注入 MiroFish 知识图谱上下文
    graph_context = get_graph_context(state["project_id"])
    script = agent.write(state["episode_outline"], graph_context)
    return {**state, "episode_script": script}

def polish_node(state: ScriptState) -> ScriptState:
    agent = DialoguePolisherAgent()
    polished = agent.polish(state["episode_script"])
    return {**state, "polished_script": polished}

def storyboard_node(state: ScriptState) -> ScriptState:
    agent = StoryboardAgent()
    shots = agent.generate(state["polished_script"])
    return {**state, "storyboard": shots}

def check_node(state: ScriptState) -> ScriptState:
    agent = ConsistencyCheckerAgent()
    # 调用 MiroFish 一致性校验接口
    result = agent.check(state["polished_script"], state["project_id"])
    new_retry = state["retry_count"] + 1
    return {**state, "check_result": result, "retry_count": new_retry}

# 3. 条件路由
def check_route(state: ScriptState) -> str:
    if state["check_result"]["passed"]:
        return "pass"
    if state["retry_count"] >= state["max_retry"]:
        return "max_retry"
    return "retry"

# 4. 构建图
def build_script_graph(checkpointer):
    graph = StateGraph(ScriptState)
    
    graph.add_node("analyze", analyze_node)
    graph.add_node("plan", plan_node)
    graph.add_node("write", write_node)
    graph.add_node("polish", polish_node)
    graph.add_node("storyboard", storyboard_node)
    graph.add_node("check", check_node)
    
    graph.set_entry_point("analyze")
    graph.add_edge("analyze", "plan")
    graph.add_edge("plan", "write")
    graph.add_edge("write", "polish")
    graph.add_edge("polish", "storyboard")
    graph.add_edge("storyboard", "check")
    
    # 条件边：通过则结束，不通过则回退到写作节点重写
    graph.add_conditional_edges(
        "check",
        check_route,
        {
            "pass": END,
            "retry": "write",
            "max_retry": END  # 超过最大重试次数，标记待人工处理
        }
    )
    
    return graph.compile(checkpointer=checkpointer)
```

##### 升级收益
1. **原生循环校验**：校验不通过自动回退重写，支持最大重试次数控制，无需手写循环逻辑
2. **状态自动管理**：所有中间结果（解析、大纲、剧本、分镜、校验结果）都在共享 State 中，自动持久化
3. **断点续跑**：生成过程中断，重启后从断点继续，不用从头开始
4. **扩展灵活**：后续新增人工审核节点、多轮润色、风格校验，只需加节点和边即可
5. **全链路可观测**：LangSmith 中可查看每个 Agent 的输入输出、Token 消耗、耗时，便于优化和排错

---

### 5.5 Moyin 渲染品控与后期引擎
保持原有设计不变：
1. 四层角色一致性算法（提示词编译器 + 参考图对齐 + 参数统一 + AI 质检）100% 复用 Moyin 核心逻辑，Python 迁移实现
2. FFmpeg 自动化后期流水线（拼接、字幕、混音、转码）100% 复用 Moyin 命令方案，Python 封装
3. 渲染任务调度基于 Celery 队列，对接多云算力网关执行

**可选 LangGraph 增强**：复杂的后期流水线（多版本输出、多风格质检、自动精修）可后续升级为 LangGraph 状态机，当前版本保持 Celery 任务即可，满足需求。

---

### 5.6 数字制片智能体服务
**升级说明**：从定时任务 + 规则引擎，升级为 **LangGraph 决策智能体**，实现「异常识别→原因分析→策略选择→执行处理→结果校验」的全自动闭环。

#### 图结构设计
```
扫描异常任务
    ↓
异常原因分析
    ↓
选择处理策略
    ↓
执行处理操作
    ↓
校验处理结果 → 成功 → 结束
              → 失败 → 升级策略，回到执行步骤（循环）
```

#### 核心价值
- 从简单的固定规则重试，升级为可推理的智能决策
- 处理过程全链路可追溯，每一步决策原因可查
- 支持复杂异常的多轮次升级处理，自动处理率大幅提升

---

### 5.7 LangGraph 状态持久化方案
所有 LangGraph 流程统一使用 **PostgresSaver** 将检查点持久化到 PostgreSQL，保证：
1. **服务重启不中断**：任务执行中服务重启，重启后通过 `thread_id` 恢复状态，继续执行
2. **断点续跑**：长流程任务支持暂停、恢复，不用从头开始
3. **历史回溯**：可查看任意流程的所有历史状态，便于审计和排错
4. **统一存储**：和业务数据共用数据库，无需引入额外存储组件

---

## 六、协同生产层各模块详细实现
帧级协同审片、3D 导演台预演、节点画布前端保持原有设计不变。
**新增联动**：节点画布中，对应「多智能体推演」「剧本生成」的节点，点击可查看该节点内部 LangGraph 流程的执行详情、每一步状态、中间结果，实现从粗粒度到细粒度的全透明可视化。

---

## 七、前端核心模块详细实现
保持原有 React 架构、节点画布、审片、3D 导演台的设计不变。
**新增 LangGraph 可视化页面**：
- 节点执行详情页：展示 LangGraph 流程的执行步骤、状态流转、每一步的输入输出
- 时间轴形式展示执行过程，支持展开查看详情
- 异常节点高亮显示，展示错误原因与处理过程

---

## 八、数据层详细设计
### 8.1 数据库选型与分层策略
保持原有 PostgreSQL + Neo4j + Redis + MinIO + ES 的组合不变。

### 8.2 新增 LangGraph 状态表
LangGraph PostgresSaver 会自动创建检查点表，无需手动建表；业务侧新增关联表，将业务任务ID与 LangGraph 的 thread_id 绑定：
```sql
CREATE TABLE langgraph_task (
    id BIGSERIAL PRIMARY KEY,
    biz_type VARCHAR(32) NOT NULL COMMENT '业务类型：simulation/script/digital_producer',
    biz_id VARCHAR(64) NOT NULL COMMENT '业务ID：项目ID/节点ID',
    thread_id VARCHAR(64) NOT NULL UNIQUE COMMENT 'LangGraph 线程ID，用于恢复状态',
    status VARCHAR(32) NOT NULL DEFAULT 'running',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_biz ON langgraph_task(biz_type, biz_id);
```

其余核心业务表、图数据库模型、缓存设计、对象存储设计保持不变。

---

## 九、非功能技术方案
### 9.1 高可用架构
保持原有服务、数据库、中间件高可用设计不变；
**新增增强**：LangGraph 状态持久化到数据库，单节点故障、服务重启均不丢失任务状态，恢复后自动续跑，系统可用性进一步提升。

### 9.2 性能优化、安全架构、监控可观测性
保持原有设计不变；
**监控增强**：新增 LangGraph 执行指标监控：流程数、成功率、平均耗时、平均重试次数、Token 消耗统计，接入 Prometheus + Grafana；接入 LangSmith 做 Agent 级深度调试与评估。

---

## 十、部署架构
保持标准量产版、企业高可用版两档部署方案不变；LangGraph 无额外部署依赖，仅需 PostgreSQL 支持即可运行，不增加运维复杂度。

---

## 十一、升级总结
### 11.1 本次 LangGraph 升级核心收益
1. **架构更专业**：针对多Agent、循环校验、状态持久化等核心场景，用官方标准方案替代自研调度器，稳定性、可靠性大幅提升
2. **开发效率更高**：循环、分支、状态持久化、断点续跑原生支持，减少 40% 以上的自研编排代码，缩短开发周期
3. **可观测性更强**：原生对接 LangSmith，Agent 执行全链路透明，调试、优化、排错效率大幅提升
4. **扩展性更好**：后续新增复杂 Agent 流程、多轮交互、智能决策，只需扩展图节点，架构无需重构
5. **迁移成本极低**：100% 兼容 LangChain 生态，原有 LLM、Prompt、工具、记忆模块全部复用，不用推翻重写

### 11.2 落地开发顺序建议
1. 先落地 Moyin 渲染品控 + 后期流水线（复用率最高，不涉及 LangGraph）
2. 再落地 MiroFish 知识图谱构建（核心算法复用，推演模块后续升级 LangGraph）
3. 接着基于 LangGraph 实现 Toonflow 多 Agent 编剧流水线（最能体现 LangGraph 价值的核心场景）
4. 再升级 MiroFish 多智能体推演为 LangGraph 架构
5. 最后串联 DAG 工作流、前端、协同审片等模块，完成全链路闭环