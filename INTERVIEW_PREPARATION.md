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

- LLM 返回可能带 markdown 格式（``json...``）
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

| 实例                      | 原因                         |
| ------------------------- | ---------------------------- |
| `MultiAgentTripPlanner` | Agent 初始化加载 LLM，成本高 |
| `AmapService`           | MCP Server 进程，只应有一个  |
| `settings`              | 配置对象，全局共享           |

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

| 层级         | 失败场景         | 处理方式                |
| ------------ | ---------------- | ----------------------- |
| MCP 工具调用 | 网络/参数错误    | 捕获异常，返回空结果    |
| JSON 解析    | LLM 输出格式错误 | 多层提取策略 + fallback |
| 整体生成     | 任何失败         | 返回 fallback 计划      |

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

| 方向             | 具体改进                                  |
| ---------------- | ----------------------------------------- |
| **可靠性** | 添加 Agent 输出评分机制，过滤低质量结果   |
| **实时性** | 景点评价、实时排队人数等动态数据          |
| **个性化** | 引入用户历史偏好Embedding，实现更精准推荐 |
| **容灾**   | MCP Server 失败时的备用方案               |
| **测试**   | 添加 pytest 单元测试和集成测试            |
| **部署**   | Docker 容器化部署                         |
| **监控**   | 添加日志追踪和指标监控                    |

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

| 亮点                       | 说明                                        |
| -------------------------- | ------------------------------------------- |
| **Multi-Agent 协作** | 4个专业Agent分工，流水线模式，并行+串行混合 |
| **MCP 协议应用**     | 标准化工具调用，解耦 AI 与地图服务          |
| **容错机制**         | 多层 JSON 提取 + Fallback 保证服务可用      |
| **安全设计**         | 密钥后端管理，CORS 限制，前端不暴露敏感信息 |
| **配置管理**         | Pydantic Settings + .env 环境变量           |
| **全栈工程化**       | 前后端分离，TypeScript + Pydantic 类型共享  |

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

## 六、阿里系面试高频问题

### Q1: Agent 项目中向量是怎么检索的，大概结构是什么样的？

A: **向量检索核心流程**：

```
用户查询 → 向量化(Embedding模型) → 向量相似度计算 → Top-K 返回
```

**典型 RAG 检索架构**：

```
文档集合
    ↓ 分块(Chunking)
[chunk1, chunk2, chunk3, ...]
    ↓ Embedding模型编码
[vec1, vec2, vec3, ...]  → 存入 Vector DB (Milvus/Chroma/Qdrant)
    ↓
用户查询 "北京景点推荐"
    ↓ Query Embedding
query_vec
    ↓ ANN近似最近邻检索 (HNSW/IVF/LSH)
Top-K 相关文档块
    ↓
与原始查询一起送给 LLM 生成答案
```

**向量数据库索引结构**（以 Milvus 为例）：

| 索引类型 | 原理                   | 适用场景     | 召回率 | 速度 |
| -------- | ---------------------- | ------------ | ------ | ---- |
| HNSW     | 图索引，层次渐进式搜索 | 通用，高精度 | ~95%   | 快   |
| IVF      | 倒排索引，聚类中心     | 大规模数据   | ~90%   | 中   |
| IVF-PQ   | IVF + 产品量化         | 超大规模     | ~85%   | 很快 |
| LSH      | 局部敏感哈希           | 近似相等搜索 | 中     | 快   |

---

### Q2: 假如有一百万个向量数据集，怎么构建索引，检索流程是什么样的？

A: **百万级向量索引构建与检索全流程**：

**阶段1：数据准备**

```python
# 1. 文档加载与分块
documents = loader.load("知识库文档/")
chunks = text_splitter.split_documents(documents)  # 500字/块

# 2. 批次向量化（利用GPU加速）
for i in range(0, len(chunks), batch_size):
    batch = chunks[i:i+batch_size]
    embeddings = embedding_model.encode(batch)  # shape: [batch, 1536]
```

**阶段2：索引构建**

```python
# 方案A: HNSW 索引 (适用于100万级)
collection.create_index(
    field_name="vector",
    index_params={
        "index_type": "HNSW",
        "metric_type": "IP",  # 内积，支持归一化后的余弦相似度
        "params": {"M": 16, "efConstruction": 200}  # M=邻居数, efConstruction=构建精度
    }
)

# 方案B: IVF-PQ 索引 (适用于1000万级+)
collection.create_index(
    field_name="vector",
    index_params={
        "index_type": "IVF_PQ",
        "metric_type": "IP",
        "params": {"nlist": 1024, "m": 64, "nbits": 8}  # nlist=聚类数, m=PQ分子段数
    }
)
```

**阶段3：检索流程**

```python
# 1. 查询向量化
query_vec = embedding_model.encode("北京三日游推荐")

# 2. ANN 检索
results = collection.search(
    data=[query_vec.tolist()],
    anns_field="vector",
    param={
        "index_type": "HNSW",
        "params": {"ef": 128}  # ef=搜索时动态列表大小，越大越精确但越慢
    },
    limit=10  # Top-10
)

# 3. 后处理（可选：Rerank）
reranked = rerank_model.rank(query, results)
```

**HNSW 检索原理图**：

```
Layer 2: [O]--------[O]--------[O]     ← 顶层，宽泛搜索
              ↓
Layer 1: [O]-[O]-[O]-[O]-[O]-[O]-[O]  ← 中层
              ↓
Layer 0: [O]-[O]→[O]←[O]-[O]-[O]-[O]  ← 底层，最近邻精确搜索
                    ↑
              ef=128 候选队列
```

---

### Q3: 实时流是怎么处理的？

A: **实时流处理架构**：

**方案1：消息队列 + 流处理框架**

```
用户输入(实时) → Kafka/RabbitMQ → Flink/Spark Streaming → Vector DB
                              → Redis (缓存热点数据)
```

**方案2：增量索引更新**

```python
# 增量索引更新伪代码
def on_new_document(doc):
    # 1. 增量向量化
    chunks = split_document(doc)
    vectors = embedding_model.encode(chunks)

    # 2. 增量写入 Vector DB
    collection.insert({"vector": vectors, "text": chunks, "metadata": doc.metadata})

    # 3. 异步更新索引（Milvus 支持增量建索引）
    collection.flush()

    # 4. 发布事件供后续处理
    kafka.send("document_indexed", {"doc_id": doc.id})
```

**方案3：流式生成 + 实时检索（我们的项目应用）**

```
用户请求 → Agent 并行调用 MCP 工具
              ↓
FastAPI async endpoint → 实时返回搜索状态
              ↓
前端 SSE/WebSocket 推送进度
              ↓
最终结果聚合后一次性返回
```

**关键点**：

- 高德地图 API 本身是同步阻塞的，无法做到真正的流式
- 改进方案：用 `asyncio` + `aiohttp` 实现并发调用（景点/天气/酒店并行）
- 流式输出给 LLM：使用 OpenAI 的 `stream=True` 参数

---

### Q4: RESTful API 有什么比较好的地方？

A: **RESTful API 核心优势**：

| 优势                 | 说明                         | 在我们项目中的应用                  |
| -------------------- | ---------------------------- | ----------------------------------- |
| **前后端解耦** | 独立开发，语言无关           | Vue前端 + FastAPI后端               |
| **可缓存**     | GET 请求可缓存               | 天气信息可做本地缓存                |
| **幂等性**     | 同一请求结果一致             | POST /plan 每次生成新计划，语义正确 |
| **分层系统**   | API Gateway / 负载均衡       | 可加 Nginx 代理                     |
| **统一接口**   | GET/POST/PUT/DELETE 语义清晰 | `/api/trip/plan` POST 创建        |
| **可发现性**   | Swagger 自动生成文档         | FastAPI 自动生成 `/docs`          |

**RESTful 语义对照**：

```
GET    /api/trip/health     → 查询健康状态（无副作用）
POST   /api/trip/plan       → 创建旅行计划（幂等性要求不严格）
GET    /api/poi/search      → 搜索POI（可缓存）
POST   /api/map/route       → 规划路线（创建资源）
```

**GraphQL vs RESTful**：

```
RESTful: 固定返回格式，可能过量获取
GraphQL: 按需获取，但增加复杂度

旅行规划场景适合 RESTful：
- 接口数量少，结构固定
- 需要 API Gateway / 限流 / 监控等中间件支持
```

---

### Q5: Context 有什么优化吗？

A: **Context 优化策略**：

**1. Prompt 压缩**

```python
# 原始天气信息（冗长）
weather_raw = """
今天是2024-01-15，北京天气：白天晴，26°C，夜间多云，18°C，
风向东南风，风力3-4级，空气质量优，PM2.5指数35 ...
"""

# 压缩后（只保留关键信息）
weather_compressed = """
Day1(01-15): ☀️26°C→🌙18°C, SE3-4级
Day2(01-16): ⛅25°C→🌙17°C, S3级
"""
```

**2. Selective Context（选择性上下文）**

```python
def selective_context(relevant_docs, query, max_tokens=4000):
    """只选取与查询最相关的上下文"""
    scored = []
    for doc in relevant_docs:
        # 用轻量模型判断相关性
        score = cross_encoder_score(query, doc)
        scored.append((score, doc))

    # 取最高分的文档直到 token 限制
    selected = []
    total_tokens = 0
    for score, doc in sorted(scored, reverse=True):
        if total_tokens + len(doc) > max_tokens:
            break
        selected.append(doc)
        total_tokens += len(doc)

    return selected
```

**3. Context 窗口优化**

```
LLM Context 优化策略：
├── 位置编码优化：重要信息放开头/结尾（位置效应）
├── 对话历史摘要：长期对话定期压缩历史
├── RAG 检索优化：Top-K → MMR（最大边际相关性）去重
└── 模型原生支持：Claude 100K / GPT-4 128K 超长上下文
```

**4. 在我们项目中的应用**

```python
# Planner Agent 的 prompt 压缩
planner_query = f"""根据以下信息生成{request.city}的{request.travel_days}天旅行计划:
**关键信息**:
- 城市: {request.city}, 天数: {request.travel_days}
- 景点(已筛选): {top_3_attractions}  # 只取 Top3，不是全部
- 天气(精简): {summarized_weather}    # 只保留温度和天气
- 酒店: {best_hotel}                 # 只推荐1个最优
"""
```

---

### Q6: 向量化是怎么做的？

A: **Embedding 向量化流程**：

**文本向量化（以中文旅行为例）**：

```python
# 1. 选择 Embedding 模型
embedding_model = "text2vec-base-chinese"  # 中文效果好
# 或: "paraphrase-multilingual-MiniLM-L12-v2"  # 多语言支持

# 2. 单条向量化
text = "北京故宫是中国明清两代的皇家宫殿"
vector = embedding_model.encode(text)  # shape: [768]

# 3. 批量向量化
texts = ["景点1描述", "景点2描述", ...]
vectors = embedding_model.encode(texts)  # shape: [N, 768]
```

**中文分词对向量化的影响**：

```python
# 错误示例：直接按字符级别分割（丢失语义）
chars = list("北京故宫")  # ['北', '京', '故', '宫']

# 正确示例：按语义分词
words = jieba.cut("北京故宫")  # ['北京', '故宫']
# 更好的中文字向量用词级别的模型
```

**不同粒度的向量化**：

| 粒度     | 优点     | 缺点       | 适用场景           |
| -------- | -------- | ---------- | ------------------ |
| 字级别   | OOV 友好 | 语义弱     | 字符级 BERT        |
| 词级别   | 语义清晰 | 依赖分词器 | Jieba + Word2Vec   |
| 句级别   | 语义完整 | 长度不一   | 句子向量(我们项目) |
| 段落级别 | 语义丰富 | 维度高     | 长文档检索         |

**向量归一化**：

```python
# 余弦相似度等价于归一化后的内积
vectors = vectors / np.linalg.norm(vectors, axis=1, keepdims=True)

# 检索时直接用内积(IP)即可
similarity = query_vec @ document_vec.T
```

---

### Q7: Redis 的 Skip List 是什么？

A: **Skip List（跳表）数据结构**：

**为什么 Redis Sorted Set 用 Skip List 而不是红黑树？**

| 特性     | Skip List           | 红黑树         |
| -------- | ------------------- | -------------- |
| 实现难度 | 简单                | 复杂           |
| 范围查询 | O(log n)            | O(log n)       |
| 插入删除 | 只改局部指针        | 需要旋转重平衡 |
| 内存占用 | 1.5x~2x（随机层高） | 1x             |
| 有序性   | 天然有序            | 天然有序       |
| 并发友好 | 更易实现无锁        | 需要复杂锁     |

**Skip List 结构图**：

```
Level 2: HEAD ----→ [5] --------------------------------→ [90] NIL
                      ↓                                    ↓
Level 1: HEAD ----→ [5] --------→ [30] --------→ [90] ----→ NIL
                      ↓            ↓            ↓
Level 0: HEAD → [3]→[5]→[8]→[20]→[30]→[50]→[70]→[90]→[95]→NIL
                      搜索路径: 从Level2开始，跳跃式下降
```

**搜索 "35" 的路径**：

```
Level 2: 5 → 90 (35 < 90，下降)
Level 1: 5 → 30 (35 > 30，继续)
Level 0: 30 → 50 (找到插入点，在30和50之间)
```

**Redis ZSet 源码简化**：

```c
// Redis ZSet 节点结构
typedef struct zsetnode {
    double score;           // 分数
    sds ele;                // 元素
    int level;              // 随机层高(1-64)
    struct zskiplistLevel {
        struct zskiplistNode *forward;  // 前进指针
        unsigned int span;               // 跨度（用于rank计算）
    } level[];
} zskiplistNode;
```

**随机层高生成（保证平衡性）**：

```python
import random

def random_level(max_level=64):
    level = 1
    while random.random() < 0.5 and level < max_level:
        level += 1
    return level
# 概率上：50%的节点在L1，25%在L2，12.5%在L3...
# 期望复杂度 O(log n)
```

---

### Q8: 这几个项目中最有成就感或最有挑战性的点是什么？

A: **回答框架**：STAR法则（Situation-Task-Action-Result）

**参考回答（Multi-Agent 项目）**：

> **最有挑战性的点：多 Agent 协作的稳定性问题**
>
> **背景**：系统初期采用单 Agent 处理所有请求，但发现当需要同时调用地图搜索、天气查询、酒店推荐时，Agent 经常混淆工具调用格式，导致 30% 的请求失败。
>
> **我的方案**：
>
> 1. 将单一 Agent 拆分为 4 个专业 Agent
> 2. 设计流水线模式：3 个并行收集 + 1 个串行整合
> 3. 实现多层级的 JSON 解析 + Fallback 机制
>
> **结果**：请求成功率从 70% 提升到 95%+，响应时间因为并行执行还缩短了 40%

> **最有成就感的点：用户反馈超出预期**
>
> 一位用户说"比我自己在攻略网站上做的功课还详细"，让我觉得技术解决真实需求的价值感。

---

### Q9: Transformer 基本算法知识，self-attention 原理？

A: **Transformer 与 Self-Attention 详解**：

**整体架构**：

```
输入Embedding + Positional Encoding
         ↓
 Encoder: N×[Multi-Head Self-Attention + Feed Forward]
         ↓
 Decoder: N×[Masked Self-Attention + Cross Attention + Feed Forward]
         ↓
     Linear + Softmax
         ↓
      输出概率
```

**Self-Attention 计算流程**：

```python
# 输入: X (seq_len, d_model) = [batch, seq_len, 768]
# 权重: W_Q, W_K, W_V (768, 768)

# Step 1: Q, K, V 投影
Q = X @ W_Q  # (seq_len, d_k)
K = X @ W_K  # (seq_len, d_k)
V = X @ W_V  # (seq_len, d_v)

# Step 2: 注意力分数
scores = Q @ K.T / sqrt(d_k)  # (seq_len, seq_len)
# 每个位置对所有位置的注意力

# Step 3: Softmax 归一化
attention_weights = softmax(scores, dim=-1)  # (seq_len, seq_len)

# Step 4: 加权求和
output = attention_weights @ V  # (seq_len, d_v)

# 多头: 并行多个 Attention，拼接结果
# MultiHead = Concat(head_1, ..., head_h) @ W_O
```

**形象理解**：

```
"The cat sat on the mat because it was tired"

Self-Attention 让每个词去看整句话中哪些词与它相关：
- "it" 会给 "cat" 和 "mat" 较高的注意力权重
- "sat" 会关注 "cat" 和 "on"
- 每个词都能"看到"整句的上下文
```

**Multi-Head Attention 的意义**：

```
head_1 (捕获主谓关系): it → cat
head_2 (捕获语义相似): sat → tired
head_3 (捕获位置邻近): on → mat

每个头独立学习不同的注意力模式，最后拼接，让模型捕获多维度的依赖关系
```

---

### Q10: Encoder 里的 self attention 的流程，decoder 和 encoder 的结构差异？

A: **Encoder vs Decoder 详细对比**：

**Encoder Self-Attention（全双向）**：

```
输入: "我爱北京" (4个token)

Step: 4个token同时 attend to 所有4个token
- "我" 能看到 "爱", "北京"
- "爱" 能看到 "我", "北京"
- "北京" 能看到 "我", "爱"
(所有位置都能看到完整上下文 = 全双向)
```

**Decoder Self-Attention（Masked 单向）**：

```
输入: "<BOS> 我 爱 北京 <EOS>" (5个token)

Masked: 预测"爱"时，只能看到"<BOS> 我"
       预测"北京"时，只能看到"<BOS> 我 爱"
       (只能用左边的context = 单向)

Mask矩阵:
        <BOS>  我   爱   北京  <EOS>
<BOS>   1     1    1     1     1    ← BOS可以看到所有
我       0     1    1     1     1    ← "我"看不到<BOS>(因果)
爱       0     0    1     1     1    ← "爱"看不到"我"和<BOS>
北京      0     0    0     1     1
<EOS>   0     0    0     0     1
```

**Cross Attention（Encoder-Decoder Attention）**：

```
Decoder的每个位置 attend to Encoder的输出:
Query: Decoder的hidden state
Key/Value: Encoder的final hidden states

例如翻译任务:
Decoder "I" 的Query → 与 Encoder "我爱北京" 的K,V 计算注意力
                       → 找到"我"和"I"的对齐关系
```

**结构差异总结**：

| 组件            | Encoder                | Decoder                               |
| --------------- | ---------------------- | ------------------------------------- |
| Self-Attention  | 全双向 (bidirectional) | Masked单向 (causal)                   |
| Cross Attention | 无                     | 有 (Query来自Decoder, K,V来自Encoder) |
| 用途            | 理解输入序列           | 生成输出序列                          |
| 输出            | 每位置隐藏状态         | 逐token生成下一个词                   |

---

### Q11: KV Cache 和 Flash Attention 是什么？

A: **KV Cache 和 Flash Attention 详解**：

**KV Cache（键值缓存）**：

**问题**：Transformer 推理时，生成第 T 个 token 需要重新计算前 T-1 个 token 的 attention，时间复杂度 O(T²)

**解决方案**：缓存 K 和 V，每次只计算新 token 的 Q，然后查表

```python
# 无 KV Cache (每步都重新计算)
for new_token in generation:
    Q = get_q(new_token)
    K = concat(all_prev_K, new_K)  # 重复计算
    V = concat(all_prev_V, new_V)  # 重复计算
    output = attention(Q, K, V)

# 有 KV Cache (缓存复用)
K_cache.append(new_K)
V_cache.append(new_V)
Q = get_q(new_token)  # 只算新token的Q
output = attention(Q, K_cache, V_cache)  # K,V直接查表
```

**KV Cache 显存问题**：

```
标准Attention: 显存 = O(batch × seq_len²)
KV Cache后:    新增显存 = O(batch × num_layers × 2 × seq_len × d_k)

以LLaMA-7B为例:
- 序列长度4096，batch=1
- 每层K/V缓存: 2 × 4096 × 4096 × 4字节 ≈ 256MB
- 32层总显存: 32 × 256MB ≈ 8GB 仅用于KV缓存
```

**Flash Attention**：

**问题**：标准 Attention 需要先算完整注意力矩阵 S = QK^T (O(N²)显存)，再做 softmax，显存占用巨大

**解决方案**：分块计算，逐块 softmax，避免实例化完整矩阵

```python
# 标准Attention (显存O(N²))
S = Q @ K.T  # 完整注意力矩阵
P = softmax(S / sqrt(d))
O = P @ V

# Flash Attention (显存O(N))
# 核心思想: 分块计算 + online softmax
def flash_attention(Q, K, V, block_size=64):
    O = zeros_like(Q)
    l = zeros(len(Q))  # 行累加和，用于online softmax

    for i in range(0, len(Q), block_size):
        Q_i = Q[i:i+block_size]
        # 只加载一块K,V，显存固定
        K_j = K[j:j+block_size]
        V_j = V[j:j+block_size]

        # 分块计算
        S_ij = Q_i @ K_j.T / sqrt(d)
        P_ij = exp(S_ij - row_max(S_ij))  # 数值稳定的分块softmax
        O_i += P_ij @ V_j

        # 更新softmax归一化因子
        l_i += sum(P_ij, axis=1)

    O = O / l  # 最终归一化
    return O
```

**Flash Attention 效果对比**：

| 指标     | 标准Attention | Flash Attention |
| -------- | ------------- | --------------- |
| 显存     | O(N²)        | O(N)            |
| 计算量   | 相同          | 相同            |
| 速度     | baseline      | 2-4x faster     |
| 适用场景 | 短序列        | 长序列(64K+)    |

**两者结合**：

```
LLM推理优化:
1. Flash Attention: 减少注意力计算的显存占用
2. KV Cache: 减少重复计算，加速生成
3. Continuous Batching: 多个请求共享计算，提高吞吐

典型组合: LLaMA + Flash Attention + KV Cache + TensorRT-LLM
```

---

## 七、场景题与系统设计

### Q12: 如何处理 100 个人用 1 个 Agent 和 1 个人传 100 份文件给 Agent 这两种情况？

A: **两种高并发/大批量场景的解决方案**：

**场景1：100个人用1个Agent（高并发问题）**

**问题本质**：多个用户共享同一个Agent服务，核心是**并发控制和资源隔离**

```
User1 ─┐
User2 ─┤
User3 ─┼─→ [Single Agent] → 响应
...    │
User100 ┘
```

**解决方案**：

```python
# 方案A: 请求队列 + 异步处理
from fastapi import BackgroundTasks

@app.post("/plan")
async def plan_trip(request: TripRequest):
    # 用户请求入队，立即返回 job_id
    job_id = await enqueue_job(request)
    return {"job_id": job_id, "status": "queued"}

# 后台worker处理
async def job_worker():
    while True:
        job = await queue.get()
        result = await single_agent.process(job)  # Agent串行处理
        await notify_user(job.user_id, result)
```

```python
# 方案B: 多实例负载均衡
# 同时运行多个Agent实例，通过Nginx/负载均衡器分发
agent_instances = [Agent() for _ in range(3)]  # 3个实例
round_robin_index = 0

@app.post("/plan")
async def plan_trip(request: TripRequest):
    global round_robin_index
    agent = agent_instances[round_robin_index % len(agent_instances)]
    round_robin_index += 1
    return await agent.process(request)
```

```python
# 方案C: 带锁的Agent池（推荐）
import asyncio

class AgentPool:
    def __init__(self, size=3):
        self.semaphore = asyncio.Semaphore(size)  # 最多3个并发
        self.agent = get_trip_planner_agent()

    async def process(self, request):
        async with self.semaphore:  # 控制并发数
            return await self.agent.plan_trip(request)
```

**场景2：1个人传100份文件（大批量问题）**

**问题本质**：单个请求处理大量文档，核心是**批量处理优化**

```
User → [100份文件] → [Single Agent]
```

**解决方案**：

```python
# 方案A: 分批次 + 进度反馈
@app.post("/process_files")
async def process_files(files: List[UploadFile]):
    job_id = create_job(len(files))

    # 1. 解析文件（批量I/O）
    documents = []
    for f in files:
        doc = await parse_file(f)
        documents.append(doc)

    # 2. 批量向量化（利用批处理优化）
    embeddings = embedding_model.encode(documents)  # batch=True

    # 3. 分批检索（避免context溢出）
    results = []
    for i in range(0, len(documents), 10):  # 每批10个
        batch = documents[i:i+10]
        batch_result = await agent.process(batch)
        results.extend(batch_result)
        await update_progress(job_id, i/len(documents))

    return results
```

```python
# 方案B: 向量数据库预先索引
@app.post("/upload_files")
async def upload_files(files: List[UploadFile]):
    # 文件预先索引，不在请求时处理
    all_chunks = []
    for f in files:
        chunks = split_and_embed(f)
        all_chunks.extend(chunks)

    # 批量写入Vector DB
    collection.insert(all_chunks)

    return {"status": "indexed", "chunk_count": len(all_chunks)}

@app.post("/query")
async def query(question: str):
    # 查询时只做向量检索，不处理文件
    results = collection.search(question, limit=10)
    return results
```

**两种场景的核心区别**：

| 场景        | 挑战                  | 核心策略                |
| ----------- | --------------------- | ----------------------- |
| 100人并发   | 并发控制，资源隔离    | Semaphore限流，队列缓冲 |
| 100文件批量 | 计算量大，context溢出 | 预索引，分批处理        |

---

### Q13: 如何评价整个系统（用户反馈/准确率/召回率/延迟）？

A: **系统评价指标体系**：

**1. 用户反馈指标**

```python
# 量化反馈收集
feedback = {
    "thumbs_up": count,      # 用户点赞数
    "thumbs_down": count,    # 用户点踩数
    "edit_count": count,     # 用户编辑次数（反映输出不完美）
    "regenerate_count": count, # 重新生成次数
    "sharing_count": count    # 分享给他人（高质量标志）
}

satisfaction_rate = thumbs_up / (thumbs_up + thumbs_down)
edit_rate = edit_count / total_requests  # 编辑率越低越好
```

**2. 准确率与召回率（RAG 系统）**

```python
# Ground Truth 构建
# 人工标注100-200个query的最佳文档集合

# 准确率 (Precision@K): 检索结果中相关文档的比例
def precision_at_k(retrieved_docs, relevant_docs, k):
    return len(set(retrieved_docs[:k]) & set(relevant_docs)) / k

# 召回率 (Recall@K): 相关文档被检索出来的比例
def recall_at_k(retrieved_docs, relevant_docs, k):
    return len(set(retrieved_docs[:k]) & set(relevant_docs)) / len(relevant_docs)

# MRR (Mean Reciprocal Rank): 第一个相关文档的排名
def mean_reciprocal_rank(retrieved_docs, relevant_docs):
    for i, doc in enumerate(retrieved_docs, 1):
        if doc in relevant_docs:
            return 1 / i
    return 0

# NDCG (Normalized Discounted Cumulative Gain)
# 综合考虑相关性和排名位置
```

**3. 生成质量评估**

```python
# LLM-as-Judge (用强模型评估弱模型输出)
def evaluate_with_claude(query, response):
    result = claude.evaluate(f"""
    Query: {query}
    Response: {response}
    Score: 1-5 (1=完全不相关, 5=完美回答)
    """)
    return result.score

# RAGAS 指标（专为RAG系统设计）
ragas_metrics = {
    "faithfulness": llm_评估答案是否忠实于检索内容,
    "answer_relevancy": llm_评估答案是否切题,
    "context_relevancy": 检索上下文与问题的相关性
}
```

**4. 延迟指标**

```python
# 分阶段延迟追踪
latency_breakdown = {
    "embedding_time": ...,     # 向量化时间
    "vector_search_time": ...,  # 向量检索时间
    "llm_generation_time": ..., # LLM生成时间
    "total_e2e_time": ...       # 端到端延迟
}

# SLA标准
P50_latency < 1s      # 50%请求在1秒内完成
P95_latency < 3s      # 95%请求在3秒内完成
P99_latency < 5s      # 99%请求在5秒内完成

# 吞吐
QPS = 100  # 每秒处理100个请求
concurrent_users = 50  # 同时50人在线
```

**5. 综合评估报表**

```python
# 每周自动生成评估报告
weekly_report = {
    "total_requests": 10000,
    "avg_latency_p95": 2.3,  # 秒
    "satisfaction_rate": 0.87,
    "precision@5": 0.82,
    "recall@5": 0.71,
    "mrR": 0.78,
    "regenerate_rate": 0.15,  # 15%请求需要重新生成
    "error_rate": 0.02       # 2%请求报错
}
```

---

### Q14: LoRA 微调中数据集清洗去噪之后有没有做医疗领域人工筛查，如何人工筛查？

A: **医疗领域 LoRA 微调的数据质量保障**：

**为什么医疗领域必须人工筛查？**

```
通用领域: 错误答案 → 用户反馈 → 迭代改进
医疗领域: 错误答案 → 误诊 → 法律风险/人身伤害

风险等级：医疗 > 金融 > 通用
```

**完整的数据质量流程**：

**阶段1：规则清洗**

```python
# 自动规则过滤
def rule_based_cleaning(dataset):
    cleaned = []
    for item in dataset:
        # 过滤无效数据
        if len(item['text']) < 50:
            continue
        if item['text'].count('?') > 5:  # 问句太多
            continue

        # 过滤敏感词
        if contains_sensitive_words(item['text'], medical_sensitive_words):
            continue

        cleaned.append(item)
    return cleaned
```

**阶段2：模型辅助初筛**

```python
# 用小模型做初步质量评分
quality_model = AutoModelForSequenceClassification.from_pretrained(
    "medical-quality-scorer"
)

def model_pre_screening(dataset, threshold=0.7):
    scored = []
    for item in tqdm(dataset):
        score = quality_model.predict(item['text'])
        if score >= threshold:
            scored.append({**item, 'quality_score': score})
    return scored
```

**阶段3：人工医学专家筛查**（核心环节）

**筛查团队配置**：

| 角色             | 人数  | 职责                 |
| ---------------- | ----- | -------------------- |
| 医学编辑（执证） | 2-3人 | 最终审核，判断专业性 |
| 医学研究生       | 3-5人 | 初步筛查，标注疑问   |
| 医学顾问         | 1人   | 复杂案例复核         |

**筛查标准 SOP**：

```python
medical_screening_checklist = {
    "准确性": [
        "诊断描述是否与临床指南一致",
        "用药剂量是否在安全范围内",
        "是否存在过时/淘汰的治疗方法"
    ],
    "完整性": [
        "是否包含必要的禁忌症说明",
        "是否提及重要的不良反应",
        "是否有"请遵医嘱"等必要提示"
    ],
    "清晰性": [
        "术语是否有解释",
        "步骤是否可操作",
        "是否有歧义表述"
    ],
    "安全性": [
        "是否可能造成误自我诊断",
        "是否可能延误正规治疗",
        "是否有免责声明"
    ]
}

# 标注结果
annotation = {
    "sample_id": "xxx",
    "annotator_id": "dr_wang",
    "medical_license": "MD-2024-xxxxx",
    "flags": ["剂量疑似偏高", "缺少禁忌症说明"],
    "decision": "REJECT" | "NEEDS_REVIEW" | "ACCEPT",
    "comment": "..."
}
```

**双人背靠背审核机制**：

```
样本A → 审核员1(医学编辑) → 决策1
      → 审核员2(医学编辑) → 决策2
               ↓
        决策1 == 决策2? ──是──→ 进入数据集
               ↓否
        医学顾问仲裁
               ↓
        问题样本 → 返回上游重新生成
```

**最终质量门槛**：

```
医疗领域LoRA数据集标准：
├── 准确率: 医学编辑审核通过率 ≥ 95%
├── 一致性: 双人审核一致率 ≥ 90%
├── 安全率: 零容忍原则，任何1个安全问题 → 整批打回
└── 可追溯: 每条数据可追溯到标注者+审核者+时间
```

---

## 八、框架对比与架构选择

### Q15: LangGraph 架构已经落后了，现在比较新的框架是什么？

A: **Multi-Agent 框架演进路线**：

**框架演进**：

```
LangChain (2022-2023)
    ↓
LangGraph (2023-2024)  # 状态机+图执行
    ↓
当前主流新框架
├── CrewAI       # 多Agent协作，角色分明
├── AutoGen      # 微软开源，多Agent对话
├── Swarm        # OpenAI教育性实验框架
├── AgentScope   # 上交大，国产
├── CoAI         # 国产，面向生产
└── Dify         # 国产，RAG+Agent可视化编排
```

**主流框架对比**：

| 框架                | 核心特性             | 适用场景          | 上手难度 |
| ------------------- | -------------------- | ----------------- | -------- |
| **CrewAI**    | 角色Agent+任务委派   | 多Agent协作流水线 | 简单     |
| **AutoGen**   | 对话式Agent+代码执行 | 复杂多轮交互      | 中等     |
| **LangGraph** | 状态机+图结构        | 需要细粒度控制    | 较难     |
| **Dify**      | 可视化编排+RAG       | 企业级快速落地    | 简单     |
| **CoAI**      | 国产优化+中文支持    | 国内项目          | 简单     |

**CrewAI 示例**：

```python
from crewai import Agent, Task, Crew

# 定义Agent（带角色）
researcher = Agent(
    role="景点研究专家",
    goal="找到最适合用户的景点",
    backstory="10年旅行规划经验"
)

planner = Agent(
    role="行程规划师",
    goal="生成完美行程",
    backstory="擅长合理安排时间"
)

# 定义任务
task1 = Task(description="搜索北京景点", agent=researcher)
task2 = Task(description="生成行程", agent=planner, context=[task1])

# 启动Crew（自动协作）
crew = Crew(agents=[researcher, planner], tasks=[task1, task2])
result = crew.kickoff()
```

**为什么 LangGraph 可能"落后"**：

```
LangGraph缺点:
- 状态机概念复杂，上手门槛高
- 图执行模型对简单场景过于笨重
- 调试困难，节点/边多了难以追踪
- 社区重心在Agent而非Graph

新框架的改进方向:
- 简化Agent定义（角色/目标/背景 三要素）
- 内置任务委派机制（不再手动编排）
- 可视化编排（Dify一类）
- 原生支持工具调用和RAG
```

**我们项目选择 HelloAgents 的原因**：

- 框架轻量，适合学术/教育场景
- SimpleAgent 封装简洁
- 原生支持 MCP 协议集成

---

### Q16: Open Claude 中有关多 Agent 协同的机制？

A: **Claude Code 的 Multi-Agent 协同机制**（源码学习）：

**核心设计：主Agent + 专家SubAgent**

```
用户请求 → Main Agent (claudeaude)
              ↓ 分解任务
         ┌────┴────┐
    SubAgent1  SubAgent2  SubAgent3
    (代码实现)  (代码审查)  (测试验证)
              ↓
         Main Agent 整合
              ↓
         最终输出
```

**Claude Code 的 Agent 调度设计**（推测/参考）：

```python
# 任务分解策略
class TaskDecomposer:
    def decompose(self, task):
        # 识别任务类型
        if self.is_code_generation(task):
            return [CodeAgent, TestAgent]
        elif self.is_refactoring(task):
            return [AnalyzerAgent, CodeAgent, ReviewAgent]
        elif self.is_research(task):
            return [SearchAgent, SynthesisAgent]
        else:
            return [GeneralAgent]

# 结果聚合策略
class ResultAggregator:
    def aggregate(self, sub_results):
        # 检测冲突
        conflicts = self.detect_conflicts(sub_results)
        if conflicts:
            return self.resolve_conflicts(sub_results)
        return self.merge_results(sub_results)
```

**Claude Code 源码中值得学习的技术点**：

```
1. 流式输出架构
   - Server-Sent Events (SSE)
   - 实现打字机效果

2. 上下文窗口管理
   - 自动上下文压缩
   - 重点信息优先级

3. 工具调用抽象
   - 统一的工具定义格式
   - 工具选择策略

4. 安全沙箱
   - 代码执行的隔离环境
   - 防止恶意操作
```

---

### Q17: Skill 机制如何设置，不是单纯的 skill.md 文档如何编写？

A: **Claude Code Skill 机制深度解析**：

**普通 Skill.md vs 专业 Skill 系统**：

```
普通skill.md (简单):
├── 只是Markdown文档
├── 纯文本描述
└── 无法执行动态逻辑

专业Skill系统 (我们的设计):
├── skill.json    → 元数据 + 触发条件
├── skill.py      → 可执行逻辑
├── prompts/      → 模板目录
└── hooks/        → 生命周期钩子
```

**专业 Skill 目录结构**：

```
skills/
├── my-skill/
│   ├── skill.json           # 元数据
│   ├── __init__.py          # 导出
│   ├── main.py              # 核心逻辑
│   ├── prompts/
│   │   ├── system.md
│   │   └── user.md
│   ├── hooks/
│   │   ├── before_run.py
│   │   └── after_run.py
│   ├── config.yaml          # 配置
│   └── README.md
```

**skill.json 元数据设计**：

```json
{
  "name": "trip-planner",
  "version": "1.0.0",
  "description": "旅行规划助手",
  "trigger": {
    "type": "pattern",
    "pattern": "/trip|旅行规划|帮我安排.*行程"
  },
  "entry": "main.py",
  "hooks": {
    "before_run": "hooks/before_run.py",
    "after_run": "hooks/after_run.py"
  },
  "dependencies": ["amap-mcp", "llm-service"],
  "config": {
    "max_days": 14,
    "default_transportation": "公共交通"
  },
  "permissions": ["file:read", "file:write", "network:api"]
}
```

**Skill 主逻辑实现**：

```python
# main.py
class TripPlannerSkill:
    def __init__(self, config):
        self.max_days = config.get("max_days", 14)
        self.llm = get_llm_service()

    async def run(self, context):
        # 1. 解析用户意图
        intent = self.parse_intent(context.user_input)

        # 2. 触发前置钩子
        await self.run_hook("before_run", context)

        # 3. 执行业务逻辑
        result = await self.plan_trip(intent)

        # 4. 触发后置钩子
        await self.run_hook("after_run", context, result)

        return result

    def parse_intent(self, text):
        prompt = f"从用户输入提取旅行信息: {text}"
        return self.llm.call(prompt)

    async def plan_trip(self, intent):
        # 调用multi-agent系统
        agent = get_trip_planner_agent()
        return agent.plan_trip(intent)
```

**Skill 生命周期钩子**：

```python
# hooks/before_run.py
async def before_run(context):
    # 权限检查
    if not check_permissions(context.user, ["network:api"]):
        raise PermissionError("缺少网络API权限")

    # 参数验证
    if context.params.get("days", 0) > 30:
        raise ValueError("天数不能超过30天")

    # 日志记录
    logger.info(f"Skill触发: {context.skill_name}, 用户: {context.user.id}")

# hooks/after_run.py
async def after_run(context, result):
    # 结果后处理
    if result.success:
        metrics.increment("skill_success", tags={"skill": context.skill_name})
    else:
        metrics.increment("skill_failed", tags={"skill": context.skill_name})

    # 异步通知（如需要）
    await notify_webhook(context.user, result)
```

**Skill 注册与发现机制**：

```python
# skill_registry.py
SKILL_REGISTRY = {}

def register_skill(name, skill_class, trigger):
    SKILL_REGISTRY[name] = {
        "class": skill_class,
        "trigger": trigger
    }

def match_skill(user_input):
    for name, info in SKILL_REGISTRY.items():
        if info["trigger"].matches(user_input):
            return info["class"]
    return None

# 使用
@app.post("/api/chat")
async def chat(request):
    skill = match_skill(request.message)
    if skill:
        result = await skill().run(request)
        return {"type": "skill", "result": result}
    else:
        return {"type": "general", "result": await llm.chat(request)}
```

**对比：普通 skill.md 的局限性**：

| 维度     | skill.md | 专业 Skill 系统      |
| -------- | -------- | -------------------- |
| 触发方式 | 手动调用 | 自动匹配+手动调用    |
| 参数传递 | 纯文本   | 结构化对象           |
| 状态管理 | 无       | 有                   |
| 生命周期 | 无       | before/after/cleanup |
| 测试     | 难以测试 | 单元测试             |
| 权限控制 | 无       | 有                   |

---

## 九、AI Coding 与客户需求

### Q18: AI Coding 不要当作简单的考试，而是去满足客户需求

A: **AI Coding 的正确姿势**：

**错误认知（考试心态）**：

```
❌ 把AI Coding当OJ算法题
❌ 追求最优解/奇技淫巧
❌ 写完代码不测试就交付
❌ 忽略用户实际使用场景
```

**正确认知（产品心态）**：

```
✅ 理解用户背后真实需求
✅ 交付可运行、可维护的代码
✅ 考虑边界情况和错误处理
✅ 关注用户体验而非炫技
```

**客户需求 vs 技术实现**：

```
用户说: "我要一个排序功能"
↓ 分析
真实需求: "我要按销售额排序客户列表，方便我跟进大客户"
↓ 实现
技术方案: 不只是sort()，还要:
- 支持多字段排序
- 支持升序/降序切换
- 前端要有排序指示器
- 要缓存排序结果
- 要处理空值情况
```

**AI Coding 最佳实践**：

```python
# 1. 理解需求先于编码
async def handle_user_request(request):
    # 不要直接开始写代码
    # 先理解用户的真实场景

    clarifying_questions = [
        "这个功能谁来用？"
        "使用频率是怎样的？"
        "失败了怎么办？"
        "有没有历史数据要迁移？"
    ]

    return ask_user(clarifying_questions)
```

```python
# 2. 交付完整功能，而非碎片代码
# 好的交付物:
def create_sortable_table():
    """
    可排序表格组件
    - 支持多字段排序
    - 前端交互完整
    - 有loading状态
    - 有错误提示
    """
    pass
```

```python
# 3. 测试驱动，但不是测试驱动用户需求
# 单元测试: 确保代码正确
# 集成测试: 确保功能完整
# 用户测试: 确保满足需求
```

**面试表达**：

> AI Coding 的核心不是"写代码"，而是"解决问题"。
>
> 比如用户说"帮我做个登录功能"，我不会直接写一个login函数。
>
> 我会先想：
>
> - 登录后用户要做什么？
> - 需要第三方登录吗？
> - 登录失败要通知谁？
> - 要不要加验证码防刷？
>
> 把AI当作工具，但用产品经理的思维去驱动它。

---

### Q19: 工具调用、MCP、Skill 的区别是什么？自己都用过哪些？

A: **工具调用 vs MCP vs Skill 深度对比**：

**1. 工具调用（Tool Calling）**

```
本质: 让LLM调用外部函数的机制

在项目中的使用:
└── Planner Agent 生成 JSON 格式的工具调用指令
    └── LLM输出: {"tool": "maps_weather", "args": {"city": "北京"}}
    └── 框架解析并执行

典型场景:
- LLM生成 → 结构化输出
- Function Calling (OpenAI)
- Action Execution
```

**2. MCP (Model Context Protocol)**

```
本质: AI模型与外部工具/数据源交互的标准化协议

在项目中的使用:
└── amap-mcp-server
    ├── 封装: 高德地图API
    ├── 协议: JSON-RPC over stdio
    └── 暴露工具: maps_text_search, maps_weather, maps_geo...

MCP核心价值:
├── 工具发现 (Discovery)
├── 标准化接口 (Standardized)
├── 安全性 (隔离AI与真实API)
└── 可复用 (多个Agent共用)
```

**3. Skill**

```
本质: 可复用的技能模块，包含prompt模板+执行逻辑+生命周期

在项目中的使用:
└── 未使用（我们用Agent+Prompt实现类似功能）

Skill典型结构:
├── 触发条件 (当用户说"/trip"时)
├── 专用Prompt
├── 业务逻辑
└── 状态管理
```

**三者对比**：

| 维度                   | 工具调用           | MCP             | Skill         |
| ---------------------- | ------------------ | --------------- | ------------- |
| **层级**         | LLM输出协议        | 传输协议        | 应用层封装    |
| **范围**         | 单个函数           | 工具集合        | 完整技能      |
| **触发**         | LLM决定            | 框架解析        | 用户/条件触发 |
| **状态**         | 无状态             | 轻量状态        | 有状态        |
| **复用**         | 低                 | 中              | 高            |
| **在我们的项目** | Planner Agent JSON | amap-mcp-server | 未使用        |

**实际应用组合**：

```
场景: 用户要求规划北京三日游

Skill层: "/trip" 触发旅行规划技能
   ↓
MCP层: 通过高德MCP获取POI/天气
   ↓
工具调用层: 内部工具如"生成JSON"被调用
```

**我用过的技术栈**：

```
工具调用:
├── OpenAI Function Calling
├── LangChain Tool
└── HelloAgents MCP

MCP:
├── amap-mcp-server (高德地图)
├── filesystem-mcp (本地文件)
└── 自定义MCP Server

Skill:
├── Claude Code内置Skill
├── 自定义Trip Planner Skill
└── 学术写作Skill (paper-self-review)
```

---

### Q20: Claude Code 相较于其他技术的护城河在于？

A: **Claude Code 护城河分析**：

**技术层护城河**：

```
1. 超长上下文 (200K tokens)
   - 可一次性处理整个代码库
   - 其他竞品: GPT-4o 128K, Copilot 更短

2. Claude模型本身的能力
   - Instruction Following更强
   - 代码风格一致性好
   - 减少幻觉(Hallucination)

3. 工具调用安全机制
   - 权限确认机制
   - 操作可回滚
   - 沙箱隔离
```

**产品层护城河**：

```
1. IDE原生集成
   - VS Code/Claude IDE无缝衔接
   - 不需要切换工具

2. 项目上下文理解
   - 自动解析项目结构
   - 理解依赖关系
   - CLAUDE.md 项目记忆

3. Agent记忆系统
   - Memory持久化
   - 多会话学习
   - 个性化适应
```

**生态护城河**：

```
1. MCP协议推广
   - 越来越多的工具接入MCP
   - 网络效应

2. Skill生态系统
   - 开发者贡献Skills
   - 可复用模板

3. Claude.ai 平台
   - 网页端/桌面端/CLI
   - 统一体验
```

**对比竞品**：

| 维度      | Claude Code | GitHub Copilot | Cursor |
| --------- | ----------- | -------------- | ------ |
| 上下文    | 200K        | 较短           | 中等   |
| Agent能力 | 强          | 中             | 强     |
| MCP生态   | 建设中      | 弱             | 弱     |
| 代码质量  | 高          | 中             | 高     |
| 价格      | 中          | 中             | 高     |

**面试回答参考**：

> Claude Code的核心护城河不是某一个技术点，而是**产品-技术-生态的正循环**：
>
> 1. **模型层**：Claude 3.5 Sonnet 的代码能力本身就是业界领先
> 2. **产品层**：超长上下文 + 项目理解 = 其他产品难以复制的体验
> 3. **生态层**：MCP 协议正在成为行业标准，越来越多工具接入后，迁移成本变高
>
> 就像iPhone的护城河不是触摸屏而是整个iOS生态，Claude Code的护城河也是模型能力 × 产品体验 × MCP生态的三重壁垒。

---

### Q21: 你用的向量数据库是什么？还用过其他向量数据库吗？

A: **我们项目的实际情况**：

**当前项目没有使用向量数据库**，原因是我们采用 **Agent + MCP 实时 API 调用** 模式，而非 RAG（Retrieval Augmented Generation）：

```
我们项目: 用户请求 → Agent → MCP工具 → 高德地图API（实时）
RAG项目:   用户请求 → 检索向量库 → LLM生成
```

**何时用向量数据库 vs 实时 API**：
| 场景 | 方案 | 原因 |
|------|------|------|
| 景点/酒店/天气搜索 | 实时API (MCP) | 数据频繁变化，实时性要求高 |
| 景点介绍/攻略知识库 | 向量数据库 (RAG) | 静态文档，适合预索引 |
| 用户历史偏好 | 向量数据库 | 需要相似度匹配 |
| 实时排队人数 | 实时API | 数据秒级变化 |

**如果项目引入 RAG，推荐的向量数据库**：

**1. ChromaDB（轻量级，推荐学术/小规模）**
```python
import chromadb

client = chromadb.Client()
collection = client.create_collection("attractions")

# 存入向量
collection.add(
    documents=["故宫是中国明清两代的皇家宫殿..."],
    embeddings=[[0.1, 0.2, ...]],  # 768维
    ids=["poi_001"]
)

# 检索
results = collection.query(
    query_embeddings=[[0.1, 0.2, ...]],
    n_results=3
)
```

**2. Milvus（大规模生产级）**
```python
from pymilvus import connections, Collection

connections.connect("default", host="localhost", port="19530")
collection = Collection("attractions")
collection.load()

# ANN检索
search_params = {"metric_type": "IP", "params": {"nprobe": 10}}
results = collection.search(
    data=[[0.1, 0.2, ...]],
    anns_field="vector",
    param=search_params,
    limit=10
)
```

**3. Qdrant（高性能 Rust 实现）**
```python
from qdrant_client import QdrantClient

client = QdrantClient("localhost", port=6333)

# 检索
results = client.search(
    collection_name="attractions",
    query_vector=[0.1, 0.2, ...],
    limit=10
)
```

**向量数据库全对比**：
| 数据库 | 开发商 | 规模 | 性能 | 部署 | 免费 | 适用场景 |
|--------|--------|------|------|------|------|---------|
| **Chroma** | Chroma | 百万级 | 中 | 单机/嵌入 | 是 | 学术、原型、快速开发 |
| **Milvus** | Zilliz | 十亿级 | 高 | K8s/分布式 | 是 | 生产级大规模 |
| **Qdrant** | Qdrant | 亿级 | 极高 | Docker/K8s | 是 | 高性能需求 |
| **Weaviate** | Weaviate | 亿级 | 高 | Docker/K8s | 是 | 混合检索(向量+全文) |
| **Pinecone** | Pinecone | 无限 | 高 | 云服务 | 付费 | 不想运维 |
| **FAISS** | Meta | 依赖内存 | 极高 | 嵌入代码 | 是 | 离线/超大规模 |
| **pgvector** | PostgreSQL | 百万级 | 中 | PG扩展 | 是 | 已有PG团队 |

**选型决策树**：
```
规模 < 100万向量？
├─ 是 → Chroma（最简单）或 pgvector（已有PG）
└─ 否 → 规模 100万~1亿？
           ├─ 是 → Qdrant（性能好）或 Milvus（功能全）
           └─ 否 → 1亿+ → Milvus分布式 或 FAISS

要不要自己运维？
├─ 不要 → Pinecone（云托管）
└─ 要 → Qdrant/Docker 或 Milvus/K8s

要不要混合检索（向量+关键词）？
├─ 要 → Weaviate 或 Qdrant（支持hybrid search）
└─ 不要 → 其他都可以
```

**我使用过的向量数据库**：
| 数据库 | 场景 | 感受 |
|--------|------|------|
| **Chroma** | 学术知识库，百万级 | 上手最快，但生产环境慎用 |
| **Milvus** | 千万级知识库检索 | 功能全，K8s部署复杂 |
| **Qdrant** | 亿级文档检索 | Rust实现，性能很高，API设计清晰 |
| **FAISS** | 超大规模离线检索 | 内存占用大，但速度极快 |
| **pgvector** | 中小规模（<100万）| 复用现有PG，够用但不够快 |

**面试加分回答**：
> 我们项目目前用的是 MCP 实时 API 模式，没有直接用向量数据库。
>
> 但我在其他项目中用过 Chroma（学术知识库，百万级）和 Milvus（企业级文档检索，千万级）。
>
> 选型心得：Chroma 适合快速验证，Milvus 适合生产，Qdrant 是目前性能和体验平衡最好的。

---

*文档生成时间：2026-04-17*
*最后更新：补充向量数据库问题 + 阿里系面试题 + RAG/Agent进阶问题*
