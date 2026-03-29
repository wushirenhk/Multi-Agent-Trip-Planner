# Multi-Agent Trip Planner - 智能旅行规划助手

基于 HelloAgents 框架的多智能体旅行规划系统,集成高德地图 MCP 服务,使用 Vue 3 + FastAPI 构建的现代化 Web 应用。

[English](README_en.md) | 简体中文

---

## 功能特点

- **多智能体协作**: 景点搜索、天气查询、酒店推荐、行程规划由专门的 Agent 分工处理
- **AI 驱动**: 基于 LLM 智能生成个性化旅行计划
- **高德地图集成**: 实时 POI 搜索、路线规划、天气预报
- **现代化前端**: Vue 3 + TypeScript + Ant Design Vue,响应式设计
- **交互式地图**: 高德地图可视化,景点标记与路线绘制
- **编辑与导出**: 支持行程编辑、图片/PDF 导出

---

## 系统架构

```
用户输入 → [Vue3 Frontend] → [FastAPI Backend] → [Multi-Agent System]
                                                    ├── Attraction Agent (景点搜索)
                                                    ├── Weather Agent (天气查询)
                                                    ├── Hotel Agent (酒店推荐)
                                                    └── Planner Agent (行程规划)
                                                              ↓
                                                    [HelloAgents MCP]
                                                              ↓
                                                    [amap-mcp-server]
```

---

## 技术栈

| 层级 | 技术 |
|------|------|
| **AI Agent** | HelloAgents (SimpleAgent) |
| **LLM** | OpenAI GPT-4 / DeepSeek 等 |
| **地图服务** | 高德地图 MCP Server + JS API |
| **后端** | FastAPI + Python 3.10+ |
| **前端** | Vue 3 + TypeScript + Vite |
| **UI** | Ant Design Vue |

---

## 项目结构

```
Multi-Agent-Trip-Planner/
├── backend/                    # FastAPI 后端
│   ├── app/
│   │   ├── agents/             # 多智能体核心
│   │   │   └── trip_planner_agent.py
│   │   ├── api/routes/        # API 路由
│   │   │   ├── trip.py         # 旅行规划接口
│   │   │   ├── poi.py          # POI 接口
│   │   │   └── map.py          # 地图接口
│   │   ├── services/           # 服务层
│   │   │   ├── amap_service.py # 高德地图封装
│   │   │   ├── llm_service.py # LLM 服务
│   │   │   └── unsplash_service.py
│   │   ├── models/schemas.py   # 数据模型
│   │   └── config.py           # 配置管理
│   ├── requirements.txt
│   └── run.py                  # 启动脚本
│
├── frontend/                   # Vue 3 前端
│   ├── src/
│   │   ├── views/
│   │   │   ├── Home.vue        # 首页表单
│   │   │   └── Result.vue      # 结果展示
│   │   ├── services/api.ts     # API 调用
│   │   └── types/index.ts      # 类型定义
│   └── package.json
│
└── docs/
    └── TUTORIAL.md             # 完整教程
```

---

## 快速开始

### 1. 环境准备

- Python 3.10+
- Node.js 16+
- 高德地图 API Key
- LLM API Key (OpenAI/DeepSeek 等)

### 2. 后端安装

```bash
cd backend

# 创建虚拟环境
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 安装依赖
pip install -r requirements.txt

# 配置环境变量
cp .env.example .env
# 编辑 .env 填入:
#   AMAP_API_KEY=你的高德地图Key
#   OPENAI_API_KEY=你的LLM密钥
```

### 3. 前端安装

```bash
cd frontend

npm install

# 配置环境变量
cp .env.example .env
# 编辑 .env 填入:
#   VITE_AMAP_WEB_JS_KEY=你的高德Web端Key
```

### 4. 启动服务

```bash
# 终端 1: 后端 (http://localhost:8000)
cd backend
python run.py

# 终端 2: 前端 (http://localhost:5173)
cd frontend
npm run dev
```

### 5. 访问应用

打开浏览器访问 `http://localhost:5173`

---

## 核心模块说明

### 多智能体系统 (`trip_planner_agent.py`)

系统包含 4 个专门的 Agent:

| Agent | 职责 | 使用工具 |
|-------|------|---------|
| Attraction Agent | 搜索景点 | `maps_text_search` |
| Weather Agent | 查询天气 | `maps_weather` |
| Hotel Agent | 推荐酒店 | `maps_text_search` |
| Planner Agent | 生成行程 | 无 (仅 LLM) |

**工作流程**:
```
用户请求
    ↓
Attraction Agent → 景点列表
Weather Agent   → 天气预报
Hotel Agent     → 酒店推荐
    ↓
Planner Agent (整合所有信息)
    ↓
最终行程 JSON
```

### MCP 工具封装 (`amap_service.py`)

`AmapService` 类封装了高德地图 MCP 服务:

- `search_poi()` - POI 搜索
- `get_weather()` - 天气查询
- `plan_route()` - 路线规划
- `geocode()` - 地理编码
- `get_poi_detail()` - POI 详情

---

## API 接口

启动后端后访问 `http://localhost:8000/docs` 查看完整 API 文档。

### 主要端点

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/trip/plan` | 生成旅行计划 |
| GET | `/api/trip/health` | 健康检查 |
| GET | `/api/poi/search` | 搜索 POI |
| GET | `/api/poi/detail/{id}` | POI 详情 |
| GET | `/api/poi/photo` | 获取景点图片 |

### 请求示例

```bash
curl -X POST http://localhost:8000/api/trip/plan \
  -H "Content-Type: application/json" \
  -d '{
    "city": "北京",
    "start_date": "2025-06-01",
    "end_date": "2025-06-03",
    "travel_days": 3,
    "transportation": "公共交通",
    "accommodation": "经济型酒店",
    "preferences": ["历史文化", "美食"]
  }'
```

---

## 配置说明

### 后端环境变量 (`.env`)

| 变量 | 说明 | 必需 |
|------|------|------|
| `AMAP_API_KEY` | 高德地图 Web 服务 API Key | 是 |
| `OPENAI_API_KEY` | LLM API 密钥 | 是 |
| `OPENAI_BASE_URL` | LLM API 地址 | 否 |
| `OPENAI_MODEL` | LLM 模型名称 | 否 |

### 前端环境变量 (`.env`)

| 变量 | 说明 | 必需 |
|------|------|------|
| `VITE_AMAP_WEB_JS_KEY` | 高德地图 Web 端(JS API) Key | 是 |

---

## 难点与解决方案

### 1. LLM 输出格式不稳定

**问题**: LLM 可能返回带 markdown 格式或额外文本的响应。

**解决**: 实现多种格式的 JSON 提取逻辑,并在失败时使用 fallback:

```python
# 尝试提取 ```json ... ``` 块
# 尝试提取 ``` ... ``` 块
# 尝试直接提取 JSON 对象
# 全部失败则使用 fallback 计划
```

### 2. MCP 工具调用格式

**问题**: Agent 需要精确格式才能正确调用工具。

**解决**: 在 prompt 中明确列出调用格式示例:

```
使用maps_text_search工具时,必须严格按照以下格式:
[TOOL_CALL:amap_maps_text_search:keywords=关键词,city=城市名]
```

### 3. 跨域与 API 安全

**问题**: 前端直接调用地图 API 会暴露密钥。

**解决**: 后端代理所有地图请求,CORS 配置限制来源。

---

## 常见问题

**Q: MCP 工具调用失败?**
- 检查 `AMAP_API_KEY` 是否正确
- 确认网络能访问高德地图
- 查看后端日志错误信息

**Q: 地图不显示?**
- 确认 `VITE_AMAP_WEB_JS_KEY` 配置正确 (是 Web 端 Key)
- 检查浏览器控制台跨域错误

**Q: LLM 返回格式错误?**
- 系统有 fallback 机制
- 可简化请求或减少天数

---

## 教程文档

详细的教程文档请查看 [docs/TUTORIAL.md](docs/TUTORIAL.md),包含:

- 系统架构详解
- 核心模块函数说明
- 关键设计决策
- 难点与解决方案
- API 完整文档
- 前端组件说明
- 常见问题解答

---

## License

MIT License
