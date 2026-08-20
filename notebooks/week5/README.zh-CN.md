# 第 5 周：与 LLM 集成的完整 RAG 系统

＃＃ 概述

第 5 周通过将 Ollama LLM 与混合搜索相集成，完成了我们的**生产级 RAG 系统**。该系统性能提升 **6 倍**（120 秒 → 15-20 秒），支持实时流式响应，并提供 Gradio Web 界面。

## 我们建造了什么

- **本地LLM集成**：Ollama 服务与 llama3.2 模型
- **性能优化**：即时降低 80%，速度提升 6 倍
- **流 API**：通过服务器发送的事件进行实时响应
- **Gradio 界面**：支持流式响应的交互式 Web UI
- **生产就绪**：具有两个重点端点的简洁 API 设计

＃＃ 建筑学

<p align="center">
  <img src="../../static/week5_rag_architecture.png" alt="Week 5 Complete RAG System Architecture" width="900">
  <br>
  <em>具有LLM生成层（Ollama）、混合检索管道和Gradio接口的完整RAG系统</em>
</p>

## 快速入门

### 1.启动服务
```bash
docker compose up --build -d
```

### 2. 测试 RAG 端点
```bash
curl -X POST "http://localhost:8000/api/v1/ask" \
  -H "Content-Type: application/json" \
  -d '{"query": "What are transformers?", "top_k": 3, "use_hybrid": true}'
```

### 3.启动Gradio界面
```bash
uv run python gradio_launcher.py
# Open http://localhost:7861
```

## API 端点

### 标准 RAG - `/api/v1/ask`
- **目的**：使用元数据完成响应
- **响应时间**：15-20 秒
- **用例**：批处理、API 集成

### 流式 RAG - `/api/v1/stream`
- **用途**：实时生成 Token
- **第一个令牌的时间**：2-3 秒
- **用例**：交互式用户界面，更好的用户体验

### 请求格式
```json
{
    "query": "Your question",
    "top_k": 3,              // Chunks to retrieve (1-10)
    "use_hybrid": true,      // BM25 + vector search
    "model": "llama3.2:1b",  // LLM model
    "categories": ["cs.AI"]  // Optional filter
}
```

＃＃ 表现

|配置|响应时间 |使用案例|
|--------------|----------------|----------|
| `top_k=1, BM25` | 〜2.4秒|快速解答 |
| `top_k=3, Hybrid` | ~15-20 秒 |均衡的品质|
| `top_k=5, Hybrid` | ~25-30 秒 |综合|

**关键优化**：
- 删除了冗余元数据（立即减少 80%）
- 共享代码架构（DRY原则）
- 针对重点答案的回复限制为 300 字
- 自动源重复数据删除

＃＃ 配置

```bash
# .env file
OLLAMA_HOST=http://ollama:11434
OLLAMA__DEFAULT_MODEL=llama3.2:1b
JINA_API_KEY=your_key_here  # For embeddings
```

## 测试

### 运行笔记本
```bash
jupyter notebook notebooks/week5/week5_complete_rag_system.ipynb
```

### 测试流式响应
```bash
curl -X POST "http://localhost:8000/api/v1/stream" \
  -H "Content-Type: application/json" \
  -d '{"query": "Explain attention mechanism", "top_k": 2}' \
  --no-buffer
```

## 故障排除

|问题 |解决方案 |
|--------|----------|
| `/stream` 上的 404 |重建 API：`docker compose build api && docker compose restart api` |
|反应慢|使用较小型号：`llama3.2:1b` 或减小 `top_k` |
|没有音响 |端口更改为7861：`http://localhost:7861` |
|奥拉玛错误 |检查服务：`docker exec rag-ollama ollama list` |

## 项目结构

```
src/
├── routers/
│   └── ask.py              # RAG endpoints
├── services/
│   └── ollama/
│       ├── client.py       # LLM client
│       └── prompts/        # System prompts
├── gradio_app.py           # Web interface
└── gradio_launcher.py      # Launcher script

notebooks/week5/
├── README.md               # This file
└── week5_complete_rag_system.ipynb
```

## 后续步骤

- **增强**：添加对话记忆、反馈循环
- **优化**：实施缓存，微调模型
- **部署**：添加身份验证、监控、负载平衡

＃＃ 资源

- [笔记本教程](./week5_complete_rag_system.ipynb)
- [API文档](http://localhost:8000/docs)
- [广播接口](http://localhost:7861)
- [奥拉玛模型](https://ollama.ai/library)
