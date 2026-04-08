# Multi-Agent Trip Planner - 面试准备指南

## 项目概述

基于 HelloAgents 框架的多智能体旅行规划系统，集成高德地图 MCP 服务，使用 Vue 3 + FastAPI 构建。

**核心功能**：用户输入目的地、日期、交通住宿偏好，系统自动生成个性化旅行计划。

---

## 一、项目难点与解决方案

### 1. Multi-Agent 协作架构设计

**难点**：
- 如何让多个专业 Agent（景点、天气、酒店、规划）有效协作
- Agent 之间如何分工、信息如何聚合
- 避免 Agent 产生冲突或重复的工作

**解决方案**：
```
用户请求
    ↓
Attraction Agent → 景点列表（并行动作）
Weather Agent   → 天气预报（并行动作）
Hotel Agent     → 酒店推荐（并行动作）
    ↓
Planner Agent（整合所有信息生成最终行程）
```

- 采用**流水线模式**：3个Agent并行执行收集信息，Planner Agent统一整合
- 每个Agent职责单一明确，通过prompt engineering确保专业性
- Planner Agent设计为"无工具Agent"，纯粹做信息整合和生成

**关键代码** (`trip_planner_agent.py:222-278`)：
```python
def plan_trip(self, request: TripRequest) -> TripPlan:
    # 并行执行3个收集Agent
    attraction_response = self.attraction_agent.run(attraction_query)
    weather_response = self.weather_agent.run(weather_query)
    hotel_response = self.hotel_agent.run(hotel_query)

    # Planner Agent统一整合
    planner_response = self.planner_agent.run(planner_query)
    return self._parse_response(planner_response, request)
```

---

### 2. LLM 输出格式不稳定的处理

**难点**：
- LLM 返回可能带 markdown 格式（```json...```）
- 可能返回额外解释文本而非纯 JSON
- JSON 中可能包含转义字符问题

**解决方案**：
实现多层级的 JSON 提取策略 (`trip_planner_agent.py:327-368`)：

```python
def _parse_response(self, response: str, request: TripRequest) -> TripPlan:
    try:
        # 策略1: 提取 ```json ... ``` 块
        if "```json" in response:
            json_start = response.find("```json") + 7
            json_end = response.find("```", json_start)
            json_str = response[json_start:json_end].strip()

        # 策略2: 提取 ``` ... ``` 块
        elif "```" in response:
            json_start = response.find("```") + 3
            json_end = response.find("```", json_start)
            json_str = response[json_start:json_end].strip()

        # 策略3: 直接查找 JSON 对象
        elif "{" in response and "}" in response:
            json_start = response.find("{")
            json_end = response.rfind("}") + 1
            json_str = response[json_start:json_end]

        # 策略4: 全部失败，使用 fallback
        else:
            raise ValueError("响应中未找到JSON数据")

        return json.loads(json_str)
    except Exception:
        return self._create_fallback_plan(request)
```

**Fallback 机制**：当 JSON 解析失败时，自动生成备用计划，保证服务不中断。

---

### 3. MCP 工具调用格式控制

**难点**：
- Agent 可能不按指定格式调用工具
- 需要确保 tool_name 和 arguments 格式完全正确
- HelloAgents 框架的工具调用语法需要精确匹配

**解决方案**：
在 prompt 中提供**精确的调用示例**和**强制格式要求**：

```python
ATTRACTION_AGENT_PROMPT = """你必须使用工具来搜索景点!

**工具调用格式:**
使用maps_text_search工具时,必须严格按照以下格式:
[TOOL_CALL:amap_maps_text_search:keywords=景点关键词,city=城市名]

**示例:**
用户: "搜索北京的历史文化景点"
你的回复: [TOOL_CALL:amap_maps_text_search:keywords=历史文化,city=北京]
"""
```

通过 prompt engineering + auto_expand 模式，让 MCP 工具自动展开为独立工具供 Agent 调用。

---

### 4. 跨域与 API 安全

**难点**：
- 前端直接调用高德地图 API 会暴露 Web JS Key
- 前端直接调用 LLM API 会暴露 API Key
- 需要支持前后端分离开发模式

**解决方案**：

**架构层面**：
```
[Vue Frontend] → [FastAPI Backend] → [LLM/高德API]
                   (代理转发)
```

**CORS 配置** (`config.py:32`)：
```python
cors_origins: str = "http://localhost:5173,http://localhost:3000,..."
```

**后端代理**：
- 所有地图请求通过后端代理
- LLM 调用通过后端 `llm_service.py` 封装
- API Key 存储在 `.env` 中，不暴露给前端

---

### 5. 高德地图 MCP Server 集成

**难点**：
- MCP Server 需要通过 `uvx` 启动
- 需要正确传递 API Key 环境变量
- MCP 工具返回的是字符串，需要二次解析

**解决方案** (`amap_service.py:12-47`)：
```python
def get_amap_mcp_tool() -> MCPTool:
    _amap_mcp_tool = MCPTool(
        name="amap",
        description="高德地图服务",
        server_command=["uvx", "amap-mcp-server"],
        env={"AMAP_MAPS_API_KEY": settings.amap_api_key},
        auto_expand=True  # 自动展开为独立工具
    )
    return _amap_mcp_tool
```

**单例模式**：全局缓存 `_amap_mcp_tool`，避免重复创建进程。

---

### 6. 单例模式与全局状态管理

**难点**：
- Agent 初始化成本高（加载模型、创建连接）
- 需要在多次请求间复用实例
- 但不能使用全局变量（不符合最佳实践）

**解决方案**：
```python
# 全局实例
_multi_agent_planner = None

def get_trip_planner_agent() -> MultiAgentTripPlanner:
    """获取多智能体旅行规划系统实例(单例模式)"""
    global _multi_agent_planner

    if _multi_agent_planner is None:
        _multi_agent_planner = MultiAgentTripPlanner()

    return _multi_agent_planner
```

同样的模式用于 `AmapService` 和 LLM 实例。

---

### 7. 配置管理与环境变量

**难点**：
- 多个组件需要独立配置
- HelloAgents 也有自己的 `.env`
- 需要合理的配置优先级

**解决方案** (`config.py:9-16`)：
```python
# 首先加载当前目录的.env
load_dotenv()

# 然后尝试加载HelloAgents的.env(如果存在)
helloagents_env = Path(__file__).parent.parent.parent.parent / "HelloAgents" / ".env"
if helloagents_env.exists():
    load_dotenv(helloagents_env, override=False)  # 不覆盖已有配置
```

使用 Pydantic `BaseSettings` 进行配置管理，支持类型验证和默认值。

---

### 8. Vue + FastAPI 数据流同步

**难点**：
- 首页表单数据需要传递给结果页
- 跨页面数据共享
- TypeScript 类型与 Python Pydantic 模型需要一致

**解决方案**：
- 使用 `sessionStorage` 在页面间传递数据
- 前后端共享相同的字段命名和类型设计
- Pydantic `Field` 的 `description` 直接生成 OpenAPI 文档

---

## 二、面试高频问题与回答

### 1. 系统架构类

**Q1: 描述一下你们的系统架构？**

A: 系统采用前后端分离架构：
- **前端**：Vue 3 + TypeScript + Vite + Ant Design Vue，负责用户交互和地图展示
- **后端**：FastAPI，提供 RESTful API，作为前端与 AI 系统的桥梁
- **AI 层**：基于 HelloAgents 的多 Agent 系统，分为景点搜索、天气查询、酒店推荐、行程规划4个专业 Agent
- **服务层**：高德地图 MCP Server 提供地图服务

请求流程：用户输入 → Frontend → FastAPI → Multi-Agent → MCP Server → 高德API

---

**Q2: 为什么要用 Multi-Agent 而不是单个 Agent？**

A: 单一 Agent 的问题：
- 职责混杂：景点、天气、酒店、规划全部在一个 Agent 中，导致 prompt 复杂、行为不稳定
- 工具调用冲突：同时需要多种工具时容易混淆
- 难以维护和扩展

Multi-Agent 优势：
- **职责分离**：每个 Agent 专注单一任务，prompt 更简单清晰
- **并行执行**：3个收集 Agent 可以并行运行，提高响应速度
- **可扩展性**：新增需求（如美食推荐）只需添加一个 Agent
- **可测试性**：每个 Agent 可以独立测试

---

**Q3: 多个 Agent 之间如何通信和协作？**

A: 采用**流水线模式**：
1. **并行收集阶段**：Attraction/Weather/Hotel Agent 同时运行，各自调用 MCP 工具获取信息
2. **串行整合阶段**：Planner Agent 接收前3个 Agent 的输出作为上下文，综合生成行程

Agent 之间不直接通信，而是通过**共享上下文**（planner_query）传递信息。这种模式：
- 降低 Agent 间依赖
- 支持失败隔离（某个 Agent 失败不影响整体）
- 便于调试和问题定位

---

### 2. 技术实现类

**Q4: 如何确保 LLM 输出的 JSON 格式正确？**

A: 采取多层策略：

1. **Prompt 工程**：在 prompt 中明确指定 JSON 格式，包含字段说明和示例
2. **多层提取**：实现 JSON 块、代码块、原始 JSON 对象的多层提取逻辑
3. **容错机制**：解析失败时使用 fallback 计划，保证服务可用
4. **Pydantic 验证**：即使 JSON 解析成功，Pydantic 也会进行字段类型验证

```python
# 即使格式正确，Pydantic 也会验证
trip_plan = TripPlan(**data)  # 类型错误会抛出异常
```

---

**Q5: MCP Server 是什么？你们如何使用它？**

A: MCP（Model Context Protocol）是一种 AI 模型与外部工具交互的协议。

**高德地图 MCP Server** (`amap-mcp-server`)：
- 通过 `uvx` 启动独立进程
- 封装了高德地图的 POI 搜索、天气查询、路线规划等能力
- 接收标准化格式的工具调用，返回结构化数据

**在我们的系统中的使用**：
```python
_amap_mcp_tool = MCPTool(
    name="amap",
    server_command=["uvx", "amap-mcp-server"],
    env={"AMAP_MAPS_API_KEY": settings.amap_api_key},
    auto_expand=True
)
# auto_expand=True 自动将 MCP 工具转换为独立工具
```

**为什么用 MCP 而不是直接调用 API**：
- 工具调用语法标准化（[TOOL_CALL:...]）
- Agent 无需关心 API 的 HTTP 请求细节
- 便于切换不同的地图服务商

---

**Q6: 你们如何处理 API 密钥的安全问题？**

A: 核心原则：**密钥绝不暴露给前端**

1. **后端存储**：所有密钥存储在 backend/.env 中
2. **后端代理**：前端不直接调用外部 API，所有请求经后端转发
3. **CORS 限制**：只允许指定的开发端口访问
4. **环境变量**：使用 pydantic-settings 管理，通过 `os.getenv` 读取

```python
# .env (已gitignore)
AMAP_API_KEY=xxx
OPENAI_API_KEY=xxx

# 代码中读取
settings = get_settings()
api_key = settings.amap_api_key  # 不暴露给前端
```

---

**Q7: 单例模式在项目中的使用场景？**

A: 用于需要全局共享且创建成本高的资源：

| 实例 | 原因 |
|------|------|
| `MultiAgentTripPlanner` | Agent 初始化加载 LLM，成本高 |
| `AmapService` | MCP Server 进程，只应有一个 |
| `settings` | 配置对象，全局共享 |

**实现方式**：
```python
# 全局变量 + 惰性初始化
_multi_agent_planner = None

def get_trip_planner_agent():
    global _multi_agent_planner
    if _multi_agent_planner is None:
        _multi_agent_planner = MultiAgentTripPlanner()
    return _multi_agent_planner
```

---

### 3. 前端相关

**Q8: Vue 3 相比 Vue 2 有哪些变化？你们如何使用？**

A: 我们使用 Vue 3 + Composition API：

**主要变化**：
- **Composition API**：`setup()` 函数中使用 `ref`、`reactive`、`computed`
- **更好的 TypeScript 支持**
- **Tree-shaking 优化**：未使用的功能不打包

**我们的使用**：
```typescript
// Home.vue
const formState = reactive({
  city: '',
  startDate: '',
  // ...
})

const travelDays = computed(() => {
  // 计算属性
})
```

**其他技术选型**：
- Vite：更快的冷启动和热更新
- Ant Design Vue：企业级 UI 组件库
- Axios：HTTP 请求封装

---

**Q9: 前后端数据如何在页面间传递？**

A: 使用 `sessionStorage`：

```typescript
// Home.vue - 提交时存储
sessionStorage.setItem('tripPlan', JSON.stringify(response.data))

// Result.vue - 读取
const tripData = JSON.parse(sessionStorage.getItem('tripPlan') || '{}')
```

**为什么不使用 Vuex/Pinia**：
- 简单场景不需要全局状态管理
- sessionStorage 更轻量，适合跨页数据
- 数据无需持久化（刷新可重新获取）

---

**Q10: 地图组件如何与后端数据联动？**

A: 分两层：

**1. 展示层（前端高德 JS API）**：
```typescript
// Result.vue
const map = new AMap.Map('container')
// 添加标记
new AMap.Marker({ position: [lng, lat] })
// 绘制路线
new AMap.Riding/Driving/Transfer(route)
```

**2. 数据层（后端 MCP）**：
- POI 搜索 → `maps_text_search`
- 地理编码 → `maps_geo`
- 路线规划 → `maps_direction_driving_by_address`

**数据流向**：
```
用户输入目的地
    ↓
后端 Attraction Agent 调用 MCP → 返回景点列表（含经纬度）
    ↓
前端用这些经纬度在地图上打标记
    ↓
用户选择具体路线时，前端调用后端 /api/map/route 获取路线
```

---

### 4. AI/LLM 相关

**Q11: 你们使用什么 LLM？Prompt 是如何设计的？**

A: **支持多 LLM**：OpenAI GPT-4 / DeepSeek，可通过配置切换。

**Prompt 设计策略**：

1. **角色设定**：明确 Agent 身份（"你是景点搜索专家"）
2. **强制工具调用**：必须使用工具，不允许自己编造
3. **格式示例**：提供精确的调用格式和示例
4. **输出规范**：明确 JSON 格式要求

```python
ATTRACTION_AGENT_PROMPT = """你是景点搜索专家。
你必须使用工具来搜索景点!不要自己编造景点信息!
[TOOL_CALL:amap_maps_text_search:keywords=关键词,city=城市名]
"""
```

---

**Q12: 如何评估生成行程的质量？**

A: 目前系统通过以下机制保证质量：

1. **Fallback 机制**：JSON 解析失败时使用备用计划
2. **Pydantic 验证**：确保返回数据完整（天数、景点、用餐等）
3. **结构化输出**：prompt 明确要求包含预算、天气、交通等字段
4. **人工审核**：前端提供编辑功能，用户可手动调整

**可改进方向**：
- 添加评分/反馈机制
- 使用 Embedding 计算与用户偏好的匹配度
- 增加合理性校验（景点开放时间、路线可行性）

---

**Q13: Agent 工具调用失败时如何处理？**

A: 三层降级策略：

| 层级 | 失败场景 | 处理方式 |
|------|---------|---------|
| MCP 工具调用 | 网络/参数错误 | 捕获异常，返回空结果 |
| JSON 解析 | LLM 输出格式错误 | 多层提取策略 + fallback |
| 整体生成 | 任何失败 | 返回 fallback 计划 |

```python
# amap_service.py
try:
    result = self.mcp_tool.run({...})
except Exception as e:
    print(f"❌ POI搜索失败: {str(e)}")
    return []  # 返回空列表，不中断流程

# trip_planner_agent.py
try:
    trip_plan = self._parse_response(...)
except Exception:
    return self._create_fallback_plan(request)  # 使用备用计划
```

---

### 5. 工程实践类

**Q14: 你们如何管理配置和依赖？**

A: **Python 端**：
- `requirements.txt`：固定版本的依赖列表
- `.env`：密钥和环境特定配置
- `pydantic-settings`：类型安全的配置管理

**Node.js 端**：
- `package.json`：依赖列表和脚本
- `.env`：Vite 环境变量（VITE_ 前缀）

**配置优先级**：
```
代码默认值 < .env 文件 < 系统环境变量
```

---

**Q15: 项目的错误处理机制？**

A: 分层处理：

**1. API 层**（FastAPI）：
```python
@router.post("/plan")
async def plan_trip(request: TripRequest):
    try:
        trip_plan = agent.plan_trip(request)
        return TripPlanResponse(success=True, data=trip_plan)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

**2. Agent 层**：
- 工具调用异常：捕获并返回空结果
- JSON 解析异常：使用 fallback 计划

**3. 前端层**：
- Axios 拦截器统一处理错误
- 友好的错误提示 UI

---

**Q16: 如何进行开发调试？**

A: **后端**：
```bash
# 直接运行，print 输出到终端
cd backend && python run.py

# 日志级别配置
LOG_LEVEL=DEBUG
```

**前端**：
```bash
# Vite 热更新，保存即刷新
cd frontend && npm run dev
```

**API 调试**：
- Swagger 文档：`http://localhost:8000/docs`
- 可直接在文档页面测试接口

---

### 6. 开放讨论类

**Q17: 这个项目有哪些可以改进的地方？**

A: **可改进方向**：

| 方向 | 具体改进 |
|------|---------|
| **可靠性** | 添加 Agent 输出评分机制，过滤低质量结果 |
| **实时性** | 景点评价、实时排队人数等动态数据 |
| **个性化** | 引入用户历史偏好Embedding，实现更精准推荐 |
| **容灾** | MCP Server 失败时的备用方案 |
| **测试** | 添加 pytest 单元测试和集成测试 |
| **部署** | Docker 容器化部署 |
| **监控** | 添加日志追踪和指标监控 |

---

**Q18: 如果要支持多语言/国际化，如何改造？**

A: 改造点：

1. **Prompt 多语言**：为每个 Agent 添加多语言 prompt 变体
2. **前端国际化**：Vue I18n 处理 UI 文本
3. **API 多语言**：高德地图支持多语言（通过参数指定）
4. **LLM 多语言**：DeepSeek 等模型原生支持多语言

---

**Q19: 如何保证生成行程的合法性（比如景点是否营业）？**

A: 当前版本未实现，可行方案：

1. **基础校验**：检查日期是否在有效范围内
2. **API 增强**：调用高德详情 API 获取营业时间
3. **Prompt 引导**：在 prompt 中要求 Agent 考虑景点开放时间
4. **Fallback**：无法获取信息时给出"建议提前确认"的提示

---

**Q20: 如果用户输入的地点不存在或模糊，如何处理？**

A: 当前策略：

1. **城市级模糊搜索**：MCP `maps_text_search` 支持关键词搜索
2. **Fallback 计划**：当所有 Agent 失败时，生成包含输入城市的通用计划
3. **前端校验**：日期格式验证、必填项检查

**可改进**：
- 添加"您是要找..."的歧义消解交互
- 调用地理编码 API 验证地点存在性

---

## 三、技术亮点总结

| 亮点 | 说明 |
|------|------|
| **Multi-Agent 协作** | 4个专业Agent分工，流水线模式，并行+串行混合 |
| **MCP 协议应用** | 标准化工具调用，解耦 AI 与地图服务 |
| **容错机制** | 多层 JSON 提取 + Fallback 保证服务可用 |
| **安全设计** | 密钥后端管理，CORS 限制，前端不暴露敏感信息 |
| **配置管理** | Pydantic Settings + .env 环境变量 |
| **全栈工程化** | 前后端分离，TypeScript + Pydantic 类型共享 |

---

## 四、准备面试的项目亮点表达

### 表达公式

> "**问题背景** + **我的方案** + **技术细节** + **效果**"

### 示例

**Q: 介绍你最有挑战的项目？**

**回答模板**：

> 这个项目我负责的是 Multi-Agent 旅行规划系统。核心难点是如何让多个 AI Agent 有效协作。
>
> 我的方案是将 Agent 分为三层：3个专业 Agent（景点、天气、酒店）并行收集信息，1个规划 Agent 串行整合。这种流水线模式让响应速度提升，同时保证了输出质量。
>
> 技术细节上，我使用 HelloAgents 框架，通过 MCP 协议调用高德地图服务。为了处理 LLM 输出不稳定的问题，我实现了多层级的 JSON 提取策略，配合 Fallback 机制保证服务不中断。
>
> 最终系统能够根据用户偏好，自动生成包含景点、住宿、餐饮、交通的完整旅行计划。

---

## 五、反向问题准备

面试结束时通常会问"你有什么问题要问？"：

1. **团队技术栈**：贵司 AI 项目的技术栈是什么？
2. **业务场景**：这个岗位主要服务的业务场景是什么？
3. **成长空间**：入职后有什么学习资源或培训机制？
4. **团队规模**：您所在的团队有多少人，如何分工？
5. **技术挑战**：目前团队面临的最大技术挑战是什么？

---

*文档生成时间：2026-04-03*
