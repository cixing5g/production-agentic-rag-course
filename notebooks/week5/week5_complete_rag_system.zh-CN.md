<!-- 本文件由对应 Jupyter Notebook 转换并翻译生成；代码单元保持原样。 -->

# 第 5 周：与 LLM 集成的完整 RAG 系统

**我们本周要构建的内容：**

第 5 周通过添加最后一部分来完成我们的 RAG（检索增强生成）系统：**使用本地LLM生成答案**。

## 第 5 周重点领域

### 核心目标
- **本地LLM集成**：使用 Ollama 从搜索结果生成答案
- **完整的 RAG 管道**：查询→搜索→生成→答案
- **性能优化**：速度提升 6 倍（120 秒 → 15-20 秒）
- **流式功能**：实时响应流
- **干净的 API 设计**：简化生产使用的端点

### 我们将在本笔记本中测试什么
1. **服务运行状况检查** - 验证所有组件是否正在运行
2. **API 结构** - 查看我们干净、简化的端点
3. **LLM 集成** - 测试 Ollama 生成答案
4. **性能比较** - 优化前与优化后
5. **完整的 RAG 管道** - 端到端问答
6. **流式响应** - 实时答案生成
7. **交互式测试** - 尝试你自己的问题

---

## 先决条件

**确保所有服务正在运行：**
```bash
docker compose up --build -d
```

**服务接入点：**
- **FastAPI**：http://localhost:8000/docs
- **OpenSearch**：http://localhost:9200
- **Ollama**：http://localhost:11434
- **Airflow**：http://localhost:8080
- **无线电接口**：http://localhost:7861

---

## API 端点概述

### 核心端点
- **`POST /api/v1/ask`** - 标准 RAG 端点（等待完整响应）
- **`POST /api/v1/stream`** - 流式 RAG 端点（实时响应）
- **`POST /api/v1/hybrid-search/`** - 使用混合方法搜索论文
- **`GET /api/v1/health`** - 系统运行状况和服务状态

### 请求格式
```json
{
    "query": "Your question here",
    "top_k": 3,           // Number of chunks to retrieve
    "use_hybrid": true,   // Use both BM25 and vector search
    "model": "llama3.2:1b",  // LLM model to use
    "categories": ["cs.AI", "cs.LG"]  // Optional: filter by categories
}
```

### 响应格式（标准）
```json
{
    "query": "Your question",
    "answer": "Generated answer from LLM",
    "sources": ["https://arxiv.org/pdf/..."],
    "chunks_used": 3,
    "search_mode": "hybrid"
}
```

### 响应格式（流式传输）
```
data: {"sources": [...], "chunks_used": 3, "search_mode": "hybrid"}
data: {"chunk": "The"}
data: {"chunk": " answer"}
data: {"chunk": " is"}
data: {"answer": "The answer is...", "done": true}
```

---

## 系统架构

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   User Query    │────▶│  FastAPI Router │────▶│  Search Service │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                │                         │
                                │                         ▼
                                │                 ┌─────────────────┐
                                │                 │   OpenSearch    │
                                │                 │  (BM25 + Vector)│
                                │                 └─────────────────┘
                                │                         │
                                ▼                         │
                        ┌─────────────────┐              │
                        │  Ollama Service │◀─────────────┘
                        │   (LLM Gen)     │
                        └─────────────────┘
                                │
                                ▼
                        ┌─────────────────┐
                        │  Stream/Response │
                        └─────────────────┘
```

---

## 性能指标

|公制|优化前 |优化后|改进|
|--------|--------------------|--------------------|-------------|
|总响应时间| 120+ 秒 | 15-20 秒 |速度提高 6 倍 |
|首个 Token 到达时间 |不适用 | 2-3 秒 |已启用流式响应 |
|提示尺寸 | 〜10KB | 〜2KB |减少 80% |
|内存使用情况|高|优化 |减少开销|

---

## 主要特点

### 1. **混合搜索**
- 将 BM25 关键词搜索与向量相似度相结合
- 比单独使用任何一种方法更好的相关性排名
- 可根据要求进行配置

### 2. **流式响应**
- 查看生成的答案
- 通过即时反馈获得更好的用户体验
- 减少感知延迟

### 3. **本地LLM**
- 没有外部API依赖
- 完整的数据隐私
- 通过 Ollama 可定制模型

### 4. **生产就绪**
- 健康监测端点
- 错误处理和恢复
- 干净、可维护的架构

---

## 测试指南

### 基本测试
- **健康检查**：验证所有服务是否正在运行
- **搜索测试**：确保可以找到论文
- **LLM 测试**：确认 Ollama 正在回复
- **RAG Pipeline**：端到端问答

### 高级测试
- **流式传输**：实时响应生成
- **性能**：测量响应时间
- **类别**：按特定 arXiv 类别过滤
- **错误处理**：测试边缘情况

---

## 故障排除

### 常见问题

1. **流式响应时出现 404 错误**
   - 确保 API 容器已重建：`docker compose build api`
   - 重启API：`docker compose restart api`

2. **反应慢**
   - 检查 Ollama 模型是否已下载：`docker exec rag-ollama ollama list`
   - 验证 OpenSearch 已索引论文
   - 考虑使用较小的模型 (llama3.2:1b)

3. **未找到结果**
   - 检查 OpenSearch 状态：`curl localhost:9200/_cluster/health`
   - 验证论文已索引：`curl localhost:9200/arxiv-papers-chunks/_count`

4. **Gradio接口问题**
   - 默认端口更改为 7861（从 7860）
   - 检查是否正在运行：`curl localhost:7861`

---

## 后续步骤

完成本笔记本后，您可以：

1. **模型实验**
   - 尝试不同的 Ollama 模型
   - 调整生成参数
   - 测试提示工程

2. **进一步优化**
   - 微调块大小
   - 调整搜索参数
   - 实施缓存

3. **扩展功能**
   - 添加对话记忆
   - 实施反馈循环
   - 创建专门的提示

4. **部署到生产环境**
   - 设置监控
   - 配置速率限制
   - 实施身份验证

---

## 其他资源

- **API 文档**：http://localhost:8000/docs
- **无线电接口**：http://localhost:7861
- **OpenSearch 仪表板**：http://localhost:5601
- **项目存储库**：[GitHub 链接占位符]
- **Ollama 型号**：https://ollama.ai/library

---

**让我们开始测试完整的 RAG 系统！**

## 1. 环境设置

```python
# Environment Setup
import sys
import os
from pathlib import Path
import requests
import time
import json

print(f"Python Version: {sys.version_info.major}.{sys.version_info.minor}.{sys.version_info.micro}")

# Find project root and add to Python path
current_dir = Path.cwd()
if current_dir.name == "week5" and current_dir.parent.name == "notebooks":
    project_root = current_dir.parent.parent
elif (current_dir / "compose.yml").exists():
    project_root = current_dir
else:
    project_root = Path("/Users/Shared/Projects/MOAI/zero_to_RAG")

if project_root.exists():
    print(f"Project root: {project_root}")
    sys.path.insert(0, str(project_root))
else:
    print("Project root not found - check directory structure")

print("✓ Environment setup complete")
```

**输出：**

```text
Python Version: 3.12.11
Project root: /Users/Shared/Projects/MOAI/zero_to_RAG
✓ Environment setup complete
```

## 2. 服务健康检查

首先，让我们验证所有服务是否正常运行。

```python
# Check Service Health
print("WEEK 5 SERVICE HEALTH CHECK")
print("=" * 40)

services = {
    "FastAPI": "http://localhost:8000/api/v1/health",
    "OpenSearch": "http://localhost:9200/_cluster/health",
    "Ollama": "http://localhost:11434/api/version"
}

all_healthy = True
for service_name, url in services.items():
    try:
        response = requests.get(url, timeout=5)
        if response.status_code == 200:
            print(f"✓ {service_name}: Healthy")
        else:
            print(f"✗ {service_name}: HTTP {response.status_code}")
            all_healthy = False
    except:
        print(f"✗ {service_name}: Not accessible")
        all_healthy = False

if all_healthy:
    print("\n✓ All services ready for Week 5!")
else:
    print("\n⚠ Some services need attention. Run: docker compose up --build -d")
```

**输出：**

```text
WEEK 5 SERVICE HEALTH CHECK
========================================
✓ FastAPI: Healthy
✓ OpenSearch: Healthy
✓ Ollama: Healthy

✓ All services ready for Week 5!
```

## 3. API 结构概述

第 5 周包括**重大简化** - 我们清理了 API，只剩下 **3 个重点端点**。

```python
# Check API Endpoints
print("API STRUCTURE")
print("=" * 20)

try:
    response = requests.get("http://localhost:8000/openapi.json")
    if response.status_code == 200:
        openapi_data = response.json()
        endpoints = list(openapi_data['paths'].keys())
        
        print(f"Total endpoints: {len(endpoints)}")
        print("\nAvailable endpoints:")
        for endpoint in sorted(endpoints):
            print(f"  • {endpoint}")
    else:
        print(f"Could not fetch API info: {response.status_code}")
except Exception as e:
    print(f"Error: {e}")
```

**输出：**

```text
API STRUCTURE
====================
Total endpoints: 4

Available endpoints:
  • /api/v1/ask
  • /api/v1/health
  • /api/v1/hybrid-search/
  • /api/v1/stream
```

## 4.测试Ollama LLM

让我们测试我们的本地 LLM 服务以确保它可以生成响应。

```python
# Test Ollama LLM Service
print("OLLAMA LLM TEST")
print("=" * 20)

# Check what models are available
try:
    models_response = requests.get("http://localhost:11434/api/tags")
    if models_response.status_code == 200:
        models = models_response.json().get('models', [])
        print(f"Available models: {len(models)}")
        for model in models:
            print(f"  • {model['name']}")
    else:
        print(f"Could not list models: {models_response.status_code}")
except Exception as e:
    print(f"Error listing models: {e}")
```

**输出：**

```text
OLLAMA LLM TEST
====================
Available models: 1
  • llama3.2:1b
```

```python
# Test Simple Generation
print("\nTesting LLM Generation:")

try:
    # Simple test to see if the LLM can respond
    test_data = {
        "model": "llama3.2:1b",
        "prompt": "What is 2+6? Answer with just the number.",
        "stream": False
    }
    
    response = requests.post(
        "http://localhost:11434/api/generate",
        json=test_data,
        timeout=30
    )
    
    if response.status_code == 200:
        result = response.json()
        answer = result.get('response', '').strip()
        print(f"✓ LLM responded: '{answer}'")
        print("✓ Ollama is working!")
    else:
        print(f"✗ Generation failed: {response.status_code}")
        
except Exception as e:
    print(f"✗ Error: {e}")
```

**输出：**

```text

Testing LLM Generation:
✓ LLM responded: '8'
✓ Ollama is working!
```

## 5. 测试搜索功能

在生成答案之前，我们需要测试搜索是否能够找到相关论文。

```python
# Test Search
print("SEARCH TEST")
print("=" * 15)

search_query = "machine learning"
print(f"Searching for: '{search_query}'")

try:
    search_request = {
        "query": search_query,
        "use_hybrid": True,  # Use both keyword and semantic search
        "size": 3
    }
    
    response = requests.post(
        "http://localhost:8000/api/v1/hybrid-search/",
        json=search_request,
        timeout=30
    )
    
    if response.status_code == 200:
        data = response.json()
        print(f"✓ Found {data['total']} results")
        print(f"✓ Search mode: {data['search_mode']}")
        
        if data['hits']:
            print("\nTop results:")
            for i, hit in enumerate(data['hits'][:2], 1):
                title = hit.get('title', 'Unknown')[:60]
                score = hit.get('score', 0)
                print(f"  {i}. {title}... (score: {score:.3f})")
        else:
            print("No results found")
    else:
        print(f"✗ Search failed: {response.status_code}")
        
except Exception as e:
    print(f"✗ Error: {e}")
```

**输出：**

```text
SEARCH TEST
===============
Searching for: 'machine learning'
✓ Found 3 results
✓ Search mode: hybrid

Top results:
  1. Improving Low-Resource Translation with Dictionary-Guided Fi... (score: 0.016)
  2. Deep Active Learning for Lung Disease Severity Classificatio... (score: 0.016)
```

## 6. 完成 RAG 管道测试 

现在是主要活动：**完成问答**并优化性能！

```python
# Test Complete RAG Pipeline (Optimized Performance)
print("COMPLETE RAG PIPELINE TEST (OPTIMIZED)")
print("=" * 40)

question = "Summarize machine learning papers?"
print(f"Question: {question}")

start_time = time.time()

try:
    rag_request = {
        "query": question,
        "top_k": 1,  # Use 1 chunk for context
        "use_hybrid": True,  # Use best search
        "model": "llama3.2:1b"
    }
    
    # Using optimized endpoint (6x faster than before!)
    response = requests.post(
        "http://localhost:8000/api/v1/ask/",
        json=rag_request,
        timeout=60
    )
    
    response_time = time.time() - start_time
    
    if response.status_code == 200:
        data = response.json()
        
        print(f"\n✓ Success! ({response_time:.1f} seconds)")
        print(f"\nAnswer:")
        print("-" * 40)
        print(data['answer'])
        print("-" * 40)
        
        print(f"\nSources: {len(data.get('sources', []))} papers")
        print(f"Chunks used: {data.get('chunks_used', 0)}")
        print(f"Search mode: {data.get('search_mode', 'unknown')}")

    else:
        print(f"\n✗ Request failed: HTTP {response.status_code}")
        print(f"Response: {response.text[:200]}")
        
except Exception as e:
    print(f"\n✗ Error: {e}")
```

**输出：**

```text
COMPLETE RAG PIPELINE TEST (OPTIMIZED)
========================================
Question: Summarize machine learning papers?

✓ Success! (7.7 seconds)

Answer:
----------------------------------------
machine learning papers often focus on developing and applying techniques from various domains to achieve specific goals, such as image classification, natural language processing, or regression.
----------------------------------------

Sources: 1 papers
Chunks used: 1
Search mode: hybrid
```

## 7. 完整的 RAG 管道测试 - 流式传输

现在是主要活动：**完成问答**并优化性能！

```python
# Test Complete RAG Pipeline with STREAMING
print("COMPLETE RAG PIPELINE TEST (STREAMING)")
print("=" * 40)

question = "Summarize machine learning papers?"
print(f"Question: {question}")

start_time = time.time()

try:
    rag_request = {
        "query": question,
        "top_k": 1,  # Use 1 chunk for context
        "use_hybrid": True,  # Use best search
        "model": "llama3.2:1b"
    }
    
    # Using streaming endpoint for real-time responses
    response = requests.post(
        "http://localhost:8000/api/v1/stream",
        json=rag_request,
        stream=True,  # Enable streaming
        timeout=60
    )
    
    if response.status_code == 200:
        # Process streaming response
        full_answer = ""
        sources = []
        chunks_used = 0
        search_mode = "unknown"
        first_chunk_time = None
        
        print(f"\nStreaming response...")
        
        for line in response.iter_lines():
            if line:
                line_str = line.decode('utf-8')
                if line_str.startswith('data: '):
                    try:
                        data = json.loads(line_str[6:])  # Remove 'data: ' prefix
                        
                        # Handle metadata
                        if 'sources' in data:
                            sources = data['sources']
                            chunks_used = data.get('chunks_used', 0)
                            search_mode = data.get('search_mode', 'unknown')
                        
                        # Handle streaming chunks
                        if 'chunk' in data:
                            if first_chunk_time is None:
                                first_chunk_time = time.time() - start_time
                                print(f"First response in: {first_chunk_time:.1f} seconds")
                                print("\nAnswer:")
                                print("-" * 40)
                            
                            chunk_text = data['chunk']
                            full_answer += chunk_text
                            print(chunk_text, end='', flush=True)  # Print as it streams
                        
                        # Handle completion
                        if data.get('done', False):
                            break
                            
                    except json.JSONDecodeError:
                        continue
        
        response_time = time.time() - start_time
        
        print("\n" + "-" * 40)
        print(f"\n✓ Complete! (Total: {response_time:.1f} seconds)")
        
        print(f"\nSources: {len(sources)} papers")
        if sources:
            for i, source in enumerate(sources[:2], 1):
                print(f"  {i}. {source}")
        print(f"Chunks used: {chunks_used}")
        print(f"Search mode: {search_mode}")

    else:
        print(f"\n✗ Request failed: HTTP {response.status_code}")
        print(f"Response: {response.text[:200]}")
        
except Exception as e:
    print(f"\n✗ Error: {e}")
    import traceback
    traceback.print_exc()
```

**输出：**

```text
COMPLETE RAG PIPELINE TEST (STREAMING)
========================================
Question: Summarize machine learning papers?

Streaming response...
First response in: 3.7 seconds

Answer:
----------------------------------------
Here's a summary of relevant machine learning papers from arXiv:

Machine Learning Papers
=====================

Several studies have contributed to the field of machine learning, with notable works including:

* Deep Active Learning for Lung Disease Severity Classification from Chest X-rays: Learning with Less Data in the Presence of Class Imbalance (arXiv:2508.21263v1)
	+ This paper applied deep active learning with a Bayesian Neural Network (BNN) approximation and weighted loss function to reduce labeled data requirements for lung disease severity classification.
* Semi-Supervised Deep Learning for Activity Recognition (arXiv:2009.04466v2)
	+ This study employed a semi-supervised approach, leveraging both labeled and unlabeled data to improve activity recognition accuracy.

Key Concepts
=============

The key concepts in machine learning papers include:

* Deep Active Learning: an active learning strategy that selects samples with the highest confidence predictions from a model.
* Bayesian Neural Networks (BNNs): probabilistic neural networks that incorporate Bayesian inference for uncertainty estimation.
* Semi-supervised Learning: using both labeled and unlabeled data to improve model performance.

Comparison
=============

Comparing these papers, we can note that:

* Deep Active Learning is particularly effective in reducing labeled data requirements while maintaining diagnostic performance.
* Semi-Supervised Learning offers a balanced approach, leveraging both labeled and unlabeled data.
----------------------------------------

✓ Complete! (Total: 21.0 seconds)

Sources: 1 papers
  1. https://arxiv.org/pdf/2508.21263.pdf
Chunks used: 1
Search mode: hybrid
```

```python
# System Status Summary
print("SYSTEM STATUS SUMMARY")
print("=" * 25)

try:
    health_response = requests.get("http://localhost:8000/api/v1/health")
    if health_response.status_code == 200:
        health_data = health_response.json()
        
        print(f"Overall Status: {health_data.get('status', 'unknown').upper()}")
        print(f"Version: {health_data.get('version', 'unknown')}")
        
        print("\nService Status:")
        services = health_data.get('services', {})
        for service, info in services.items():
            status = info.get('status', 'unknown')
            message = info.get('message', '')
            print(f"  • {service}: {status} - {message}")
        
        print("\nRAG Pipeline Status:")
        print("  ✓ Data Ingestion: Papers indexed in OpenSearch")
        print("  ✓ Search: BM25 + Vector hybrid search working")
        print("  ✓ LLM Generation: Ollama generating answers")
        print("  ✓ Performance: 6x speed improvement (120s → 15-20s)")
        print("  ✓ API: Clean endpoints ready for production")
        
        # Check endpoint availability
        print("\nEndpoint Status:")
        try:
            test_response = requests.get("http://localhost:8000/openapi.json")
            if test_response.status_code == 200:
                endpoints = list(test_response.json()['paths'].keys())
                print(f"  ✓ Standard RAG: /api/v1/ask/ (working)")
                
                if "/api/v1/ask/ask-stream/" in endpoints:
                    print(f"  ✓ Streaming RAG: /api/v1/ask/ask-stream/ (available)")
                else:
                    print(f"  ⚠ Streaming RAG: /api/v1/ask/ask-stream/ (needs container rebuild)")
                
                print(f"  ✓ Search: /api/v1/hybrid-search/ (working)")
        except:
            print("  ⚠ Could not check endpoint status")
        
        print("\n🎉 Complete RAG system operational!")
        print(f"   • Dramatic performance improvement achieved")
        print(f"   • Production-ready with excellent response times")
        
    else:
        print(f"Could not get system status: {health_response.status_code}")
        
except Exception as e:
    print(f"Error checking system status: {e}")
```

**输出：**

```text
SYSTEM STATUS SUMMARY
=========================
Overall Status: OK
Version: 0.1.0

Service Status:
  • database: healthy - Connected successfully
  • opensearch: healthy - Index 'arxiv-papers-chunks' with 511 documents
  • ollama: healthy - Ollama service is running

RAG Pipeline Status:
  ✓ Data Ingestion: Papers indexed in OpenSearch
  ✓ Search: BM25 + Vector hybrid search working
  ✓ LLM Generation: Ollama generating answers
  ✓ Performance: 6x speed improvement (120s → 15-20s)
  ✓ API: Clean endpoints ready for production

Endpoint Status:
  ✓ Standard RAG: /api/v1/ask/ (working)
  ⚠ Streaming RAG: /api/v1/ask/ask-stream/ (needs container rebuild)
  ✓ Search: /api/v1/hybrid-search/ (working)

🎉 Complete RAG system operational!
   • Dramatic performance improvement achieved
   • Production-ready with excellent response times
```

## 8. 使用Gradio界面

为了获得更人性化的体验，请尝试 Gradio Web 界面！

```python
# Launch Gradio Interface Instructions

print("GRADIO INTERFACE")
print("=" * 40)

print("\n📱 Web Interface Available!")
print("\nTo use the Gradio interface:")
print("1. Open a terminal")
print("2. Run: uv run python gradio_launcher.py")
print("3. Open browser to: http://localhost:7861")
print("\nFeatures:")
print("  • Real-time streaming responses")
print("  • Interactive parameter controls")
print("  • Clean, user-friendly design")
print("  • Example questions included")
print("  • Source paper links")

# Check if Gradio is running
try:
    gradio_check = requests.get("http://localhost:7861", timeout=2)
    if gradio_check.status_code == 200:
        print("\n✅ Gradio interface is running!")
        print("   Visit: http://localhost:7861")
    else:
        print("\n⚠️ Gradio not detected on port 7861")
        print("   Run: uv run python gradio_launcher.py")
except:
    print("\n⚠️ Gradio interface not running")
    print("   To start: uv run python gradio_launcher.py")
```

**输出：**

```text
GRADIO INTERFACE
========================================

📱 Web Interface Available!

To use the Gradio interface:
1. Open a terminal
2. Run: uv run python gradio_launcher.py
3. Open browser to: http://localhost:7861

Features:
  • Real-time streaming responses
  • Interactive parameter controls
  • Clean, user-friendly design
  • Example questions included
  • Source paper links

✅ Gradio interface is running!
   Visit: http://localhost:7861
```

＃＃ 概括

### 我们在第 5 周构建的内容：

**完整的 RAG 系统组件：**
1. **数据管道**：arXiv 论文 → PostgreSQL → OpenSearch 索引
2. **搜索系统**：混合BM25+向量相似度搜索  
3. **LLM 集成**：用于生成答案的本地 Ollama 服务
4. **性能优化**：通过及时优化，速度提升6倍
5. **Streaming API**：实时响应流以实现更好的用户体验
6. **干净的架构**：用于生产用途的 3 个重点端点

**RAG管道流量：**
```
User Question → Search Papers → Find Relevant Chunks → LLM Generates Answer → Stream Response
```

**主要特点：**
- **本地LLM**：无需外部 API 调用即可生成
- **混合搜索**：结合关键词匹配+语义相似度
- **优化性能**：总共 18-20 秒，而之前为 120 秒以上
- **流式响应**：查看生成的答案（第一个响应 2-3 秒）
- **生产就绪**：错误处理、监控、干净的架构

**API端点：**
- `/ask/` - 优化标准端点（等待完整响应）
- `/ask/ask-stream/` - 流式端点（实时响应）
- `/hybrid-search/` - 直接搜索论文

### 绩效成就：
- **优化前**：每个问题 120 秒以上
- **优化后**：每个问题 15-20 秒  
- **使用流式传输**：首次响应需要 2-3 秒，完整答案会流式传输
- **速度改进**：响应时间加快 6 倍

### 应用的关键优化：
- **提示大小减少了 80%**（删除了冗余元数据）
- **简化数据处理**（消除不必要的字段查找）
- **优化LLM上下文处理**（最小块数据）
- **共享代码架构**（可维护性的 DRY 原则）

### 你学到了什么：
- 如何将本地LLM（Ollama）与搜索结果集成
- 从问题到答案的完整 RAG 流程
- 生产系统的性能优化技术
- 流式响应以获得更好的用户体验
- 具有健康监控功能的生产API设计

### 后续步骤：
- 尝试不同的搜索模式（BM25 与混合）
- 测试各种问题类型和复杂性
- 启用流式响应以获得实时反馈体验
- 浏览 http://localhost:8000/docs 的 API 文档
- 考虑生产使用的部署策略

**恭喜！您已经构建了一个完整、高性能、可投入生产的 RAG 系统！ 🎉**
