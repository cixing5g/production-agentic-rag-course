<!-- 本文件由对应 Jupyter Notebook 转换并翻译生成；代码单元保持原样。 -->

# 第 4 周：文档分块和混合搜索

**我们本周要构建的内容：**

第 4 周重点关注文档分块策略和混合搜索实现，该实现结合了 BM25 关键词搜索和向量相似性搜索的优点，以实现卓越的检索准确性。

## 第 4 周重点领域

### 核心目标
- **基于章节的分块**：利用文档结构进行智能分段
- **重叠策略**：维护具有重叠段的块之间的上下文
- **向量嵌入**：生成用于语义相似性搜索的嵌入
- **混合搜索架构**：使用分数融合将 BM25 和向量搜索结合起来

### 我们将在本笔记本中实现什么
1. **基于章节的分块** - 具有重叠的生产就绪分块
2. **独立嵌入生成** - 直接 Jina AI 集成
3. **统一搜索测试** - 测试 BM25、向量和混合搜索模式
4. **性能分析** - 比较搜索方法

---

## 架构要点
- **单一统一索引**：一个OpenSearch索引支持所有搜索模式
- **Consolidated Client**：简化的架构，没有单独的索引
- **生产就绪**：包括错误处理和后备策略

## ⚠️ 重要提示：第​​ 4 周新鲜容器设置

**新用户或集成更新**：第 4 周需要全新的容器状态和正确的环境配置。

### 全新开始（第 4 周需要）
```bash
# Complete clean slate - removes all data but ensures correct hybrid search state
docker compose down -v

# Build fresh containers with latest Week 4 code
docker compose up --build -d
```

### 创建 .env 文件
```bash
# Copy the environment configuration (if not already done)
cp .env.example .env
```

### 所需的环境变量

将这些添加到您的 `.env` 文件中：

```bash
# Core Services
POSTGRES_DATABASE_URL=postgresql+psycopg2://rag_user:rag_password@postgres:5432/rag_db
OPENSEARCH__HOST=http://opensearch:9200

# Jina AI Embeddings (Required for Vector/Hybrid Search)
JINA_API_KEY=your_jina_api_key_here
```

### 🔑 获取您的 Jina AI API 密钥

1. **注册Jina AI**：访问https://jina.ai/embeddings/
2. **生成 API 密钥**：转到仪表板并创建一个新密钥
3. **添加到.env文件**：`JINA_API_KEY=jina_your_actual_api_key_here`

**注意**：如果没有 API 密钥，笔记本将使用虚拟嵌入进行演示。

```python
# Environment Setup and Health Check
import sys
import os
from pathlib import Path
import requests
import json

print(f"Python Version: {sys.version_info.major}.{sys.version_info.minor}.{sys.version_info.micro}")

# Find project root and add to Python path
current_dir = Path.cwd()
if current_dir.name == "week4" and current_dir.parent.name == "notebooks":
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
    exit()

# Set environment variables for notebook execution (localhost instead of container names)
os.environ["POSTGRES_DATABASE_URL"] = "postgresql+psycopg2://rag_user:rag_password@localhost:5432/rag_db"
os.environ["OPENSEARCH__HOST"] = "http://localhost:9200"
# Use the working API key for real embeddings demonstration
os.environ["JINA_API_KEY"] = "jina_f25c4c4ca3514b17b089f7dce4640d96HEz1QNZznFF2kWOowimt_Amycq1X"

# Health check
print("\nWEEK 4 PREREQUISITE CHECK")
print("=" * 50)

services_to_test = {
    "FastAPI": "http://localhost:8000/api/v1/health",
    "PostgreSQL (via API)": "http://localhost:8000/api/v1/health",
    "OpenSearch": "http://localhost:9200/_cluster/health"
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

if all_healthy:
    print("\n✓ All services healthy! Ready for Week 4 development.")
    print("✓ Database URL configured for notebook: localhost:5432")
    print("✓ OpenSearch URL configured for notebook: localhost:9200")
    print("✓ Jina API key configured for real embeddings")
else:
    print("\n✗ Some services need attention. Please ensure containers are running.")
```

## 1. 获取用于分块的样本论文

```python
# Get Sample Papers from Database
from src.db.factory import make_database
from src.models.paper import Paper

print("FETCHING SAMPLE PAPERS")
print("=" * 50)

database = make_database()

with database.get_session() as session:
    # Get papers with processed text
    papers = session.query(Paper).filter(
        Paper.raw_text != None,
        Paper.raw_text != ""
    ).limit(3).all()
    
    if papers:
        print(f"Found {len(papers)} papers with processed text:\n")
        sample_papers = []
        
        for i, paper in enumerate(papers, 1):
            print(f"{i}. [{paper.arxiv_id}] {paper.title[:60]}...")
            print(f"   Text length: {len(paper.raw_text):,} characters")
            print(f"   Sections available: {'Yes' if paper.sections else 'No'}\n")
            
            sample_papers.append({
                'arxiv_id': paper.arxiv_id,
                'title': paper.title,
                'abstract': paper.abstract,
                'raw_text': paper.raw_text,
                'sections': paper.sections,
                'authors': paper.authors,
                'categories': paper.categories,
                'published_date': paper.published_date
            })
        
        test_paper = sample_papers[0]
        print(f"Selected paper for analysis: {test_paper['arxiv_id']}")
        
    else:
        print("No papers with processed text found.")
        print("Please run the Airflow DAG 'arxiv_paper_ingestion' first.")
        test_paper = None
        sample_papers = []
```

## 2. 基于部分的重叠分块

我们的生产分块系统利用文档结构，同时添加智能重叠来维护块之间的上下文。

```python
# Section-Based Chunking Implementation
import re

def section_based_chunking(text: str, sections_data=None, target_words: int = 600, overlap_words: int = 100):
    """Production-ready section-based chunking with overlaps.
    
    Args:
        text: The full text to chunk
        sections_data: List of section dictionaries with 'title' and 'content' keys, or dict (optional)
        target_words: Target number of words per chunk (default: 600)
        overlap_words: Number of words to overlap between chunks (default: 100)
    
    Returns:
        List of chunk dictionaries with metadata
    """
    chunks = []
    
    if not sections_data:
        # Fallback: Use paragraph boundaries if no sections
        paragraphs = re.split(r'\n\s*\n', text.strip())
        paragraphs = [p.strip() for p in paragraphs if p.strip()]
        
        current_chunk = ""
        chunk_index = 0
        
        for para in paragraphs:
            combined_text = current_chunk + " " + para if current_chunk else para
            if len(combined_text.split()) <= target_words:
                current_chunk = combined_text
            else:
                if current_chunk:
                    chunks.append({
                        'index': chunk_index,
                        'text': current_chunk.strip(),
                        'word_count': len(current_chunk.split()),
                        'section': 'content'
                    })
                    chunk_index += 1
                current_chunk = para
        
        if current_chunk:
            chunks.append({
                'index': chunk_index,
                'text': current_chunk.strip(),
                'word_count': len(current_chunk.split()),
                'section': 'content'
            })
    else:
        # Handle both list and dict formats
        chunk_index = 0
        
        if isinstance(sections_data, list):
            # Sections data is a list of dictionaries with 'title' and 'content'
            sections_items = [(item.get('title', f'section_{i}'), item.get('content', '')) 
                            for i, item in enumerate(sections_data) if isinstance(item, dict)]
        else:
            # Sections data is a dictionary
            sections_items = list(sections_data.items())
        
        for section_name, section_content in sections_items:
            if not section_content or len(str(section_content).strip()) < 50:
                continue
                
            section_text = str(section_content).strip()
            words = section_text.split()
            
            if len(words) <= target_words:
                # Small section fits in one chunk
                chunks.append({
                    'index': chunk_index,
                    'text': section_text,
                    'word_count': len(words),
                    'section': section_name
                })
                chunk_index += 1
            else:
                # Large section needs splitting with overlap
                start = 0
                while start < len(words):
                    end = start + target_words
                    chunk_words = words[start:end]
                    chunk_text = ' '.join(chunk_words)
                    
                    chunks.append({
                        'index': chunk_index,
                        'text': chunk_text,
                        'word_count': len(chunk_words),
                        'section': section_name,
                        'has_overlap': start > 0
                    })
                    chunk_index += 1
                    start += (target_words - overlap_words)
                    
                    if end >= len(words):
                        break
    
    return chunks

# Test the chunking system
if test_paper:
    print("SECTION-BASED CHUNKING RESULTS")
    print("=" * 50)
    
    chunks = section_based_chunking(
        text=test_paper['raw_text'], 
        sections_data=test_paper.get('sections'),
        target_words=600,
        overlap_words=100
    )
    
    print(f"Paper: {test_paper['arxiv_id']}")
    print(f"Original text: {len(test_paper['raw_text'].split()):,} words")
    print(f"Total chunks created: {len(chunks)}")
    print(f"Average chunk size: {sum(c['word_count'] for c in chunks) / len(chunks):.0f} words")
    
    # Show sample chunks
    print("\nSample chunks:")
    for i in range(min(3, len(chunks))):
        chunk = chunks[i]
        print(f"\nChunk {i+1}: {chunk['section']}")
        print(f"  Words: {chunk['word_count']}")
        print(f"  Text preview: {chunk['text'][:150]}...")
    
    # Show section distribution
    section_counts = {}
    for chunk in chunks:
        section_counts[chunk['section']] = section_counts.get(chunk['section'], 0) + 1
    
    print(f"\nChunks per section (top 5):")
    for section, count in list(section_counts.items())[:5]:
        print(f"  {section}: {count} chunks")
        
else:
    print("No test paper available. Please check database connection.")
```

## 3. 重叠策略分析

```python
# Compare Different Overlap Strategies
def compare_overlap_strategies(text: str, sections_data=None):
    """Compare chunking with different overlap amounts."""
    
    overlap_sizes = [0, 50, 100, 150]
    results = []
    
    print("COMPARING OVERLAP STRATEGIES")
    print("=" * 40)
    
    for overlap in overlap_sizes:
        chunks = section_based_chunking(
            text=text, 
            sections_data=sections_data,
            target_words=600,
            overlap_words=overlap
        )
        
        avg_words = sum(chunk['word_count'] for chunk in chunks) / len(chunks) if chunks else 0
        
        results.append({
            'overlap': overlap,
            'chunks': len(chunks),
            'avg_words': avg_words
        })
        
        print(f"Overlap {overlap:3d} words: {len(chunks):3d} chunks, avg {avg_words:.0f} words/chunk")
    
    print("\nRecommendation: 100-word overlap provides best balance")
    print("- Sufficient context preservation")
    print("- Minimal redundancy")
    print("- Optimal for retrieval accuracy")
    
    return results

if test_paper:
    overlap_results = compare_overlap_strategies(
        text=test_paper['raw_text'],
        sections_data=test_paper.get('sections')
    )
else:
    print("No test paper available for overlap comparison.")
```

## 4. 独立嵌入生成

```python
# Standalone Embedding Generation
import httpx
import asyncio
from typing import List

class JinaEmbeddingsGenerator:
    """Standalone Jina AI embeddings generator."""
    
    def __init__(self, api_key: str = None, model: str = "jina-embeddings-v3"):
        self.api_key = api_key or os.getenv("JINA_API_KEY")
        self.model = model
        self.base_url = "https://api.jina.ai/v1/embeddings"
        self.embedding_dimension = 1024
        
        if not self.api_key:
            print("Warning: No Jina API key found. Using dummy embeddings.")
    
    async def generate_embeddings(self, texts: List[str]) -> List[List[float]]:
        """Generate embeddings for a list of texts."""
        if not self.api_key:
            # Return dummy embeddings for demonstration
            return [[0.1] * self.embedding_dimension for _ in texts]
        
        headers = {
            "Content-Type": "application/json",
            "Authorization": f"Bearer {self.api_key}"
        }
        
        payload = {
            "model": self.model,
            "input": texts,
            "task": "retrieval.passage"
        }
        
        async with httpx.AsyncClient() as client:
            try:
                response = await client.post(
                    self.base_url,
                    headers=headers,
                    json=payload,
                    timeout=30.0
                )
                response.raise_for_status()
                
                result = response.json()
                embeddings = [item["embedding"] for item in result["data"]]
                return embeddings
                
            except Exception as e:
                print(f"Error generating embeddings: {e}")
                return [[0.1] * self.embedding_dimension for _ in texts]

# Test embedding generation
print("TESTING EMBEDDING GENERATION")
print("=" * 40)

embeddings_generator = JinaEmbeddingsGenerator()

# Test with sample chunks
if test_paper and 'chunks' in locals():
    test_texts = [chunk['text'][:500] for chunk in chunks[:3]]  # First 3 chunks
    
    embeddings = await embeddings_generator.generate_embeddings(test_texts)
    
    print(f"Generated embeddings for {len(embeddings)} chunks")
    print(f"Embedding dimension: {len(embeddings[0])}")
    
    for i, embedding in enumerate(embeddings):
        norm = sum(x*x for x in embedding)**0.5
        print(f"\nChunk {i+1} embedding:")
        print(f"  Preview: [{embedding[0]:.3f}, {embedding[1]:.3f}, ...]")
        print(f"  Norm: {norm:.3f}")
else:
    # Test with simple example
    test_texts = [
        "Machine learning is a subset of artificial intelligence.",
        "Neural networks are computational models inspired by biology."
    ]
    
    embeddings = await embeddings_generator.generate_embeddings(test_texts)
    print(f"Generated {len(embeddings)} embeddings")
    print(f"Dimension: {len(embeddings[0]) if embeddings else 0}")
```

## 5.统一搜索系统测试

```python
# Test Unified Search System
from src.services.opensearch.factory import make_opensearch_client_fresh
from opensearchpy import OpenSearch

print("UNIFIED SEARCH SYSTEM TEST")
print("=" * 40)

# Create unified OpenSearch client
opensearch_client = make_opensearch_client_fresh()

# Configure for notebook execution
opensearch_client.host = "http://localhost:9200"
opensearch_client.client = OpenSearch(
    hosts=["http://localhost:9200"],
    use_ssl=False,
    verify_certs=False,
    ssl_show_warn=False,
)

# Check index health
stats = opensearch_client.get_index_stats()
print(f"Index: {stats['index_name']}")
print(f"Documents: {stats['document_count']}")
print(f"Health: {'Healthy' if opensearch_client.health_check() else 'Unhealthy'}")

if stats['document_count'] > 0:
    print("\n✓ Index contains data. Ready for search testing!")
else:
    print("\n⚠ Index is empty. Please run the Airflow DAG first:")
    print("  1. Open http://localhost:8080 (admin/admin)")
    print("  2. Trigger 'arxiv_paper_ingestion' DAG")
    print("  3. Wait for completion (~10 minutes)")
```

## 6. BM25 关键词搜索

```python
# Test BM25 Keyword Search
print("BM25 KEYWORD SEARCH TEST")
print("=" * 40)

test_queries = [
    "machine learning",
    "neural networks",
    "artificial intelligence"
]

for query in test_queries:
    print(f"\nQuery: '{query}'")
    try:
        results = opensearch_client.search_papers(
            query=query,
            size=3
        )
        
        print(f"  Found: {results.get('total', 0)} results")
        
        for i, hit in enumerate(results.get('hits', [])[:2], 1):
            title = hit.get('title', 'N/A')[:50]
            score = hit.get('score', 0)
            
            print(f"    {i}. {title}... (score: {score:.2f})")
            
    except Exception as e:
        print(f"  Error: {e}")

print("\n✓ BM25 search completed!")
```

## 7.向量相似度搜索

```python
# Test Vector Search
print("VECTOR SIMILARITY SEARCH TEST")
print("=" * 40)

test_queries = [
    "deep learning models",
    "transformer architecture"
]

for query in test_queries:
    print(f"\nQuery: '{query}'")
    try:
        # Generate query embedding
        query_embedding = await embeddings_generator.generate_embeddings([query])
        
        if query_embedding:
            results = opensearch_client.search_chunks_vector(
                query_embedding=query_embedding[0],
                size=3
            )
            
            print(f"  Found: {results.get('total', 0)} results")
            
            for i, hit in enumerate(results.get('hits', [])[:2], 1):
                title = hit.get('title', 'N/A')[:50]
                score = hit.get('score', 0)
                
                print(f"    {i}. {title}... (score: {score:.3f})")
    
    except Exception as e:
        print(f"  Error: {e}")

print("\n✓ Vector search completed!")
```

## 8. 混合搜索（BM25 + 向量）

```python
# Test Hybrid Search
print("HYBRID SEARCH TEST (BM25 + VECTOR)")
print("=" * 40)

test_queries = [
    "machine learning algorithms",
    "neural network optimization"
]

for query in test_queries:
    print(f"\nQuery: '{query}'")
    try:
        # Generate query embedding
        query_embedding = await embeddings_generator.generate_embeddings([query])
        
        if query_embedding:
            results = opensearch_client.search_chunks_hybrid(
                query=query,
                query_embedding=query_embedding[0],
                size=3
            )
            
            print(f"  Found: {results.get('total', 0)} results")
            print(f"  (60% BM25 + 40% Vector fusion)")
            
            for i, hit in enumerate(results.get('hits', [])[:2], 1):
                title = hit.get('title', 'N/A')[:50]
                score = hit.get('score', 0)
                
                print(f"    {i}. {title}... (hybrid score: {score:.3f})")
                
    except Exception as e:
        print(f"  Error: {e}")

print("\n✓ Hybrid search completed!")
```

## 9. 性能比较

```python
# Performance Comparison
import time

print("SEARCH PERFORMANCE COMPARISON")
print("=" * 50)

query = "machine learning artificial intelligence"
print(f"Test query: '{query}'\n")

results_summary = []

# Test BM25
start = time.time()
try:
    bm25_results = opensearch_client.search_papers(query=query, size=5)
    bm25_time = time.time() - start
    results_summary.append({
        'method': 'BM25',
        'time': bm25_time,
        'results': bm25_results.get('total', 0)
    })
except:
    results_summary.append({'method': 'BM25', 'time': 0, 'results': 0})

# Test Vector
start = time.time()
try:
    query_embedding = await embeddings_generator.generate_embeddings([query])
    if query_embedding:
        vector_results = opensearch_client.search_chunks_vector(
            query_embedding=query_embedding[0], size=5
        )
        vector_time = time.time() - start
        results_summary.append({
            'method': 'Vector',
            'time': vector_time,
            'results': vector_results.get('total', 0)
        })
except:
    results_summary.append({'method': 'Vector', 'time': 0, 'results': 0})

# Test Hybrid
start = time.time()
try:
    if query_embedding:
        hybrid_results = opensearch_client.search_chunks_hybrid(
            query=query, query_embedding=query_embedding[0], size=5
        )
        hybrid_time = time.time() - start
        results_summary.append({
            'method': 'Hybrid',
            'time': hybrid_time,
            'results': hybrid_results.get('total', 0)
        })
except:
    results_summary.append({'method': 'Hybrid', 'time': 0, 'results': 0})

# Display results
print(f"{'Method':<10} {'Time (s)':<10} {'Results':<10}")
print("-" * 30)
for result in results_summary:
    print(f"{result['method']:<10} {result['time']:<10.3f} {result['results']:<10}")

print("\nRecommendations:")
print("• BM25: Best for exact keyword matching")
print("• Vector: Best for semantic similarity")
print("• Hybrid: Best overall accuracy")
```

## 10. 生产 API 端点测试

现在让我们测试用户将在生产中与之交互的实际 FastAPI 端点。

```python
# Test Production API Endpoints
import requests
import json

print("PRODUCTION API ENDPOINT TESTING")
print("=" * 50)

# Test BM25-only search
print("\n1. Testing BM25-Only Search:")
try:
    bm25_request = {
        "query": "machine learning transformer",
        "use_hybrid": False,
        "size": 3
    }
    
    bm25_response = requests.post(
        "http://localhost:8000/api/v1/hybrid-search/",
        json=bm25_request
    )
    
    if bm25_response.status_code == 200:
        bm25_data = bm25_response.json()
        print(f"✓ Search mode: {bm25_data['search_mode']}")
        print(f"✓ Total results: {bm25_data['total']}")
        print(f"✓ Top result score: {bm25_data['hits'][0]['score']:.2f}")
        print(f"✓ Top result: {bm25_data['hits'][0]['title'][:60]}...")
    else:
        print(f"✗ BM25 search failed: {bm25_response.status_code}")
except Exception as e:
    print(f"✗ BM25 search error: {e}")

# Test Hybrid search with real embeddings
print("\n2. Testing Hybrid Search (BM25 + Vector):")
try:
    hybrid_request = {
        "query": "neural network architecture",
        "use_hybrid": True,
        "size": 3
    }
    
    hybrid_response = requests.post(
        "http://localhost:8000/api/v1/hybrid-search/",
        json=hybrid_request
    )
    
    if hybrid_response.status_code == 200:
        hybrid_data = hybrid_response.json()
        print(f"✓ Search mode: {hybrid_data['search_mode']}")
        print(f"✓ Total results: {hybrid_data['total']}")
        if hybrid_data['hits']:
            print(f"✓ Top result score: {hybrid_data['hits'][0]['score']:.4f}")
            print(f"✓ Top result: {hybrid_data['hits'][0]['title'][:60]}...")
            print(f"✓ Chunk info available: {'chunk_text' in hybrid_data['hits'][0]}")
        else:
            print("⚠ No results returned")
    else:
        print(f"✗ Hybrid search failed: {hybrid_response.status_code}")
        print(f"Response: {hybrid_response.text}")
except Exception as e:
    print(f"✗ Hybrid search error: {e}")

print("\n✓ Production API testing completed!")
```

## 11. 增强性能比较

```python
# Enhanced Performance Comparison - Client vs API
import time

print("COMPREHENSIVE SEARCH PERFORMANCE COMPARISON")
print("=" * 60)

query = "machine learning artificial intelligence"
print(f"Test query: '{query}'\n")

results_summary = []

# Test 1: Low-level OpenSearch client tests
print("1. LOW-LEVEL OPENSEARCH CLIENT TESTS:")
print("-" * 40)

# BM25 via OpenSearch client
start = time.time()
try:
    bm25_results = opensearch_client.search_papers(query=query, size=5)
    bm25_time = time.time() - start
    results_summary.append({
        'method': 'Client BM25',
        'time': bm25_time,
        'results': bm25_results.get('total', 0)
    })
    print(f"✓ BM25 (client): {bm25_time:.3f}s, {bm25_results.get('total', 0)} results")
except Exception as e:
    results_summary.append({'method': 'Client BM25', 'time': 0, 'results': 0})
    print(f"✗ BM25 (client): {e}")

# Vector via OpenSearch client
start = time.time()
try:
    query_embedding = await embeddings_generator.generate_embeddings([query])
    if query_embedding:
        vector_results = opensearch_client.search_chunks_vector(
            query_embedding=query_embedding[0], size=5
        )
        vector_time = time.time() - start
        results_summary.append({
            'method': 'Client Vector',
            'time': vector_time,
            'results': vector_results.get('total', 0)
        })
        print(f"✓ Vector (client): {vector_time:.3f}s, {vector_results.get('total', 0)} results")
except Exception as e:
    results_summary.append({'method': 'Client Vector', 'time': 0, 'results': 0})
    print(f"✗ Vector (client): {e}")

# Test 2: Production API endpoints
print("\n2. PRODUCTION API ENDPOINTS:")
print("-" * 40)

# BM25 via API
start = time.time()
try:
    api_bm25_response = requests.post("http://localhost:8000/api/v1/hybrid-search/", json={
        "query": query,
        "use_hybrid": False,
        "size": 5
    })
    api_bm25_time = time.time() - start
    if api_bm25_response.status_code == 200:
        api_bm25_data = api_bm25_response.json()
        results_summary.append({
            'method': 'API BM25',
            'time': api_bm25_time,
            'results': api_bm25_data['total']
        })
        print(f"✓ BM25 (API): {api_bm25_time:.3f}s, {api_bm25_data['total']} results")
    else:
        print(f"✗ BM25 (API): HTTP {api_bm25_response.status_code}")
except Exception as e:
    results_summary.append({'method': 'API BM25', 'time': 0, 'results': 0})
    print(f"✗ BM25 (API): {e}")

# Hybrid via API
start = time.time()
try:
    api_hybrid_response = requests.post("http://localhost:8000/api/v1/hybrid-search/", json={
        "query": query,
        "use_hybrid": True,
        "size": 5
    })
    api_hybrid_time = time.time() - start
    if api_hybrid_response.status_code == 200:
        api_hybrid_data = api_hybrid_response.json()
        results_summary.append({
            'method': 'API Hybrid',
            'time': api_hybrid_time,
            'results': api_hybrid_data['total']
        })
        print(f"✓ Hybrid (API): {api_hybrid_time:.3f}s, {api_hybrid_data['total']} results")
        print(f"  → Search mode: {api_hybrid_data['search_mode']}")
        print(f"  → Real embeddings: {'Yes' if api_hybrid_data['search_mode'] == 'hybrid' else 'No'}")
    else:
        print(f"✗ Hybrid (API): HTTP {api_hybrid_response.status_code}")
except Exception as e:
    results_summary.append({'method': 'API Hybrid', 'time': 0, 'results': 0})
    print(f"✗ Hybrid (API): {e}")

# Display comprehensive results
print(f"\n3. PERFORMANCE SUMMARY:")
print("=" * 50)
print(f"{'Method':<15} {'Time (s)':<12} {'Results':<10} {'Notes'}")
print("-" * 55)
for result in results_summary:
    notes = ""
    if "API" in result['method']:
        notes = "Production endpoint"
    elif "Client" in result['method']:
        notes = "Direct client"
    print(f"{result['method']:<15} {result['time']:<12.3f} {result['results']:<10} {notes}")

print("\nKey Insights:")
print("• API endpoints include additional processing (validation, error handling)")
print("• Hybrid search with real embeddings provides semantic relevance")
print("• BM25 excels at keyword matching with larger result sets")
print("• Production API is what users actually interact with")
```

＃＃ 概括

### 我们取得的成就：

1. **基于章节的分块**：实现了生产就绪分块：
   - 使用解析部分尊重文档结构
   - 保留 100 个单词重叠的上下文
   - 处理结构化和非结构化文档
   - 平均创建约 348 个单词的块，具有智能边界

2. **真实嵌入生成**：创建工作嵌入系统：
   - **生产 Jina AI 集成**，具有真实的 1024 维向量
   - FastAPI 端点中自动嵌入生成
   - 可直接使用 API 的独立嵌入代码
   - 回退到虚拟嵌入，无需 API 密钥即可进行测试

3. **统一搜索架构**：全面测试所有搜索模式：
   - **BM25 关键词搜索**：快速（约 50 毫秒）并具有广泛的召回率
   - **向量相似性搜索**：与真实嵌入的语义匹配
   - **混合搜索**：结合两种方法的 RRF 融合（约 2-4 秒，包括嵌入生成）
   - **生产 API 端点**：现实世界的 `/api/v1/hybrid-search/` 集成

4. **生产就绪实施**：
   - ✅ **单一混合索引** (`arxiv-papers-chunks`) 支持所有搜索类型
   - ✅ **真正的嵌入与 Jina AI API 集成一起工作**
   - ✅ **RRF 混合搜索** 具有手动融合回退功能以实现 OpenSearch 兼容性
   - ✅ **FastAPI 端点** 具有适当的错误处理和验证
   - ✅ **81 个文档块被索引**并可从 3 篇研究论文中搜索

### 关键技术成果：

- **混合搜索模式检测**：API 自动检测并报告搜索模式（`bm25` 与 `hybrid`）
- **真实与演示比较**：显示虚拟嵌入和生产 Jina AI 嵌入之间的差异
- **端到端测试**：从原始文档→分块→嵌入→索引→搜索
- **性能分析**：客户端级与 API 级性能的综合比较

### 架构亮点：

- **整合设计**：单一客户端、单一索引、统一搜索，不复杂
- **生产 API**：`/api/v1/hybrid-search/` 端点已准备好用于实际应用程序
- **回退策略**：当嵌入失败时从混合 → BM25 优雅降级
- **真实数据**：使用实际的 arXiv 论文，而不是合成测试数据

### 搜索性能结果：

```
Method          Time (s)     Results    Notes
-------------------------------------------------
Client BM25     ~0.050s     53         Direct client
API BM25        ~0.150s     53         Production endpoint
Client Vector   ~0.005s     5          Direct client + embeddings
API Hybrid      ~2.500s     1-5        Production with RRF fusion
```

### 第 5 周的后续步骤：

- **LLM集成**：连接 Ollama 以从搜索结果生成答案
- **完整的 RAG 管道**：查询→搜索→上下文→生成→响应
- **生产部署**：Docker 编排和扩展注意事项
- **高级功能**：查询扩展、结果重新排序、对话记忆

第 4 周的实施提供了**生产级混合搜索基础**，具有真正的嵌入、全面的测试和强大的架构，为第 5 周的生成式 AI 集成做好了准备。
