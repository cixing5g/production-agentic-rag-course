<!-- 本文件由对应 Jupyter Notebook 转换并翻译生成；代码单元保持原样。 -->

# arXiv 论文助手 - 第一周：基础设施设置

使用 Docker、PostgreSQL、OpenSearch、FastAPI、Airflow 和 Ollama 构建生产级 RAG 系统。

## 技术栈
|组件|用途|端口|
|------------|---------|------|
| **FastAPI** |REST API | 8000 |
| **PostgreSQL** |论文元数据存储 | 5432 |
| **OpenSearch** |混合搜索引擎| 9200/5601 |
| **Apache Airflow** |工作流程自动化 | 8080|
| **Ollama** |本地LLM推理| 11434 | 11434

## 学习材料

**核心技术：**
- **Docker**：[教程视频](https://www.youtube.com/watch?v=pg19Z8LL06w) | [Docker Compose](https://www.youtube.com/watch?v=SXwC9fSwct8)
- **FastAPI**：[YouTube 系列](https://www.youtube.com/playlist?list=PLK8U0kF0E_D6l19LhOGWhVZ3sQ6ujJKq_) | [文档](https://fastapi.tiangolo.com/tutorial/)
- **PostgreSQL**：[初学者指南](https://www.youtube.com/watch?v=SpfIwlAYaKk) | [FastAPI + PostgreSQL](https://www.youtube.com/watch?v=398DuQbQJq0)
- **OpenSearch**：[入门](https://docs.opensearch.org/latest/getting-started/)
- **Apache Airflow**：[教程视频](https://www.youtube.com/watch?v=Y_vQyMljDsE)

**开发工具：**
- **VS Code 设置**：[视频指南](https://www.youtube.com/watch?v=mpk4Q5feWaw)
- **Git 基础知识**：[教程](https://www.youtube.com/watch?v=zTjRZNkhiEU)
- **UV 包管理器**：[设置视频](https://www.youtube.com/watch?v=AMdG7IjgSPM)

## 先决条件

**所需软件：**
- Python 3.12+（[下载](https://www.python.org/downloads/)）
- UV 包管理器（[安装指南](https://docs.astral.sh/uv/getting-started/installation/)）
- Docker 桌面（[下载](https://docs.docker.com/get-docker/)）
- Git ([下载](https://git-scm.com/downloads))

**系统要求：**
- 8GB+ 内存（推荐 16GB）
- 20GB+可用磁盘空间

## 设置说明

**运行单元格之前：**
1. 将项目提取/克隆到您的系统
2.在项目根目录中打开终端（包含`compose.yml`）
3.运行：`uv sync`
4.启动Jupyter：`uv run jupyter notebook`
5.验证内核显示项目环境（.venv）

```python
# Environment Check
import sys
from pathlib import Path

python_version = sys.version_info
print(f"Python Version: {python_version.major}.{python_version.minor}.{python_version.micro}")
print(f"Environment: {sys.executable}")

if python_version >= (3, 12):
    print("✓ Python version compatible")
else:
    print("✗ Need Python 3.12+")
    exit()
```

**输出：**

```text
Python Version: 3.12.11
Environment: /Users/Shared/Projects/MOAI/zero_to_RAG/.venv/bin/python
✓ Python version compatible
```

```python
# Find Project Root
current_dir = Path.cwd()

if current_dir.name == "week1" and current_dir.parent.name == "notebooks":
    project_root = current_dir.parent.parent
elif (current_dir / "compose.yml").exists():
    project_root = current_dir
else:
    project_root = None

if project_root and (project_root / "compose.yml").exists():
    print(f"✓ Project root: {project_root}")
else:
    print("✗ Missing compose.yml - check directory")
    exit()
```

**输出：**

```text
✓ Project root: /Users/Shared/Projects/MOAI/zero_to_RAG
```

```python
# Check Docker
import subprocess

try:
    result = subprocess.run(["docker", "--version"], capture_output=True, text=True, timeout=5)
    if result.returncode == 0:
        print(f"✓ Docker: {result.stdout}")
    else:
        print("✗ Docker: Not working")
        exit()
except:
    print("✗ Docker: Not found")
    exit()
```

**输出：**

```text
✓ Docker: Docker version 28.1.1, build 4eba377
```

```python
# Check Docker Compose
try:
    result = subprocess.run(["docker", "compose", "version"], capture_output=True, text=True, timeout=5)
    if result.returncode == 0:
        print(f"✓ Docker Compose: {result.stdout.split()[3]}")
    else:
        print("✗ Docker Compose: Not working")
        exit()
except:
    print("✗ Docker Compose: Not found")
    exit()
```

**输出：**

```text
✓ Docker Compose: v2.35.1-desktop.1
```

```python
# Check UV Package Manager
try:
    result = subprocess.run(["uv", "--version"], capture_output=True, text=True, timeout=5)
    if result.returncode == 0:
        print(f"✓ UV: {result.stdout.strip()}")
        print("\n✓ All required software ready!")
    else:
        print("✗ UV: Not working")
        exit()
except:
    print("✗ UV: Not found")
    exit()
```

**输出：**

```text
✓ UV: uv 0.7.13 (Homebrew 2025-06-12)

✓ All required software ready!
```

## 启动服务

**运行命令（在终端中）：**
```bash
cd [project-root]
docker compose up -d
```

**其作用：** 下载图像（第一次）并在后台启动所有服务。

```python
# Check Docker Running
try:
    result = subprocess.run(["docker", "info"], capture_output=True, timeout=5)
    if result.returncode == 0:
        print("✓ Docker is running")
    else:
        print("✗ Docker not running - start Docker Desktop")
        exit()
except:
    print("✗ Docker daemon not accessible")
    exit()
```

**输出：**

```text
✓ Docker is running
```

```python
# Check Current Containers
import json

try:
    result = subprocess.run(
        ["docker", "compose", "ps", "--format", "json"],
        cwd=str(project_root),
        capture_output=True,
        text=True,
        timeout=10
    )
    
    if result.returncode == 0 and result.stdout.strip():
        print("Current containers:")
        for line in result.stdout.strip().split('\n'):
            if line.strip():
                try:
                    container = json.loads(line)
                    service = container.get('Service', 'unknown')
                    state = container.get('State', 'unknown')
                    print(f"  • {service}: {state}")
                except:
                    pass
    else:
        print("No containers running")
        
except Exception as e:
    print("Could not check containers")
```

**输出：**

```text
Current containers:
  • airflow: running
  • api: running
  • opensearch-dashboards: running
  • ollama: running
  • opensearch: running
  • postgres: running
```

## 服务健康验证

所有服务都会自动启动。检查他们的健康状况：

```python
# Service Health Check
EXPECTED_SERVICES = {
    'api': 'FastAPI REST API server',
    'postgres': 'PostgreSQL database',
    'opensearch': 'OpenSearch search engine', 
    'opensearch-dashboards': 'OpenSearch web dashboard',
    'ollama': 'Local LLM inference server',
    'airflow': 'Workflow automation (optional - may be off)'
}

try:
    result = subprocess.run(
        ["docker", "compose", "ps", "--format", "json"],
        cwd=str(project_root),
        capture_output=True,
        text=True,
        timeout=15
    )
    
    if result.returncode == 0:
        print("SERVICE STATUS")
        print("=" * 70)
        print(f"{'Service':<20} {'State':<15} {'Status':<15} {'Notes'}")
        print("-" * 70)
    else:
        print("Could not get service status")
        exit()
        
except Exception as e:
    print(f"Error checking services: {e}")
    exit()

# Parse Service Status
found_services = set()
service_states = {}

if result.stdout.strip():
    for line in result.stdout.strip().split('\n'):
        if line.strip():
            try:
                container = json.loads(line)
                service = container.get('Service', 'unknown')
                state = container.get('State', 'unknown')
                health = container.get('Health', 'no check')
                
                found_services.add(service)
                service_states[service] = {'state': state, 'health': health}
                
                if state == 'running' and health in ['healthy', 'no check']:
                    indicator = "✓"
                    notes = "Ready"
                elif state == 'running' and health == 'unhealthy':
                    indicator = "⚠"
                    notes = "Starting up..."
                elif state == 'exited':
                    indicator = "✗"
                    notes = "Failed to start"
                else:
                    indicator = "?"
                    notes = f"Status: {state}"
                
                print(f"{indicator} {service:<18} {state:<14} {health:<14} {notes}")
                
            except json.JSONDecodeError:
                pass
```

**输出：**

```text
SERVICE STATUS
======================================================================
Service              State           Status          Notes
----------------------------------------------------------------------
✓ airflow            running        healthy        Ready
✓ api                running        healthy        Ready
✓ opensearch-dashboards running        healthy        Ready
✓ ollama             running        healthy        Ready
✓ opensearch         running        healthy        Ready
✓ postgres           running        healthy        Ready
```

```python
# Check Missing Services
missing_services = set(EXPECTED_SERVICES.keys()) - found_services

if missing_services:
    print("\nMISSING SERVICES:")
    print("-" * 70)
    for service in missing_services:
        description = EXPECTED_SERVICES[service]
        if service == 'airflow':
            print(f"⚠ {service:<18} not running    {'(Optional)':<14} {description}")
        else:
            print(f"✗ {service:<18} not running    {'Required':<14} {description}")

failed_services = [s for s, info in service_states.items() 
                  if info['state'] in ['exited', 'restarting'] or info['health'] == 'unhealthy']

if failed_services:
    print(f"\nTROUBLESHOOTING:")
    for service in failed_services:
        print(f"   docker compose logs {service}")
elif missing_services and 'airflow' not in missing_services:
    print(f"\nACTION NEEDED:")
    print("Start missing services: docker compose up -d")
```

### 1.FastAPI - REST API 服务

**互动探索：**

您可以通过多种方式探索和测试 FastAPI 服务：
- **API 文档**：http://localhost:8000/docs（交互式 Swagger UI）
- **替代文档**：http://localhost:8000/redoc（ReDoc 接口）
- **源代码**：位于 `src/routers/` 目录

让我们测试 API 端点并浏览文档：

```python
# Test FastAPI Health
import requests

try:
    response = requests.get("http://localhost:8000/api/v1/health", timeout=5)
    if response.status_code == 200:
        data = response.json()
        print("✓ FastAPI is responding")
        print(f"Status: {data.get('status', 'unknown')}")
    else:
        print(f"⚠ API returned status: {response.status_code}")
except requests.exceptions.ConnectionError:
    print("✗ API not responding - wait 1-2 minutes")
except Exception as e:
    print(f"✗ API test error: {e}")
```

**输出：**

```text
✓ FastAPI is responding
Status: ok
```

```python
# PRODUCTION INSIGHTS
print("\n" + "="*60)
print("  PRODUCTION INSIGHT (Online Sessions Only)")
print("="*60)
print("❓ How are they scaled?")
print("❓ What are the bottlenecks?")
print("❓ How are they monitored and managed?")
print("❓ How are they integrated with other systems?")
print("❓ What are the best practices for using these systems?")
print("❓ How are these systems used and deployed in production?")
print("❓ How are they tested? in terms of load and performance?")
print("→ Learn these production secrets in our online walkthrough sessions!")
print("="*60)
```

**输出：**

```text

============================================================
  PRODUCTION INSIGHT (Online Sessions Only)
============================================================
❓ How are they scaled?
❓ What are the bottlenecks?
❓ How are they monitored and managed?
❓ How are they integrated with other systems?
❓ What are the best practices for using these systems?
❓ How are these systems used and deployed in production?
❓ How are they tested? in terms of load and performance?
→ Learn these production secrets in our online walkthrough sessions!
============================================================
```

### 2. Apache Airflow - 工作流程自动化

**互动探索：**

Apache Airflow 管理数据管道和自动化工作流程。您可以通过以下方式探索它：
- **Web 仪表板**：http://localhost:8080 
- **登录**：用户名：`admin`，密码：在容器中找到（参见下面的测试）
- **源代码**：位于 `airflow/dags/` 目录

**简单密码位置：**
Airflow 3.0 将管理员密码存储在可预测的文件中：
```
/opt/airflow/simple_auth_manager_passwords.json.generated
```

下面的测试会自动为你读取这个文件！

让我们测试 Airflow 并获取密码：

```python
# Get Airflow Password
import json
from pathlib import Path

password_file = project_root / "airflow" / "simple_auth_manager_passwords.json.generated"

try:
    if password_file.exists():
        with open(password_file, 'r') as f:
            data = json.load(f)
            password = data.get("admin")
        print(f"✓ Airflow password: {password}")
    else:
        print(f"⚠ Password file not found")
        password = None
except Exception as e:
    print(f"✗ Could not read password: {e}")
    password = None
```

**输出：**

```text
✓ Airflow password: sBtDW9ffYBgETMqR
```

```python
# Test Airflow Health
try:
    response = requests.get("http://localhost:8080/api/v1/health", timeout=5)
    if response.status_code == 200:
        print("✓ Airflow is healthy")
        
        if password:
            print(f"\nAirflow Login:")
            print(f"URL: http://localhost:8080")
            print(f"Username: admin")
            print(f"Password: {password}")
    else:
        print(f"⚠ Airflow returned: {response.status_code}")
        
except requests.exceptions.ConnectionError:
    print("✗ Airflow not responding - wait 2-3 minutes")
except Exception as e:
    print(f"✗ Airflow test error: {e}")
```

**输出：**

```text
✓ Airflow is healthy

Airflow Login:
URL: http://localhost:8080
Username: admin
Password: sBtDW9ffYBgETMqR
```

### 3. OpenSearch - 混合数据库

**互动探索：**

OpenSearch 提供全文搜索和分析功能：
- **API端点**：http://localhost:9200 
- **仪表板 UI**：http://localhost:5601（Web 界面）
- **源代码**：位于 `src/services/opensearch/` 目录

**对学生很重要：** 
- ✅ 使用 http://localhost:5601 作为网页界面
- ✅ 使用仪表板中的开发工具进行 API 查询

让我们测试 OpenSearch 并探索其功能：

```python
# Test 1: Check OpenSearch Dashboards Web Interface
# This is the proper way for students to interact with OpenSearch

dashboards_url = "http://localhost:5601"

try:
    # Test if Dashboards is accessible
    response = requests.get(f"{dashboards_url}/api/status", timeout=10, allow_redirects=True)
    if response.status_code == 200:
        print("✓ OpenSearch Dashboards is accessible!")
        print("✓ Web interface is ready for exploration")
        
        print("\n Web Interface Access:")
        print("=" * 40)
        print(f"Main Dashboard: {dashboards_url}")
        print(f"Dev Tools: {dashboards_url}/app/dev_tools")
        print("=" * 40)
        
        print("\n Student Learning Activities:")
        print("1. Explore the Dashboard:")
        print("   • Visit http://localhost:5601")
        print("   • Navigate through the interface")
        print("   • Check out the 'Discover' tab")
        
        print("\n2. Use Dev Tools for API Queries:")
        print("   • Go to Dev Tools")
        print("   • Try: GET /_cluster/health")
        print("   • Try: GET /_cat/indices?v")
        print("   • Try: GET /_cluster/stats")
        print("   • Check the learning material for more information")
        
    else:
        print(f"⚠ Dashboards returned status: {response.status_code}")
        print("Interface may still be starting up")
        
except requests.exceptions.ConnectionError:
    print("✗ OpenSearch Dashboards not accessible yet")
    print("Wait 2-3 minutes for full startup")
    
except requests.exceptions.Timeout:
    print("⚠ Dashboards request timed out")
    print("This is normal during startup - try again in a few minutes")
    
except Exception as e:
    print(f"✗ Error accessing Dashboards: {e}")
    print("Check container status: docker compose ps")
```

**输出：**

```text
✓ OpenSearch Dashboards is accessible!
✓ Web interface is ready for exploration

 Web Interface Access:
========================================
Main Dashboard: http://localhost:5601
Dev Tools: http://localhost:5601/app/dev_tools
========================================

 Student Learning Activities:
1. Explore the Dashboard:
   • Visit http://localhost:5601
   • Navigate through the interface
   • Check out the 'Discover' tab

2. Use Dev Tools for API Queries:
   • Go to Dev Tools
   • Try: GET /_cluster/health
   • Try: GET /_cat/indices?v
   • Try: GET /_cluster/stats
   • Check the learning material for more information
```

```python
# PRODUCTION DEPLOYMENT INSIGHT
print("\n" + "="*60)
print("🎯 PRODUCTION INSIGHT (Online Sessions Only)")
print("="*60)
print("❓ Why companies use OpenSearch?")
print("❓ What all is achievable with OpenSearch?")
print("❓ How does OpenSearch handle billions of documents?")
print("❓ How do companies search through billions of documents?")
print("❓ How do e-commerce giants search millions of products instantly?")
print("→ Learn these production secrets in our online walkthrough sessions!")
print("="*60)
```

**输出：**

```text

============================================================
🎯 PRODUCTION INSIGHT (Online Sessions Only)
============================================================
❓ Why companies use OpenSearch?
❓ What all is achievable with OpenSearch?
❓ How does OpenSearch handle billions of documents?
❓ How do companies search through billions of documents?
❓ How do e-commerce giants search millions of products instantly?
→ Learn these production secrets in our online walkthrough sessions!
============================================================
```

### 4. Ollama - 本地 LLM 推理引擎

**互动探索：**

Ollama 在您的机器上本地运行大型语言模型：
- **API端点**：http://localhost:11434
- **命令行**：在容器内可用
- **隐私**：所有人工智能处理都在本地进行（无外部 API）

让我们测试一下 Ollama，看看有哪些型号可用：

```python
# Test 1: Check Ollama Service Status
# Let's see if Ollama is running and what models are available

import requests
import json

ollama_url = "http://localhost:11434/api/tags"

try:
    response = requests.get(ollama_url, timeout=5)
    if response.status_code == 200:
        models_data = response.json()
        models = models_data.get('models', [])
        
        print("✓ Ollama is running!")
        print(f"Available models: {len(models)}")
        
        if models:
            print("\nInstalled Models:")
            for model in models:
                name = model.get('name', 'unknown')
                size = model.get('size', 0)
                size_gb = round(size / (1024**3), 1)
                print(f"  • {name} ({size_gb} GB)")
        else:
            print("\n  No models installed yet")
            print("   This is normal - models are large files (3-7 GB each)")
            print("   In Week 4, we'll install a model like llama3.2")
            
        print("\n  Try This Later (Week 4):")
        print("1. docker exec -it rag-ollama ollama pull llama3.2")
        print("2. docker exec -it rag-ollama ollama list")
        print("3. docker exec -it rag-ollama ollama run llama3.2")
        
    else:
        print(f"⚠ Ollama returned status: {response.status_code}")
        
except requests.exceptions.ConnectionError:
    print("✗ Ollama is not responding yet")
    print("Ollama service might still be starting")
    
except requests.exceptions.Timeout:
    print("✗ Ollama request timed out")
    print("Service might still be initializing")
    
except Exception as e:
    print(f"✗ Unexpected error testing Ollama: {e}")
    print("Try again in a few minutes")
```

**输出：**

```text
✓ Ollama is running!
Available models: 1

Installed Models:
  • llama3.2:1b (1.2 GB)

  Try This Later (Week 4):
1. docker exec -it rag-ollama ollama pull llama3.2
2. docker exec -it rag-ollama ollama list
3. docker exec -it rag-ollama ollama run llama3.2
```

```python
# Test 2: Check Ollama Version and Health
# Let's verify Ollama is properly configured

import requests
import json

ollama_version_url = "http://localhost:11434/api/version"

try:
    response = requests.get(ollama_version_url, timeout=5)
    if response.status_code == 200:
        version_data = response.json()
        version = version_data.get('version', 'unknown')
        
        print("✓ Ollama API is healthy!")
        print(f"Version: {version}")
        
        print("\n  What is Ollama?")
        print("• Runs AI models completely on your local machine")
        print("• No data sent to external services (privacy-first)")
        print("• No API fees or rate limits")
        print("• Supports models like Llama, Mistral, Phi, etc.")
        
        print("\n  Coming in Week 4:")
        print("• Install and run a local language model")
        print("• Generate answers to research questions")
        print("• Summarize academic papers")
        print("• All processing stays on your computer!")
        
    else:
        print(f"⚠ Ollama version check returned: {response.status_code}")
        
except requests.exceptions.ConnectionError:
    print("✗ Could not check Ollama version")
    print("Service might still be starting up")
    
except requests.exceptions.Timeout:
    print("✗ Ollama request timed out")
    print("Service might still be initializing")
    
except Exception as e:
    print(f"✗ Unexpected error checking version: {e}")
    print("Try again in a few minutes")
```

**输出：**

```text
✓ Ollama API is healthy!
Version: 0.11.2

  What is Ollama?
• Runs AI models completely on your local machine
• No data sent to external services (privacy-first)
• No API fees or rate limits
• Supports models like Llama, Mistral, Phi, etc.

  Coming in Week 4:
• Install and run a local language model
• Generate answers to research questions
• Summarize academic papers
• All processing stays on your computer!
```

```python
# PRODUCTION DEPLOYMENT INSIGHT
print("\n" + "="*60)
print("🎯 PRODUCTION INSIGHT (Online Sessions Only)")
print("="*60)
print("❓ What are the real issues with LLMs when in production?")
print("❓ What is the difference between fine-tuned LLM and RAG?")
print("❓ How do companies serve LLMs without burning through cash?")
print("→ Learn these production secrets in our online walkthrough sessions!")
print("="*60)
```

**输出：**

```text

============================================================
🎯 PRODUCTION INSIGHT (Online Sessions Only)
============================================================
❓ What are the real issues with LLMs when in production?
❓ What is the difference between fine-tuned LLM and RAG?
❓ How do companies serve LLMs without burning through cash?
→ Learn these production secrets in our online walkthrough sessions!
============================================================
```

```python
# HANDS-ON: Pull and Test Llama 3.2 (Small Model)

import requests
import subprocess
import time

print("DOWNLOADING LLAMA 3.2:1B MODEL")
print("=" * 50)
print("This is a small 1.3GB model - perfect for testing!")
print("Download will take 2-5 minutes depending on your internet speed...")

try:
    result = subprocess.run(
        ["docker", "exec", "rag-ollama", "ollama", "pull", "llama3.2:1b"],
        cwd=str(project_root),
        capture_output=True,
        text=True,
        timeout=600
    )
    
    if result.returncode == 0:
        print("Llama 3.2:1b model downloaded successfully!")
    else:
        print(f"Download issue: {result.stderr}")
        
except subprocess.TimeoutExpired:
    print("Download timed out - this is normal for slow connections")
    print("The download continues in the background")
except Exception as e:
    print(f"Error downloading model: {e}")
    print("Make sure Ollama container is running: docker compose ps")
```

**输出：**

```text
DOWNLOADING LLAMA 3.2:1B MODEL
==================================================
This is a small 1.3GB model - perfect for testing!
Download will take 2-5 minutes depending on your internet speed...
Llama 3.2:1b model downloaded successfully!
```

```python
# Test Llama 3.2:1b API

def test_ollama_model(model_name, prompt, max_wait_time=60):
    """Test an Ollama model with a prompt."""
    print(f"Testing {model_name} with prompt: '{prompt}'")
    print("-" * 60)
    
    url = "http://localhost:11434/api/generate"
    data = {
        "model": model_name,
        "prompt": prompt,
        "stream": False
    }
    
    try:
        print("Generating response (this may take 10-30 seconds)...")
        start_time = time.time()
        
        response = requests.post(url, json=data, timeout=max_wait_time)
        
        if response.status_code == 200:
            result = response.json()
            response_text = result.get('response', '').strip()
            
            elapsed_time = time.time() - start_time
            print(f"Response generated in {elapsed_time:.1f} seconds")
            print("\nRESPONSE:")
            print("=" * 40)
            print(response_text)
            print("=" * 40)
            
            if 'model' in result:
                print(f"\nModel: {result['model']}")
            if 'total_duration' in result:
                duration_ms = result['total_duration'] / 1000000
                print(f"Generation time: {duration_ms:.0f}ms")
                
            return True
            
        else:
            print(f"API error: {response.status_code}")
            print(f"Response: {response.text}")
            return False
            
    except requests.exceptions.ConnectionError:
        print("Could not connect to Ollama API")
        print("Make sure Ollama is running: docker compose ps")
        return False
    except requests.exceptions.Timeout:
        print("Request timed out")
        print("Model might be loading for the first time (this is normal)")
        return False
    except Exception as e:
        print(f"Unexpected error: {e}")
        return False

test_prompt = "What is machine learning in one sentence?"
success = test_ollama_model("llama3.2:1b", test_prompt)

if success:
    print("\nSUCCESS! Your local AI model is working!")
    print("\nTry more prompts:")
    print('• test_ollama_model("llama3.2:1b", "Explain neural networks simply")')
    print('• test_ollama_model("llama3.2:1b", "Write a Python function to sort a list")')
else:
    print("\nTroubleshooting:")
    print("1. Make sure model downloaded: docker exec rag-ollama ollama list")
    print("2. Check Ollama logs: docker compose logs ollama")
    print("3. Try again - first run takes longer to load model into memory")
```

**输出：**

```text
Testing llama3.2:1b with prompt: 'What is machine learning in one sentence?'
------------------------------------------------------------
Generating response (this may take 10-30 seconds)...
Response generated in 2.9 seconds

RESPONSE:
========================================
Machine learning is a subfield of artificial intelligence that enables computers to learn from data, make predictions or decisions without being explicitly programmed, by analyzing patterns and relationships within the data.
========================================

Model: llama3.2:1b
Generation time: 2929ms

SUCCESS! Your local AI model is working!

Try more prompts:
• test_ollama_model("llama3.2:1b", "Explain neural networks simply")
• test_ollama_model("llama3.2:1b", "Write a Python function to sort a list")
```

### 5. PostgreSQL - 数据库存储

**互动探索：**

PostgreSQL 存储我们应用程序的所有结构化数据：
- **连接**：本地主机：5432
- **数据库**：rag_db
- **用户名/密码**：rag_user / rag_password
- **GUI工具推荐**：DBeaver（免费数据库客户端）

让我们测试数据库连接并探索架构：

```python
# Test 1: Check PostgreSQL Connection (Basic)
# Let's verify PostgreSQL is accepting connections

def test_postgres_connection():
    """Test PostgreSQL connection using simple socket check."""
    import socket
    
    try:
        # Test if PostgreSQL port is open
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(3)
        result = sock.connect_ex(('localhost', 5432))
        sock.close()
        
        if result == 0:
            print("✓ PostgreSQL is accepting connections on port 5432!")
            return True
        else:
            print("✗ PostgreSQL port is not accessible")
            return False
            
    except Exception as e:
        print(f"✗ Could not test PostgreSQL: {e}")
        return False

postgres_available = test_postgres_connection()

if postgres_available:
    print("\n  Database Connection Details:")
    print("• Host: localhost")
    print("• Port: 5432") 
    print("• Database: rag_db")
    print("• Username: rag_user")
    print("• Password: rag_password")
    
    print("\n  Recommended GUI Tools:")
    print("• DBeaver (Free): https://dbeaver.io/download/")
    print("• pgAdmin: https://www.pgadmin.org/download/")
```

**输出：**

```text
✓ PostgreSQL is accepting connections on port 5432!

  Database Connection Details:
• Host: localhost
• Port: 5432
• Database: rag_db
• Username: rag_user
• Password: rag_password

  Recommended GUI Tools:
• DBeaver (Free): https://dbeaver.io/download/
• pgAdmin: https://www.pgadmin.org/download/
```

```python
# Test PostgreSQL Connection
try:
    import psycopg2
    
    conn = psycopg2.connect(
        host="localhost",
        port=5432,
        database="rag_db", 
        user="rag_user",
        password="rag_password"
    )
    
    print("✓ PostgreSQL connected")
    cursor = conn.cursor()
    
except ImportError:
    print("⚠ psycopg2 not installed - basic connection only")
    exit()
except Exception as e:
    print(f"✗ Database connection failed: {e}")
    exit()
```

**输出：**

```text
✓ PostgreSQL connected
```

```python
# Check Database Tables
cursor.execute("""
    SELECT table_name 
    FROM information_schema.tables 
    WHERE table_schema = 'public'
    ORDER BY table_name;
""")

all_tables = cursor.fetchall()

app_tables = []
airflow_tables = []

for (table_name,) in all_tables:
    if table_name in ['papers', 'users', 'embeddings']:
        app_tables.append(table_name)
    else:
        airflow_tables.append(table_name)

print(f"Found {len(all_tables)} total tables")
print(f"Application tables: {len(app_tables)}")
print(f"Airflow tables: {len(airflow_tables)}")

for table in app_tables:
    print(f"  • {table}")

if not app_tables:
    print("  No application tables yet (expected in Week 1)")
    
cursor.close()
conn.close()
```

**输出：**

```text
Found 46 total tables
Application tables: 1
Airflow tables: 45
  • papers
```

```python
# PRODUCTION DEPLOYMENT INSIGHT
print("\n" + "="*60)
print("🎯 PRODUCTION INSIGHT (Online Sessions Only)")
print("="*60)
print("❓ How do companies handle millions of transactions with PostgreSQL?")
print("❓ What's the secret to zero-downtime database migrations?")
print("→ Learn these production secrets in our online walkthrough sessions!")
print("="*60)
```

**输出：**

```text

============================================================
🎯 PRODUCTION INSIGHT (Online Sessions Only)
============================================================
❓ How do companies handle millions of transactions with PostgreSQL?
❓ What's the secret to zero-downtime database migrations?
→ Learn these production secrets in our online walkthrough sessions!
============================================================
```

### 服务运行状况摘要和后续步骤

基于上面的交互测试：

**如果所有服务均显示 ✓**： 
- 🎉恭喜！您的基础设施已准备就绪
- 所有服务均健康且响应正确
- 您可以使用提供的链接和说明探索每项服务

**如果某些服务显示 ✗**：
- 别担心！服务需要时间才能启动
- 等待 2-3 分钟并重新运行测试单元
- OpenSearch 和 Airflow 花费的时间最长（最多 5 分钟）

**服务接入点：**
- **FastAPI 文档**：http://localhost:8000/docs - 交互式 API 测试
- **Airflow 仪表板**：http://localhost:8080（管理员/管理员） - 工作流程管理
- **OpenSearch 仪表板**：http://localhost:5601 - 仪表板和用户界面 + 分析
- **OpenSearch API**：http://localhost:9200 - 直接 API 访问
- **Ollama API**：http://localhost:11434 - 本地 LLM 推理
- **PostgreSQL**：http://localhost:5432 - 使用 DBeaver 或类似工具

**实践学习活动：**

1. **FastAPI**：交互式文档中的测试端点
2. **Airflow**：手动登录并触发DAG  
3. **OpenSearch**：在开发工具中尝试查询
4. **Ollama**：准备第 6 周模型安装
5. **PostgreSQL**：安装DBeaver并探索数据库结构

**常见问题：**
- “连接被拒绝”→ 服务仍在启动
- “端口正在使用”→另一个应用程序正在使用该端口  
- 容器重启→使用`docker compose logs [service-name]`检查日志

## 故障排除

**常见问题：**
- **连接被拒绝** → 服务仍在启动（等待 2-3 分钟）
- **正在使用的端口** → 停止冲突的应用程序或更改端口
- **容器重启** → 检查日志：`docker compose logs [service-name]`
- **内存不足** → 增加 Docker Desktop 内存分配

**重置所有内容：** `docker compose down && docker compose up -d`

## 第 1 周完成

**服务接入点：**
- **API**：http://localhost:8000/docs
- **Airflow**：http://localhost:8080（管理/sBtDW9ffYBgETMqR）  
- **OpenSearch**：http://localhost:5601
- **PostgreSQL**：本地主机：5432（rag_user/rag_password）

**成功标准：**
- [ ] 所有服务在状态检查中均正常
- [ ] API 文档可访问
- [ ] Airflow 仪表板加载
- [ ] OpenSearch 界面有效

**下一步：** 使用 `docker compose up -d` 保持服务运行或重新启动

## 项目命令

**Makefile 快捷方式：**
```bash
make start    # Start all services  
make status   # Check service status
make logs     # View logs
make health   # Check service health
make stop     # Stop all services
make help     # View all commands
```

**下一步：** 阅读主要的 `README.md` 以获取完整的项目文档。
