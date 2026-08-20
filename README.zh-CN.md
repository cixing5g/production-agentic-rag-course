# AI 项目之母
## 第一阶段：RAG 系统——arXiv 论文助手

<div align="center">
  <h3>面向学习者的生产级 RAG 系统实战课程</h3>
  <p>从零动手实现，学习如何构建现代 AI 系统</p>
  <p>掌握目前需求旺盛的 AI 工程技能：<strong>RAG（检索增强生成）</strong></p>
</div>

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.12+-blue.svg" alt="Python 版本">
  <img src="https://img.shields.io/badge/FastAPI-0.115+-green.svg" alt="FastAPI">
  <img src="https://img.shields.io/badge/OpenSearch-2.19-orange.svg" alt="OpenSearch">
  <img src="https://img.shields.io/badge/Docker-Compose-blue.svg" alt="Docker">
  <img src="https://img.shields.io/badge/Status-Week%207%20Advanced%20Features-brightgreen.svg" alt="状态">
</p>

</br>

<p align="center">
  <a href="#-课程简介">
    <img src="static/mother_of_ai_project_rag_architecture.gif" alt="RAG 架构" width="700">
  </a>
</p>

## 📖 课程简介

这是一套**以学习者为中心的实战课程**。你将从头构建一个完整的科研助手：它可以自动获取学术论文、理解论文内容，并运用 RAG 技术回答研究问题。

在 **arXiv 论文助手**项目中，你将按照业界常用的做法，构建一套**可用于实际项目的 RAG 系统**。很多教程一开始就讲向量搜索，而本课程会先带你打好关键词搜索的基础，再加入向量搜索，完成混合检索。

> **🎯 本课程的不同之处：** 我们采用成熟团队常用的方式来构建 RAG 系统——先做好搜索，再用 AI 增强搜索能力，而不是跳过搜索基础，一开始就完全依赖 AI。

学完课程后，你不仅会拥有一个自己的 AI 科研助手，还会掌握为其他领域构建实用 RAG 系统所需的核心技能。

### **🎓 你将完成什么**

- **第 1 周：** 使用 Docker、FastAPI、PostgreSQL、OpenSearch 和 Airflow 搭建完整的运行环境
- **第 2 周：** 构建自动化数据流程，从 arXiv 获取并解析学术论文
- **第 3 周：** 使用筛选和相关性评分，实现可用于实际项目的 BM25 关键词搜索
- **第 4 周：** 对长文进行合理切分，并将关键词搜索与语义搜索结合起来
- **第 5 周：** 接入本地大语言模型，实现完整的 RAG 流程、流式回答和 Gradio 界面
- **第 6 周：** 使用 Langfuse 跟踪系统运行情况，并用 Redis 缓存提升性能
- **第 7 周：** **使用 LangGraph 构建 Agentic RAG，并接入 Telegram 机器人，方便在移动设备上使用**

---

## 🏗️ 系统架构的演进

### 第 7 周：Agentic RAG 与 Telegram 机器人

<div align="center">
  <img src="static/week7_telegram_and_agentic_ai.png" alt="第 7 周 Telegram 与 Agentic AI 架构" width="800">
  <p><em>第 7 周完整架构：Telegram 机器人与 Agentic RAG 系统的连接方式</em></p>
</div>

### LangGraph Agentic RAG 工作流程

<div align="center">
  <img src="static/langgraph-mermaid.png" alt="LangGraph Agentic RAG 流程" width="800">
  <p><em>LangGraph 的详细工作流程，包括判断、文档相关性检查和检索方法调整</em></p>
</div>

**第 7 周代码讲解与博客：** [使用 LangGraph 和 Telegram 构建 Agentic RAG](https://jamwithai.substack.com/p/agentic-rag-with-langgraph-and-telegram)

**第 7 周的主要功能：**

- **自主判断：** Agent 可以评估搜索结果，并按需要调整检索方法
- **文档筛选：** 自动判断文档与问题是否相关
- **问题改写：** 搜索结果不理想时，自动换一种更合适的问法
- **安全检查：** 识别超出系统知识范围的问题，减少模型编造答案
- **移动端使用：** 通过 Telegram 机器人，在不同设备上与 AI 对话
- **过程可查：** 记录 Agent 的每一步判断，便于排查问题，也让回答更可信

---

## 🚀 快速开始

### **📋 准备工作**

- **Docker Desktop**（包含 Docker Compose）
- **Python 3.12 或更高版本**
- **UV 包管理器**（[安装说明](https://docs.astral.sh/uv/getting-started/installation/)）
- **至少 8GB 内存和 20GB 可用磁盘空间**

### **⚡ 开始使用**

```bash
# 1. 克隆代码并进入项目目录
git clone <repository-url>
cd arxiv-paper-curator

# 2. 准备环境变量（重要）
cp .env.example .env
# .env 文件包含 OpenSearch、arXiv API 和各项服务所需的配置。
# 默认配置可以直接运行。
# 你还需要填写免费的 Jina Embeddings API Key 和 Langfuse Key（详情见博客）。

# 3. 安装依赖
uv sync

# 4. 启动全部服务
docker compose up --build -d

# 5. 检查服务是否正常
curl http://localhost:8000/api/v1/health
```

### **📚 每周学习安排**

| 周次 | 主题 | 博客文章 | 对应代码 |
|------|------|----------|----------|
| **第 0 周** | AI 项目之母：六个阶段 | [AI 项目之母](https://jamwithai.substack.com/p/the-mother-of-ai-project) | - |
| **第 1 周** | 基础环境 | [支撑 RAG 系统运行的基础设施](https://jamwithai.substack.com/p/the-infrastructure-that-powers-rag) | [week1.0](https://github.com/jamwithai/arxiv-paper-curator/releases/tag/week1.0) |
| **第 2 周** | 数据导入流程 | [为 RAG 构建数据导入流程](https://jamwithai.substack.com/p/bringing-your-rag-system-to-life) | [week2.0](https://github.com/jamwithai/arxiv-paper-curator/releases/tag/week2.0) |
| **第 3 周** | 将数据写入 OpenSearch，并使用 BM25 检索 | [每个 RAG 系统都需要的搜索基础](https://jamwithai.substack.com/p/the-search-foundation-every-rag-system) | [week3.0](https://github.com/jamwithai/arxiv-paper-curator/releases/tag/week3.0) |
| **第 4 周** | **文本切分与混合检索** | [让混合检索发挥作用的文本切分方法](https://jamwithai.substack.com/p/chunking-strategies-and-hybrid-rag) | [week4.0](https://github.com/jamwithai/arxiv-paper-curator/releases/tag/week4.0) |
| **第 5 周** | **完整的 RAG 系统** | [完整的 RAG 系统](https://jamwithai.substack.com/p/the-complete-rag-system) | [week5.0](https://github.com/jamwithai/arxiv-paper-curator/releases/tag/week5.0) |
| **第 6 周** | **运行监控与缓存** | [可用于实际项目的 RAG：监控与缓存](https://jamwithai.substack.com/p/production-ready-rag-monitoring-and) | [week6.0](https://github.com/jamwithai/arxiv-paper-curator/releases/tag/week6.0) |
| **第 7 周** | **Agentic RAG 与 Telegram 机器人** | [使用 LangGraph 和 Telegram 构建 Agentic RAG](https://jamwithai.substack.com/p/agentic-rag-with-langgraph-and-telegram) | [week7.0](https://github.com/jamwithai/arxiv-paper-curator/releases/tag/week7.0) |

**📥 获取某一周的代码：**

```bash
# 克隆某一周的代码
git clone --branch <WEEK_TAG> https://github.com/jamwithai/arxiv-paper-curator
cd arxiv-paper-curator
uv sync
docker compose down -v
docker compose up --build -d

# 将 <WEEK_TAG> 换成 week1.0、week2.0 等标签
```

### **📊 服务地址**

| 服务 | 地址 | 用途 |
|------|------|------|
| **API 文档** | http://localhost:8000/docs | 在线查看和测试 API |
| **Gradio RAG 界面** | http://localhost:7861 | 简单易用的聊天界面 |
| **Langfuse 控制台** | http://localhost:3000 | 查看 RAG 各步骤的运行情况 |
| **Airflow 控制台** | http://localhost:8080 | 管理自动化任务 |
| **OpenSearch Dashboards** | http://localhost:5601 | 查看和管理混合检索 |

#### **注意：** Airflow 的用户名和密码请查看 `airflow/simple_auth_manager_passwords.json.generated`

---

## 📚 第 1 周：搭建基础环境 ✅

**从这里开始！** 学习如何搭建现代 RAG 系统所需的运行环境。

### **🎯 学习目标**

- 使用 Docker Compose 搭建完整环境
- 使用 FastAPI 开发接口，自动生成文档并添加健康检查
- 配置和管理 PostgreSQL 数据库
- 配置支持混合检索的 OpenSearch
- 配置 Ollama 本地大语言模型服务
- 统一启动各项服务，并检查它们的运行状态
- 配置代码格式化、类型检查和测试工具

### **🏗️ 架构概览**

<p align="center">
  <img src="static/week1_infra_setup.png" alt="第 1 周基础环境" width="800">
</p>

**主要服务：**

- **FastAPI：** 支持异步处理的 REST API（端口 8000）
- **PostgreSQL 16：** 保存论文信息（端口 5432）
- **OpenSearch 2.19：** 搜索引擎及其管理界面（端口 9200、5601）
- **Apache Airflow 3.0：** 管理自动化任务（端口 8080）
- **Ollama：** 本地大语言模型服务（端口 11434）

### **📓 配置指南**

```bash
# 启动第 1 周的 Notebook
uv run jupyter notebook notebooks/week1/week1_setup.ipynb
```

**学习指南：** 按照[第 1 周 Notebook](notebooks/week1/week1_setup.ipynb) 中的步骤完成配置和检查。

### **📖 延伸阅读**

**博客文章：** [支撑 RAG 系统运行的基础设施](https://jamwithai.substack.com/p/the-infrastructure-that-powers-rag)——包含详细操作说明和实际项目经验

---

## 📚 第 2 周：数据导入流程 ✅

**接着使用第 1 周搭建的环境：** 学习如何自动获取、处理和保存学术论文。

### **🎯 学习目标**

- 接入 arXiv API，并加入访问频率限制和失败重试
- 使用 Docling 解析学术 PDF
- 使用 Apache Airflow 自动执行数据导入任务
- 提取并保存论文信息
- 完成从 API 获取论文到写入数据库的全部步骤

### **🏗️ 架构概览**

<p align="center">
  <img src="static/week2_data_ingestion_flow.png" alt="第 2 周数据导入架构" width="800">
</p>

**数据处理组件：**

- **MetadataFetcher：** 协调完整数据处理流程
- **ArxivClient：** 获取论文，包含访问频率限制和失败重试
- **PDFParserService：** 使用 Docling 处理学术文档
- **Airflow DAG：** 每天自动导入论文
- **PostgreSQL：** 保存整理后的论文信息和正文内容

### **📓 实现指南**

```bash
# 启动第 2 周的 Notebook
uv run jupyter notebook notebooks/week2/week2_arxiv_integration.ipynb
```

**学习指南：** 按照[第 2 周 Notebook](notebooks/week2/week2_arxiv_integration.ipynb) 中的步骤完成开发和检查。

### **📖 延伸阅读**

**博客文章：** [为 RAG 构建数据导入流程](https://jamwithai.substack.com/p/bringing-your-rag-system-to-life)——介绍 arXiv API 接入和 PDF 处理

---

## 📚 第 3 周：先做好关键词搜索

**接着使用前两周完成的系统：** 实现 RAG 系统经常使用的关键词搜索。

### **🎯 学习目标**

- 理解关键词搜索对 RAG 系统的重要性
- 管理和优化 OpenSearch 索引及搜索配置
- 理解 BM25 算法及其相关数学原理
- 使用 Query DSL 编写包含筛选和加权的复杂搜索条件
- 用精确率、召回率等指标衡量搜索效果和性能
- 学习真实项目中常用的实现方式

### **🏗️ 架构概览**

<p align="center">
  <img src="static/week3_opensearch_flow.png" alt="第 3 周 OpenSearch 流程架构" width="800">
</p>

**搜索相关组件：**

- **OpenSearch 服务：** `src/services/opensearch/`——搜索功能的主要代码
- **搜索 API：** `src/routers/search.py`——使用 BM25 评分的搜索接口
- **学习资料：** `notebooks/week3/`——完整的 OpenSearch 接入指南
- **质量指标：** 精确率、召回率和相关性评分

### **📓 配置指南**

```bash
# 启动第 3 周的 Notebook
uv run jupyter notebook notebooks/week3/week3_opensearch.ipynb
```

**学习指南：** 按照[第 3 周 Notebook](notebooks/week3/week3_opensearch.ipynb) 中的步骤配置 OpenSearch 并实现 BM25 搜索。

### **📖 延伸阅读**

**博客文章：** [每个 RAG 系统都需要的搜索基础](https://jamwithai.substack.com/p/the-search-foundation-every-rag-system)——使用 OpenSearch 完整实现 BM25

---

## 📚 第 4 周：文本切分与混合检索

**接着使用第 3 周完成的关键词搜索：** 加入语义搜索，让系统不只匹配相同的词，也能理解意思相近的表达。

### **🎯 学习目标**

- 按文章章节合理切分文档
- 接入 Jina AI 生成向量，并在服务不可用时使用备用方案
- 使用 RRF 合并关键词搜索和语义搜索的结果
- 用一个 API 支持多种搜索方式
- 比较不同搜索方法的速度、效果和适用场景

### **🏗️ 架构概览**

<p align="center">
  <img src="static/week4_hybrid_opensearch.png" alt="第 4 周混合检索架构" width="800">
</p>

**混合检索相关组件：**

- **文本切分工具：** `src/services/indexing/text_chunker.py`——按章节切分文本，并保留适量的上下文重叠
- **向量服务：** `src/services/embeddings/`——通过 Jina AI 生成向量
- **混合检索 API：** `src/routers/hybrid_search.py`——用同一个 API 支持各种搜索方式
- **学习资料：** `notebooks/week4/`——完整的混合检索实现指南

### **📓 配置指南**

```bash
# 启动第 4 周的 Notebook
uv run jupyter notebook notebooks/week4/week4_hybrid_search.ipynb
```

**学习指南：** 按照[第 4 周 Notebook](notebooks/week4/week4_hybrid_search.ipynb) 中的步骤完成开发和检查。

### **📖 延伸阅读**

**博客文章：** [让混合检索发挥作用的文本切分方法](https://jamwithai.substack.com/p/chunking-strategies-and-hybrid-rag)——介绍适合实际项目的文本切分方法和 RRF 结果合并方法

---

## 📚 第 5 周：接入大语言模型，完成 RAG 系统

**接着使用第 4 周完成的混合检索：** 接入大语言模型，把搜索系统变成可以连续对话的智能助手。

### **🎯 学习目标**

- 使用 Ollama 接入本地大语言模型，保护数据隐私
- 将提示词长度减少 80%，让回答速度提升约 6 倍
- 使用 Server-Sent Events 实时返回生成内容
- 同时提供普通接口和流式接口
- 使用 Gradio 创建交互界面，并提供常用参数设置

### **🏗️ 架构概览**

<p align="center">
  <img src="static/week5_complete_rag.png" alt="第 5 周完整 RAG 系统架构" width="900">
</p>

**完整 RAG 系统的主要组件：**

- **RAG 接口：** `src/routers/ask.py`——两个接口：`/api/v1/ask` 和 `/api/v1/stream`
- **Ollama 服务：** `src/services/ollama/`——大语言模型客户端和优化后的提示词
- **系统提示词：** `src/services/ollama/prompts/rag_system.txt`——为学术论文问答优化
- **Gradio 界面：** `src/gradio_app.py`——支持流式回答的网页界面
- **启动脚本：** `gradio_launcher.py`——用于快速启动 Gradio，运行在 7861 端口

### **📓 配置指南**

```bash
# 启动第 5 周的 Notebook
uv run jupyter notebook notebooks/week5/week5_complete_rag_system.ipynb

# 启动 Gradio 界面
uv run python gradio_launcher.py
# 打开 http://localhost:7861
```

**学习指南：** 按照[第 5 周 Notebook](notebooks/week5/week5_complete_rag_system.ipynb) 中的步骤接入大语言模型并完成 RAG 流程。

### **📖 延伸阅读**

**博客文章：** [完整的 RAG 系统](https://jamwithai.substack.com/p/the-complete-rag-system)——介绍如何接入本地大语言模型，以及如何提升系统性能

---

## 📚 第 6 周：运行监控与缓存

**接着使用第 5 周完成的 RAG 系统：** 加入运行记录、性能优化和监控功能，让系统更适合实际使用。

### **🎯 学习目标**

- 接入 Langfuse，记录和查看 RAG 的完整处理过程
- 使用 Redis 缓存，并设置合理的缓存键和过期时间
- 通过实时控制台查看响应时间和费用
- 学习实际项目常用的监控与优化方法
- 分析费用并减少大语言模型调用；命中缓存时可提速 150～400 倍

### **🏗️ 架构概览**

<p align="center">
  <img src="static/week6_monitoring_and_caching.png" alt="第 6 周监控与缓存架构" width="900">
</p>

**监控与缓存相关组件：**

- **Langfuse 服务：** `src/services/langfuse/`——记录 RAG 处理过程和相关指标
- **缓存服务：** `src/services/cache/`——使用 Redis 保存完全相同问题的回答；Redis 不可用时，系统仍可继续运行
- **更新后的接口：** `src/routers/ask.py`——已经接入运行记录和缓存
- **Docker 配置：** `docker-compose.yml`——新增 Redis 服务和本地 Langfuse 服务
- **学习资料：** `notebooks/week6/`——完整的监控与缓存实现指南

### **📓 配置指南**

```bash
# 启动第 6 周的 Notebook
uv run jupyter notebook notebooks/week6/week6_cache_testing.ipynb
```

**学习指南：** 按照[第 6 周 Notebook](notebooks/week6/week6_cache_testing.ipynb) 中的步骤接入 Langfuse 和 Redis 缓存。

### **📖 延伸阅读**

**博客文章：** [可用于实际项目的 RAG：监控与缓存](https://jamwithai.substack.com/p/production-ready-rag-monitoring-and)——介绍 RAG 系统的监控与缓存

---

## 📚 第 7 周：使用 LangGraph 和 Telegram 构建 Agentic RAG

**接着使用第 6 周完成的系统：** 让 Agent 能分步骤判断和调整检索方法，并接入 Telegram 机器人，方便在手机上使用。

### **🎯 学习目标**

- 使用 LangGraph 按当前处理状态安排 Agent 的下一步操作
- 检查用户问题是否有效、是否属于系统支持的范围
- 判断搜索到的文档与问题是否相关
- 在搜索结果不理想时自动改写问题
- 多次尝试不同的检索方法，并在需要时采用备用方案
- 接入 Telegram 机器人，处理异步操作和错误
- 展示 Agent 的判断过程，让系统行为更容易理解

### **🏗️ 架构概览**

<p align="center">
  <img src="static/week7_telegram_and_agentic_ai.png" alt="第 7 周 Agentic RAG 与 Telegram 架构" width="900">
</p>

**Agentic RAG 相关组件：**

- **Agent 步骤：** `src/services/agents/nodes/`——负责检查问题、检索、判断文档、改写问题和生成回答
- **工作流程：** `src/services/agents/agentic_rag.py`——使用 LangGraph 组织各个处理步骤
- **Telegram 机器人：** `src/services/telegram/`——处理命令和消息
- **Agentic RAG 接口：** `src/routers/agentic_ask.py`——Agentic RAG API
- **学习资料：** `notebooks/week7/`——第 7 周的学习资料和示例

### **📓 配置指南**

```bash
# 启动第 7 周的 Notebook
uv run jupyter notebook notebooks/week7/week7_agentic_rag.ipynb
```

**学习指南：** 按照[第 7 周 Notebook](notebooks/week7/week7_agentic_rag.ipynb) 中的步骤，使用 LangGraph 实现 Agentic RAG 并接入 Telegram 机器人。

### **📖 延伸阅读**

**博客文章：** [使用 LangGraph 和 Telegram 构建 Agentic RAG](https://jamwithai.substack.com/p/agentic-rag-with-langgraph-and-telegram)——介绍如何让 Agent 自主判断、调整检索方法，并通过移动设备使用

---

## ⚙️ 配置

**准备配置文件：**

```bash
cp .env.example .env
# 根据你的运行环境编辑 .env
```

**主要环境变量：**

- `JINA_API_KEY`——第 4 周及之后的内容需要，用于生成向量和混合检索
- `TELEGRAM__BOT_TOKEN`——第 7 周需要，用于接入 Telegram 机器人
- `LANGFUSE__PUBLIC_KEY` 和 `LANGFUSE__SECRET_KEY`——第 6 周可选，用于监控

**完整配置：** 所有可用选项和详细说明请查看 [.env.example](.env.example)。

---

## 🔧 开发参考

### **🛠️ 技术栈**

| 服务 | 用途 | 状态 |
|------|------|------|
| **FastAPI** | REST API 和自动生成的接口文档 | ✅ 可用 |
| **PostgreSQL 16** | 保存论文信息和正文 | ✅ 可用 |
| **OpenSearch 2.19** | 混合检索（BM25 + 向量） | ✅ 可用 |
| **Apache Airflow 3.0** | 自动执行数据处理任务 | ✅ 可用 |
| **Jina AI** | 生成向量（第 4 周） | ✅ 可用 |
| **Ollama** | 运行本地大语言模型（第 5 周） | ✅ 可用 |
| **Redis** | 高速缓存（第 6 周） | ✅ 可用 |
| **Langfuse** | 查看 RAG 各步骤的运行情况（第 6 周） | ✅ 可用 |

**开发工具：** UV、Ruff、MyPy、Pytest、Docker Compose

### **🏗️ 项目目录**

```text
arxiv-paper-curator/
├── src/                    # 应用主代码
│   ├── routers/            # API 接口（搜索、问答、论文）
│   ├── services/           # 主要功能（OpenSearch、Ollama、Agent、缓存）
│   ├── models/             # 数据库模型（SQLAlchemy）
│   ├── schemas/            # Pydantic 数据校验模型
│   └── config.py           # 环境配置
├── notebooks/              # 每周学习资料（第 1～7 周）
├── airflow/                # 自动化任务（DAG）
├── tests/                  # 测试代码
└── compose.yml             # Docker 服务配置
```

### **📡 API 接口**

| 接口 | 方法 | 说明 | 周次 |
|------|------|------|------|
| `/health` | GET | 检查服务是否正常 | 第 1 周 |
| `/api/v1/papers` | GET | 列出已保存的论文 | 第 2 周 |
| `/api/v1/papers/{id}` | GET | 获取指定论文 | 第 2 周 |
| `/api/v1/search` | POST | BM25 关键词搜索 | 第 3 周 |
| `/api/v1/hybrid-search/` | POST | 混合检索（BM25 + 向量） | **第 4 周** |

**API 文档：** 打开 [http://localhost:8000/docs](http://localhost:8000/docs)，可在线查看和测试接口。

### **🔧 常用命令**

#### **使用 Makefile（推荐）**

```bash
# 查看所有可用命令
make help

# 常用操作
make start         # 启动全部服务
make health        # 检查全部服务是否正常
make test          # 运行测试
make stop          # 停止服务
```

#### **全部 Make 命令**

| 命令 | 说明 |
|------|------|
| `make start` | 启动全部服务 |
| `make stop` | 停止全部服务 |
| `make restart` | 重启全部服务 |
| `make status` | 查看服务状态 |
| `make logs` | 查看服务日志 |
| `make health` | 检查全部服务是否正常 |
| `make setup` | 安装 Python 依赖 |
| `make format` | 格式化代码 |
| `make lint` | 检查代码格式和类型 |
| `make test` | 运行测试 |
| `make test-cov` | 运行测试并统计覆盖率 |
| `make clean` | 清理运行环境和生成的文件 |

#### **直接运行命令**

```bash
# 如果你更喜欢直接使用 Docker 和 UV 命令
docker compose up --build -d    # 启动服务
docker compose ps               # 查看状态
docker compose logs             # 查看日志
uv run pytest                   # 运行测试
```

### **🎓 适合谁学习**

| 人群 | 可以学到什么 |
|------|--------------|
| **AI/机器学习工程师** | 学习比入门教程更完整、更适合实际项目的 RAG 架构 |
| **软件工程师** | 使用常用工程方法，完整构建一款 AI 应用 |
| **数据科学家** | 使用现代工具实现可实际运行的 AI 系统 |

---

## 🛠️ 常见问题

**服务无法启动？**

- 等待 2～3 分钟，然后运行 `docker compose logs` 查看日志
- 如果端口被占用，请停止占用 8000、8080、5432 或 9200 端口的其他服务
- 如果内存不足，请在 Docker Desktop 中增加可用内存

**需要帮助？**

- 查看第 1 周 Notebook 中的详细排错说明
- 运行 `docker compose logs [service-name]` 查看指定服务的日志
- 如需完全重新启动环境，运行 `docker compose down --volumes && docker compose up --build -d`

---

## 💰 费用

**本课程完全免费！** 只有选择使用外部服务时，才可能产生少量费用：

- **本地运行：** 0 美元，所有服务都在本地运行
- **可选的云 API：** 如果使用外部大语言模型服务，预计约 2～5 美元

---

<div align="center">
  <h3>🎉 准备好开始 AI 工程学习之旅了吗？</h3>
  <p><strong>从第 1 周的配置 Notebook 开始，动手构建你的第一个实用 RAG 系统！</strong></p>

  <p><em>献给希望掌握现代 AI 工程的学习者</em></p>
  <p><strong>由 <a href="https://www.linkedin.com/in/shirin-khosravi-jam/">Shirin Khosravi Jam</a> 和 <a href="https://www.linkedin.com/in/shantanuladhwe/">Shantanu Ladhwe</a> 用心制作</strong></p>
</div>

---

## Star 数量变化

[![Star 数量变化图](https://api.star-history.com/svg?repos=jamwithai/production-agentic-rag-course&type=Date)](https://star-history.com/#jamwithai/production-agentic-rag-course&Date)

---

## 📄 许可证

本项目采用 MIT 许可证，详情请查看 [LICENSE](LICENSE) 文件。
