<!-- 本文件由对应 Jupyter Notebook 转换并翻译生成；代码单元保持原样。 -->

# 第 7 周：Agentic RAG 与 LangGraph

**我们本周测试的内容：**

第 7 周使用 LangGraph 的代理架构以及护栏验证和迭代查询细化，通过**智能、自适应检索**扩展了我们的 RAG 系统。

## Agentic RAG 功能

### 传统 RAG 与 Agentic RAG

**传统 RAG（第 5-6 周）**：
```
Query → Always Retrieve → Generate Answer
```

**Agentic RAG（第 7 周）**：
```
Query → Guardrail Validation (Score 0-100)
  ├─ Score < 60 → Out of Scope (reject with helpful message)
  └─ Score >= 60 → Retrieve Documents
       ↓
     Grade Documents
       ├─ Relevant → Generate Answer
       └─ Not Relevant → Rewrite Query → Retry (max 2 attempts)
```

### 关键功能

1. **Guardrail 验证** - LLM 在检索之前验证查询范围（0-100 分）
   - 分数 < 60: Query is out-of-scope (e.g., "What is a dog?")
   - Score >= 60：查询与 ML/NLP 研究论文相关
2. **范围外处理** - 自动拒绝 ML/NLP 域之外的查询
3. **文档分级** - 验证检索到的论文是否相关
4. **查询细化** - 重写模糊查询以获得更好的结果
5. **推理透明度** - 显示智能体的决策步骤
6. **迭代改进** - 如果需要，可以重试更好的查询（最多 2 次尝试）

### 架构：LangGraph 工作流程

![LangGraph Agentic RAG 工作流程](../../static/langgraph-mermaid.png)

**工作流程节点：**
- **开始** → **护栏**（LLM得分0-100）
- **retrieve** → **tool_retrieve** （执行搜索）
- **grade_documents**（LLM相关性检查）
- **rewrite_query**（如果文档不相关则查询细化）
- **结束**（以回答或拒绝结束）

### 新响应字段

- `reasoning_steps`：详细决策轨迹
- `retrieval_attempts`：搜索尝试次数（0-2）
- `rewritten_query`：细化后的查询（如果重写）

### 配置（GraphConfig）

- `max_retrieval_attempts`：2
- `guardrail_threshold`：60/100
- `model`：“llama3.2:1b”
- `temperature`：0.0
- `top_k`：3

---

## 1.先决条件

### 1.环境变量设置

**复制示例文件并添加您的 API 密钥：**

```bash
cp .env.example .env
```

然后编辑 `.env` 并添加您的：
- `JINA_API_KEY` - 从[Jina AI](https://jina.ai/)获取进行混合搜索
- `LANGFUSE_PUBLIC_KEY` - 设置后从 Langfuse UI 获取（参见下面的步骤 2）
- `LANGFUSE_SECRET_KEY` - 设置后从 Langfuse UI 获取（参见下面的步骤 2）

`.env.example` 中的其他值暂时可以保持原样。

### 2. Langfuse v3 自托管设置

该项目使用**Langfuse v3**（自托管），其中包括：
- **langfuse-web**：Web UI 位于 http://localhost:3001
- **langfuse-worker**：后台作业处理器
- **langfuse-postgres**：跟踪数据库
- **langfuse-redis**：缓存和队列管理
- **langfuse-minio**：S3兼容的对象存储
- **clickhouse**：分析数据库

**首次设置：**
1. 确保 `.env` 拥有 `.env.example` 自动生成的所有机密
2.启动服务：`docker compose up langfuse-web langfuse-worker langfuse-postgres langfuse-redis langfuse-minio clickhouse -d`
3.访问http://localhost:3001并创建您的第一个用户
4. 进入设置 → API 密钥获取 `LANGFUSE_PUBLIC_KEY` 和 `LANGFUSE_SECRET_KEY`
5. 将这些密钥复制到您的 `.env` 文件

**注意：** 如果 Langfuse 密钥丢失，跟踪将被禁用，但 API 仍然可以工作。

### 3.Ollama 模型设置

**启动 Docker 服务时会自动拉取 `llama3.2:1b` 模型。**

如果需要手动拉取：
```bash
# Pull model in the Ollama container
docker exec rag-ollama ollama pull llama3.2:1b

# Or if running Ollama locally
ollama pull llama3.2:1b
```

**验证型号是否可用：**
```bash
docker exec rag-ollama ollama list
```

### 4.启动所有服务

**确保所有服务正在运行：**
```bash
docker compose up --build -d
```

**服务接入点：**
- **FastAPI**：http://localhost:8000/docs
- **OpenSearch**：http://localhost:9200
- **Ollama**：http://localhost:11434
- **Langfuse UI**：http://localhost:3001

---

## 2. 服务健康检查

```python
import sys
import os
from pathlib import Path
import requests
import time

print(f"Python Version: {sys.version_info.major}.{sys.version_info.minor}.{sys.version_info.micro}")

# Find project root
current_dir = Path.cwd()
if current_dir.name == "week7" and current_dir.parent.name == "notebooks":
    project_root = current_dir.parent.parent
elif (current_dir / "compose.yml").exists():
    project_root = current_dir
else:
    project_root = current_dir.parent.parent

if project_root.exists():
    print(f"Project root: {project_root}")
    sys.path.insert(0, str(project_root))
else:
    print("⚠ Project root not found - check directory structure")

# Load .env file if it exists
env_file = project_root / ".env"
if env_file.exists():
    print(f"\n✓ Loading environment from: {env_file}")
    with open(env_file) as f:
        for line in f:
            line = line.strip()
            if line and not line.startswith('#') and '=' in line:
                key, value = line.split('=', 1)
                if key not in os.environ:
                    os.environ[key] = value
    print("✓ Environment variables loaded")
else:
    print(f"\n⚠ No .env file found at: {env_file}")
    print("  Run: cp .env.example .env")
    print("  Then add your JINA_API_KEY, LANGFUSE_PUBLIC_KEY, and LANGFUSE_SECRET_KEY")

# Configuration for notebook tests
REQUEST_TIMEOUT = 300
TRUNCATE_ANSWERS = True
TRUNCATE_LENGTH = 200

print("\n✓ Setup complete")
```

```python
print("WEEK 7 SERVICE HEALTH CHECK")
print("=" * 40)

services = {
    "FastAPI": "http://localhost:8000/api/v1/health",
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

# Check if Ollama model is available
print("\nChecking Ollama model availability...")
try:
    response = requests.get("http://localhost:11434/api/tags", timeout=5)
    if response.status_code == 200:
        models = [m['name'] for m in response.json().get('models', [])]
        if 'llama3.2:1b' in models:
            print("✓ llama3.2:1b model is available")
        else:
            print("⚠ llama3.2:1b not found. Run: docker exec rag-ollama ollama pull llama3.2:1b")
            all_healthy = False
except:
    print("⚠ Could not check Ollama models")

if all_healthy:
    print("\n✓ All services ready for Week 7!")
else:
    print("\n⚠ Some services need attention. Run: docker compose up --build -d")
```

## 3. 测试传统 RAG（基线）

首先，我们测试传统的 RAG 端点以建立基线。

```python
print("TRADITIONAL RAG TEST (Baseline)")
print("=" * 40)

question = "What are attention mechanisms?"
print(f"Question: {question}\n")

start_time = time.time()

try:
    response = requests.post(
        "http://localhost:8000/api/v1/ask",
        json={
            "query": question,
            "top_k": 3,
            "use_hybrid": True,
            "model": "llama3.2:3b"
        },
        timeout=REQUEST_TIMEOUT
    )
    
    elapsed = time.time() - start_time
    
    if response.status_code == 200:
        data = response.json()
        print(f"✓ Traditional RAG ({elapsed:.1f}s)")
        
        # Display answer with configurable truncation
        answer = data['answer']
        if TRUNCATE_ANSWERS and len(answer) > TRUNCATE_LENGTH:
            print(f"\nAnswer: {answer[:TRUNCATE_LENGTH]}...")
            print(f"(truncated, full length: {len(answer)} chars)")
        else:
            print(f"\nAnswer: {answer}")
        
        # Display sources with validation
        sources = data.get('sources', [])
        print(f"\nSources: {len(sources)} papers")
        if sources:
            for i, source in enumerate(sources[:3], 1):  # Show first 3
                if isinstance(source, dict):
                    print(f"  {i}. {source.get('title', 'Unknown')}")
                else:
                    print(f"  {i}. {source}")
        
        print(f"Search mode: {data.get('search_mode', 'unknown')}")
    else:
        print(f"✗ Request failed: {response.status_code}")
        
except Exception as e:
    print(f"✗ Error: {e}")
```

## 4. 测试Agentic RAG - 场景 1：超出范围的拒绝

测试护栏是否正确拒绝 ML/NLP 域之外的查询。

```python
print("AGENTIC RAG - SCENARIO 1: Out-of-Scope Rejection")
print("=" * 50)

question = "What is a dog?"
print(f"Question: {question}")
print("Expected: Guardrail should reject (score < 60) and explain scope\n")

start_time = time.time()

try:
    response = requests.post(
        "http://localhost:8000/api/v1/ask-agentic",
        json={
            "query": question,
            "top_k": 3,
            "use_hybrid": True,
        },
        timeout=REQUEST_TIMEOUT
    )
    
    elapsed = time.time() - start_time
    
    if response.status_code == 200:
        data = response.json()
        print(f"✓ Agentic RAG ({elapsed:.1f}s)")
        print(f"\nAnswer: {data['answer']}")
        print(f"\nRetrieval attempts: {data.get('retrieval_attempts', 0)}")
        print(f"\nReasoning steps:")
        for i, step in enumerate(data.get('reasoning_steps', []), 1):
            print(f"  {i}. {step}")
        
        # Check if guardrail score is in reasoning steps
        guardrail_step = next(
            (s for s in data.get('reasoning_steps', []) if 'validated' in s.lower() and 'score' in s.lower()),
            None
        )
        if guardrail_step:
            print(f"\nGuardrail validation: {guardrail_step}")
        
        if data.get('retrieval_attempts', 0) == 0:
            print("\n✓ SUCCESS: Query correctly rejected by guardrail (no retrieval)!")
        else:
            print("\n⚠ UNEXPECTED: Query should have been rejected without retrieval")
    else:
        print(f"✗ Request failed: {response.status_code}")
        print(f"Response: {response.text}")
        
except Exception as e:
    print(f"✗ Error: {e}")
```

## 5. 测试Agentic RAG - 场景 2：成功检索

测试代理是否正确检索研究问题的文档并对其进行评分。

```python
print("AGENTIC RAG - SCENARIO 2: Successful Retrieval")
print("=" * 50)

question = "What are transformers in machine learning?"
print(f"Question: {question}")
print("Expected: Agent should pass guardrail, retrieve documents and generate answer\n")

start_time = time.time()

try:
    response = requests.post(
        "http://localhost:8000/api/v1/ask-agentic",
        json={
            "query": question,
            "top_k": 3,
            "use_hybrid": True,
            "model": "llama3.2:3b"
        },
        timeout=REQUEST_TIMEOUT
    )
    
    elapsed = time.time() - start_time
    
    if response.status_code == 200:
        data = response.json()
        print(f"✓ Agentic RAG ({elapsed:.1f}s)")
        
        # Display answer with better formatting
        answer = data.get('answer', '')
        print(f"\nAnswer:\n{'-'*50}")
        if TRUNCATE_ANSWERS and len(answer) > 500:  # Use longer limit for detailed answers
            print(answer[:500] + "...")
            print(f"(truncated, full length: {len(answer)} chars)")
        else:
            print(answer)
        print('-'*50)
        
        # Display sources with validation
        sources = data.get('sources', [])
        print(f"\nSources: {len(sources)} papers")
        if sources:
            for i, source in enumerate(sources, 1):
                if isinstance(source, dict):
                    print(f"  {i}. {source.get('title', source.get('id', 'Unknown'))}")
                elif isinstance(source, str):
                    print(f"  {i}. {source}")
                else:
                    print(f"  {i}. {str(source)}")
        
        print(f"\nRetrieval attempts: {data.get('retrieval_attempts', 0)}")
        print(f"\nReasoning steps:")
        for i, step in enumerate(data.get('reasoning_steps', []), 1):
            print(f"  {i}. {step}")
        

        # Check rewritten_query field
        if data.get('rewritten_query') is None:
            print("\n✓ Query was not rewritten (worked on first attempt)")
        else:
            print(f"\n→ Query was rewritten to: {data['rewritten_query']}")
        
        if data.get('retrieval_attempts', 0) >= 1:
            print("\n✓ SUCCESS: Agent retrieved and used documents!")
        else:
            print("\n⚠ UNEXPECTED: Agent didn't retrieve for research question")
    else:
        print(f"✗ Request failed: {response.status_code}")
        print(f"Response: {response.text}")
        
except Exception as e:
    print(f"✗ Error: {e}")
```

## 6. 测试 Agentic RAG - 场景 3：查询重写

测试代理是否重写模糊查询以获得更好的结果。

```python
print("AGENTIC RAG - SCENARIO 3: Query Rewriting")
print("=" * 50)

question = "Tell me about ML stuff"
print(f"Question: {question}")
print("Expected: Agent may rewrite query if documents aren't relevant\n")

start_time = time.time()

try:
    response = requests.post(
        "http://localhost:8000/api/v1/ask-agentic",
        json={
            "query": question,
            "top_k": 3,
            "use_hybrid": True,
            "model": "llama3.2:3b"
        },
        timeout=REQUEST_TIMEOUT
    )
    
    elapsed = time.time() - start_time
    
    if response.status_code == 200:
        data = response.json()
        print(f"✓ Agentic RAG ({elapsed:.1f}s)")
        
        # Display answer with better formatting
        answer = data.get('answer', '')
        print(f"\nAnswer:\n{'-'*50}")
        if TRUNCATE_ANSWERS and len(answer) > 500:
            print(answer[:500] + "...")
            print(f"(truncated, full length: {len(answer)} chars)")
        else:
            print(answer)
        print('-'*50)
        
        print(f"\nRetrieval attempts: {data.get('retrieval_attempts', 0)}")
        print(f"\nReasoning steps:")
        for i, step in enumerate(data.get('reasoning_steps', []), 1):
            print(f"  {i}. {step}")
        
        # Check for guardrail validation step
        print("\nValidating guardrail and rewrite steps:")
        reasoning_steps = data.get('reasoning_steps', [])
        if any("validated" in step.lower() for step in reasoning_steps):
            guardrail_step = next(s for s in reasoning_steps if "validated" in s.lower())
            print(f"  ✓ Guardrail validation: {guardrail_step}")
        else:
            print("  ⚠ Guardrail validation step missing")
        
        # Check for query rewriting
        if data.get('rewritten_query'):
            print(f"\n✓ Query was rewritten!")
            print(f"  Original: {question}")
            print(f"  Rewritten: {data['rewritten_query']}")
        elif data.get('retrieval_attempts', 0) > 1:
            print("\n→ Multiple retrieval attempts detected")
            if any("rewritten" in step.lower() for step in reasoning_steps):
                print("  ✓ Rewrite step found in reasoning")
            else:
                print("  ⚠ Multiple attempts but no rewrite info")
        else:
            print("\n→ Query worked on first attempt (no rewrite needed)")
        
        if data.get('retrieval_attempts', 0) > 1:
            print(f"\n✓ Agent performed {data['retrieval_attempts']} retrieval attempts")
    else:
        print(f"✗ Request failed: {response.status_code}")
        print(f"Response: {response.text}")
        
except Exception as e:
    print(f"✗ Error: {e}")
```

```python
print("AGENTIC RAG - SCENARIO 4: Multiple Out-of-Scope Queries")
print("=" * 50)

test_queries = [
    ("What is a dog?", "Biology question"),
    ("What's the weather today?", "Weather question"),
    ("Hello, how are you?", "Greeting"),
]

print("Testing guardrail rejection with various non-ML/NLP queries:\n")

for query, description in test_queries:
    print(f"Query: {query}")
    print(f"Type: {description}")
    
    try:
        response = requests.post(
            "http://localhost:8000/api/v1/ask-agentic",
            json={"query": query, "top_k": 3, "use_hybrid": True},
            timeout=30
        )
        
        if response.status_code == 200:
            data = response.json()
            
            # Check if rejected (no retrieval)
            is_rejected = data['retrieval_attempts'] == 0
            
            # Get guardrail score from reasoning if available
            guardrail_step = next(
                (s for s in data['reasoning_steps'] if 'validated' in s.lower() and 'score' in s.lower()),
                None
            )
            
            print(f"Result: {'✓ REJECTED' if is_rejected else '✗ ACCEPTED'} (attempts: {data['retrieval_attempts']})")
            if guardrail_step:
                print(f"Guardrail: {guardrail_step}")
        else:
            print(f"✗ Request failed: {response.status_code}")
    except Exception as e:
        print(f"✗ Error: {e}")
    
    print("-" * 50)
```

## 8. 交互式测试

尝试你自己的问题！

```python
def ask_agentic(question: str, show_full_answer: bool = False):
    """Helper function to test agentic RAG.
    
    Args:
        question: The question to ask
        show_full_answer: If True, show full answer regardless of TRUNCATE_ANSWERS setting
    """
    print(f"Question: {question}\n")
    
    start = time.time()
    
    try:
        response = requests.post(
            "http://localhost:8000/api/v1/ask-agentic",
            json={"query": question, "top_k": 3, "use_hybrid": True},
            timeout=REQUEST_TIMEOUT
        )
        
        elapsed = time.time() - start
        
        if response.status_code == 200:
            data = response.json()
            print(f"✓ Response in {elapsed:.1f}s\n")
            
            # Display answer
            answer = data.get('answer', '')
            print(f"Answer:\n{'-'*50}")
            if not show_full_answer and TRUNCATE_ANSWERS and len(answer) > 500:
                print(answer[:500] + "...")
                print(f"(truncated, full length: {len(answer)} chars)")
            else:
                print(answer)
            print('-'*50)
            
            # Display metadata
            print(f"\nRetrieval attempts: {data.get('retrieval_attempts', 0)}")
            
            # Display sources with validation
            sources = data.get('sources', [])
            print(f"Sources: {len(sources)}")
            if sources:
                for i, source in enumerate(sources[:3], 1):  # Show first 3
                    if isinstance(source, dict):
                        print(f"  {i}. {source.get('title', source.get('id', 'Unknown'))}")
                    elif isinstance(source, str):
                        print(f"  {i}. {source}")
            
            # Display reasoning
            print(f"\nReasoning:")
            for step in data.get('reasoning_steps', []):
                print(f"  • {step}")
        else:
            print(f"✗ Error: {response.status_code}")
            print(response.text)
    except Exception as e:
        print(f"✗ Exception: {e}")

# Try it!
ask_agentic("How does BERT differ from GPT?")
```

```python
# Try more questions
ask_agentic("What is the capital of France?")  # Should reject as out-of-scope
```

```python
ask_agentic("Explain self-attention mechanisms")  # Should retrieve papers
```

＃＃ 概括

### 我们在第 7 周测试了什么：

**Agentic RAG 功能**：
1. ✅ **Guardrail Validation** - LLM 在检索之前验证查询范围（0-100 分）
2. ✅ **范围外处理** - 自动拒绝 ML/NLP 域之外的查询
3. ✅ **文档分级** - 验证检索到的论文的相关性
4. ✅ **查询重写** - 如果需要的话改进查询
5. ✅ **推理透明度** - 显示决策步骤
6. ✅ **迭代改进** - 可以使用更好的查询重试（最多 2 次尝试）

### 相对于传统 RAG 的主要改进：

|特色 |传统 RAG |特工 RAG |
|--------|----------------|-------------|
| **查询验证** |无 |护栏评分（0-100）|
| **超出范围的处理** |无 |自动拒绝并提供有用的消息 |
| **检索决定** |总是检索 |仅当护栏通过时（分数 >= 60）|
| **相关性检查** |无 |基于LLM的文件分级|
| **查询细化** |无 |基于LLM的重写|
| **迭代** |单程 |最多 2 次检索尝试 |
| **透明度** |黑匣子|详细推理步骤|
| **配置** |硬编码|具有阈值的 GraphConfig |

### 架构：7 节点 LangGraph 工作流程

```
LangGraph Workflow:
  START
    ↓
  guardrail (LLM scoring 0-100)
    ├─ score < 60 → out_of_scope → END (rejection message)
    └─ score >= 60 → retrieve
         ↓
       tool_retrieve (ToolNode - executes search)
         ↓
       grade_documents (LLM relevance check)
         ├─ Relevant → generate_answer → END
         └─ Not relevant → rewrite_query → retrieve (retry, max 2 attempts)
```

### 推理步骤格式：

新的Agentic RAG 返回结构化推理步骤：

1. **“验证的查询范围（分数：X/100）”** - Guardrail验证结果
2. **“检索的文档（N次尝试）”** - 检索尝试的次数
3. **“分级文档（N相关）”** - 文档相关性检查
4. **“重写查询以获得更好的结果”** - 查询细化（如果需要）
5. **“根据上下文生成答案”** - 最终答案生成

### 配置参数（GraphConfig）：

- `max_retrieval_attempts`：2 - 最大重试次数
- `guardrail_threshold`：60/100 - 继续进行的最低分数
- `model`：“llama3.2:1b” - 默认LLM模型
- `temperature`：0.0 - 确定性生成
- `top_k`：3 - 要检索的文档

### 后续步骤：

- **使用不同的问题类型和查询复杂性进行实验**
- **监控**推理步骤以了解代理决策
- **与传统 RAG 比较**性能和准确性
- **根据您的域要求调整**护栏阈值
- **使用附加工具进行扩展**（网络搜索、计算、代码执行）

**第 7 周完成！您现在拥有一个具有护栏验证功能的智能自适应 RAG 系统！ 🎉**
