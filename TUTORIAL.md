# Multi-Agent Trip Planner - 完整使用教程

基于 HelloAgents 框架的智能旅行规划助手,集成了高德地图 MCP 服务,使用多智能体协作生成个性化旅行计划。

---

## 目录

1. [项目概述](#项目概述)
2. [系统架构](#系统架构)
3. [核心模块详解](#核心模块详解)
4. [关键设计决策](#关键设计决策)
5. [难点与解决方案](#难点与解决方案)
6. [API 接口文档](#api-接口文档)
7. [前端组件说明](#前端组件说明)
8. [快速开始](#快速开始)
9. [常见问题](#常见问题)

---

## 项目概述

### 什么是 Multi-Agent Trip Planner?

这是一个**多智能体协作的旅行规划系统**,用户输入目的地、日期、偏好等信息后,系统会自动:

1. 调用高德地图 API 搜索景点、酒店
2. 查询旅行期间的天气预报
3. 由 LLM 整合所有信息生成完整行程

### 技术栈

| 层级 | 技术 |
|------|------|
| **AI Agent 框架** | HelloAgents (SimpleAgent) |
| **LLM** | OpenAI GPT-4 / DeepSeek (可配置) |
| **地图服务** | 高德地图 MCP Server |
| **图片服务** | Unsplash API |
| **后端** | FastAPI + Python 3.10+ |
| **前端** | Vue 3 + TypeScript + Vite + Ant Design Vue |
| **地图前端** | 高德地图 JavaScript API |

---

## 系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                          用户界面 (Vue 3)                        │
│  ┌──────────────────┐    ┌──────────────────────────────────┐  │
│  │   Home.vue       │    │   Result.vue                     │  │
│  │   - 输入表单      │    │   - 行程展示                      │  │
│  │   - 日期选择      │    │   - 地图标记                      │  │
│  │   - 偏好标签      │    │   - 编辑/导出功能                  │  │
│  └──────────────────┘    └──────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                      FastAPI 后端服务                            │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                      API Routes                             ││
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐  ││
│  │  │ /trip/   │  │  /poi/   │  │ /map/    │  │ /health   │  ││
│  │  │ plan     │  │ detail   │  │ weather  │  │           │  ││
│  │  └──────────┘  └──────────┘  └──────────┘  └───────────┘  ││
│  └─────────────────────────────────────────────────────────────┘│
│                                  │                              │
│                                  ▼                              │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                 Multi-Agent Trip Planner                    ││
│  │                                                              ││
│  │   ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐    ││
│  │   │ Attraction  │  │  Weather    │  │     Hotel       │    ││
│  │   │   Agent     │  │   Agent     │  │     Agent       │    ││
│  │   │ (景点搜索)   │  │  (天气查询)  │  │    (酒店推荐)    │    ││
│  │   └─────────────┘  └─────────────┘  └─────────────────┘    ││
│  │           │                │                  │              ││
│  │           └────────────────┼──────────────────┘              ││
│  │                            ▼                                ││
│  │                   ┌─────────────────┐                       ││
│  │                   │   Planner       │                       ││
│  │                   │   Agent         │                       ││
│  │                   │  (行程规划)      │                       ││
│  │                   └─────────────────┘                       ││
│  └─────────────────────────────────────────────────────────────┘│
│                                  │                              │
│                                  ▼                              │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                      MCP Tools                               ││
│  │  ┌──────────────────────────────────────────────────────┐  ││
│  │  │              amap-mcp-server (高德地图)                 │  ││
│  │  │  • maps_text_search    • maps_weather                  │  ││
│  │  │  • maps_direction_*   • maps_geo                       │  ││
│  │  └──────────────────────────────────────────────────────┘  ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

---

## 核心模块详解

### 1. 多智能体系统 (`trip_planner_agent.py`)

这是项目**最核心的模块**,实现了多智能体协作架构。

#### Agent 角色定义

| Agent | Prompt 用途 | 工具能力 |
|-------|-----------|---------|
| `attraction_agent` | 搜索景点 POI | `maps_text_search` |
| `weather_agent` | 查询天气预报 | `maps_weather` |
| `hotel_agent` | 推荐酒店 | `maps_text_search` |
| `planner_agent` | 生成行程计划 | 无 (仅 LLM) |

#### 核心函数

**`MultiAgentTripPlanner.__init__()`**
- 初始化 4 个 SimpleAgent 实例
- 创建共享的 MCP 工具 (`amap-mcp-server`)
- 设置各 Agent 的 system prompt

**`MultiAgentTripPlanner.plan_trip(request)`**
```
用户请求 → 4步协作 → 最终行程
   │
   ├─→ [Step 1] attraction_agent.run() → 景点列表
   │
   ├─→ [Step 2] weather_agent.run()    → 天气预报
   │
   ├─→ [Step 3] hotel_agent.run()      → 酒店推荐
   │
   └─→ [Step 4] planner_agent.run()   → 整合生成最终行程
```

**`_build_attraction_query()` / `_build_planner_query()`**
- 构建发送给 LLM 的 prompt
- **关键**: 需要包含工具调用格式 `[TOOL_CALL:tool_name:params]`

**`_parse_response()`**
- 从 LLM 输出中提取 JSON
- 支持三种格式: ` ```json ... ``` `、直接 JSON 对象、包含 `{}` 的文本

**`_create_fallback_plan()`**
- 当 Agent 执行失败时的备用方案
- 生成基于请求参数的简单行程

---

### 2. MCP 工具封装 (`amap_service.py`)

#### `AmapService` 类方法

| 方法 | 功能 | 底层调用 |
|------|------|---------|
| `search_poi()` | 搜索 POI | `maps_text_search` |
| `get_weather()` | 查询天气 | `maps_weather` |
| `plan_route()` | 路线规划 | `maps_direction_*` |
| `geocode()` | 地址转坐标 | `maps_geo` |
| `get_poi_detail()` | POI 详情 | `maps_search_detail` |

#### 单例模式

使用全局变量 `_amap_service` 确保 MCP 工具只初始化一次:
```python
_amap_service = None

def get_amap_service() -> AmapService:
    global _amap_service
    if _amap_service is None:
        _amap_service = AmapService()
    return _amap_service
```

---

### 3. LLM 服务 (`llm_service.py`)

#### `HelloAgentsLLM`

项目使用 HelloAgents 框架的 `HelloAgentsLLM`,它会自动从环境变量读取配置:

| 环境变量 | 说明 | 默认值 |
|---------|------|-------|
| `OPENAI_API_KEY` | API 密钥 | - |
| `OPENAI_BASE_URL` | API 地址 | `https://api.openai.com/v1` |
| `OPENAI_MODEL` | 模型名称 | `gpt-4` |

#### 单例模式

`_llm_instance` 全局变量确保 LLM 实例只创建一次。

---

### 4. 数据模型 (`schemas.py`)

#### 请求模型

```python
TripRequest {
    city: str              # 目的地城市
    start_date: str        # 开始日期 (YYYY-MM-DD)
    end_date: str          # 结束日期 (YYYY-MM-DD)
    travel_days: int       # 旅行天数 (1-30)
    transportation: str     # 交通方式
    accommodation: str     # 住宿偏好
    preferences: List[str] # 旅行偏好标签
    free_text_input: str   # 额外要求
}
```

#### 响应模型

```python
TripPlan {
    city: str
    start_date: str
    end_date: str
    days: List[DayPlan]    # 每日行程
    weather_info: List[WeatherInfo]
    overall_suggestions: str
    budget: Budget         # 预算信息
}

DayPlan {
    date: str
    day_index: int
    description: str
    transportation: str
    accommodation: str
    hotel: Hotel
    attractions: List[Attraction]
    meals: List[Meal]
}
```

#### Pydantic 验证

`WeatherInfo` 中的温度字段使用 `field_validator` 自动解析字符串中的数字:
```python
@field_validator('day_temp', 'night_temp', mode='before')
def parse_temperature(cls, v):
    if isinstance(v, str):
        v = v.replace('°C', '').replace('℃', '').replace('°', '').strip()
        try:
            return int(v)
        except ValueError:
            return 0
    return v
```

---

### 5. 前端 Vue 组件

#### `Home.vue` - 输入表单

**表单分区**:
1. **目的地与日期**: 城市选择、日期范围、天数自动计算
2. **偏好设置**: 交通方式、住宿类型、旅行偏好标签
3. **额外要求**: 自由文本输入

**关键逻辑**:
- `watch([start_date, end_date])`: 自动计算并限制旅行天数 (≤30)
- `handleSubmit()`: 模拟进度更新,调用 API,跳转到结果页

#### `Result.vue` - 结果展示

**功能模块**:
1. **侧边导航**: 锚点链接到各 section
2. **行程概览**: 城市、日期、总体建议
3. **预算明细**: 景点、酒店、餐饮、交通分项
4. **景点地图**: 高德地图 + 标记 + 路线
5. **每日行程**: 可折叠面板,含景点/酒店/餐饮
6. **天气信息**: 每日天气预报卡片

**编辑功能**:
- `toggleEditMode()`: 切换编辑状态
- `moveAttraction()`: 调整景点顺序
- `deleteAttraction()`: 删除景点

**导出功能**:
- `exportAsImage()`: 使用 html2canvas 导出 PNG
- `exportAsPDF()`: 使用 jspdf 导出 PDF

**地图功能**:
- `initMap()`: 加载高德地图 JS API
- `addAttractionMarkers()`: 添加标记和信息窗口
- `drawRoutes()`: 按天分组绘制路线

---

## 关键设计决策

### 1. 多 Agent 协作 vs 单一 Agent

**选择**: 多 Agent 协作

**原因**:
- 单一 Agent 难以同时处理多个工具调用
- 分工明确: 搜索 Agent 专注搜索,规划 Agent 专注整合
- 便于单独调试和扩展

**权衡**:
- 需要设计 Agent 间的数据传递格式
- Prompt 工程需要针对每个 Agent 单独优化

### 2. MCP 协议集成

**选择**: 通过 `hello_agents.tools.MCPTool` 封装高德地图

**原因**:
- MCP 协议是标准化的 Agent 工具调用协议
- HelloAgents 框架提供了现成的 MCP 支持
- 可扩展: 未来可接入更多 MCP 服务

### 3. 单例模式

**选择**: 全局变量 + 函数封装

```python
_service = None
def get_service():
    global _service
    if _service is None:
        _service = Service()
    return _service
```

**原因**:
- MCP 工具初始化成本高,避免重复创建
- FastAPI 的依赖注入不适用于长连接服务
- 简化测试: 便于 mock 和重置

---

## 难点与解决方案

### 难点 1: LLM 输出格式不稳定

**问题**: LLM 返回的 JSON 可能包含 markdown 格式、额外文本、甚至格式错误。

**解决方案**:
```python
def _parse_response(self, response: str, request: TripRequest) -> TripPlan:
    # 尝试多种格式
    if "```json" in response:
        # 提取 ```json ... ``` 块
        json_str = extract_json_block(response)
    elif "```" in response:
        # 提取 ``` ... ``` 块
        json_str = extract_code_block(response)
    elif "{" in response and "}" in response:
        # 直接查找 JSON 对象
        json_str = extract_json_object(response)
    else:
        raise ValueError("响应中未找到JSON数据")

    # 解析 JSON (失败时使用 fallback)
    try:
        data = json.loads(json_str)
        return TripPlan(**data)
    except Exception:
        return self._create_fallback_plan(request)
```

### 难点 2: MCP 工具调用格式

**问题**: Agent 需要精确的格式才能正确调用 MCP 工具。

**解决方案**:
- 在 system prompt 中明确列出**工具调用格式示例**
- 使用 `[TOOL_CALL:tool_name:param=value]` 格式
- 提示 Agent "必须使用工具,不要自己编造"

示例 Prompt:
```
**工具调用格式:**
使用maps_text_search工具时,必须严格按照以下格式:
`[TOOL_CALL:amap_maps_text_search:keywords=景点关键词,city=城市名]`

**示例:**
用户: "搜索北京的历史文化景点"
你的回复: [TOOL_CALL:amap_maps_text_search:keywords=历史文化,city=北京]
```

### 难点 3: 跨域和 API 安全

**问题**: 前端直接调用高德地图 API 会暴露密钥。

**解决方案**:
- 后端通过 FastAPI 代理高德地图请求
- 使用环境变量存储密钥 (`AMAP_API_KEY`)
- CORS 配置限制允许的来源

### 难点 4: 地图服务可靠性

**问题**: 高德地图 API 可能因网络问题失败。

**解决方案**:
- `AmapService` 方法包含 try-except 错误处理
- 失败时返回空列表或默认值
- 前端使用占位图和错误处理

```python
# 前端图片加载失败处理
const handleImageError = (event: Event) => {
  const img = event.target as HTMLImageElement
  img.src = 'data:image/svg+xml,...占位图...'
}
```

### 难点 5: 行程数据在页面间传递

**问题**: Vue 路由跳转时需要传递大量行程数据。

**解决方案**:
- 使用 `sessionStorage` 存储 JSON 字符串
- 页面加载时从 sessionStorage 读取
- 优点: 页面刷新不丢失数据,简单易用

```javascript
// Home.vue - 保存
sessionStorage.setItem('tripPlan', JSON.stringify(response.data))

// Result.vue - 读取
const data = sessionStorage.getItem('tripPlan')
if (data) {
  tripPlan.value = JSON.parse(data)
}
```

---

## API 接口文档

### 1. 生成旅行计划

**POST** `/api/trip/plan`

**请求体**:
```json
{
  "city": "北京",
  "start_date": "2025-06-01",
  "end_date": "2025-06-03",
  "travel_days": 3,
  "transportation": "公共交通",
  "accommodation": "经济型酒店",
  "preferences": ["历史文化", "美食"],
  "free_text_input": "希望多安排一些博物馆"
}
```

**响应**:
```json
{
  "success": true,
  "message": "旅行计划生成成功",
  "data": {
    "city": "北京",
    "start_date": "2025-06-01",
    "end_date": "2025-06-03",
    "days": [...],
    "weather_info": [...],
    "overall_suggestions": "...",
    "budget": {...}
  }
}
```

### 2. 健康检查

**GET** `/api/trip/health`

**响应**:
```json
{
  "status": "healthy",
  "service": "trip-planner",
  "agent_name": "旅行规划助手",
  "tools_count": 5
}
```

### 3. POI 搜索

**GET** `/api/poi/search?keywords=故宫&city=北京`

### 4. POI 详情

**GET** `/api/poi/detail/{poi_id}`

### 5. 景点图片

**GET** `/api/poi/photo?name=故宫`

---

## 前端组件说明

### 目录结构

```
frontend/src/
├── main.ts              # Vue 应用入口
├── App.vue              # 根组件
├── services/
│   └── api.ts           # API 调用封装
├── types/
│   └── index.ts        # TypeScript 类型定义
└── views/
    ├── Home.vue         # 首页 (输入表单)
    └── Result.vue       # 结果页 (行程展示)
```

### API 服务封装 (`api.ts`)

```typescript
// 生成旅行计划
export const generateTripPlan = async (data: TripFormData): Promise<TripPlanResponse> => {
  const response = await axios.post('/api/trip/plan', data)
  return response.data
}

// 获取景点图片
export const getAttractionPhoto = async (name: string): Promise<any> => {
  const response = await axios.get(`/api/poi/photo?name=${encodeURIComponent(name)}`)
  return response.data
}
```

### 路由配置

使用 Vue Router,主要路由:
- `/`: Home.vue (首页)
- `/result`: Result.vue (结果页)

---

## 快速开始

### 1. 环境准备

```bash
# Python 3.10+
python --version

# Node.js 16+
node --version

# uv (推荐) 或 pip
pip install uv
```

### 2. 获取 API 密钥

| 服务 | 获取地址 | 用途 |
|------|---------|------|
| 高德地图 | https://console.amap.com/dev/key/app |
| OpenAI/DeepSeek | https://platform.openai.com 或对应平台 |

### 3. 后端安装

```bash
cd backend

# 创建虚拟环境
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 安装依赖
pip install -r requirements.txt

# 配置环境变量
cp .env.example .env
# 编辑 .env,填入 API 密钥
```

### 4. 前端安装

```bash
cd frontend

# 安装依赖
npm install

# 配置环境变量
cp .env.example .env
# 编辑 .env,填入高德地图 Web JS API Key
```

### 5. 启动服务

```bash
# 终端 1: 启动后端
cd backend
python run.py
# 访问 http://localhost:8000/docs 查看 API 文档

# 终端 2: 启动前端
cd frontend
npm run dev
# 访问 http://localhost:5173
```

---

## 常见问题

### Q: MCP 工具调用失败怎么办?

**A**: 检查以下几点:
1. `AMAP_API_KEY` 是否正确配置
2. 网络能否访问高德地图 API
3. 查看后端日志中的错误信息
4. 确认 `amap-mcp-server` 是否通过 `uvx` 正常安装

### Q: LLM 返回格式错误?

**A**: 这是常见问题,系统内置了 fallback 机制:
1. 检查 prompt 是否清晰
2. 查看日志中 LLM 的原始输出
3. 尝试简化请求或减少天数

### Q: 地图不显示?

**A**: 检查:
1. `VITE_AMAP_WEB_JS_KEY` 是否配置 (注意是 Web 端 Key,不是 Web 服务 Key)
2. 浏览器控制台是否有跨域错误
3. 景点是否有有效的经纬度坐标

### Q: 如何扩展新的 Agent?

**A**: 在 `trip_planner_agent.py` 中:
1. 定义新的 `*_AGENT_PROMPT`
2. 在 `__init__` 中创建 `SimpleAgent` 实例
3. 在 `plan_trip()` 中调用新的 Agent

---

## 附录: 目录结构

```
Multi-Agent-Trip-Planner/
├── backend/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── config.py              # 配置管理
│   │   ├── agents/
│   │   │   ├── __init__.py
│   │   │   └── trip_planner_agent.py  # 多智能体核心
│   │   ├── api/
│   │   │   ├── __init__.py
│   │   │   ├── main.py           # FastAPI 应用
│   │   │   └── routes/
│   │   │       ├── __init__.py
│   │   │       ├── trip.py        # 旅行规划 API
│   │   │       ├── poi.py         # POI 相关 API
│   │   │       └── map.py         # 地图 API
│   │   ├── models/
│   │   │   ├── __init__.py
│   │   │   └── schemas.py        # Pydantic 数据模型
│   │   └── services/
│   │       ├── __init__.py
│   │       ├── amap_service.py    # 高德地图服务
│   │       ├── llm_service.py     # LLM 服务
│   │       └── unsplash_service.py # 图片服务
│   ├── requirements.txt
│   ├── .env.example
│   └── run.py                     # 启动脚本
├── frontend/
│   ├── src/
│   │   ├── main.ts
│   │   ├── App.vue
│   │   ├── services/
│   │   │   └── api.ts
│   │   ├── types/
│   │   │   └── index.ts
│   │   └── views/
│   │       ├── Home.vue
│   │       └── Result.vue
│   ├── package.json
│   ├── vite.config.ts
│   └── .env.example
└── docs/
    └── TUTORIAL.md               # 本教程
```

---

*最后更新: 2026-03-29*
