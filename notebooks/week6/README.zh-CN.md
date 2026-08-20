# 第 6 周：使用 Langfuse 和 Redis 进行生产监控和缓存

＃＃ 概述

第 6 周为我们的 RAG 系统添加了生产级监控和智能缓存。我们集成 Langfuse 以实现完整的管道可观察性，并集成 Redis 以实现高性能响应缓存。

## 我们建造了什么

- **Langfuse 集成**：端到端 RAG 管道跟踪和分析
- **Redis 缓存**：重复查询的响应速度提高 150-400 倍
- **性能监控**：实时指标和系统运行状况
- **生产就绪**：企业级可观察性和优化

＃＃ 建筑学

<p align="center">
  <img src="../../static/week6_monitoring_and_caching.png" alt="Week 6 Monitoring & Caching Architecture" width="900">
  <br>
  <em>具有 Langfuse 跟踪和 Redis 缓存集成的第 6 周架构</em>
</p>

### 数据流
```
Query → Cache Check → [Hit: ~100ms] | [Miss: Full Pipeline ~15s] → Cache Store → Langfuse Trace
```

## 主要特点

### **Langfuse可观测性**
- 完整的 RAG 管道跟踪和性能故障
- 用户分析、查询模式和成功率跟踪
- 带有成本和使用指标的实时监控仪表板
- 具有答案相关性和来源归属的质量见解

### **Redis智能缓存**
- **精确匹配策略**：用于精确匹配的参数感知缓存键
- **性能**：重复查询的响应速度提高 150-400 倍（约 100 毫秒 vs 15-20 秒）
- **TTL 管理**：24 小时默认过期时间，可配置设置
- **未来增强**：可以升级为语义相似度缓存以进行模糊匹配

## 快速入门

### 环境设置
```bash
# Required environment variables
LANGFUSE__SECRET_KEY=sk_lf_your_secret_key
LANGFUSE__PUBLIC_KEY=pk_lf_your_public_key
REDIS__HOST=redis
REDIS__TTL_HOURS=24
```

### 启动服务
```bash
docker compose up --build -d
```

### 测试缓存性能
```bash
# First request (cache miss ~15-20s)
curl -X POST "http://localhost:8000/api/v1/ask" \
  -H "Content-Type: application/json" \
  -d '{"query": "What are transformers?", "top_k": 3}'

# Second identical request (cache hit ~100ms)
curl -X POST "http://localhost:8000/api/v1/ask" \
  -H "Content-Type: application/json" \
  -d '{"query": "What are transformers?", "top_k": 3}'
```

## 性能基准

|场景 |响应时间 |改进|
|----------|----------------|-------------|
| **缓存未命中** | 15-20 秒 |基线|
| **缓存命中** | 50-100 毫秒 | **快 150-400 倍** |
| **监控开销** | <2% |影响可忽略不计|

## 测试

### 运行笔记本
```bash
jupyter notebook notebooks/week6/week6_cache_testing.ipynb
```

### 监控系统健康状况
```bash
# Check Redis connectivity
redis-cli ping

# View cache statistics  
curl "http://localhost:8000/api/v1/health"

# Access Langfuse dashboard
# Visit: https://cloud.langfuse.com (or your self-hosted instance)
```

## 故障排除

|问题 |解决方案 |
|--------|----------|
| **缓存不工作** |检查Redis：`redis-cli ping` |
| **无朗福斯痕迹** |验证环境变量：`LANGFUSE__*` |
| **反应慢** |监控缓存命中率和系统资源|

## 后续步骤

- **增强缓存**：升级到语义相似性缓存以进行模糊匹配
- **高级分析**：自定义仪表板和 A/B 测试框架  
- **生产扩展**：分布式缓存和自动监控
- **质量优化**：用户反馈整合和答案评分

＃＃ 资源

- **笔记本**：[week6_cache_testing.ipynb](./week6_cache_testing.ipynb)
- **Langfuse仪表板**：https://cloud.langfuse.com
- **Redis 文档**：https://redis.io/docs

---

第 6 周将您的 RAG 系统转变为生产级服务，性能提高 150-400 倍并具有全面的可观察性。