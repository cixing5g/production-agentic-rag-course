<!-- 本文件由对应 Jupyter Notebook 转换并翻译生成；代码单元保持原样。 -->

# 第 3 周：首先搜索关键词 - 关键基础

> ** 90% 的问题：** 大多数 RAG 系统直接跳到向量搜索，而错过了为最佳检索系统提供动力的基础。我们做对了！

## 基础设置——请先完成此操作！

**在运行任何单元之前，请确保您的环境已正确配置：**

```bash
# 1. CRITICAL: Copy the environment configuration
cp .env.example .env

# 2. Verify these Week 3 settings are in your .env:
# OPENSEARCH__HOST=http://opensearch:9200
# OPENSEARCH__INDEX_NAME=arxiv-papers
# ARXIV__MAX_RESULTS=15
```

**重要提示：** 第 3 周需要使用 `.env` 文件配置 OpenSearch 连接和相关服务。`.env.example` 中的默认设置无需修改即可使用！

**为什么要先进行关键词搜索？**
- **精确匹配能力：** 精确查找特定技术术语和论文 ID
- **速度和效率：** BM25 速度快，不需要昂贵的嵌入模型
- **可解释：** 您确切地了解论文被检索的原因
- **生产现实：** 像 Elasticsearch 这样的公司使用关键词搜索作为其基础

---

# 第 3 周：OpenSearch 集成和 BM25 搜索

**我们本周要构建的内容：**

第 3 周的重点是使用 BM25 评分实施 OpenSearch 集成以实现全文搜索功能。这将我们的系统从简单的存储解决方案转变为可搜索的知识库。

## 第 3 周重点领域

### 核心目标
- **OpenSearch 集成**：将我们的 FastAPI 应用程序连接到 OpenSearch 集群
- **索引管理**：使用适当的映射创建和管理 arxiv-papers 索引
- **BM25 搜索**：通过相关性评分实施全文搜索
- **数据管道**：将论文从 PostgreSQL 传输到 OpenSearch
- **搜索 API**：通过 REST 端点公开搜索功能

### 我们将在本笔记本中测试什么
1. **基础设施验证** - 确保第 1-2 周的所有服务都在运行
2. **OpenSearch 服务集成** - 测试客户端创建和运行状况检查
3. **索引创建和管理** - 使用适当的映射创建 arxiv-papers 索引
4. **数据管道** - 将论文从 PostgreSQL 传输到 OpenSearch
5. **BM25 搜索功能** - 通过相关性评分测试搜索查询
6. **搜索 API 端点** - 验证 FastAPI 搜索端点是否正常工作

### 成功指标
- OpenSearch集群健康且可访问
- 使用正确的映射创建 arxiv-papers 索引
- 论文成功从 PostgreSQL 索引
- BM25搜索返回带有分数的相关结果
- 搜索API端点正确响应
- 所有组件均可供生产使用

---

## 第 3 周组件状态
|组件|目的|状态 |
|------------|---------|--------|
| **OpenSearch 客户端** |连接到 OpenSearch 集群 | ✅ 完整 |
| **指数管理** |创建和管理搜索索引 | ✅ 完整 |
| **查询生成器** |构建复杂的搜索查询 | ✅ 完整 |
| **数据管道** |将论文转移到 OpenSearch | ✅ 完整 |
| **搜索API** |用于搜索的 REST 端点 | ✅ 完整 |
| **BM25 评分** |基于相关性的搜索结果 | ✅ 完整 |

## 重要提示：第​​ 3 周 Docker 服务重新启动

**新用户或集成冲突**：第 3 周引入了需要新容器状态的 OpenSearch 集成。使用这种干净的重启方法：

### 全新开始（建议第 3 周）
```bash
# Complete clean slate - removes all data but ensures correct OpenSearch state
docker compose down -v

# Build fresh containers with latest code
docker compose up --build -d
```

**何时使用这个：**
- 首次运行第 3 周内容 
- OpenSearch 连接问题
- 索引冲突或映射错误
- 想要以干净的 OpenSearch 状态开始

**注意**：这会破坏现有数据，但可确保您拥有正确的第 3 周配置以及正确的 OpenSearch 集成。

---

## 先决条件检查

**开始之前：**
1. 第一周基础设施完成
2. 第 2 周 arXiv 集成工作
3. 紫外线环境激活
4. Docker桌面运行
5. 第 2 周以来 PostgreSQL 中已有的一些论文

**为什么要使用新容器？** 第 3 周包括 OpenSearch 集成，该集成需要正确的集群初始化，并且可能与现有索引状态发生冲突。

**服务接入点：**
- **FastAPI**：http://localhost:8000/docs（API 文档）
- **PostgreSQL**：通过 API 或 `docker exec -it rag-postgres psql -U rag_user -d rag_db`
- **OpenSearch**：http://localhost:9200/_cluster/health
- **Ollama**：http://localhost:11434（LLM服务）
- **Airflow**：http://localhost:8080（用户名：`admin`，密码：`admin`）

## 环境设置

```python
# Environment Setup and Path Configuration
import sys
from pathlib import Path
import json
import requests

print(f"Python Version: {sys.version_info.major}.{sys.version_info.minor}.{sys.version_info.micro}")
print(f"Environment: {sys.executable}")

# Find project root and add to Python path
current_dir = Path.cwd()
if current_dir.name == "week3" and current_dir.parent.name == "notebooks":
    project_root = current_dir.parent.parent
elif (current_dir / "compose.yml").exists():
    project_root = current_dir
else:
    project_root = None

if project_root and (project_root / "compose.yml").exists():
    print(f"Project root: {project_root}")
    sys.path.insert(0, str(project_root))
else:
    print("Missing compose.yml - check directory")
    exit()
```

## 1.基础设施验证

```python
# Service Health Verification
print("WEEK 3 PREREQUISITE CHECK")
print("=" * 50)

services_to_test = {
    "FastAPI": "http://localhost:8000/api/v1/health",
    "PostgreSQL (via API)": "http://localhost:8000/api/v1/health", 
    "OpenSearch": "http://localhost:9200/_cluster/health",
    "Airflow": "http://localhost:8080/health"  
}

all_healthy = True

for service_name, url in services_to_test.items():
    try:
        response = requests.get(url, timeout=5)
        if response.status_code == 200:
            print(f"✓ {service_name}: Healthy")
        else:
            print(f"✗ {service_name}: HTTP {response.status_code}")
            all_healthy = False
    except requests.exceptions.ConnectionError:
        print(f"✗ {service_name}: Not accessible")
        all_healthy = False
    except Exception as e:
        print(f"✗ {service_name}: {type(e).__name__}")
        all_healthy = False

print()
if all_healthy:
    print("All services healthy! Ready for Week 3 OpenSearch integration.")
else:
    print("Some services need attention. Please run: docker compose up --build")
```

## 2. OpenSearch 客户端设置

```python
# OpenSearch Client Setup
from src.services.opensearch.factory import make_opensearch_client
from opensearchpy import OpenSearch

print("OPENSEARCH CLIENT SETUP")
print("=" * 40)

# Create OpenSearch client using factory pattern
opensearch_client = make_opensearch_client()

# Override for notebook execution (localhost instead of container hostname)
opensearch_client.host = "http://localhost:9200"
opensearch_client.client = OpenSearch(
    hosts=["http://localhost:9200"],
    http_compress=True,
    use_ssl=False,
    verify_certs=False,
    ssl_assert_hostname=False,
    ssl_show_warn=False,
)

print(f"Client configured with host: {opensearch_client.host}")
print(f"Index name: {opensearch_client.index_name}")

# Test health check
is_healthy = opensearch_client.health_check()
if is_healthy:
    print("✓ OpenSearch health check: PASSED")
else:
    print("✗ OpenSearch health check: FAILED")
```

## 索引配置

```python
# Display Index Configuration
from src.services.opensearch.index_config import ARXIV_PAPERS_INDEX, ARXIV_PAPERS_MAPPING

print("INDEX CONFIGURATION")
print("=" * 40)
print(f"Index Name: {ARXIV_PAPERS_INDEX}")
print(f"\nKey Features:")
print("• Custom text analyzers for better search")
print("• Multi-field mapping (text + keyword)")
print("• 10 specialized fields for papers")
print("\nField Types:")

properties = ARXIV_PAPERS_MAPPING["mappings"]["properties"]
for field_name, config in properties.items():
    field_type = config.get("type")
    analyzer = config.get("analyzer", "")
    if analyzer:
        print(f"  • {field_name}: {field_type} [{analyzer}]")
    else:
        print(f"  • {field_name}: {field_type}")
```

### 创建索引

```python
# Create Index if it doesn't exist
print("INDEX CREATION")
print("=" * 40)

try:
    # Check if index already exists
    index_exists = opensearch_client.client.indices.exists(index=opensearch_client.index_name)
    
    if index_exists:
        print(f"✓ Index '{opensearch_client.index_name}' already exists")
        
        # Get current index statistics
        stats = opensearch_client.get_index_stats()
        if stats and 'error' not in stats:
            print(f"\nCurrent Statistics:")
            print(f"   Documents: {stats.get('document_count', 0)}")
            print(f"   Size: {stats.get('size_in_bytes', 0):,} bytes")
    else:
        print(f"Creating new index: {opensearch_client.index_name}")
        
        # Create the index with our custom mapping
        success = opensearch_client.create_index()
        
        if success:
            print(f"✓ Index created successfully!")
        else:
            print(f"✗ Index creation failed")
            
except Exception as e:
    print(f"✗ Error with index management: {e}")
```

## 3. 数据管道 - 运行 Airflow DAG

**arxiv_paper_ingestion** DAG 自动：
1. 从 arXiv API 获取论文
2. 在PostgreSQL中存储论文
3. **将论文索引到 OpenSearch**

＃＃＃ 指示：

**继续之前，运行 Airflow DAG：**

1. 打开Airflow UI：http://localhost:8080
2、登录：用户名`admin`，密码`admin`
3.找到**`arxiv_paper_ingestion`** DAG
4.点击DAG名称将其打开
5. 点击**“触发DAG”**按钮（▶️播放图标）
6. 等待约 10 分钟完成
7. 检查所有任务是否变绿

然后运行下面的单元格来验证：

```python
# Verify Data Pipeline Results
print("VERIFYING DATA PIPELINE")
print("=" * 40)

stats = opensearch_client.get_index_stats()

if stats and 'error' not in stats:
    doc_count = stats.get('document_count', 0)
    
    if doc_count > 0:
        print(f"✓ Success! Found {doc_count} documents in OpenSearch")
        
        # Show sample papers
        sample = opensearch_client.search_papers("*", size=3)
        if sample.get('hits'):
            print(f"\nSample papers:")
            for i, paper in enumerate(sample['hits'], 1):
                title = paper.get('title', 'Unknown')[:60]
                print(f"  {i}. {title}...")
    else:
        print("⚠️  No documents in OpenSearch yet")
        print("\nPlease run the Airflow DAG first (see instructions above)")
else:
    print("✗ Could not retrieve index stats")
```

## 4. 简单的 BM25 搜索

让我们从一个简单的搜索开始来演示 BM25 评分：

```python
# Simple BM25 Search
print("SIMPLE BM25 SEARCH")
print("=" * 40)

# Change this to any word from your papers
search_term = "learning"  # Try different terms!

print(f"Searching for: '{search_term}'\n")

results = opensearch_client.search_papers(
    query=search_term,
    size=5
)

if results.get('hits'):
    print(f"Found {results.get('total', 0)} total matches\n")
    
    for i, paper in enumerate(results['hits'], 1):
        print(f"{i}. {paper.get('title', 'Unknown')[:70]}...")
        print(f"   Score: {paper.get('score', 0):.2f}")
        print(f"   arXiv ID: {paper.get('arxiv_id', 'N/A')}\n")
else:
    print("No results found. Try searching for:")
    print("  • 'neural', 'model', 'algorithm'")
    print("  • Use '*' to see all papers")
```

## 5. 高级 OpenSearch 查询

现在让我们直接使用 OpenSearch Python 客户端探索不同的查询类型。这足见BM25的强大，无需向量！

### 5.1 匹配查询

`match` 查询是单个字段全文搜索的标准查询：

```python
# Match Query - Search in title field
print("MATCH QUERY - Single Field Search")
print("=" * 40)

query = {
    "query": {
        "match": {
            "title": "machine learning"
        }
    },
    "size": 3
}

response = opensearch_client.client.search(
    index=opensearch_client.index_name,
    body=query
)

print(f"Found {response['hits']['total']['value']} results\n")

for hit in response['hits']['hits']:
    print(f"Title: {hit['_source']['title'][:70]}...")
```

### 5.2 多匹配查询

同时搜索多个字段：

```python
# Multi-Match Query - Search across multiple fields
print("MULTI-MATCH QUERY - Search Multiple Fields")
print("=" * 40)

query = {
    "query": {
        "multi_match": {
            "query": "AI Agents",
            "fields": ["title^2", "abstract", "authors"],  # ^2 boosts title field
            "type": "best_fields"
        }
    },
    "size": 3
}

response = opensearch_client.client.search(
    index=opensearch_client.index_name,
    body=query
)

print(f"Found {response['hits']['total']['value']} results\n")

for hit in response['hits']['hits']:
    print(f"Title: {hit['_source']['title'][:70]}...")
    print(f"Score: {hit['_score']:.2f}")
    print(f"Authors: {', '.join(hit['_source']['authors'][:2])}...\n")
```

### 5.3 增强查询

提升某些结果同时降低其他结果：

```python
# Boosting Query - Promote and demote results
print("BOOSTING QUERY - Promote/Demote Results")
print("=" * 40)

query = {
    "query": {
        "boosting": {
            "positive": {
                "match": {
                    "abstract": "deep learning"
                }
            },
            "negative": {
                "match": {
                    "abstract": "multimodal"
                }
            },
            "negative_boost": 0.1  # Reduce score of negative matches
        }
    },
    "size": 3
}

response = opensearch_client.client.search(
    index=opensearch_client.index_name,
    body=query
)

print(f"Query: Boost 'deep learning', demote 'survey' papers\n")
print(f"Found {response['hits']['total']['value']} results\n")

for hit in response['hits']['hits']:
    title = hit['_source']['title'][:70]
    abstract_snippet = hit['_source']['abstract'][:100]
    print(f"Title: {title}...")
    print(f"Score: {hit['_score']:.2f}")
    print(f"Abstract: {abstract_snippet}...\n")
```

### 5.4 过滤查询

按特定条件过滤结果（不影响评分）：

```python
# Filter Query - Filter by categories
print("FILTER QUERY - Category Filtering")
print("=" * 40)

query = {
    "query": {
        "bool": {
            "must": [
                {
                    "match": {
                        "abstract": "neural"
                    }
                }
            ],
            "filter": [
                {
                    "terms": {
                        "categories": ["cs.AI"]
                    }
                }
            ]
        }
    },
    "size": 3
}

response = opensearch_client.client.search(
    index=opensearch_client.index_name,
    body=query
)

print(f"Found {response['hits']['total']['value']} results\n")

for hit in response['hits']['hits']:
    title = hit['_source']['title'][:70]
    categories = ', '.join(hit['_source']['categories'])
    print(f"Title: {title}...")
    print(f"Categories: {categories}")
    print(f"Score: {hit['_score']:.2f}\n")
```

### 5.5 排序查询

按不同标准对结果进行排序：

```python
# Sorting Query - Sort by publication date
print("SORTING QUERY - Latest Papers First")
print("=" * 40)

query = {
    "query": {
        "match_all": {}  # Get all papers
    },
    "sort": [
        {
            "published_date": {
                "order": "desc"  # Latest first
            }
        }
    ],
    "size": 5
}

response = opensearch_client.client.search(
    index=opensearch_client.index_name,
    body=query
)

print(f"Query: All papers sorted by publication date (newest first)\n")

for hit in response['hits']['hits']:
    title = hit['_source']['title'][:70]
    pub_date = hit['_source']['published_date'][:10]
    print(f"Date: {pub_date} | {title}...")
```

### 5.6 组合查询

组合多种查询类型以进行复杂搜索：

```python
# Combined Query - Complex search with multiple criteria
print("COMBINED QUERY - Complex Search")
print("=" * 40)

query = {
    "query": {
        "bool": {
            "must": [
                {
                    "multi_match": {
                        "query": "transformer",
                        "fields": ["title^3", "abstract"],
                        "type": "best_fields"
                    }
                }
            ],
            "filter": [
                {
                    "range": {
                        "published_date": {
                            "gte": "2024-01-01"
                        }
                    }
                }
            ],
            "should": [
                {
                    "match": {
                        "categories": "cs.AI"
                    }
                }
            ]
        }
    },
    "sort": [
        "_score",
        {"published_date": {"order": "desc"}}
    ],
    "size": 3
}

response = opensearch_client.client.search(
    index=opensearch_client.index_name,
    body=query
)

print(f"Complex Query:")
print(f"  • Must contain 'transformer' (title boosted 3x)")
print(f"  • Filter: published after 2024-01-01")
print(f"  • Prefer: cs.AI category")
print(f"  • Sort: by relevance, then date\n")

print(f"Found {response['hits']['total']['value']} results\n")

for hit in response['hits']['hits']:
    title = hit['_source']['title'][:70]
    pub_date = hit['_source']['published_date'][:10]
    score = hit['_score']
    categories = ', '.join(hit['_source']['categories'][:2])
    
    print(f"Title: {title}...")
    print(f"  Date: {pub_date} | Score: {score:.2f}")
    print(f"  Categories: {categories}\n")
```

＃＃ 概括

### 我们展示了什么

**BM25 搜索功能强大！** 无需任何向量嵌入，我们可以：

1. **简单搜索**：带有相关性评分的基本关键词搜索
2. **匹配查询**：搜索特定字段
3. **多重匹配**：通过提升跨多个字段搜索
4. **提升**：提升或降低某些结果
5. **过滤**：应用过滤器而不影响分数
6. **排序**：按日期、分数或其他字段对结果进行排序
7. **复杂查询**：结合所有技术进行复杂搜索

### 要点

- **BM25 对于许多搜索用例来说效果很好**
- **不需要向量**即可进行有效的全文搜索
- 与基于嵌入的方法相比**简单快速**
- **过滤和排序**使搜索精确且相关
- **字段提升**有助于优先考虑重要内容

### 何时使用 BM25 与向量

**在以下情况下使用 BM25：**
- 搜索特定关键词或短语
- 需要快速、简单的实施
- 拥有良好的文本字段和清晰的术语
- 想要可解释的搜索结果

**在以下情况下考虑向量：**
- 需要语义相似性（概念，而不是关键词）
- 处理同义词和释义
- 跨语言搜索要求
- 非常简短的查询或文档

请记住：**您还可以结合两者**（混合搜索）以获得最佳结果！
我们将在下周看到这一点:)
