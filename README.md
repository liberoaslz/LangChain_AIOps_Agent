# OpsPilot AIOps 智能运维分析系统

基于 **LangChain ReAct Agent + RAG + Tool Calling + FastAPI + React** 构建的 AIOps 智能运维分析项目，面向运维知识问答、故障排查、告警分析、指标分析、日志诊断、服务拓扑分析和运行报告生成等场景。

本项目不是普通聊天机器人，而是一个面向运维工作流设计的 Agent 系统。用户可以用自然语言提出问题，系统由 Agent 判断任务类型，并按需调用知识库检索、告警查询、指标查询、日志摘要、服务拓扑和报告生成等工具，最终输出结构化分析结果。

> 说明：当前告警、指标、日志和拓扑数据为项目内置模拟数据，主要用于验证 Agent 工具调用和 AIOps 分析流程。后续可替换为 Prometheus、Elasticsearch、Loki、CMDB、告警平台等真实数据源。

---

## 效果预览

### 空会话界面

左侧为会话管理、接口信息和常用问题，右侧为智能运维对话区，底部输入框固定展示，支持文件上传、快速对话和流式对话切换。

<img src="photo/ui-empty-chat.png" width="900"/>

### 运维分析结果界面

用户提交服务异常分析问题后，Agent 会结合告警、指标、日志、拓扑和知识库信息生成结构化分析结果。

<img src="photo/ui-aiops-analysis.png" width="900"/>

---

## 项目定位

项目目标是构建一个具备以下能力的 AIOps Agent：

- 理解用户的运维问题和故障描述；
- 基于 RAG 检索内部运维文档、SOP 和故障处理经验；
- 自动调用告警、指标、日志、拓扑等工具补充上下文；
- 对服务异常进行初步根因分析和影响范围判断；
- 输出排查步骤、处理建议和结构化运行报告；
- 通过 FastAPI 对外提供接口，并通过 React 前端完成交互。

典型问题示例：

```text
CPU 使用率过高怎么排查
分析一下 order-service 最近1小时的异常情况
payment-service 今天有哪些高频异常
order-service 的问题可能影响哪些下游服务
生成 inventory-service 本月运行报告
```

---

## 当前架构

```text
用户浏览器
  |
  |  http://localhost:5173
  v
React / Vite 前端
  |
  |  /api 代理到 http://localhost:6872
  v
FastAPI 后端
  |
  v
LangChain ReAct Agent
  |
  +-- RAG 知识库工具
  +-- 告警查询工具
  +-- 指标查询工具
  +-- 日志摘要工具
  +-- 服务拓扑工具
  +-- 报告数据工具
  |
  v
LLM 汇总分析并返回结果
```

核心分层：

- **前端交互层**：基于 React/Vite 实现聊天界面、快速对话、流式对话、文件上传、AIOPS 按钮、历史会话管理。
- **API 服务层**：基于 FastAPI 暴露 `/api/chat`、`/api/chat_stream`、`/api/ai_ops`、`/api/upload` 接口。
- **Agent 层**：基于 LangChain `create_agent` 构建 ReAct Agent，负责理解任务、选择工具、聚合上下文并生成答案。
- **工具调用层**：封装 RAG、告警、指标、日志、拓扑和报告工具，供 Agent 动态调用。
- **RAG 知识库层**：支持文档加载、文本切分、Embedding 向量化、Chroma 存储、BM25 检索、RRF 融合和 Rerank 精排。
- **工程支撑层**：提供 YAML 配置、日志记录、查询缓存、MD5 去重和 RAG 评估脚本。

---

## 核心功能

### 1. 快速对话

前端调用 `/api/chat`，后端根据会话 `Id` 保存多轮消息，并调用 Agent 返回完整答案。适合普通知识问答、运维排查建议和服务异常分析。

### 2. 流式对话

前端调用 `/api/chat_stream`，后端以 SSE 形式返回数据。当前实现是先生成完整答案，再按固定长度切分返回，属于前端体验上的流式展示，不是模型原生 token 级流式输出。

### 3. AIOps 一键分析

前端右上角 AIOPS 按钮调用 `/api/ai_ops`，后端根据服务名和时间范围构造运维分析问题，Agent 结合告警、指标、日志、拓扑和 RAG 信息生成分析结果。

### 4. 知识库文件上传

前端点击输入框左侧加号上传文件，后端 `/api/upload` 将文件保存到 `data/` 目录，并触发 `VectorStoreService.load_document()` 更新 Chroma 知识库。

当前知识库加载配置支持 `txt` 和 `pdf` 文件。前端上传入口目前偏向 Markdown 文件交互，如果需要让 `.md` 真正进入知识库索引，需要在后端文件类型配置和 loader 中补充 Markdown 支持。

### 5. 历史会话管理

前端支持新建会话、切换历史会话和删除会话。后端也维护了基于 `Id` 的内存会话消息列表，用于多轮对话上下文。

---

## API 接口

后端服务默认启动在：

```text
http://localhost:6872
```

### POST /api/chat

快速对话接口。

请求示例：

```json
{
  "Id": "session-001",
  "Question": "CPU 使用率过高怎么排查"
}
```

返回核心字段：

```json
{
  "message": "OK",
  "data": {
    "Answer": "..."
  }
}
```

### POST /api/chat_stream

流式对话接口，使用 SSE 返回事件。

事件类型：

- `connected`：连接建立；
- `message`：回答内容分片；
- `done`：输出结束。

### POST /api/ai_ops

AIOps 一键分析接口。

请求示例：

```json
{
  "service_name": "order-service",
  "time_range": "最近1小时",
  "question": null
}
```

如果 `question` 为空，后端会根据 `service_name` 和 `time_range` 自动构造分析问题。

### POST /api/upload

知识库文件上传接口。后端保存文件后触发知识库加载。

---

## RAG 检索链路

项目的 RAG 检索不是简单向量检索，而是多阶段检索增强：

```text
用户问题
  -> 查询缓存
  -> Query Rewrite 生成多个改写 query
  -> 改写 query 相似度过滤
  -> 原始 query + 有效改写 query
  -> 每个 query 并行执行 Chroma 向量检索 + BM25 关键词检索
  -> 对单个 query 的两路结果做 RRF 融合
  -> 对多个 query 的候选结果再次 RRF 融合
  -> 使用原始问题做 embedding 相似度 Rerank
  -> 拼接 Top 文档上下文
  -> LLM 基于上下文生成回答
  -> 写入缓存
```

### Chroma 向量检索

文档进入知识库时，会先被切分成 chunk，然后通过 DashScope Embedding 模型转换为向量表示，存入 Chroma。用户提问时，也会把 query 转成向量，再检索语义相近的文档片段。

### BM25 关键词检索

BM25 用于处理运维场景中的精确关键词，例如：

```text
OOM
HTTP 500
df -h
Slow SQL
Too many connections
No space left on device
```

这类问题只靠向量检索可能不稳定，BM25 可以增强对命令、错误码、指标名和英文报错的召回能力。

### RRF 融合

RRF 根据排名融合多路检索结果，不直接相加原始分数。因为 BM25 分数和向量相似度不是同一个量纲，直接加分没有意义。

项目中 RRF 的计算方式是：

```text
score += 1 / (k + rank)
```

其中 `k` 默认为 60，`rank` 是文档在某一路检索结果中的排名。如果同一篇文档同时被向量检索和 BM25 命中，它的分数会累加，最终排名更靠前。

### Rerank 精排

RRF 融合后得到候选文档，项目会再使用原始用户问题做精排：

```text
原始 query -> embedding
候选文档 -> embedding
计算余弦相似度
按相似度排序
返回 Top 5
```

这里的 Rerank 是基于 embedding 相似度实现的，不是单独接入 cross-encoder reranker 模型。

---

## Agent 工具体系

当前 Agent 注册的工具包括：

| 工具 | 作用 |
| --- | --- |
| `rag_summarize` | 检索运维知识、SOP、故障经验和系统说明 |
| `get_target_service` | 当问题缺少服务名时，补全或获取目标服务 |
| `get_time_range` | 当问题缺少时间范围时，补全默认分析时间 |
| `fetch_alert_data` | 获取指定服务在指定时间内的告警信息 |
| `fetch_metric_data` | 获取 CPU、内存、QPS、RT、错误率、Pod 重启次数等指标 |
| `fetch_log_summary` | 获取日志摘要和高频错误信息 |
| `fetch_service_topology` | 获取上下游依赖关系和中间件拓扑 |
| `fetch_report_data` | 聚合报告生成需要的告警、指标、日志和拓扑数据 |
| `fill_context_for_report` | 标记进入报告模式，配合中间件切换提示词 |

工具层当前使用结构化模拟数据验证流程，真实落地时可以把工具内部数据获取逻辑替换为真实系统接口。

---

## 中间件机制

项目使用 LangChain Agent middleware 实现工具监控、模型调用日志和动态提示词切换。

- `monitor_tool`：拦截工具调用，记录工具名和参数；当检测到调用 `fill_context_for_report` 时，将 `runtime.context["report"]` 设置为 `True`。
- `log_before_model`：模型调用前记录消息数量和当前消息内容，便于调试。
- `report_prompt_switch`：每次模型调用前检查 `runtime.context["report"]`，如果为 `True` 则加载 `report_prompt.txt`，否则加载 `main_prompt.txt`。

因此报告模式的触发流程是：

```text
用户请求生成报告
  -> Agent 根据主提示词调用 fill_context_for_report
  -> middleware 将 report 标记设置为 True
  -> dynamic_prompt 切换为报告提示词
  -> Agent 获取报告数据并生成结构化报告
```

---

## 项目目录

```text
.
├── api_server.py                 # FastAPI 后端服务入口
├── app.py                        # 早期本地交互入口，当前主要使用 FastAPI + React
├── agent/
│   ├── react_agent.py            # LangChain ReAct Agent 封装
│   └── tools/
│       ├── agent_tools.py        # RAG / 告警 / 指标 / 日志 / 拓扑 / 报告工具
│       └── middleware.py         # 工具监控、日志、动态提示词切换
├── config/
│   ├── agent.yml                 # Agent 相关配置
│   ├── chroma.yml                # Chroma、文档路径、分块参数配置
│   ├── prompts.yml               # prompt 文件路径配置
│   └── rag.yml                   # LLM 和 embedding 模型配置
├── data/                         # 知识库源文件目录
├── frontend/
│   ├── src/                      # React 前端源码
│   ├── package.json              # 前端依赖和启动脚本
│   └── vite.config.ts            # Vite 配置，代理 /api 到后端 6872
├── model/
│   └── factory.py                # ChatTongyi 和 DashScope Embeddings 工厂
├── prompts/
│   ├── main_prompt.txt           # 普通问答和工具调用主提示词
│   ├── report_prompt.txt         # 报告生成提示词
│   └── rag_summarize.txt         # RAG 总结提示词
├── rag/
│   ├── bm25_retriever.py         # BM25 关键词检索
│   ├── query_rewriter.py         # Query 改写
│   ├── rag_service.py            # RAG 检索与总结主流程
│   ├── reranker.py               # embedding 相似度精排
│   ├── rrf_fusion.py             # RRF 排名融合
│   ├── semantic_checker.py       # 改写问题语义相似度校验
│   └── vector_store.py           # Chroma 向量库加载与检索
├── tests/
│   ├── rag_evaluation.py         # RAG 召回评估脚本
│   └── test_cases.json           # RAG 测试用例
├── utils/
│   ├── cache.py                  # 查询缓存
│   ├── config_handler.py         # YAML 配置读取
│   ├── file_handler.py           # 文件加载与 MD5 去重
│   ├── logger_handler.py         # 日志模块
│   ├── path_tool.py              # 路径工具
│   └── prompt_loader.py          # prompt 加载器
├── requirements.txt              # Python 依赖
└── README.md
```

---

## 技术栈

- **后端语言**：Python
- **Agent 框架**：LangChain ReAct Agent
- **模型接入**：Tongyi / DashScope，当前聊天模型配置为 `qwen3-max`
- **Embedding 模型**：`text-embedding-v4`
- **向量数据库**：Chroma
- **RAG 检索**：Query Rewrite、Chroma、BM25、RRF、Embedding Rerank
- **Web 后端**：FastAPI、Uvicorn、SSE
- **Web 前端**：React、Vite、TypeScript、lucide-react
- **工程支撑**：YAML 配置、日志、缓存、MD5 去重、RAG 评估

说明：项目代码中引入了 LangGraph runtime/types 用于 middleware 相关类型和 Agent 底层运行机制，但当前没有手写 LangGraph 图编排流程。

---

## 怎么运行项目

项目运行时需要同时启动两个服务：

```text
后端 FastAPI：负责 Agent、RAG、工具调用和模型请求，端口 6872
前端 React/Vite：负责浏览器页面展示和交互，端口 5173
```

所以完整启动方式是：**先启动后端，再启动前端，最后浏览器访问前端页面**。

### 第一次运行

第一次运行需要先安装依赖、配置模型 Key、初始化知识库。

#### 1. 进入项目目录

```powershell
cd D:\agent\AIOps
```

#### 2. 安装 Python 后端依赖

```powershell
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
```

如果之前报过 `No module named 'fastapi'`，就是后端依赖没有装完整，执行这一步即可。

#### 3. 配置 DashScope API Key

项目调用通义千问 / DashScope 模型时，会从环境变量 `DASHSCOPE_API_KEY` 读取 Key。不要把真实 Key 写死在代码里。

方式一：临时配置，只对当前 PowerShell 窗口有效：

```powershell
$env:DASHSCOPE_API_KEY="your-api-key"
```

其中 `your-api-key` 要替换成你自己的真实 DashScope API Key，不能直接照抄。

方式二：永久配置到当前 Windows 用户环境变量：

```powershell
[Environment]::SetEnvironmentVariable("DASHSCOPE_API_KEY", "your-api-key", "User")
```

同样需要把 `your-api-key` 替换成真实 Key。

永久配置后，需要**关闭当前 PowerShell，重新打开一个新的 PowerShell**，再检查是否生效：

```powershell
echo $env:DASHSCOPE_API_KEY
```

如果能输出你的 Key，说明环境变量已经生效。后续启动后端时就不需要每次再执行 `$env:DASHSCOPE_API_KEY=...`。

#### 4. 初始化或更新知识库

把运维文档放到：

```text
D:\agent\AIOps\data
```

当前后端知识库加载支持 `txt` 和 `pdf` 文件。

然后执行：

```powershell
python -c "from rag.vector_store import VectorStoreService; v=VectorStoreService(); v.load_document(); print(v.vector_store._collection.count())"
```

这一步会把文档切分、生成 embedding 向量并写入 Chroma。系统会用 MD5 记录已经处理过的文件，避免重复入库。

#### 5. 安装前端依赖

前端需要 Node.js 和 npm。先检查当前终端是否能识别：

```powershell
node -v
npm -v
```

如果提示 `node` 或 `npm` 无法识别，先安装 Node.js LTS：

```powershell
winget install OpenJS.NodeJS.LTS --accept-package-agreements --accept-source-agreements
```

安装完成后，**重新打开一个 PowerShell**，再执行：

```powershell
cd D:\agent\AIOps\frontend
npm install
```

如果你习惯使用 `pnpm`，也可以执行：

```powershell
pnpm install
```

前端本质上是 Vite 项目，不强制必须用 `pnpm`。只要电脑里有 Node.js 和 npm，就可以用 `npm install` 和 `npm run dev` 启动。

---

### 日常启动

以后每次运行项目，通常只需要启动后端和前端。

#### 1. 启动后端

打开第一个 PowerShell 终端。如果你已经把 `DASHSCOPE_API_KEY` 永久配置到系统环境变量里，直接执行：

```powershell
cd D:\agent\AIOps
uvicorn api_server:app --host 0.0.0.0 --port 6872 --reload
```

如果没有永久配置环境变量，也可以在当前终端临时设置后再启动：

```powershell
cd D:\agent\AIOps
$env:DASHSCOPE_API_KEY="your-api-key"
uvicorn api_server:app --host 0.0.0.0 --port 6872 --reload
```

注意：这里的 `your-api-key` 是占位符，必须替换成真实 Key，否则尽管后端启动完成，请求模型时也会报 `401 InvalidApiKey`。

看到类似下面的信息，说明后端启动成功：

```text
Uvicorn running on http://0.0.0.0:6872
```

此时可以打开后端接口文档：

```text
http://localhost:6872/docs
```

注意：这个终端不要关，关了后端服务就停了。

#### 2. 启动前端

再打开第二个 PowerShell 终端：

```powershell
cd D:\agent\AIOps\frontend
npm run dev
```

看到类似下面的信息，说明前端启动成功：

```text
Local:   http://localhost:5173/
```

然后在浏览器访问：

```text
http://localhost:5173
```

这就是项目的主页面。

---

### 运行后的访问关系

浏览器访问的是前端：

```text
http://localhost:5173
```

前端请求接口时，会通过 Vite 代理转发到后端：

```text
/api/chat        -> http://localhost:6872/api/chat
/api/chat_stream -> http://localhost:6872/api/chat_stream
/api/ai_ops      -> http://localhost:6872/api/ai_ops
/api/upload      -> http://localhost:6872/api/upload
```

因此，前端页面能打开不代表整个项目都可用；只有后端也启动了，聊天、AIOPS 分析和文件上传才会正常工作。

---

## 最小验证

后端启动后，可以在浏览器访问：

```text
http://localhost:6872/docs
```

也可以在前端页面测试以下问题：

```text
CPU 使用率过高怎么排查
```

```text
分析一下 order-service 最近1小时的异常情况
```

```text
payment-service 今天有哪些高频异常
```

```text
生成 inventory-service 本月运行报告
```

如果能够返回结构化结果，说明前端、FastAPI、Agent、工具调用和 RAG 链路已经跑通。

---

## RAG 评估

项目提供 RAG 评估脚本：

```powershell
python -m tests.rag_evaluation
```

评估内容包括：

- `Recall@1`：正确文档是否出现在 Top 1；
- `Recall@3`：正确文档是否出现在 Top 3；
- `Recall@5`：正确文档是否出现在 Top 5；
- `Keyword Hit Rate`：生成答案是否命中预期关键词。

测试用例位于：

```text
tests/test_cases.json
```

当前项目记录的 RAG 评测结果：

```text
Recall@1: 80%
Recall@3: 96.67%
```

---

## 常见问题

### 1. 为什么后端要先启动？

前端只负责页面交互，真正的 Agent、RAG、工具调用和模型请求都在 FastAPI 后端中执行。前端请求 `/api/chat`、`/api/chat_stream`、`/api/ai_ops`、`/api/upload` 时，需要后端服务在 `6872` 端口可用。

### 2. 为什么端口是 6872 和 5173？

`6872` 是启动 FastAPI 时指定的后端端口：

```powershell
uvicorn api_server:app --host 0.0.0.0 --port 6872 --reload
```

`5173` 是 `frontend/package.json` 和 Vite 配置中指定的前端开发端口。

### 3. 为什么上传文件后模型不一定知道？

上传文件只是把文件保存到 `data/` 并触发知识库加载。当前后端知识库配置只处理 `txt/pdf`，如果上传 Markdown 文件但没有扩展后端 loader，它可能不会被真正索引进 Chroma。

### 4. 当前是真实运维数据吗？

当前告警、指标、日志和拓扑是模拟数据，用于验证 Agent 工具调用链路。真实生产接入时，可以将工具内部逻辑替换为 Prometheus、Elasticsearch、Loki、CMDB 或告警平台接口。

### 5. `/api/chat_stream` 是真正的大模型流式输出吗？

当前不是。它先调用 Agent 得到完整答案，再切分为 SSE 片段返回给前端，因此是接口层面的流式展示。若要实现真正 token 级流式输出，需要启用模型 streaming 并改造 Agent 调用方式。

---

## 可扩展方向

- 接入 Prometheus 获取实时指标；
- 接入 Elasticsearch / Loki 获取真实日志；
- 接入告警平台获取真实告警；
- 接入 CMDB 或服务拓扑平台；
- 支持 Markdown 文档入库；
- 将当前模拟工具替换为真实后端接口；
- 增加用户认证、会话持久化和权限控制；
- 支持真正 token 级流式输出；
- 增加更多 RAG 评估指标和可视化评估报告。

---

## License

本项目适合作为 Agent、AIOps、RAG 和 Tool Calling 学习与实践项目使用。
