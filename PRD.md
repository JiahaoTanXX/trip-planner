# 智能旅游规划 Agent - 产品需求文档

**文档版本**：v1.0.16
**最后更新**：2026-04-28

## 1. 产品概述

### 1.1 产品定位
一款基于 AI 的智能旅游规划工具，为"懒人规划"用户服务。通过表单向导收集用户需求，LLM 自动生成个性化行程方案。

### 1.2 核心价值
- 节省用户 80% 的规划时间
- 一键生成完整行程
- 整合多源数据，提供一站式体验

### 1.3 目标用户
- 不想花时间研究路线的旅行爱好者
- 懒人规划：时间宝贵、预算充足、不想自己做攻略

> **用户特征**：追求省时省力，希望一键生成完整行程

### 1.4 适用范围
- 初期：中国境内城市
- 后期：可扩展至境外城市
- 平台：Web 端（H5 / 桌面浏览器）

---

## 2. 用户需求

### 2.1 用户画像与痛点

| 用户类型 | 特征 | 痛点 |
|---------|------|------|
| 懒人规划 | 时间宝贵，预算充足，不想自己做攻略 | 希望一键生成完整行程，省时省力 |

> **说明**：初期仅针对懒人规划用户，其他用户类型在 V2 后扩展

### 2.2 核心功能需求

#### 2.2.1 目的地选择
- 支持搜索中国境内城市
- 显示城市基本信息（著名景点、代表美食、最佳季节）
- **说明**：美食信息仅用于城市介绍，不参与行程规划（餐饮规划暂不考虑）
- **错误码区分**：
  - E001：城市名称在支持列表中完全不存在
  - E004：城市存在但可推荐景点 < 阈值（< 1 个）

#### 2.2.2 行程参数配置
| 参数 | 类型 | 说明 |
|------|------|------|
| 目的地 | 必填 | 城市名称（直接填入，校验格式：去除空白后至少2个中文字符） |
| 出发日期 | 必填 | 日期选择器（**不可选择过去日期**） |
| 旅行天数 | 必填 | 1-7 天 |
| 人数 | 必填 | 1-5 人（成人，**不含儿童/婴幼儿**；儿童/婴幼儿不计入人数） |
| 预算范围 | 必填 | 区间选择（枚举索引）：0=<1000元 / 1=1000-3000元 / 2=3000-5000元 / 3=>5000元（整个行程总预算，区间为左闭右开：0 表示 0≤预算<1000，1 表示 1000≤预算<3000，以此类推） |
| 出行风格 | 必填 | 轻松随意（随性游玩）/ 经典打卡（必去著名景点）/ 深度体验（少而精） |

> **出行风格对生成策略的影响**：
> - `casual`（轻松随意）：景点数量偏少（每天 1-2 个），不强制打卡，留有休息时间
> - `classic`（经典打卡）：优先选择评分高、知名度高的景点，每天 2-3 个
> - `deep`（深度体验）：每天最多 2 个景点，每个景点停留时间更长，增加游览细节

> **目的地校验规则**：去除首尾空白后进行校验
> - 空字符串或仅空格：提示"请输入目的地"
> - 少于2个字符：提示"目的地名称过短"
> - 重复字符（如"北京北京"）：后端校验失败返回 E001
>
> **人数与预算关联提示**：当 人数 × 天数 × 100元 > 实际预算时，给出提示（如"2人7天选择1000元预算可能不够，建议调整"）；最低日均消费按 100元/天/人 计算；预算为枚举索引 0-3，分别对应实际金额 500、2000、4000、6000 元（区间中值）
>
> **预算类型**：区间选择对应枚举值 0-3，后端根据索引映射到实际金额区间
>
> **表单数据持久化**：用户填写表单时，数据自动暂存至 localStorage；刷新页面后可恢复已填写数据；点击"生成行程"后清除暂存数据

#### 2.2.3 偏好设置
| 偏好类型 | 选项 |
|---------|------|
| 必去景点 | 历史文化、自然风光、购物、娱乐、夜生活 |
| 排除景点 | 游乐场、密室逃脱、温泉、SPA、采摘 |
| 交通方式 | 步行优先、公共交通、自驾 |

> **必去景点标签关联说明**：
> - 这些标签是**软约束**，作为 LLM 生成行程的参考信息
> - 不保证每个标签每天强制安排一个景点，LLM 会综合考虑标签、景点评分、距离等因素生成最优行程
> - 如果某标签下无可用景点，LLM 会自动跳过并选择其他合适景点

#### 2.2.4 行程生成
- 每日行程有主题（如"经典打卡日"、"自然探索日"）
- **主题生成规则**：根据当日主要景点类型生成（如：当日景点以历史文化为主，则主题为"历史探秘日"）
- 每项包含：时间段、景点、时长、门票信息、游览建议
- **每天最多 3 个景点**（不含餐饮，餐饮规划暂不考虑）
- **住宿安排**：每天安排住宿地点（酒店/民宿）
- 行程卡片显示：预计花费、景点距离
- 生成策略：优先选择评分高、距离近的景点，确保行程紧凑合理
- **路过顺便看看的地点不计入景点数量**

#### 2.2.5 行程展示
- 行程卡片列表视图
- 地图可视化（景点标注 + 路线）
- **行程卡片显示**：每日景点列表、预计花费、景点距离
- **行程导出功能**：
  - 图片导出：包含行程卡片列表（可滚动截取）+ 封面信息（目的地、天数、总预算），无水印，2x 分辨率
  - PDF 导出：长图格式，包含完整行程卡片 + 地图截图（高德静态地图 API，服务端生成）+ 封面信息，无水印
  - 最大高度：2000px，超过时按天分页；分页时每页包含该天行程和页眉（目的地+第X天）
  - 不包含住宿详情卡（住宿信息已在每日行程卡片中体现）

### 2.3 扩展功能（Should Have）

| 功能 | 说明 |
|------|------|
| 行程重新生成 | 点击"重新生成"按钮，基于**用户最后一次确认的偏好**重新生成完整行程；不保存历史记录；用户修改过行程后仍可重新生成 |
| 目的地建议 | 基于偏好推荐目的地、显示热门榜单（V2） |

> **"原始偏好"定义**：指用户在向导 Step 4 点击"确认生成"时提交的表单数据。如果用户在结果页修改了行程后再点击重新生成，仍使用 Step 4 确认时的偏好，不使用用户修改后的行程数据。

---

## 3. 产品功能

### 3.1 信息架构

```
首页
└── 目的地输入框 + 开始规划按钮

向导页（4步表单）
├── Step 1: 目的地选择
├── Step 2: 出行信息（日期、天数、人数、预算、出行风格）
├── Step 3: 偏好设置（必去/排除景点、交通方式）
└── Step 4: 确认生成

结果页
├── 行程卡片列表
├── 地图可视化
└── 操作按钮（导出/重新生成）
```

### 3.2 用户流程

```
[打开应用]
      ↓
[首页：输入目的地城市 + 点击开始规划]
      ↓
[向导 Step 1] → 确认城市名称（查询不到则提示）
      ↓
[向导 Step 2] → 填写日期、天数、人数、预算、选择出行风格
      ↓
[向导 Step 3] → 设置偏好标签（必去/排除景点、交通方式）
      ↓
[向导 Step 4] → 确认信息，点击生成行程
      ↓
[显示加载状态 + AI 整合数据]
      ↓
[结果页：展示行程卡片 + 地图]
      ↓
[用户可：导出图片或PDF / 重新生成]
```

### 3.3 页面原型说明

#### 3.3.1 首页
- 顶部：产品名称 + Logo
- 中部：目的地输入框（直接填入城市名）+ "开始规划"按钮

#### 3.3.2 向导页
- 左侧/顶部：步骤进度条（1→2→3→4）
- 右侧/中部：当前步骤表单
- 底部：上一步/下一步按钮（Step 4 为"确认生成"按钮，非"下一步"）
- **Step 4 特殊说明**：底部按钮为"确认生成"（非"下一步"），点击后触发行程生成

#### 3.3.3 结果页
- 顶部：目的地 + 天数概览
- 左侧：每日行程卡片（可折叠/展开）
- 右侧：地图（高德地图嵌入）
- **地图路线规划**：景点按 items 时间顺序连接（方案A：按时间顺序；方案B：路径优化算法由高德 API 自动决定）
- 底部：导出按钮（图片/PDF）+ 重新生成按钮
- **移动端布局**：地图展示在行程卡片下方（上下布局），地图高度固定为屏幕的 40%，行程卡片可滚动查看

---

## 4. 数据需求

### 4.1 用户输入数据

```json
{
  "destination": "上海",
  "departure_date": "2026-05-01",
  "days": 3,
  "travelers": 2,
  "budget": 1,
  "travel_style": "classic",
  "preferences": {
    "include_tags": ["历史文化", "自然风光"],
    "exclude_tags": ["游乐场"],
    "transport": "public"
  }
}
```

> **budget 字段说明**：枚举索引（0-3），对应区间：0=<1000 / 1=1000-3000 / 2=3000-5000 / 3=>5000 元（区间为左闭右开）
> **travel_style 可选值**：`casual`（轻松随意）/ `classic`（经典打卡）/ `deep`（深度体验）
> **include_tags 说明**：不含美食（餐饮规划暂不考虑）
> **days.day 说明**：从 1 开始，表示第几天（day=1 为第一天）

### 4.2 系统输出数据

```json
{
  "trip_id": "uuid-string",
  "destination": "上海",
  "days": [
    {
      "day": 1,
      "theme": "经典城市之旅",
      "items": [
        {
          "type": "attraction",
          "time": "09:00-12:00",
          "poi": {
            "name": "外滩",
            "address": "中山东路",
            "lat": 31.2405,
            "lng": 121.4901,
            "ticket": null,
            "rating": 4.8,
            "phone": null,
            "website": null
          },
          "estimated_cost": 0,
          "distance_from_previous": "2.3km",
          "note": "建议清晨拍摄，避开人流"
        },
        {
          "type": "attraction",
          "time": "14:00-17:00",
          "poi": {
            "name": "豫园",
            "address": "豫园路",
            "lat": 31.2276,
            "lng": 121.4895,
            "ticket": 40,
            "rating": 4.6,
            "phone": "021-63260802",
            "website": "https://yuyuan.trip.com"
          },
          "estimated_cost": 80,
          "distance_from_previous": "1.5km",
          "note": "建议提前购票，错峰游览"
        }
      ],
      "accommodation": {
        "name": "上海外滩酒店",
        "address": "中山东一路",
        "lat": 31.2450,
        "lng": 121.4950,
        "check_in_time": "18:00",
        "phone": "021-63200000"
      }
    }
  ],
  "summary": "3天2晚上海经典之旅，涵盖外滩、豫园等经典景点。",
  "total_estimated_cost": 380
}
```

> **ticket 字段说明**：数字表示门票价格（元），`null` 表示免费入场
> **estimated_cost 字段说明**：该景点的预估门票/人
> **distance_from_previous 字段说明**：与上一个景点的距离（单位：公里 km）
> **type 可选值**：attraction（景点）、accommodation（住宿）
> **poi.phone/poi.website 字段说明**：景点联系电话和网址，`null` 表示无数据
> **accommodation.phone 字段说明**：住宿联系电话，`null` 表示无数据
> **accommodation.check_in_time 字段说明**：建议入住时间（当用户 9:00 到达但无法提前入住时，系统安排 18:00 入住；最晚入住时间为当日 22:00）；18:00 之前用户可自由活动（如午餐、购物），不受住宿限制
> **total_estimated_cost 说明**：整个行程的景点门票总预估（不含住宿和餐饮）
> **summary 用途说明**：用于行程分享时的预览文本、分享链接的 og:description、以及导出 PDF 的封面副标题

### 4.3 外部 API

| 数据源 | 用途 | 关键接口 |
|--------|------|----------|
| 高德地图 | 地理服务 + 地图展示 | 地理编码、POI 搜索、路径规划（Web API）；地图渲染、交互（JS API） |
| 携程 | 产品数据 | 景点详情、酒店、门票 |

> **高德 API 选择策略**：
> - **Web API**：用于地理编码、POI 搜索、路径规划（服务端调用）
> - **JS API**：用于前端地图渲染和交互（客户端加载）
>
> **API 失败策略**：高德/携程 API 调用失败时，**不降级**为纯 LLM 知识库生成。旅游计划依赖实时数据（天气、开放时间等），降级会导致行程不准确。此时返回错误码 E002，提示用户稍后重试。若频繁失败，建议客户端实现指数退避重试机制。
>
> **单 Agent 架构（V1 简化）**：
> - Planner Service：整合意图解析 + 行程生成，单次 MiniMax 调用完成
> - Data Service：并行调用高德/携程 API 获取 POI 数据
> - 超时策略：单步 API 超时 5s，整体行程生成超时 10s
> - **待确认**：如 Multi-Agent 协调复杂度过高，可进一步简化为单 LLM 调用

### 4.4 错误码与异常处理

| 错误码 | 场景 | 用户提示 | 处理方式 |
|--------|------|----------|----------|
| E001 | 目的地无数据 | "暂未支持该城市的旅游规划" | 引导用户选择其他城市 |
| E002 | 高德/携程 API 超时或失败 | "网络不稳定，请稍后重试" | 显示重试按钮（**不降级生成**） |
| E003 | LLM 生成失败 | "行程生成失败，请稍后重试" | 初始生成失败自动重试一次；若仍失败显示用户提示（并非直接显示"E003"，而是显示"行程生成失败，请稍后重试"） |
| E004 | 可选景点不足 | "可选景点不足，请调整偏好或选择其他城市" | 引导用户调整偏好或换城市 |

> **重试机制**：
> - 初始行程生成：LLM 调用失败时自动重试一次；若仍失败，返回 E003
> - 用户触发重新生成：最多重试 3 次
> - 每次重试后**不保存用户已填写的数据**，用户需重新填写表单
> - 外部 API（高德/携程）失败时，不降级为纯 LLM 生成，直接返回 E002
>
> **边界条件**：
> - 景点数量不足：用户偏好排除后可选景点 < 1 个时，返回 E004（deep 模式 2 个景点为合法最低数量）
> - 行程修改后重新生成：用户修改行程后点击重新生成，修改内容不保留，基于原始偏好重新生成
> - 住宿安排策略：假设第一天 9:00 到达，最后一天 18:00 离开；住宿根据天数推断（3 天行程 = 2 晚住宿）；**最后一天 18:00 离开时，当晚不需要住宿**；住宿位置优先选择靠近当日最后一个景点
>
> **AI 输出校验**：LLM 生成的行程需符合 JSON Schema 约束，校验失败时触发 E003。

### 4.5 项目架构

#### 4.5.1 项目目录结构

```
Trip-planner-v1/
├── backend/                      # FastAPI 后端
│   ├── app/
│   │   ├── main.py              # FastAPI 入口
│   │   ├── config.py            # 配置管理（环境变量加载）
│   │   ├── api/v1/              # API 路由
│   │   ├── models/              # Pydantic 数据模型
│   │   ├── services/            # 外部 API 封装（高德/携程/MiniMax）
│   │   ├── planner/             # 单 Agent（V1 简化）：意图解析 + 行程生成
│   │   ├── prompts/             # Prompt 模板
│   │   └── schemas/             # JSON Schema
│   ├── tests/                   # 测试
│   ├── .env.example             # 环境变量示例（API Key 配置）
│   └── pyproject.toml
├── frontend/                    # Vue 3 前端
│   ├── src/
│   │   ├── views/              # 页面
│   │   ├── components/          # 组件
│   │   ├── stores/             # Pinia 状态
│   │   ├── api/                # API 调用
│   │   └── utils/              # 工具函数（导出）
│   └── package.json
└── docker-compose.yml
```

#### 4.5.2 模块职责

| 目录/模块 | 职责 |
|-----------|------|
| `backend/app/agents/planner.py` | **单 Agent 架构**（V1 简化）：整合意图解析 + 行程生成，一个 MiniMax 调用完成 |
| `backend/app/services/` | MiniMax/高德/携程 API 封装 |
| `backend/app/prompts/` | Prompt 模板集中管理 |
| `backend/app/schemas/` | LLM 输出 JSON Schema 定义 |
| `backend/app/api/v1/` | API 路由（trips/cities） |
| `frontend/src/views/` | 页面：HomeView/WizardView/ResultView |
| `frontend/src/stores/` | Pinia 状态管理 |
| `frontend/src/utils/` | 导出功能（图片/PDF） |

#### 4.5.3 单 Agent 数据流（V1 简化）

```
用户表单 → FastAPI 参数校验
              ↓
         单 Planner Agent
         - MiniMax 解析需求 + 生成行程（单次调用）
              ↓
         Data Service（并行获取 POI）
         - 高德 POI 搜索景点
         - 高德 POI 搜索住宿
         - 携程获取景点详情
              ↓
         JSON Schema 校验 → 返回行程
```

#### 4.5.4 Orchestrator 状态机设计

> **待确认问题**：V1 是否需要异步 Orchestrator？
> - **方案A（推荐）**：同步 Orchestrator，重新生成预计 10s 内完成，直接返回结果（去掉 202 Accepted）
> - **方案B**：异步 Orchestrator，使用 FastAPI BackgroundTasks，保留 202 Accepted + 轮询

| 状态 | 说明 |
|------|------|
| `pending` | 初始状态，接收请求 |
| `processing` | 执行中（Intent 解析 / Data 获取 / Planner 生成） |
| `completed` | 成功，返回行程 |
| `failed` | 失败，返回错误码 |

> **超时控制**：整体行程生成超时 10s，超时后返回 E003

#### 4.5.5 重试退避策略

| 场景 | 重试次数 | 间隔时间 | 说明 |
|------|----------|----------|------|
| 初始生成 LLM 失败 | 1 | 1s | 失败后直接返回 E003 |
| 重新生成 LLM 失败 | 3 | 1s → 2s → 4s | 指数退避（1s/2s/4s），仍失败返回 E003 |
| 外部 API 失败 | 0 | - | 不降级，直接返回 E002 |

> **降级策略**：连续失败 2 次后可简化 Prompt（减少生成内容），仍失败返回 E003

---

## 5. 非功能需求

### 5.1 性能要求
| 指标 | 目标 | 说明 |
|------|------|------|
| 页面加载 | < 2s（WiFi）/ < 4s（4G） | 首屏加载时间，桌面浏览器 |
| API 响应 | < 500ms | 简单查询（地理编码、单 POI）；复杂查询（多 POI 路径规划）< 2s |
| 行程生成 | < 10s | 包含 AI 行程生成 + 数据整合的总时间 |

### 5.2 可用性要求
| 指标 | 目标 | 说明 |
|------|------|------|
| 核心流程成功率 | > 95% | 统计口径：从点击"生成行程"到结果页展示成功的比率；排除用户主动取消和网络异常 |
| 错误提示清晰度 | 用户可理解、可操作 | 错误提示需包含原因说明和下一步操作引导 |

### 5.3 兼容性要求
- 浏览器：Chrome、Firefox、Safari 最新两个版本
- 屏幕尺寸：支持响应式设计
  - 移动端：< 768px（手机竖屏）
  - 平板：768px - 1024px（平板/手机横屏）
  - 桌面：> 1024px
- **平台**：V1 仅支持 Web 端（H5 / 桌面浏览器）

---

## 6. 验收标准

### 6.1 功能验收

| 功能 | 验收条件 |
|------|----------|
| 目的地输入 | 用户直接填入城市名，查询不到则提示 |
| 表单必填校验 | 未填写时提示清晰，无法提交；日期不可选过去；人数×天数×日均消费与预算关联提示 |
| 行程生成 | 点击后 10 秒内返回结果 |
| 行程卡片 | 展示每日时间线、景点、建议、预计花费、景点距离 |
| 地图展示 | 景点标注准确，路线连续 |
| 行程导出 | 支持导出为图片（2x 分辨率）和 PDF（长图格式），无水印；导出失败时显示友好错误提示并提供重试按钮 |

### 6.2 交互验收

| 场景 | 验收条件 |
|------|----------|
| 加载状态 | 生成过程中显示加载动画和提示 |
| 错误处理 | API 失败时显示友好错误页面 |
| 空状态 | 无数据时显示引导提示 |

### 6.3 技术风险与缓解方案

| 风险 | 描述 | 缓解方案 |
|------|------|----------|
| MiniMax JSON Schema 校验失败 | MiniMax 输出格式可能不稳定，需要多次校验和重试 | 增加格式校验和自动修复机制，重试 3 次仍失败返回 E003 |
| 高德静态地图 API 限额 | 静态地图 API 每日调用限额 | 记录使用量，设计降级方案（如使用默认地图图片） |
| html2canvas 截取地图成功率 | 地图是 canvas 元素，截取可能只获取空白 | 测试不同浏览器和设备，制定兼容性方案；PDF 使用服务端静态地图 API 替代 |

---

## 7. 未来规划

| 阶段 | 功能 |
|------|------|
| V2 | 用户系统、行程预订（酒店、门票一键下单） |
| V3 | 实时天气/人流预警、行程分享 |
| V4 | 境外城市支持、多语言 |

> **扩展预留**：数据模型预留扩展字段，V2 可平滑接入用户系统和预订功能

---

## 8. 附录

### 8.1 术语表

| 术语 | 定义 |
|------|------|
| POI | Point of Interest，兴趣点（V1 中指景点和住宿） |
| LLM | Large Language Model，大语言模型 |
| 向导式 | Wizard，指多步骤表单引导用户完成输入 |
| travel_style | 出行风格：casual（轻松随意）/ classic（经典打卡）/ deep（深度体验） |

### 8.2 技术栈

| 层级 | 技术 |
|------|------|
| 后端 | Python 3.11+ / FastAPI |
| 数据库 | MySQL 8.0 / SQLAlchemy 2.0（**同步 ORM**，V1 阶段暂不使用异步） |
| 数据校验 | Pydantic v2 |
| 前端 | Vue 3.4+ / TypeScript 5.x / Vite 5.x |
| 状态管理 | Pinia |
| 路由 | Vue Router 4 |
| UI 组件 | Naive UI |
| AI | MiniMax（对话补全）+ **单 Agent 架构**（V1 阶段简化 Multi-Agent） |
| 地图 | 高德地图 Web API / JS API / 静态地图 API |
| 外部 API | 携程 API（景点/住宿） |
| 导出库 | html2canvas（图片导出）+ jsPDF（PDF 生成）；PDF 地图使用高德静态地图 API（服务端生成） |
| **安全** | 环境变量存储 API Key（`.env` 文件，不提交 Git）；SQLAlchemy ORM 参数化查询（禁止原始 SQL 拼接） |

### 8.3 系统组件

#### 后端组件

| 组件 | 职责 | 技术实现 |
|------|------|----------|
| API Gateway | 接收请求、路由分发 | FastAPI |
| Planner Service | **单 Agent**（V1 简化）：意图解析 + 行程生成，单次 MiniMax 调用 | FastAPI + MiniMax |
| Data Service | 获取景点/住宿 POI 数据 | 高德/携程 API 调用（并行） |
| Trip Service | 行程业务逻辑 | FastAPI Service |
| Export Service | 导出 PDF（地图截图） | FastAPI + 高德静态地图 API |
| User Service | 用户管理（V2） | FastAPI |

#### 前端组件

| 组件 | 职责 |
|------|------|
| HomeView | 首页（目的地输入 + 开始规划） |
| WizardView | 向导表单（4步：目的地/出行信息/偏好/确认） |
| ResultView | 结果展示（行程卡片 + 地图 + 导出） |
| tripStore | Pinia 状态：表单数据、行程结果、加载状态 |
| amapStore | Pinia 状态：地图标注、路线数据 |

#### 外部服务

| 服务 | 用途 | 调用方 |
|------|------|--------|
| MiniMax API | LLM 对话补全 | Backend（Planner Service） |
| 高德 Web API | 地理编码、POI搜索、路径规划 | Backend（Data Service） |
| 高德 JS API | 地图渲染、交互 | Frontend |
| 高德静态地图 API | PDF 地图截图 | Backend（Export Service） |
| 携程 API | 景点详情、酒店数据 | Backend（Data Service） |

### 8.4 架构处理流程

#### 行程生成流程

```
1. 用户提交表单（POST /api/v1/trips/generate）
       ↓
2. FastAPI 参数校验（Pydantic）
       ↓
3. Orchestrator.start()
       ↓
4. Intent Agent → MiniMax 解析需求，生成结构化描述
       ↓
5. Data Agent 并行调用：
   - 高德 POI 搜索景点（基于目的地 + include_tags）
   - 高德 POI 搜索住宿（基于目的地）
   - 携程获取景点详情（门票、评分）
       ↓
6. Planner Agent → MiniMax 基于 POI 数据生成行程 JSON
       ↓
7. JSON Schema 校验输出（失败则触发 E003 重试）
       ↓
8. 存储行程到 MySQL（trips → days → trip_items/accommodations）
       ↓
9. 返回行程给前端
```

#### 导出流程

```
图片导出：
1. 用户点击"导出图片"
       ↓
2. 前端调用 html2canvas 截取行程卡片 DOM
       ↓
3. 合成 2x 分辨率图片
       ↓
4. 触发浏览器下载

PDF 导出：
1. 用户点击"导出 PDF"
       ↓
2. 前端请求后端获取高德静态地图截图（Export Service）
       ↓
3. 前端使用 jsPDF 合成：封面 + 地图截图 + 行程卡片
       ↓
4. 触发浏览器下载
```

### 8.5 数据库设计

#### ER 图

```
users (V2)
   │
   │ 1:N
   ▼
trips ──────── 1:N ──────── days
                              │
                         ┌────┴────┐
                         ▼         ▼
                    trip_items  accommodations
```

#### 表结构

##### users（V2 预留）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT PK | |
| username | VARCHAR(50) | 唯一 |
| email | VARCHAR(100) | 唯一 |
| password_hash | VARCHAR(255) | |
| created_at | TIMESTAMP | |

##### trips
| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT PK | |
| user_id | BIGINT NULL | V1 可为空（匿名行程），V2 必填 |
| destination | VARCHAR(50) | INDEX |
| departure_date | DATE | INDEX |
| days | TINYINT | 1-7 |
| travelers | TINYINT | 1-5 |
| budget | TINYINT | 枚举索引 0-3 |
| travel_style | ENUM | casual/classic/deep |
| preferences | JSON | include_tags, exclude_tags, transport |
| status | ENUM | generating/completed/failed |
| summary | TEXT | |
| total_estimated_cost | DECIMAL | |
| booking_status | ENUM | **预留 V2**（pending/confirmed/cancelled） |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP |

> **索引说明**：
> - `INDEX idx_destination_date (destination, departure_date)` - 高频查询优化
> - `INDEX idx_user_id (user_id)` - V2 用户行程查询

##### days
| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT PK | |
| trip_id | BIGINT FK | → trips.id，INDEX |
| day_num | TINYINT | 1, 2, 3... |
| theme | VARCHAR(50) | |
| created_at | TIMESTAMP | |

##### trip_items
| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT PK | |
| day_id | BIGINT FK | → days.id，INDEX |
| trip_id | BIGINT FK | → trips.id（反范式设计，方便直接查询） |
| item_type | ENUM | **attraction**（restaurant 已删除，预留 V2） |
| time_range | VARCHAR(20) | '09:00-12:00' |
| time_range | VARCHAR(20) | '09:00-12:00' |
| name | VARCHAR(100) | |
| address | VARCHAR(200) | |
| lat | DECIMAL(10,7) | |
| lng | DECIMAL(10,7) | |
| ticket | DECIMAL(10,2) | NULL=免费 |
| rating | DECIMAL(2,1) | 1.0-5.0 |
| estimated_cost | DECIMAL(10,2) | 预估门票/人 |
| distance_from_previous | VARCHAR(20) | '2.3km' |
| note | TEXT | |
| sort_order | TINYINT | |
| created_at | TIMESTAMP | |

##### accommodations
| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT PK | |
| day_id | BIGINT FK | → days.id |
| trip_id | BIGINT FK | → trips.id |
| name | VARCHAR(100) | |
| address | VARCHAR(200) | |
| lat | DECIMAL(10,7) | |
| lng | DECIMAL(10,7) | |
| check_in_time | TIME | |
| phone | VARCHAR(20) | |
| sort_order | TINYINT | |
| created_at | TIMESTAMP | |

### 8.6 关键流程逻辑

#### 8.6.1 行程生成流程（核心 - V1 简化版）

```
[1. 用户提交表单]
        ↓
[2. 参数校验] → 失败 → 返回 422 + E005
        ↓
[3. 检查城市是否支持] → 失败 → 返回 400 + E001
        ↓
[4. Data Service: 并行获取 POI 数据]
   - 高德搜索景点 POI（超时5s）
   - 高德搜索住宿 POI（超时5s）
   - 携程获取景点详情（超时5s）
   任一失败 → 返回 502 + E002
        ↓
[5. 检查景点数量是否满足最低阈值] → 不足 → 返回 422 + E004
        ↓
[6. 单 Planner Agent: MiniMax 生成行程] → 失败 → 重试1次（间隔1s） → 返回 500 + E003
        ↓
[7. JSON Schema 校验] → 失败 → 重试1次（间隔1s） → 返回 500 + E003
        ↓
[8. 计算总预估费用]
[9. 存储行程到 MySQL]
[10. 返回行程给用户] → 201 Created
```

> **超时控制**：整体行程生成超时 10s，超时后返回 E003

#### 8.6.2 参数校验规则

| 参数 | 校验规则 | 错误提示 |
|------|----------|----------|
| destination | 去除空白后长度 ≥ 2，**只允许中文字符和字母数字** | "目的地名称过短" |
| departure_date | 日期 ≥ 今天 | "出发日期不能选择过去" |
| days | 1 ≤ days ≤ 7 | "旅行天数需为1-7天" |
| travelers | 1 ≤ travelers ≤ 5 | "人数需为1-5人" |
| budget | ∈ {0, 1, 2, 3} | "预算区间选择错误" |
| travel_style | ∈ {casual, classic, deep} | "出行风格选择错误" |

> **安全要求**：destination 字段**禁止原始 SQL 拼接**，必须使用 SQLAlchemy ORM 参数化查询
> **人数×天数×100 > 实际预算金额?** → 显示警告提示（不阻止提交）

#### 8.6.3 子流程：POI 数据获取（Data Service）

```
输入：目的地、用户偏好（include_tags/exclude_tags）
输出：景点 POI 列表 + 住宿 POI 列表

并行执行（超时：5秒/每个）：

A. 高德搜索景点 POI
   - keywords: include_tags
   - city: 目的地
   - 数量限制: 最多 50 个

B. 高德搜索住宿 POI
   - keywords: [酒店、民宿]
   - city: 目的地
   - 数量限制: 最多 20 个

C. 携程获取景点详情
   - 获取: 门票价格、评分、电话

失败策略：任一 API 超时/失败 → 整体失败 → E002（不降级）
```

#### 8.6.4 子流程：单 Planner Agent（V1 简化）

```
输入：
- 用户原始偏好（destination, days, travelers, budget, travel_style, preferences）
- Data Service 获取的 POI 数据（景点列表、住宿列表）

输出：结构化行程 JSON

1. 构建 Prompt（包含目的地、可选 POI、用户偏好、行程规则）
2. 调用 MiniMax API（temperature=0.8, max_tokens=4000）
3. JSON 解析响应 → 失败则重试（间隔1s）
4. 输出校验：
   - 必需字段存在
   - days 数量正确
   - 每天景点数量符合 travel_style
   - 失败则重试（间隔1s）
```

#### 8.6.6 导出行程流程

**图片导出：**
```
[1. 用户点击"导出图片"]
        ↓
[2. html2canvas 截取行程卡片 DOM]
        ↓
[3. 合成 2x 分辨率图片]
        ↓
[4. 触发浏览器下载]
```

**PDF 导出：**
```
[1. 用户点击"导出 PDF"]
        ↓
[2. 前端请求后端: GET /api/v1/trips/{trip_id}/export/map]
        ↓
[3. 后端调用高德静态地图 API]
        ↓
[4. 前端使用 jsPDF 合成: 封面 + 地图截图 + 行程卡片]
        ↓
[5. 触发浏览器下载]
```

#### 8.6.7 重新生成流程

> **待确认问题**：重新生成使用同步返回（10s 内）还是异步返回（202 + 轮询）？

**方案A（推荐 - 同步返回）**：
```
[1. 用户点击"重新生成"]
        ↓
[2. 检查重试次数: regenerate_count ≥ 3?] → 是 → 返回 429
        ↓
[3. 递增重试次数（间隔退避：1s → 2s → 4s）]
[4. 执行行程生成流程（同步，10s 内返回）]
[5. 返回 201 或错误]
```

**方案B（异步返回）**：
```
[1. 用户点击"重新生成"]
        ↓
[2. 检查重试次数: regenerate_count ≥ 3?] → 是 → 返回 429
        ↓
[3. 递增重试次数，更新 status='regenerating']
[4. 后台执行行程生成流程]
[5. 返回 202 Accepted]
        ↓
[6. 前端轮询 GET /api/v1/trips/{trip_id}（间隔2s，超时60s）]
   - status='completed' → 展示新行程
   - status='failed' → 显示错误
```

#### 8.6.8 城市验证流程

```
[1. 用户输入目的地]
        ↓
[2. 前端校验: 去除空白后长度 < 2?] → 是 → 提示"目的地名称过短"
        ↓
[3. 调用 POST /api/v1/cities/validate]
        ↓
[4. 后端调用高德地理编码 API]
        ↓
[5. 返回验证结果]
   - valid=true → 显示城市简介，继续下一步
   - valid=false → 提示"暂未支持该城市"
```

#### 8.6.9 异常处理策略

| 异常场景 | 处理策略 | 返回码 |
|----------|----------|--------|
| LLM 调用失败（初始生成） | 重试1次（间隔1s），仍失败 → E003 | 500 |
| LLM 调用失败（重新生成） | 重试3次（间隔退避：1s→2s→4s），仍失败 → E003 | 500/429 |
| 外部 API 超时 | 不降级，直接返回错误 | 502 |
| 景点数量不足 | 返回 E004，引导调整偏好 | 422 |
| JSON Schema 校验失败 | 重试1次（间隔1s），仍失败 → E003 | 500 |
| 行程生成超时（10s） | 返回 E003 | 500 |

> **重试退避**：重新生成场景使用指数退避（1s → 2s → 4s），避免频繁重试加剧 API 压力

### 8.7 RESTful API 文档

#### 基础规范

- **基础路径**：`/api/v1`
- **认证方式**：API Key（Header: `X-API-Key`），V2 扩展 JWT
- **数据格式**：JSON
- **字符编码**：UTF-8

#### 错误响应格式

```json
{
  "error": {
    "code": "E001",
    "message": "错误描述",
    "details": {}
  }
}
```

#### 错误状态码

| 状态码 | 含义 | 说明 |
|--------|------|------|
| 200 | OK | 请求成功 |
| 201 | Created | 资源创建成功 |
| 202 | Accepted | 请求已接受，正在处理 |
| 204 | No Content | 请求成功，无返回内容 |
| 400 | Bad Request | 请求参数错误 |
| 401 | Unauthorized | 认证失败或未提供 API Key |
| 403 | Forbidden | 无权限访问 |
| 404 | Not Found | 资源不存在 |
| 422 | Unprocessable Entity | 请求格式正确但语义错误 |
| 429 | Too Many Requests | 请求过于频繁，触发限流 |
| 500 | Internal Server Error | 服务器内部错误 |
| 502 | Bad Gateway | 外部服务不可用 |
| 503 | Service Unavailable | 服务暂时不可用 |
| 504 | Gateway Timeout | 外部服务超时 |

#### 业务错误码

| 错误码 | HTTP状态码 | 说明 |
|--------|------------|------|
| E001 | 400 | 目的地无数据 |
| E002 | 502 | 高德/携程 API 超时或失败 |
| E003 | 500 | LLM 生成失败（重试后仍失败） |
| E004 | 422 | 可选景点不足 |
| E005 | 422 | 行程为空（校验失败） |

---

#### API 接口

##### 1. 健康检查

**GET** `/api/v1/health`

检查服务状态。

**Response 200:**
```json
{
  "status": "healthy",
  "timestamp": "2026-04-28T10:00:00Z"
}
```

---

##### 2. 城市验证

**POST** `/api/v1/cities/validate`

验证城市是否支持旅游规划。

**Request Body:**
```json
{
  "name": "上海"
}
```

**Response 200 (有效):**
```json
{
  "valid": true,
  "city": {
    "name": "上海",
    "description": "国际化大都市..."
  }
}
```

**Response 200 (无效):**
```json
{
  "valid": false,
  "message": "暂未支持该城市的旅游规划"
}
```

**错误码：** E001

---

##### 3. 生成行程

**POST** `/api/v1/trips/generate`

生成旅游行程（核心接口）。

**Request Body:**
```json
{
  "destination": "上海",
  "departure_date": "2026-05-01",
  "days": 3,
  "travelers": 2,
  "budget": 1,
  "travel_style": "classic",
  "preferences": {
    "include_tags": ["历史文化", "自然风光"],
    "exclude_tags": ["游乐场"],
    "transport": "public"
  }
}
```

**Response 201:**
```json
{
  "trip_id": "uuid-string",
  "status": "completed",
  "destination": "上海",
  "days": [
    {
      "day": 1,
      "theme": "经典城市之旅",
      "items": [
        {
          "type": "attraction",
          "time": "09:00-12:00",
          "poi": {
            "name": "外滩",
            "address": "中山东路",
            "lat": 31.2405,
            "lng": 121.4901,
            "ticket": null,
            "rating": 4.8,
            "phone": null,
            "website": null
          },
          "estimated_cost": 0,
          "distance_from_previous": null,
          "note": "建议清晨拍摄"
        }
      ],
      "accommodation": {
        "name": "上海外滩酒店",
        "address": "中山东一路",
        "lat": 31.2450,
        "lng": 121.4950,
        "check_in_time": "18:00",
        "phone": "021-63200000"
      }
    }
  ],
  "summary": "3天2晚上海经典之旅",
  "total_estimated_cost": 380
}
```

**错误码：** E002（API失败）、E003（生成失败）、E004（景点不足）

---

##### 4. 获取行程

**GET** `/api/v1/trips/{trip_id}`

获取已生成行程详情。

**Response 200:**
```json
{
  "trip_id": "uuid-string",
  "status": "completed",
  "destination": "上海",
  "departure_date": "2026-05-01",
  "days": 3,
  "travelers": 2,
  "budget": 1,
  "travel_style": "classic",
  "preferences": {
    "include_tags": ["历史文化", "自然风光"],
    "exclude_tags": ["游乐场"],
    "transport": "public"
  },
  "days": [...],
  "summary": "3天2晚上海经典之旅",
  "total_estimated_cost": 380,
  "created_at": "2026-04-28T10:00:00Z"
}
```

**Response 404:**
```json
{
  "error": {
    "code": "E404",
    "message": "行程不存在"
  }
}
```

---

##### 5. 重新生成行程

**POST** `/api/v1/trips/{trip_id}/regenerate`

基于原始偏好重新生成行程。

**Response 202:**
```json
{
  "trip_id": "uuid-string",
  "status": "regenerating",
  "message": "行程重新生成中，请稍后查询"
}
```

轮询 `GET /api/v1/trips/{trip_id}` 获取结果。

**Response 404:**
```json
{
  "error": {
    "code": "E404",
    "message": "行程不存在"
  }
}
```

**错误码：** E003（重试次数超限，返回 429）

---

##### 6. 导出行程 - 获取地图截图

**GET** `/api/v1/trips/{trip_id}/export/map`

获取行程地图截图（用于 PDF 导出）。

**Query Parameters:**
| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| width | int | 600 | 图片宽度 |
| height | int | 400 | 图片高度 |

**Response 200:**
```json
{
  "image_url": "https://restapi.amap.com/...",
  "expires_at": "2026-04-28T11:00:00Z"
}
```

**Response 404:**
```json
{
  "error": {
    "code": "E404",
    "message": "行程不存在"
  }
}
```

**错误码：** E002（地图API失败，返回 502）

---

##### 7. 删除行程

**DELETE** `/api/v1/trips/{trip_id}`

删除指定行程。

**Response 204:** 空

**Response 404:**
```json
{
  "error": {
    "code": "E404",
    "message": "行程不存在"
  }
}
```
