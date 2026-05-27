# Amazon Inventory Agent Demo

一个面向 Amazon 跨境电商运营场景的库存 Agent MVP Demo。

本项目不是一个简单的 CRUD 后台，而是一个具备完整数据闭环的系统级库存 Agent Demo：从模拟业务数据出发，经过 MySQL 数据层、规则引擎、库存分析服务、任务生成服务、FastAPI 接口层，最终在 React 前端中展示库存风险、补货建议和运营任务。

## 1. 项目定位

Amazon Inventory Agent Demo 目标是模拟一个跨境电商卖家日常库存运营中的智能分析系统，帮助运营人员快速识别：

- 哪些 SKU 有断货风险
- 哪些 SKU 已经断货
- 哪些 SKU 存在滞销风险
- 哪些 SKU 在途库存过高
- 哪些 SKU 存在不可售库存异常
- 哪些 SKU 缺少关键数据
- 哪些任务需要运营人员立即处理

当前版本是 MVP 阶段，重点验证“库存 Agent 的业务闭环”，不接入真实 Amazon SP-API，不使用 LLM，不做复杂权限系统，也不做 SaaS 化部署。

## 2. 系统整体链路

```text
Excel 模拟数据
    ↓
MySQL 数据层
    ↓
规则引擎 risk_rules.py
    ↓
库存 Agent 分析服务
    ↓
任务生成服务
    ↓
FastAPI API
    ↓
React 前端控制台
```

系统将库存、销量、补货配置和商品主数据拆分为不同的数据源，再通过规则引擎统一计算风险等级、补货建议和数据质量状态。分析结果会写入 `inventory_agent_analysis`，再由任务服务生成 `inventory_agent_tasks`，形成从数据到运营动作的闭环。

这也是它区别于普通后台项目的核心：它不仅展示数据，还会根据业务规则主动判断风险并生成可执行任务。

## 3. 技术栈

### 后端

- Python 3.11+
- FastAPI
- SQLAlchemy
- PyMySQL
- pandas
- openpyxl
- python-dotenv
- pytest

### 数据库

- MySQL

### 前端

- React
- Vite
- Axios
- 普通 CSS

### 技术选择说明

- FastAPI 用于快速构建清晰的 REST API，并自动提供 Swagger 文档。
- SQLAlchemy engine + text 用于轻量数据库访问，当前阶段不引入 ORM Model，避免过早复杂化。
- pandas + openpyxl 用于生成和导入 Excel 模拟数据，适合 MVP 阶段快速构造业务场景。
- React + Vite 用于快速实现运营控制台页面。
- pytest 用于验证核心规则函数，保证 Agent 判断逻辑可测试、可复用。

## 4. 项目目录结构

```text
amazon_inventory_agent/
├─ backend/
│  ├─ app/
│  │  ├─ api/                    # FastAPI 路由
│  │  ├─ core/                   # 配置与数据库连接
│  │  ├─ repositories/           # 数据访问层
│  │  ├─ schemas/                # API 数据结构
│  │  ├─ services/               # Agent 分析与任务生成服务
│  │  ├─ utils/                  # 规则引擎
│  │  └─ main.py                 # FastAPI 入口
│  ├─ data/                      # 生成的 Excel 模拟数据
│  ├─ scripts/                   # 数据生成、导入、Agent 运行脚本
│  ├─ sql/                       # MySQL 建表 SQL
│  ├─ tests/                     # 单元测试
│  ├─ .env.example               # 环境变量示例
│  └─ requirements.txt           # Python 依赖
├─ docs/                         # 产品、架构、数据库、规则与开发计划文档
├─ frontend/
│  ├─ src/
│  │  ├─ api/                    # Axios API 封装
│  │  ├─ components/             # 通用展示组件
│  │  ├─ pages/                  # Dashboard、Inventory、Tasks、SKU Detail
│  │  ├─ styles/                 # 全局样式
│  │  ├─ App.jsx
│  │  └─ main.jsx
│  ├─ index.html
│  ├─ package.json
│  └─ vite.config.js
└─ README.md
```

## 5. 核心数据表

项目使用 6 张核心表：

| 表名 | 作用 |
|---|---|
| `amazon_product_master` | 商品主数据，保存 SKU、ASIN、标题、品牌、分类等基础信息 |
| `amazon_inventory_snapshot` | 库存快照，保存可售库存、在途库存、不可售库存等数据 |
| `amazon_sales_summary` | 销量汇总，保存 7 日、30 日销量和日均销量 |
| `inventory_replenishment_config` | 补货配置，保存安全库存天数、目标库存天数、采购周期、箱规等 |
| `inventory_agent_analysis` | Agent 分析结果，保存风险等级、建议动作、建议补货量、原因说明 |
| `inventory_agent_tasks` | Agent 自动生成的运营任务，保存任务类型、优先级、状态、处理人备注 |

业务主键围绕以下字段组合：

```text
seller_id + marketplace_id + seller_sku
```

该组合能够唯一定位一个卖家在一个 Amazon 站点下的一个 SKU，是库存、销量、配置和分析结果之间的核心关联方式。

当前 SQL 不使用外键，原因是 MVP 阶段更关注数据导入、分析和任务生成流程，同时便于后续接入外部平台数据、处理缺失数据和保留历史快照。

## 6. 模拟数据系统

脚本：

```text
backend/scripts/generate_demo_excel.py
```

生成 4 个 Excel 文件：

```text
backend/data/product_master.xlsx
backend/data/inventory_snapshot.xlsx
backend/data/sales_summary.xlsx
backend/data/replenishment_config.xlsx
```

模拟数据包含 30 个 SKU，覆盖 7 类典型库存运营场景：

| 场景 | 数量 | 说明 |
|---|---:|---|
| 正常库存 SKU | 8 | 库存充足，销量稳定 |
| 高断货风险 SKU | 6 | 可售库存少，销量较高，补货周期长 |
| 已断货 SKU | 3 | `fulfillable_quantity = 0` |
| 滞销 SKU | 5 | 库存高，销量低 |
| 在途库存较多 SKU | 4 | `inbound_shipped_quantity` 或 `inbound_receiving_quantity` 较高 |
| 不可售异常 SKU | 2 | `total_unfulfillable_quantity` 较高 |
| 数据缺失 SKU | 2 | 用于测试数据质量判断 |

其中 `sales_summary.xlsx` 和 `replenishment_config.xlsx` 各故意缺少 1 条数据，用于验证 `missing_sales` 和 `missing_config` 场景。

## 7. 数据导入系统

脚本：

```text
backend/scripts/import_demo_data.py
```

功能：

- 读取 `backend/data/` 下的 4 个 Excel 文件
- 使用 `pandas.read_excel()` 加载数据
- 使用 `pandas.to_sql()` 写入 MySQL
- 导入前先清空前 4 张业务基础表
- 不清空 analysis 和 tasks 表
- 不自动建表
- 不使用 ORM

导入顺序：

```text
amazon_product_master
amazon_inventory_snapshot
amazon_sales_summary
inventory_replenishment_config
```

这种设计适合 MVP 阶段快速验证数据链路：表结构由 SQL 明确控制，导入脚本只负责把模拟数据放进数据库。

## 8. Agent 规则引擎

文件：

```text
backend/app/utils/risk_rules.py
```

核心函数包括：

| 函数 | 作用 |
|---|---|
| `calculate_daily_sales_for_risk` | 优先使用 7 日日均销量，缺失时回退到 30 日日均销量 |
| `calculate_available_days` | 计算可售库存还能支撑多少天 |
| `calculate_total_cover_days` | 计算总库存覆盖天数 |
| `calculate_effective_inbound_quantity` | 计算有效在途库存，只包含 shipped 和 receiving |
| `calculate_estimated_stockout_date` | 估算断货日期 |
| `judge_stockout_risk` | 判断断货风险等级 |
| `judge_overstock_risk` | 判断滞销风险等级 |
| `round_up_to_carton` | 按箱规向上取整 |
| `calculate_recommended_replenishment_quantity` | 计算建议补货数量 |
| `get_recommended_action` | 输出建议动作 |
| `judge_need_manual_approval` | 判断是否需要人工审批 |
| `build_action_reason` | 生成中文原因说明 |
| `judge_data_quality` | 判断数据质量状态 |

当前版本是规则驱动 Agent，不调用 LLM。这样可以保证结果稳定、可解释、可测试，也更适合库存风控、补货建议这类需要明确业务逻辑的场景。

## 9. Agent 分析服务

文件：

```text
backend/app/services/inventory_analysis_service.py
```

分析流程：

1. 读取所有商品主数据 SKU。
2. 为每个 SKU 获取最新库存快照。
3. 获取最新销量汇总。
4. 获取补货配置。
5. 判断数据质量。
6. 调用 `risk_rules.py` 计算风险指标。
7. 生成建议补货数量和建议动作。
8. 生成中文原因说明。
9. 写入 `inventory_agent_analysis`。

每次运行会生成一个 `analysis_batch_id`，用于标记同一批次的分析结果，便于追踪本次 Agent 运行产生的数据。

## 10. 任务生成服务

文件：

```text
backend/app/services/task_generation_service.py
```

任务生成规则：

| 触发条件 | 任务类型 |
|---|---|
| 断货风险为 `critical` 或 `high` | `stockout_warning` |
| 建议动作为 `replenish_now` 或 `prepare_replenishment` | `replenishment_suggestion` |
| 滞销风险为 `high` | `overstock_warning` |
| 不可售库存大于 0 | `unfulfillable_inventory_alert` |
| 数据质量不是 `complete` | `data_missing_alert` |

任务会写入 `inventory_agent_tasks`，默认状态为 `pending`。这张表代表 Agent 从“分析系统”进入“运营工作流”的起点。

## 11. FastAPI 接口

后端入口：

```text
backend/app/main.py
```

启动命令：

```bash
uvicorn backend.app.main:app --reload
```

默认地址：

```text
http://127.0.0.1:8000
```

Swagger：

```text
http://127.0.0.1:8000/docs
```

主要接口：

| 接口 | 作用 |
|---|---|
| `GET /` | 健康检查 |
| `GET /api/dashboard/summary` | 获取 Dashboard 汇总指标 |
| `GET /api/inventory/analysis` | 获取库存分析列表，支持筛选 |
| `GET /api/inventory/analysis/{seller_sku}` | 获取单个 SKU 最新分析结果 |
| `GET /api/tasks` | 获取任务列表，支持筛选 |
| `POST /api/tasks/{task_id}/resolve` | 将任务标记为已解决 |
| `POST /api/tasks/{task_id}/ignore` | 将任务标记为忽略 |
| `POST /api/agent/run-inventory-analysis` | 重新运行库存 Agent 分析 |

Swagger 对这个 Demo 很重要，因为它能让面试官、合作方或前端开发者直接看到 Agent 系统对外暴露的能力。

## 12. React 前端

前端目录：

```text
frontend/
```

启动命令：

```bash
cd frontend
npm install
npm run dev
```

默认地址：

```text
http://127.0.0.1:5173
```

页面：

| 页面 | 作用 |
|---|---|
| Dashboard | 展示总 SKU、断货风险、滞销风险、数据缺失、任务数等指标 |
| InventoryList | 展示 30 条 SKU 分析结果，支持风险和 SKU 筛选 |
| TaskList | 展示 Agent 生成的任务，支持标记已解决和忽略 |
| SkuDetail | 查看单个 SKU 的详细库存分析 |

组件：

| 组件 | 作用 |
|---|---|
| `MetricCard` | Dashboard 指标卡片 |
| `RiskBadge` | 根据风险等级显示不同样式 |
| `InventoryTable` | 库存分析表格 |
| `TaskCard` | 任务展示卡片 |

前端不是简单的数据列表，而是一个轻量级 AI Agent 控制台：运营人员可以查看风险、理解原因、处理任务，并重新运行 Agent。

## 13. 本地运行步骤

### 1. 创建 MySQL 数据库

```sql
CREATE DATABASE amazon_inventory_agent
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

### 2. 执行建表 SQL

```bash
mysql -u root -p amazon_inventory_agent < backend/sql/01_create_tables.sql
```

### 3. 安装后端依赖

```bash
cd backend
pip install -r requirements.txt
```

### 4. 配置环境变量

复制示例文件：

```bash
cp .env.example .env
```

填写本地 MySQL 配置：

```text
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=amazon_inventory_agent
```

不要把真实 `.env` 文件提交到 GitHub。

### 5. 生成 Excel 模拟数据

```bash
python backend/scripts/generate_demo_excel.py
```

### 6. 导入 Excel 到 MySQL

```bash
python backend/scripts/import_demo_data.py
```

### 7. 运行库存 Agent

```bash
python backend/scripts/run_inventory_agent.py
```

### 8. 启动后端 API

```bash
uvicorn backend.app.main:app --reload
```

### 9. 启动前端

```bash
cd frontend
npm install
npm run dev
```

## 14. 测试

运行规则引擎单元测试：

```bash
python -m pytest backend/tests/test_inventory_rules.py
```

测试覆盖场景：

- 正常库存
- 已断货
- 高断货风险
- 中等断货风险
- 无销量
- 滞销
- 箱规取整
- 数据缺失
- 无效销量
- 推荐动作判断

## 15. 验证 SQL

可以通过以下 SQL 检查数据：

```sql
SELECT COUNT(*) FROM amazon_product_master;
SELECT COUNT(*) FROM amazon_inventory_snapshot;
SELECT COUNT(*) FROM amazon_sales_summary;
SELECT COUNT(*) FROM inventory_replenishment_config;
SELECT COUNT(*) FROM inventory_agent_analysis;
SELECT COUNT(*) FROM inventory_agent_tasks;
```

当前 Demo 预期数据量：

| 表 | 预期行数 |
|---|---:|
| `amazon_product_master` | 30 |
| `amazon_inventory_snapshot` | 30 |
| `amazon_sales_summary` | 29 |
| `inventory_replenishment_config` | 29 |
| `inventory_agent_analysis` | 30 |
| `inventory_agent_tasks` | 65 |

其中 `sales_summary` 和 `replenishment_config` 各少 1 条是故意设计的，用于测试数据缺失场景。

## 16. 当前完成度

已完成：

- 产品规划文档
- 系统架构文档
- 数据库设计文档
- Agent 规则文档
- MySQL 建表 SQL
- Excel 模拟数据生成
- Excel 数据导入 MySQL
- 规则引擎
- pytest 单元测试
- Agent 分析服务
- 任务生成服务
- FastAPI 接口
- Swagger
- React 前端控制台
- Dashboard / Inventory / Tasks / SKU Detail 页面
- Agent 手动重跑
- 任务 resolve / ignore 状态更新

未完成：

- 接入真实 Amazon SP-API
- 用户系统
- 权限系统
- 多租户
- 自动定时调度
- LLM 分析
- 广告 Agent
- 利润 Agent
- SaaS 化部署
- 云端部署

当前项目处于“可运行的本地 MVP Demo”阶段。

## 17. 为什么这个项目超过普通 CRUD Demo

普通 CRUD 项目通常只完成数据的增删改查，而本项目已经具备：

- 数据层：商品、库存、销量、补货配置、分析结果、任务数据
- 策略层：可测试的库存风险规则
- Agent 层：自动分析 SKU 并生成建议
- 任务层：把分析结果转化为运营动作
- API 层：提供可集成的后端接口
- 可视化层：提供运营人员可以使用的前端控制台

它的核心价值不是“调用一个模型”，而是把业务数据、规则判断、任务生成和前端操作串成一个完整闭环。

## 18. 后续扩展方向

高优先级：

1. 接入 Amazon SP-API，替换模拟 Excel 数据。
2. 增加定时任务，让 Agent 每天自动运行。
3. 增加用户与权限系统，支持多人运营。
4. 引入 LLM，用于生成更自然的运营解释和策略摘要。
5. 增加任务处理记录，形成完整运营审计链路。

中长期方向：

1. 广告 Agent：结合广告花费、ACOS、转化率判断投放动作。
2. 利润 Agent：结合采购成本、FBA 费用、售价判断真实利润。
3. 多 Agent 协同：库存、广告、利润、价格联动决策。
4. SaaS 化部署：支持多卖家、多店铺、多站点。
5. AI Copilot：让运营人员通过自然语言查询库存风险和任务。
6. 自动执行运营动作：在审批后自动创建补货计划或运营动作。

## 19. 项目总结

Amazon Inventory Agent Demo 本质上是一个系统级 AI Agent MVP。

它没有停留在“调用 GPT API 生成一句建议”的层面，而是搭建了一个完整的业务闭环：

```text
业务数据 → 风险规则 → Agent 分析 → 任务生成 → API → 前端控制台 → 人工处理
```

当前版本虽然还没有接入真实 Amazon 数据和 LLM，但已经具备一个 AI Agent 产品最重要的基础结构：数据层、策略层、任务层、接口层和可视化层。
