# LangChain ReAct Agent AIOps 智能运维系统

基于 **LangChain + ReAct Agent + RAG + Tool Calling** 构建的 AIOps 智能运维项目，面向运维知识问答、告警分析、日志辅助诊断、监控指标解读、服务拓扑分析与运行报告生成等场景。

本项目不是通用聊天机器人，而是一个面向运维工作流设计的智能体系统。系统通过 Agent 统一调度 RAG 知识库、告警工具、指标工具、日志工具、拓扑工具和报告生成工具，支持对 `order-service`、`payment-service`、`inventory-service` 等业务服务进行自然语言分析。

## 效果预览

### 1. 聊天界面展示

展示用户在前端输入问题后，系统返回问答结果的整体效果，左侧为历史会话记录。

<img src="photo/1.png" width="700"/>

### 2. Agent 工具调用过程

展示 Agent 在任务处理中自动选择并调用工具的过程。

<img src="photo/2.png" width="700"/>

### 3. 报告生成示例

展示系统根据服务运行数据生成结构化运维分析报告的结果页面。

<img src="photo/3.png" width="700"/>

---

## 1. 项目定位

本项目的目标是构建一个具备 **运维知识理解、故障辅助诊断、多工具协同分析和结构化报告生成能力** 的 AIOps 智能体系统。

系统能够完成以下任务：

- 理解用户输入的运维问题
- 识别服务名、时间范围和故障类型
- 调用 RAG 知识库回答运维基础知识与故障 SOP
- 调用告警、指标、日志、拓扑等工具补充上下文
- 对故障现象进行初步根因推测
- 输出排查步骤、处理建议和运行报告

典型应用场景包括：

- 运维知识问答
- 服务异常分析
- 日志辅助诊断
- 监控指标解读
- 服务依赖影响分析
- 日报、周报、月报生成

---

## 2. 核心能力

### 2.1 ReAct Agent 多工具协同

项目使用 `create_agent(...)` 构建 ReAct 智能体，将模型、系统提示词、工具集合和中间件统一组合起来。Agent 可以根据用户问题自动选择调用路径，例如：

- 查询运维知识时调用 `rag_summarize`
- 分析服务异常时调用告警、指标、日志和拓扑工具
- 生成运行报告时调用报告上下文工具和报告数据聚合工具

这种设计使系统具备从“自然语言问题”到“工具调用”再到“结构化答案生成”的完整链路。

### 2.2 RAG 运维知识库

项目通过 `Chroma` 构建本地向量知识库，支持从 `txt / pdf` 运维文档中加载知识，覆盖 CPU 高、内存高、OOM、磁盘满、服务不可用、接口响应慢、数据库连接异常、Redis 缓存问题、MQ 堆积、发布回滚、`order-service` 业务服务说明等内容。

RAG 链路用于解决通用模型不了解本地运维资料、内部服务概念和项目私有知识的问题。

### 2.3 混合检索与精排

项目在 RAG 检索阶段引入多种检索策略：

- `QueryRewriter`：将口语化问题改写为更适合检索的运维表达
- `SemanticChecker`：校验改写问题是否偏离原始语义
- `BM25Retriever`：对 `OOM`、`df -h`、`HTTP 500`、`Too many connections` 等关键词进行精确召回
- `Chroma`：基于 embedding 完成语义检索
- `RRF Fusion`：融合向量检索和 BM25 关键词检索结果
- `Reranker`：对候选文档进行最终相关性排序
- `QueryCache`：缓存重复查询结果，减少重复检索和模型调用

该设计兼顾了语义相似问题和精确关键词问题，适合运维场景中“口语化提问 + 专业术语 + 英文报错”混合出现的情况。

### 2.4 动态提示词切换

项目将普通问答和报告生成拆成两套提示词：

- `main_prompt.txt`：用于普通运维问答、故障排查和工具调用规范
- `report_prompt.txt`：用于正式报告生成，约束章节结构、语言风格和输出格式
- `rag_summarize.txt`：用于约束 RAG 回答必须基于参考资料生成

当用户请求“生成报告 / 今日运行情况 / 本周分析 / 本月分析 / 系统健康度分析”等内容时，中间件会将 Agent 切换到报告模式。

### 2.5 面向运维的结构化工具集

当前工具层包含以下能力：

- `rag_summarize`：运维知识库问答
- `get_target_service`：获取或补全目标服务名
- `get_time_range`：获取或补全分析时间范围
- `fetch_alert_data`：获取告警信息
- `fetch_metric_data`：获取监控指标
- `fetch_log_summary`：获取日志摘要和高频错误
- `fetch_service_topology`：获取服务上下游依赖和中间件拓扑
- `fetch_report_data`：聚合运行报告数据
- `fill_context_for_report`：注入报告生成上下文

工具层当前使用结构化模拟数据验证 Agent 工作流，后续可将工具内部的数据获取逻辑替换为 Prometheus、Elasticsearch、Loki、CMDB、告警平台或服务拓扑平台接口。

---

## 3. 适用场景

本项目适合以下典型场景：

- 分析某个服务最近 1 小时 / 今日 / 本月的异常情况
- 结合告警、指标和日志进行初步根因推测
- 查询 CPU、内存、磁盘、数据库、Redis、MQ 等运维基础知识
- 解释 `order-service`、`payment-service`、`inventory-service` 等业务服务含义
- 生成服务运行日报、周报、月报和健康度报告
- 分析服务上下游依赖关系和潜在影响范围

示例问题：

```text
啥是 order_service
CPU 使用率过高一般怎么排查
分析一下 order-service 最近1小时的异常情况
payment-service 今天有哪些高频异常
order-service 的问题可能会影响哪些下游服务
生成 inventory-service 本月运行报告
```

---

## 4. 项目架构

```bash
.
├── agent/                       # Agent 核心逻辑
│   ├── react_agent.py           # ReAct Agent 主入口
│   └── tools/
│       ├── agent_tools.py       # 工具定义：RAG / 告警 / 指标 / 日志 / 拓扑 / 报告
│       └── middleware.py        # 中间件：工具监控、日志、动态提示词切换
├── config/                      # YAML 配置
│   ├── agent.yml
│   ├── chroma.yml
│   ├── prompts.yml
│   └── rag.yml
├── data/                        # 运维知识库文档目录
├── model/
│   └── factory.py               # 聊天模型与 Embedding 模型工厂
├── prompts/
│   ├── main_prompt.txt          # 普通问答提示词
│   ├── report_prompt.txt        # 报告生成提示词
│   └── rag_summarize.txt        # RAG 总结提示词
├── rag/
│   ├── bm25_retriever.py        # BM25 关键词检索
│   ├── query_rewriter.py        # Query 改写
│   ├── rag_service.py           # RAG 检索总结服务
│   ├── reranker.py              # 候选文档精排
│   ├── rrf_fusion.py            # RRF 结果融合
│   ├── semantic_checker.py      # 改写问题相似度校验
│   └── vector_store.py          # Chroma 向量库加载与检索
├── tests/
│   ├── rag_evaluation.py        # RAG 召回与回答质量评估
│   └── test_cases.json          # RAG 测试用例
├── utils/
│   ├── cache.py                 # 查询缓存
│   ├── config_handler.py        # 配置读取
│   ├── file_handler.py          # 文件加载与 MD5 去重
│   ├── logger_handler.py        # 日志模块
│   ├── path_tool.py             # 路径工具
│   └── prompt_loader.py         # 提示词加载器
├── app.py                       # Streamlit 交互入口
├── requirements.txt
└── README.md
```

---

## 5. 工作流程

### 5.1 普通运维分析流程

1. 用户在前端输入运维问题
2. `app.py` 将用户问题传给 `ReactAgent.execute_stream`
3. Agent 根据系统提示词判断任务类型
4. 如果问题涉及运维知识或故障 SOP，调用 `rag_summarize`
5. 如果问题涉及服务异常分析，调用告警、指标、日志、拓扑等工具
6. Agent 汇总工具结果和上下文
7. 模型输出结构化分析结果
8. 前端展示最终回答

普通分析输出通常包括：

- 问题分析
- 可能原因
- 排查步骤
- 解决建议

### 5.2 RAG 知识库问答流程

```text
用户问题
↓
Query 改写
↓
语义相似度校验
↓
原问题 + 有效改写问题
↓
Chroma 向量检索 + BM25 关键词检索
↓
RRF 融合候选文档
↓
Reranker 精排
↓
拼接参考资料 context
↓
LLM 基于参考资料生成回答
↓
缓存结果并返回
```

RAG 适合回答以下问题：

```text
order_service 是什么
CPU 使用率过高怎么排查
No space left on device 是什么问题
服务不可用应该怎么处理
接口响应慢怎么定位
Too many connections 怎么排查
```

### 5.3 报告生成流程

当用户请求“生成报告 / 今日运行情况 / 本周分析 / 本月分析 / 系统健康度分析”等内容时，Agent 会进入报告模式：

1. 必要时补全服务名与时间范围
2. 调用 `fill_context_for_report` 设置报告上下文
3. 中间件切换到 `report_prompt.txt`
4. 优先调用 `fetch_report_data` 聚合报告数据
5. 如信息不足，再补充调用告警、指标、日志、拓扑或 RAG 工具
6. 按报告提示词输出正式结构化报告

报告结构通常包括：

- 报告概览
- 核心指标表现
- 告警与异常情况
- 日志与根因分析
- 依赖与影响分析
- 综合判断
- 运维建议

---

## 6. 技术栈

- **开发语言**：Python
- **Agent 框架**：LangChain、LangGraph、ReAct Agent
- **模型接入**：Tongyi / DashScope
- **Embedding 模型**：DashScope Embeddings
- **向量数据库**：Chroma
- **关键词检索**：BM25、jieba
- **检索融合**：RRF
- **文档处理**：TextLoader、PyPDFLoader
- **文本切分**：RecursiveCharacterTextSplitter
- **前端交互**：Streamlit
- **工程能力**：YAML 配置管理、日志系统、模块化结构、RAG 评估

---

## 7. 关键设计

### 7.1 Agent 与工具解耦

Agent 主流程不直接写死业务逻辑，而是通过工具描述和系统提示词选择工具。告警、指标、日志、拓扑、报告和 RAG 均封装为独立工具，便于后续替换数据源或扩展新工具。

### 7.2 多源检索增强 RAG

项目同时使用向量检索和 BM25 关键词检索。向量检索适合语义相似问题，BM25 适合英文报错、命令、指标名等精确关键词问题，例如：

```text
OOM
df -h
HTTP 500
Slow SQL
Too many connections
No space left on device
```

两类检索结果通过 RRF 融合，再通过 Reranker 精排，提高最终召回文档与用户问题的相关性。

### 7.3 Query 改写与相似度校验

运维用户的提问经常比较口语化，例如：

```text
服务挂了咋办
CPU 飙高了
接口慢得很
磁盘快爆了
```

系统通过 Query 改写将其转换为更适合检索的标准表达，例如“服务不可用排查流程”“CPU 使用率过高排查方法”。为了避免改写跑偏，系统会对原问题和改写问题做语义相似度校验，只保留相关改写。

### 7.4 知识库加载去重

向量库加载时会对知识文件计算 MD5，并记录已处理文件，避免同一文档重复入库，保证知识库构建过程稳定可控。

### 7.5 提示词分层设计

项目将提示词拆分为主提示词、报告提示词和 RAG 总结提示词，不同场景使用不同约束：

- 普通问答强调简洁、专业、可执行
- 报告生成强调章节完整和结构规范
- RAG 回答强调必须基于参考资料，不编造内容

---

## 8. RAG 评估

项目提供 RAG 评估脚本：

```bash
python -m tests.rag_evaluation
```

评估内容包括：

- `Recall@1`：正确文档是否出现在 Top 1
- `Recall@3`：正确文档是否出现在 Top 3
- `Recall@5`：正确文档是否出现在 Top 5
- `Keyword Hit Rate`：生成回答是否命中预期关键词

测试用例位于：

```text
tests/test_cases.json
```

测试问题覆盖：

- 标准运维问题
- 口语化问题
- 英文报错
- 命令关键词
- 多现象混合问题
- 业务服务概念问题

示例测试问题：

```text
CPU 使用率过高怎么排查
服务老是 OOM 重启，该查什么
No space left on device 是什么问题
发布后服务打不开，应该先回滚还是查日志
Too many connections 之后服务访问失败，应该查应用还是数据库
```

---

## 9. 快速开始

### 9.1 环境要求

- Python 3.10+
- 可用的 DashScope / Tongyi API Key
- 已安装项目依赖

### 9.2 安装依赖

```bash
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
```

### 9.3 配置 API Key

Windows PowerShell：

```powershell
$env:DASHSCOPE_API_KEY="your-api-key"
```

Windows CMD：

```cmd
set DASHSCOPE_API_KEY=your-api-key
```

Linux / macOS：

```bash
export DASHSCOPE_API_KEY="your-api-key"
```

### 9.4 检查配置文件

首次运行前建议确认：

- `config/rag.yml`
- `config/chroma.yml`
- `config/prompts.yml`
- `config/agent.yml`

其中需要重点关注：

- 聊天模型名称
- Embedding 模型名称
- 向量库存储目录
- 知识库数据目录
- 提示词文件路径

### 9.5 准备知识库文档

将运维文档、SOP、系统说明、故障案例等放入：

```text
data/
```

项目支持 `txt / pdf` 文档加载。对于 RAG 检索，优先推荐使用 UTF-8 编码的 `txt` 文档。

### 9.6 初始化或更新知识库

当 `data/` 中新增知识文档后，需要将文档写入 Chroma 向量库。可以执行：

```powershell
python -c "from rag.vector_store import VectorStoreService; v=VectorStoreService(); v.load_document(); print(v.vector_store._collection.count())"
```

如果文档已经被 MD5 记录为处理过，系统会跳过重复入库。

### 9.7 启动项目

如果项目以 Streamlit 方式运行：

```bash
streamlit run app.py
```

如果本地入口直接封装在 `app.py` 中：

```bash
python app.py
```

---

## 10. 最小化验证

启动后可以优先测试以下问题：

### 运维知识问答

```text
啥是 order_service
```

```text
CPU 使用率过高怎么排查
```

```text
No space left on device 是什么问题
```

### 服务异常分析

```text
分析一下 order-service 最近1小时的异常情况
```

```text
payment-service 今天有哪些高频异常
```

### 报告生成

```text
生成 inventory-service 本月运行报告
```

```text
请给我一份 order-service 今日系统健康度分析
```

如果以上问题能够返回结构化结果，说明 Agent、工具调用、RAG 和前端展示链路已经跑通。

---

## 11. 常见问题

### Q1：为什么把文档放进 data 后模型还是不知道？

因为文档放入 `data/` 后不会自动进入知识库，需要执行向量库加载流程，将文档切分、向量化并写入 Chroma。

### Q2：为什么建议 RAG 使用 txt 文档？

PDF 文本提取可能受字体、编码和扫描格式影响，容易出现提取乱码或提取不完整。UTF-8 编码的 `txt` 文档更适合作为知识库来源。

### Q3：为什么 Agent 有时没有调用 RAG？

Agent 没有单独的固定路由模块，而是根据提示词和工具描述进行工具选择。对于“是什么、怎么排查、SOP、故障处理方法”等知识类问题，应通过提示词约束 Agent 优先调用 RAG 工具。

### Q4：当前工具数据为什么是固定的？

当前工具层使用结构化模拟数据验证 Agent 工作流。真实落地时可以将工具内部的数据获取逻辑替换为 Prometheus、Elasticsearch、Loki、CMDB 或告警平台接口。

---

## 12. 可扩展方向

- 接入 Prometheus 获取实时指标
- 接入 Elasticsearch / Loki 获取真实日志
- 接入告警平台进行实时告警分析
- 接入 CMDB 或服务拓扑平台
- 支持多服务联合根因分析
- 支持工单生成和故障升级
- 支持自动化修复建议
- 支持更多 RAG 评估指标和可视化评估报告

---

## 13. License

本项目适合作为 Agent / AIOps / RAG / Tool Calling 学习与实践项目使用。
