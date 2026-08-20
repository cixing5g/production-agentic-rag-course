# 第 7 周：Agentic RAG 与 LangGraph + Telegram Bot

＃＃ 概述

第 7 周为 arXiv Paper Curator 添加了两项主要增强功能：

1. **🤖 Agentic RAG with LangGraph** - 智能、自适应检索和决策
2. **💬 Telegram Bot 集成** - 用于移动/桌面访问的对话界面

---

## 🧠 第 1 部分：使用 LangGraph 的 Agentic RAG

### 什么是 Agentic RAG？

**传统 RAG**（第 5-6 周）：
```
Query → Always Retrieve → Generate Answer
```

**Agentic RAG**（第 7 周）：
```
Query → Agent Decides:
  ├─ Simple question? → Respond directly (faster!)
  └─ Research needed? → Retrieve
       ├─ Relevant docs? → Generate answer
       └─ Not relevant? → Rewrite query → Try again
```

### 主要特点

- **🎯 智能决策** - LLM 决定何时实际需要检索
- **📊 文档评分** - 验证检索到的论文是否相关
- **🔄 查询细化** - 重写模糊查询以获得更好的结果
- **🔍推理透明度** - 显示智能体的决策步骤
- **♻️ 迭代改进** - 如果需要，可以重试更好的查询

### 我们建造了什么

```
src/services/agents/
├── tools.py            # Retriever tool wrapping OpenSearch
├── nodes.py            # 4 graph nodes (query, grade, rewrite, generate)
├── agentic_rag.py      # LangGraph workflow + service
├── prompts.py          # LLM prompt templates
└── factory.py          # Dependency injection

src/routers/
└── agentic_ask.py      # FastAPI endpoint

Total: ~750 LOC following SOLID, KISS, DRY principles
```

＃＃＃ 建筑学

```
LangGraph Workflow:

START
  ↓
generate_query_or_respond
  ├─ No retrieval needed → END (direct response)
  └─ Needs retrieval → retrieve (ToolNode)
       ↓
     grade_documents
       ├─ Relevant → generate_answer → END
       └─ Not relevant → rewrite_query → (loop back)
```

### 新的 API 端点

**`POST /api/v1/ask-agentic`**

```json
// Request
{
  "query": "What are transformers in ML?",
  "top_k": 3,
  "use_hybrid": true
}

// Response
{
  "query": "What are transformers in ML?",
  "answer": "Transformers are neural network architectures...",
  "sources": ["https://arxiv.org/pdf/1706.03762.pdf"],
  "chunks_used": 3,
  "search_mode": "hybrid",
  "reasoning_steps": [
    "Decided to retrieve relevant papers",
    "Retrieved documents from database",
    "Generated answer from relevant documents"
  ],
  "retrieval_attempts": 1
}
```

### 快速入门：Agentic RAG

**1.确保服务正在运行：**
```bash
docker compose up --build -d
```

**2.使用 cURL 进行测试：**
```bash
# Simple question (should respond directly)
curl -X POST http://localhost:8000/api/v1/ask-agentic \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is 2+2?",
    "top_k": 3,
    "use_hybrid": true
  }'

# Research question (should retrieve papers)
curl -X POST http://localhost:8000/api/v1/ask-agentic \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are attention mechanisms?",
    "top_k": 3,
    "use_hybrid": true
  }'
```

**3.交互式测试：**
```bash
# Open Jupyter notebook
jupyter notebook notebooks/week7/week7_agentic_rag.ipynb
```

### 比较：传统与代理

|特色 |传统 RAG |特工 RAG |
|--------|----------------|-------------|
| **检索** |总是检索 |需要时决定 |
| **相关性检查** |无 |成绩文件 |
| **查询细化** |无 |如果需要的话重写 |
| **迭代** |单程 |多次尝试 |
| **透明度** |黑匣子|展示推理|
| **简单问题** | ~15-20 秒 | ~2-5s（无检索）|
| **复杂问题** |单次尝试 |迭代细化 |

### 测试场景

**场景 1：直接响应（无检索）**
- 查询：“5 + 7 是什么？”
- 预期：特工响应“12”而不检索文件
- 推理：“直接回复，无需检索”

**场景2：成功检索**
- 查询：“机器学习中的 Transformer 是什么？”
- 预期：代理检索论文、相关评分、生成答案
- 推理：“决定检索”→“检索到的文档”→“生成答案”

**场景 3：查询重写**
- 查询：“告诉我有关 ML 的事情”（含糊）
- 预期：代理检索、评级为不相关、重写查询、重试
- 推理：“检索”→“不相关”→“重写查询”→“再次检索”→“生成答案”

### 遵循的设计原则

- ✅ **SOLID** - 单一责任、依赖倒置、组合
- ✅ **KISS** - 节点简单（<30行），逻辑清晰
- ✅ **DRY** - 重用现有服务（OpenSearch、Ollama、Jina）
- ✅ **YAGNI** - 只实现了需要的内容
- ✅ **显式** - 类型提示、文档字符串、清晰的名称
- ✅ **2025 最佳实践** - MessagesState、ToolNode、tools_condition

### 文档

- **实施计划**：`docs/AGENTIC_RAG_IMPLEMENTATION_PLAN.md`
- **测试计划**：`docs/AGENTIC_RAG_TESTING_PLAN.md`
- **LangGraph 2025 模式**：`docs/LANGGRAPH_2025_BEST_PRACTICES.md`
- **设计原理**：`docs/DESIGN_PRINCIPLES.md`
- **交互式笔记本**：`notebooks/week7/week7_agentic_rag.ipynb`

---

## 💬 第 2 部分：Telegram 机器人集成

### 我们建造了什么

- **🤖 完整的 Telegram Bot 集成**：具有命令支持的对话界面
- **💬自然语言查询**：用简单的语言提出问题，通过来源获得答案
- **⚡ 所有第 6 周的功能**：Redis 缓存（加速 150-400 倍）和 Langfuse 跟踪
- **🎯交互命令**：`/start`、`/help`、`/ask`、`/search`、`/settings`、`/status`
- **👤 用户会话管理**：每个用户的偏好和对话历史记录
- **📱 移动优先**：具有可点击 arXiv 链接的丰富消息格式
- **🔐可选访问控制**：如果需要，将特定用户列入白名单

＃＃ 建筑学

### 数据流
```
Telegram User
    ↓
Telegram Bot (Polling/Webhook)
    ↓
TelegramService + Handlers
    ↓ [Langfuse Tracing]
Cache Check (Redis)
    ├─ Hit → Instant Response (~100ms)
    └─ Miss → Full RAG Pipeline
        ↓
Hybrid Search (OpenSearch BM25 + Vector)
        ↓
LLM Generation (Ollama)
        ↓
Cache Store (Redis)
        ↓
Format Response → Send to Telegram
```

### 新组件

```
src/services/telegram/
├── client.py           # Main Telegram bot service
├── handlers.py         # Command and message handlers
├── formatters.py       # Message formatting utilities
├── keyboards.py        # Interactive inline keyboards
├── user_manager.py     # User settings and sessions
└── factory.py          # Factory function

src/schemas/telegram/
├── messages.py         # Message validation schemas
├── commands.py         # Command schemas
└── user_settings.py    # User preferences schema
```

## 主要特点

### **对话界面**
- 自然语言查询：只需发送消息
- 具有历史记录跟踪的上下文对话
- 自动查询路由（命令与常规消息）

### **丰富的命令**
```
/start    - Welcome message with bot capabilities
/help     - Detailed usage instructions
/ask      - Explicitly ask a research question
/search   - Quick paper search by keywords
/settings - Customize preferences (search mode, results count)
/status   - Check system health and statistics
/clear    - Clear conversation history
```

### **交互设置**
- 搜索模式：混合（BM25 + 语义）或仅 BM25
- 每个查询的结果：3、5 或 10 篇论文
- 类别过滤器：全部、cs.AI、cs.LG、cs.CL 等。
- 模型选择：选择LLM模型
- 切换选项：流式响应、来源显示、紧凑模式

### **用户体验**
- **在处理过程中键入指标**
- **丰富的格式**，支持 Markdown
- arXiv 论文的**可点击链接**
- **针对长响应的自动消息分割**
- **来源引文**以及论文元数据
- **缓存指示器** (⚡) 用于即时响应

## 快速入门

### 先决条件

1. **Telegram 帐户** - 在您的手机或电脑上安装 Telegram
2. **第 1-6 周服务全部运行** - 完整的 RAG 堆栈必须可运行

### 第 1 步：创建您的 Telegram 机器人

1. **打开 Telegram** 并搜索 `@BotFather`
2. **发送** `/newbot` 到 BotFather
3. **按照提示操作**：
   - 选择一个名称（例如“我的 arXiv Curator”）
   - 选择一个用户名（例如，“my_arxiv_curator_bot” - 必须以“bot”结尾）
4. **复制机器人令牌** - 您将收到类似以下内容的内容：
   ````
   1234567890：ABCdefGHIjklMNOpqrsTUVwxyz-1234567
   ````

### 步骤2：配置环境

将这些变量添加到您的 `.env` 文件中：

```bash
# Enable Telegram bot
TELEGRAM__ENABLED=true
TELEGRAM__BOT_TOKEN=your_token_from_botfather_here

# Optional: Restrict to specific users (comma-separated Telegram user IDs)
# Leave empty to allow all users
TELEGRAM__ALLOWED_USER_IDS=

# Use polling mode for development (webhook requires HTTPS)
TELEGRAM__USE_WEBHOOK=false
```

### 步骤 3：安装依赖项

```bash
# Install python-telegram-bot library
uv sync
```

### 第 4 步：启动服务

```bash
# Start all services (includes Telegram bot)
docker compose up --build -d

# Check logs to verify Telegram bot started
docker compose logs -f api
```

你应该看到：
```
INFO - Telegram bot started successfully
INFO - Starting Telegram bot in polling mode
INFO - Bot commands set successfully
```

### 第 5 步：测试您的机器人

1. **打开 Telegram** 并搜索您的机器人用户名
2. **发送** `/start` 到您的机器人
3. **尝试问**：“机器学习中的 Transformer 是什么？”
4. **验证**您收到带有来源的答案！

## 测试说明

### 手动测试场景

#### 场景 1：基本命令

**测试`/start`命令：**
```
You: /start
Bot: 👋 Welcome to arXiv Paper Curator!
     [Shows capabilities and quick commands]
```

**测试`/help`命令：**
```
You: /help
Bot: 📚 arXiv Paper Curator Help
     [Shows detailed command documentation]
```

**测试`/status`命令：**
```
You: /status
Bot: 🔧 System Status
     ✅ OPENSEARCH
     ✅ OLLAMA
     ✅ CACHE
     [Shows system health]
```

#### 场景 2：RAG 问答

**测试简单问题：**
```
You: What are attention mechanisms?
Bot: [15-20s first time]
     *Answer:*
     Attention mechanisms allow models to dynamically focus on...

     📚 *Sources:*
     [1] *Attention Is All You Need*
         🔗 Read on arXiv - 1706.03762
         📊 Score: 12.456

     [2] *Neural Machine Translation...*
         🔗 Read on arXiv - 1409.0473
         📊 Score: 11.234

     ⚙️ Mode: hybrid
```

**测试缓存查询：**
```
You: What are attention mechanisms?
Bot: [~100ms second time ⚡]
     [Same answer as above]
     ⚙️ Mode: hybrid ⚡ Cached
```

#### 场景 3：搜索命令

**试卷搜索：**
```
You: /search transformer neural networks
Bot: 📖 Found 145 papers (showing top 10)

     1. *Attention Is All You Need*
        🔗 Read on arXiv - 1706.03762
        📊 Score: 12.456

     2. *BERT: Pre-training of Deep Bidirectional...*
        🔗 Read on arXiv - 1810.04805
        📊 Score: 11.234
     [...]
```

#### 场景 4：设置自定义

**测试设置命令：**
```
You: /settings
Bot: ⚙️ Your Settings

     *Search Mode:* HYBRID
     *Results per query:* 3 papers
     *Model:* llama3.2:1b
     *Categories:* All

     [Interactive buttons appear]:
     [🔍 Hybrid Search] [⚡ BM25 Only]
     [3 Results] [5 Results] [10 Results]
     [All Categories] [cs.AI] [cs.LG]
```

**测试更改设置：**
```
You: [Click "5 Results" button]
Bot: ✅ Results per query: 5

You: [Click "cs.AI" button]
Bot: ✅ Category filter: cs.AI
```

#### 场景 5：自然对话

**测试多轮对话：**
```
You: Tell me about BERT
Bot: [Provides answer about BERT with sources]

You: How does it differ from GPT?
Bot: [Answers about BERT vs GPT differences]

You: /clear
Bot: 🗑️ Conversation history cleared!
     Your settings have been preserved.
```

#### 场景 6：错误处理

**测试无效查询：**
```
You: asdfghjkl
Bot: ❌ No relevant papers found.
     Try different keywords or check your category filters.
```

**服务关闭时测试：**
```
You: test query
Bot: ❌ Error processing your question:
     [User-friendly error message]
```

### 验证清单

- [ ] **机器人响应 `/start`** - 显示欢迎消息
- [ ] **机器人响应所有命令** - `/help`、`/ask`、`/search`、`/settings`、`/status`、`/clear`
- [ ] **自然语言查询有效** - 无需命令即可提出问题
- [ ] **RAG 答案包括来源** - 通过 arXiv 链接引用的论文
- [ ] **缓存有效** - 重复查询立即返回（⚡指示器）
- [ ] **设置保留** - 对话中的更改仍然存在
- [ ] **交互式按钮有效** - 可以单击内联键盘按钮
- [ ] **长响应正确分割** - 消息不超过 Telegram 限制
- [ ] **Markdown 格式有效** - 粗体、斜体、链接正确呈现
- [ ] **Langfuse 显示痕迹** - 检查 http://localhost:3000 的 Telegram 事件
- [ ] **正在键入指示器显示** - 处理过程中出现“机器人正在键入...”
- [ ] **错误消息是友好的** - 没有向用户公开堆栈跟踪

### 性能测试

**缓存性能：**
```bash
# First query (cache miss)
You: What is machine learning?
Bot: [Response in ~15-20 seconds]

# Identical query (cache hit)
You: What is machine learning?
Bot: [Response in ~100ms ⚡]

# Verify 150-400x speedup!
```

**并发用户数：**
```bash
# Test with multiple Telegram accounts
# Each user should have independent settings and sessions
```

**会话管理：**
```bash
# Stop using bot for 30 minutes (default timeout)
# Verify session cleanup happens automatically
```

＃＃ 配置

### 环境变量

```bash
# Enable/Disable Bot
TELEGRAM__ENABLED=true  # Set to false to disable

# Bot Token (Required)
TELEGRAM__BOT_TOKEN=your_token_here

# Deployment Mode
TELEGRAM__USE_WEBHOOK=false  # true for production
TELEGRAM__WEBHOOK_URL=https://your-domain.com
TELEGRAM__WEBHOOK_PATH=/telegram/webhook

# Access Control (Optional)
TELEGRAM__ALLOWED_USER_IDS=123456789,987654321  # Empty = allow all

# Behavior Settings
TELEGRAM__MAX_MESSAGE_LENGTH=4000  # Telegram limit is 4096
TELEGRAM__ENABLE_STREAMING=true
TELEGRAM__SESSION_TIMEOUT_MINUTES=30
TELEGRAM__RATE_LIMIT_MESSAGES_PER_MINUTE=20

# Default User Preferences
TELEGRAM__DEFAULT_TOP_K=3
TELEGRAM__DEFAULT_USE_HYBRID=true
TELEGRAM__DEFAULT_MODEL=llama3.2:1b
```

### 用户设置（每个用户可自定义）

每个用户都可以定制：
- **搜索模式**：混合（BM25 + 语义）或仅限 BM25
- **结果计数**：每个查询 3、5 或 10 篇论文
- **类别**：全部、cs.AI、cs.LG、cs.CL、cs.CV、cs.NE
- **模型**：用于答案生成的LLM模型
- **显示**：显示源、紧凑模式、流式传输

设置跨会话保留并存储在内存中（对于生产，请考虑 Redis/PostgreSQL 存储）。

## 架构细节

### 服务集成

Telegram 机器人与所有现有的第 1-6 周服务集成：

1. **OpenSearch** - 相关论文的混合搜索
2. **Jina Embeddings** - 语义搜索功能
3. **Ollama LLM** - 答案生成
4. **Redis 缓存** - 重复查询加速 150-400 倍
5. **Langfuse** - 完整追踪 Telegram 交互
6. **PostgreSQL** - 论文元数据（通过 OpenSearch）

### 消息流

```python
# Simplified message flow
async def handle_message(update, context):
    1. Extract user_id, chat_id, text
    2. Check rate limits
    3. Get/create user settings
    4. Show "typing..." indicator
    5. Check cache (Redis)
    6. If cache hit → format and send
    7. If cache miss:
        a. Generate embedding (Jina)
        b. Search papers (OpenSearch)
        c. Build prompt with context
        d. Generate answer (Ollama)
        e. Cache result (Redis)
        f. Trace interaction (Langfuse)
    8. Format response (Markdown)
    9. Split if too long
    10. Send to Telegram
    11. Update conversation history
```

### 错误处理

该机器人实现了优雅的降级：

- **Markdown 格式失败** → 退回到纯文本
- **嵌入生成失败** → 退回到 BM25 搜索
- **缓存不可用** → 不缓存而继续
- **Langfuse 跟踪失败** → 记录警告，继续
- **服务错误** → 用户友好的错误消息

### 速率限制

内置速率限制可防止滥用：
- **每用户限制**：20 条消息/分钟（可配置）
- **自动节流**：由 Telegram 库强制执行
- **会话超时**：30 分钟不活动

## 故障排除

|问题 |解决方案 |
|--------|----------|
| **机器人没有响应** |检查 `TELEGRAM__ENABLED=true` 和有效的 `BOT_TOKEN` |
| **“未经授权”错误** | Bot 令牌无效，请使用 @BotFather 重新生成 |
| **机器人有响应但没有答案** |检查 OpenSearch、Ollama 和嵌入服务 |
| **反应慢** |第一个查询总是慢，后续查询使用缓存 |
| **Markdown 格式损坏** |机器人自动回退到纯文本 |
| **找不到机器人** |确保机器人用户名正确，以“bot”结尾|
| **“禁止”错误** |检查 `ALLOWED_USER_IDS` 中的 user_id 限制 |
| **内存问题** |用户会话存储在内存中，监控 RAM 使用情况 |

### 调试模式

启用详细日志记录：

```bash
# In .env
DEBUG=true

# Check bot logs
docker compose logs -f api | grep telegram
```

### 健康检查

```bash
# Check if bot is running
curl http://localhost:8000/api/v1/health

# Should show telegram_service status
```

### 常见修复

**机器人未启动：**
```bash
# 1. Check token is valid
# 2. Restart API service
docker compose restart api

# 3. Check logs for errors
docker compose logs api
```

**响应太慢：**
```bash
# 1. Check cache is working
docker exec rag-redis redis-cli ping  # Should return PONG

# 2. Check cache hit rate
# Look for ⚡ indicator in bot responses

# 3. Verify Langfuse tracing not blocking
LANGFUSE__ENABLED=false  # Temporarily disable
```

## 性能基准

|公制|价值|笔记|
|--------|--------|--------|
| **第一次查询** | 15-20 岁 |完整的 RAG 管道执行 |
| **缓存查询** | 50-100 毫秒 | **快 150-400 倍** |
| **打字指示器** | <500 毫秒 |立即显示 |
| **仅搜索** | 2-3秒| `/search` 命令 |
| **状态检查** | <1秒| `/status` 命令 |
| **设置更新** | <100 毫秒 |即时按钮响应 |
| **并发用户** | 10+ |同时测试|

### 缓存命中率（预期）

- **重复精确查询**：100%命中率
- **热门问题**：命中率60-80%
- **唯一查询**：0% 命中率（第一次）

### 资源使用情况

- **内存**：每个活动用户会话约 50MB
- **CPU**：最小（<5% 空闲，生成期间出现峰值）
- **网络**：每条消息约 10KB（不包括 LLM 生成）

## 生产部署

### Webhook 模式（推荐用于生产）

```bash
# .env for production
TELEGRAM__USE_WEBHOOK=true
TELEGRAM__WEBHOOK_URL=https://your-domain.com
TELEGRAM__WEBHOOK_PATH=/telegram/webhook

# Requires:
# - HTTPS domain with valid certificate
# - Nginx/Caddy for TLS termination
# - Public IP or reverse proxy
```

### 安全最佳实践

1. **限制用户**：使用`ALLOWED_USER_IDS`作为私人机器人
2. **速率限制**：保留默认限制（20 条消息/分钟）
3. **环境变量**：切勿提交包含真实 Token 的 `.env`
4. **仅 HTTPS**：在生产中使用 webhook 模式
5. **监控**：通过 Langfuse 仪表板跟踪使用情况

### 扩展考虑因素

对于高流量部署：

1. **用户存储**：从内存迁移到Redis/PostgreSQL
2. **Cache**：增加Redis内存分配
3. **负载均衡**：多个API实例（webhook模式）
4. **速率限制**：根据您的要求进行调整
5. **监控**：设置错误和延迟警报

## 后续步骤

### 可选增强功能（7.1 周以上）

- **📸 图像支持**：上传纸质 PDF，获取摘要
- **🗣️语音消息**：通过语音提问（语音转文本）
- **📊 用户分析**：使用模式仪表板
- **🤝 群聊**：多用户讨论
- **🌍多语言**：国际化支持
- **🔔通知**：类别中新论文的推送提醒
- **📈个性化**：基于机器学习的论文推荐
- **🔗语义缓存**：相似查询的模糊匹配

### 整合想法

- **Slack Bot**：移植到具有相同架构的 Slack
- **Discord Bot**：扩展到 Discord 社区
- **WhatsApp 机器人**：使用 WhatsApp Business API
- **Web Widget**：嵌入网站
- **API 访问**：公开 RESTful API 以进行集成

＃＃ 资源

- **Telegram Bot API**：https://core.telegram.org/bots/api
- **python-telegram-bot**：https://python-telegram-bot.org
- **@BotFather**：https://t.me/botfather
- **Langfuse 文档**：https://langfuse.com/docs
- **Redis 文档**：https://redis.io/docs

## 代码结构

```python
# Entry point: src/main.py
telegram_service = make_telegram_service(...)
await telegram_service.start()

# Service: src/services/telegram/client.py
class TelegramService:
    async def start() -> Start bot in polling/webhook mode
    async def stop() -> Stop bot gracefully
    async def health_check() -> Check bot status

# Handlers: src/services/telegram/handlers.py
class TelegramHandlers:
    async def start_command() -> /start
    async def help_command() -> /help
    async def ask_command() -> /ask
    async def search_command() -> /search
    async def settings_command() -> /settings
    async def handle_message() -> Regular text messages

# Formatters: src/services/telegram/formatters.py
format_rag_response() -> Rich Markdown formatting
format_search_results() -> Search result display
format_welcome_message() -> /start message
escape_markdown_v2() -> Telegram MarkdownV2 escaping
split_long_message() -> Auto-split >4000 chars
```

## 成功标准

第 7 周在以下情况下完成：

- ✅ 机器人响应所有命令
- ✅ 自然语言查询返回 RAG 答案
- ✅ 缓存提供 150-400 倍加速
- ✅ 设置在会话中保持不变
- ✅ 交互式键盘工作
- ✅ Langfuse 显示 Telegram 痕迹
- ✅ 错误处理很优雅
- ✅ Markdown 格式正确呈现
- ✅ 长消息自动分割
- ✅ 多个用户可以同时使用

---

**第 7 周将您的 RAG 系统转变为移动优先的对话式研究助手，可通过 Telegram 在任何地方访问！** 🚀

---

＃＃ 常问问题

**问：我需要 Telegram 机器人的公共 IP 吗？**
答：不！轮询模式非常适合开发和低流量部署。 Webhook 需要 HTTPS。

**问：多个用户可以同时使用机器人吗？**
答：是的！每个用户都有独立的设置和会话。

**问：运行机器人需要多少钱？**
答：免费！ Telegram bot API 完全免费，没有任何限制。

**问：我可以将机器人限制为特定用户吗？**
答：是的！使用逗号分隔的 Telegram 用户 ID 设置 `TELEGRAM__ALLOWED_USER_IDS=123456,789012`。

**问：如何获取我的 Telegram 用户 ID？**
答：在 Telegram 上发送消息 `@userinfobot` 或发送消息后检查机器人日志。

**问：机器人是否存储对话历史记录？**
答：是的，内存中存储每个用户的最后 10 条消息。对于生产，请迁移到 Redis/PostgreSQL。

**问：我可以在没有 Docker 的服务器上部署它吗？**
答：是的！只需在使用 `uv sync` 安装依赖项后运行 `python src/main.py` 即可。

**问：如何更新机器人令牌？**
A：更新`.env`中的`TELEGRAM__BOT_TOKEN`并重启：`docker compose restart api`

**问：我可以自定义机器人的个性吗？**
答：是的！编辑 `src/services/ollama/prompts/` 中的提示和 `src/services/telegram/formatters.py` 中的消息模板。

**问：这适用于其他 LLM 模型吗？**
答：是的！更改 `OLLAMA_MODEL` 或使用 `/settings` 命令选择不同的 Ollama 型号。

---

享受新的 RAG 对话界面！ 🎉
