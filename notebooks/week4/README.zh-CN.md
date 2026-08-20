# 第 4 周：文档分块和混合搜索

＃＃ 概述

第 4 周实现了**生产级混合搜索系统**，它将 BM25 关键词搜索的精度与向量嵌入的语义理解相结合。该系统通过智能地将文档分解为可搜索的块并启用多种搜索模式，为检索增强生成（RAG）奠定了基础。

## 我们建造了什么

### 🧩 **基于部分的文档分块**
- **智能分段**：利用文档结构（已解析的部分）实现自然块边界
- **上下文保留**：块之间 100 个单词的重叠保持语义连续性
- **自适应处理**：处理结构化（带部分）和非结构化（基于段落）文档
- **最佳大小**：以 600 个单词的块为目标，最小阈值为 100 个单词

### 🔍 **统一混合搜索系统**
- **单一索引架构**：一个 OpenSearch 索引 (`arxiv-papers-chunks`) 支持所有搜索模式
- **多种搜索类型**：
  - **BM25 关键词搜索**：快速（~50ms）传统文本匹配
  - **向量相似性搜索**：使用 1024 维嵌入进行语义搜索
  - **混合搜索**：RRF（倒数排名融合）结合了两种方法
- **生产 API**：具有全面验证的 RESTful 端点 `/api/v1/hybrid-search/`

### 🤖 **真正的嵌入集成**
- **Jina AI Embeddings**：针对检索进行优化的生产级 1024 维向量
- **自动生成**：FastAPI端点自动生成查询嵌入
- **回退策略**：当嵌入不可用时优雅降级到 BM25
- **性能优化**：高效的嵌入生成和存储

＃＃ 建筑学

### 系统概述

<p align="center">
  <img src="../../static/week4_hybrid_opensearch.png" alt="Week 4 Hybrid Search Architecture" width="800">
  <br>
  <em>完整的第 4 周架构，展示了具有分块、嵌入和 RRF 融合的混合搜索</em>
</p>

### 数据流
```
Raw Papers → PDF Parsing → Section Extraction → Chunking → Embedding → Indexing → Search
```

上图展示了完整的第 4 周实施：
- **数据处理管道**：arXiv 论文通过分块和嵌入生成流程
- **统一 OpenSearch 索引**：支持 BM25、向量和混合搜索模式的单一索引  
- **混合检索管道**：将关键词精度与语义理解相结合的 RRF 融合
- **生产 API 层**：具有自动嵌入生成功能的 FastAPI 端点

### 系统组件

#### **1。文档处理管道**
```python
# Located in: src/services/indexing/text_chunker.py
TextChunker.chunk_paper(
    title="Paper Title",
    abstract="Abstract text",
    full_text="Complete paper content",
    sections=parsed_sections_dict,
    target_words=600,
    overlap_words=100
)
```

#### **2。嵌入服务**
```python
# Located in: src/services/embeddings/factory.py
embeddings_service = make_embeddings_service()
vectors = await embeddings_service.embed_query(["query text"])
```

#### **3。统一搜索客户端**
```python
# Located in: src/services/opensearch/client.py
results = opensearch_client.search_unified(
    query="machine learning",
    query_embedding=vector,
    use_hybrid=True,
    size=10
)
```

#### **4。生产API**
```python
# Located in: src/routers/hybrid_search.py
POST /api/v1/hybrid-search/
{
  "query": "neural networks",
  "use_hybrid": true,
  "size": 5
}
```

## 主要特点

### **混合搜索模式**

|模式|速度|回忆|精密|使用案例|
|------|--------|--------|------------|----------|
| **仅限 BM25** | 〜50ms |高|中等|关键词精准匹配 |
| **仅向量** |约 100 毫秒 |中等|高|语义相似度|
| **混合 (RRF)** | 〜2-4秒|高|高|最佳整体相关性 |

### **RRF（倒数秩融合）**
- **算法**：使用倒数排名融合结合 BM25 和向量搜索排名
- **实现**：手动融合算法（OpenSearch 2.19 兼容性）
- **权重**：关键词和语义相关性之间的可配置平衡
- **回退**：如果向量搜索失败，自动回退到 BM25

### **基于章节的分块策略**

```python
# Chunking Parameters (optimized through testing)
CHUNK_SIZE = 600        # Target words per chunk
OVERLAP_SIZE = 100      # Words overlapping between chunks  
MIN_CHUNK_SIZE = 100    # Minimum viable chunk size
SECTION_BASED = True    # Use document structure when available
```

**好处**：
- **语义一致性**：块尊重自然文档边界
- **上下文保留**：重叠可防止边界处的信息丢失
- **检索准确性**：更好地将用户查询与相关内容匹配
- **可扩展性**：处理 1,000 到 100,000 多个单词的文档

## 实施细节

### **OpenSearch 索引配置**

**索引名称**：`arxiv-papers-chunks`

**关键词段**：
```json
{
  "arxiv_id": "2508.18563v1",
  "title": "Paper title",
  "chunk_text": "Chunk content...",
  "chunk_id": "unique_chunk_identifier", 
  "section_name": "Introduction",
  "embedding": [0.123, 0.456, ...],  // 1024 dimensions
  "paper_categories": ["cs.AI", "cs.LG"],
  "published_date": "2025-08-25T23:43:33"
}
```

### **搜索查询结构**

**BM25查询**（关键词匹配）：
```json
{
  "query": {
    "bool": {
      "should": [
        {"match": {"chunk_text": {"query": "machine learning", "fuzziness": "AUTO"}}},
        {"match": {"title": {"query": "machine learning", "boost": 2.0}}},
        {"match": {"abstract": {"query": "machine learning", "boost": 1.5}}}
      ]
    }
  }
}
```

**混合查询**（RRF手动融合）：
1.执行BM25查询→得到排名结果
2.执行向量查询→得到排序结果  
3.应用RRF融合算法→合并排名
4. 返回具有混合分数的合并结果

### **环境配置**

**所需变量**：
```bash
# Core Services
POSTGRES_DATABASE_URL=postgresql+psycopg2://rag_user:rag_password@postgres:5432/rag_db
OPENSEARCH__HOST=http://opensearch:9200

# Embeddings (Required for Hybrid Search)
JINA_API_KEY=jina_your_api_key_here

# Chunking Configuration
CHUNKING__CHUNK_SIZE=600
CHUNKING__OVERLAP_SIZE=100
CHUNKING__MIN_CHUNK_SIZE=100
CHUNKING__SECTION_BASED=true

# OpenSearch Configuration
OPENSEARCH__INDEX_NAME=arxiv-papers
OPENSEARCH__CHUNK_INDEX_SUFFIX=chunks
OPENSEARCH__VECTOR_DIMENSION=1024
```

## API 参考

### **混合搜索端点**

**端点**：`POST /api/v1/hybrid-search/`

**请求正文**：
```json
{
  "query": "transformer neural networks",
  "use_hybrid": true,
  "size": 10,
  "from": 0,
  "categories": ["cs.AI", "cs.LG"],
  "latest_papers": false,
  "min_score": 0.0
}
```

**回复**：
```json
{
  "query": "transformer neural networks",
  "total": 15,
  "hits": [
    {
      "arxiv_id": "2508.18563v1",
      "title": "Paper Title",
      "authors": "Author Names",
      "abstract": "Paper abstract...",
      "score": 0.8542,
      "chunk_text": "Relevant chunk content...",
      "chunk_id": "chunk_uuid",
      "section_name": "Related Work"
    }
  ],
  "size": 10,
  "from": 0,
  "search_mode": "hybrid"
}
```


## 性能基准

**测试环境**：3篇论文，81个块，单节点OpenSearch

|搜索类型 |平均响应时间 |吞吐量|回忆@10 |精度@10 |
|-------------|--------------------|------------|------------|----------------|
|仅 BM25 | 52 毫秒 | ~200 请求/秒 | 0.78 | 0.78 0.65 | 0.65
|仅向量 | 105 毫秒 | ~95 请求/秒 | 0.82 | 0.82 0.71 | 0.71
|混合动力 (RRF) | 2.4秒| ~25 请求/秒 | 0.89 | 0.89 0.84 | 0.84

**关键见解**：
- **混合搜索**以响应时间为代价提供最佳相关性
- **BM25** 非常适合高吞吐量关键词匹配
- **向量搜索** 良好的语义理解，速度适中
- **嵌入生成**大约需要 2 秒的混合搜索时间

## 生产部署

### **扩展考虑因素**

**开放搜索集群**：
```yaml
# Recommended minimum for production
opensearch:
  image: opensearchproject/opensearch:2.19.0
  environment:
    - cluster.name=rag-cluster
    - node.name=rag-node-1
    - discovery.type=single-node
    - OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g
  deploy:
    resources:
      limits:
        memory: 2G
      reservations:
        memory: 1G
```

**嵌入服务优化**：
- **批处理**：每个 API 调用处理多个嵌入
- **缓存**：缓存频繁的查询嵌入
- **速率限制**：遵守 Jina AI API 限制（1000 个请求/分钟）
- **后备策略**：嵌入不可用时仅使用 BM25 模式

### **监控和可观察性**

**关键指标**：
- 搜索请求延迟（p50、p95、p99）
- 嵌入生成成功率
- 索引文档数量和大小
- 搜索模式使用分布（BM25 与 Hybrid）

**健康检查**：
- OpenSearch集群健康状况
- 嵌入服务可用性
- 索引文档计数验证
- 示例查询执行

## 故障排除

### **常见问题**

**1.混合搜索返回 BM25 模式**
```bash
# Check embedding service
curl -X POST "http://localhost:8000/api/v1/hybrid-search/" \
  -H "Content-Type: application/json" \
  -d '{"query": "test", "use_hybrid": true}'

# Check logs for embedding errors
docker compose logs api | grep -i embedding
```

**2.空搜索结果**
```bash
# Verify index exists and has documents
curl "http://localhost:9200/arxiv-papers-chunks/_count"

# Check index mapping
curl "http://localhost:9200/arxiv-papers-chunks/_mapping"
```

**3.缓慢的嵌入生成**
```bash
# Check Jina API key configuration
docker compose exec api env | grep JINA

# Test direct embedding service
curl -X POST "https://api.jina.ai/v1/embeddings" \
  -H "Authorization: Bearer $JINA_API_KEY" \
  -d '{"model": "jina-embeddings-v3", "input": ["test"]}'
```

## 测试

### **运行第 4 周笔记本**

1. **启动服务**：
```bash
docker compose up --build -d
```

2. **打开笔记本**：
```bash
cd notebooks/week4
jupyter notebook week4_hybrid_search.ipynb
```

3. **执行所有单元**：笔记本包括：
   - 环境设置和健康检查
   - 基于章节的分块演示
   - 使用 Jina AI 进行真正的嵌入生成
   - 所有搜索模式（BM25、向量、混合）
   - 生产API端点测试
   - 性能比较

### **手动测试**

**测试 BM25 搜索**：
```bash
curl -X POST "http://localhost:8000/api/v1/hybrid-search/" \
  -H "Content-Type: application/json" \
  -d '{"query": "machine learning", "use_hybrid": false, "size": 3}'
```

**测试混合搜索**：
```bash
curl -X POST "http://localhost:8000/api/v1/hybrid-search/" \
  -H "Content-Type: application/json" \
  -d '{"query": "neural networks", "use_hybrid": true, "size": 3}'
```

## 后续步骤（第 5 周）

第 4 周为第 5 周的 LLM 整合提供搜索基础：

1. **LLM 集成**：连接 Ollama 以生成答案
2. **RAG管道**：查询→搜索→上下文→生成→响应  
3. **上下文管理**：优化LLM输入检索到的块
4. **答案质量**：实施引用和来源归属
5. **对话记忆**：支持多轮对话

混合搜索系统**可用于生产**，并提供高质量 RAG 应用程序所需的检索精度。

## 文件结构

```
src/
├── routers/
│   └── hybrid_search.py          # FastAPI endpoints
├── services/
│   ├── opensearch/
│   │   ├── client.py              # Unified search client
│   │   ├── factory.py             # Client factory
│   │   └── index_config_hybrid.py # Index configuration
│   ├── indexing/
│   │   ├── text_chunker.py        # Section-based chunking
│   │   ├── hybrid_indexer.py      # Document indexing
│   │   └── factory.py             # Indexing service factory
│   └── embeddings/
│       ├── jina_client.py         # Jina AI client
│       └── factory.py             # Embedding service factory
├── schemas/
│   └── api/
│       └── search.py              # Request/response models
└── config.py                      # Configuration management

notebooks/week4/
├── README.md                      # This document
├── week4_hybrid_search.ipynb      # Interactive tutorial
└── data/                          # Sample data directory
```

＃＃ 资源

- **OpenSearch 文档**：https://opensearch.org/docs/
- **Jina AI 嵌入**：https://jina.ai/embeddings/
- **FastAPI 文档**：https://fastapi.tiangolo.com/
- **倒数等级融合纸**：https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf
- **第 4 周笔记本**：[week4_hybrid_search.ipynb](./week4_hybrid_search.ipynb)
