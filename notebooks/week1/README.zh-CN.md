# 第 1 周：基础设施搭建与验证

本目录包含 arXiv 论文助手项目第 1 周的学习资料，重点是搭建并验证完整的基础设施栈。

## 内容

### `week1_setup.ipynb`

这是一份完整的 Jupyter Notebook，将引导学生学习：

1. **系统要求与搭建**
   - 理解各项技术组件及其用途
   - 跨平台安装说明（Windows、macOS、Linux）
   - 通过自动检查验证前置条件

2. **基础设施架构**
   - 全面了解多服务架构
   - 理解 Docker 容器之间如何通信
   - 数据持久化与卷管理概念

<p align="center">
  <img src="../../static/week1_infra_setup.png" alt="第 1 周基础设施搭建" width="700">
</p>

**架构概览：**

- **FastAPI**（端口 8000）：支持异步处理和自动生成文档的 REST API
- **PostgreSQL 16**（端口 5432）：用于存储论文元数据和内容的主数据库
- **OpenSearch 2.19**（端口 9200、5601）：带管理仪表板的混合搜索引擎
- **Apache Airflow 3.0**（端口 8080）：使用 DAG 和 PostgreSQL 后端的工作流编排系统
- **Ollama**（端口 11434）：供后续 RAG 实现使用的本地 LLM 服务器
- **Docker 网络**：所有服务通过 `rag-network` 通信，并使用持久化卷

3. **逐项搭建服务**
   - 用 PostgreSQL 数据库存储论文元数据
   - 用 OpenSearch 实现全文搜索
   - 用 Apache Airflow 实现工作流自动化
   - 用 Ollama 进行本地 LLM 推理
   - 用 FastAPI 提供 REST API 端点

4. **验证与测试**
   - 对所有服务执行自动健康检查
   - 分步骤完成验证
   - 模块化 Ollama 测试（4 个针对性测试单元格）
   - 常见故障排查场景及解决方案

## 学习目标

完成本周内容后，学生将能够：

- 理解容器化和 Docker Compose 编排
- 学会搭建生产级基础设施栈
- 获得数据库设计与 API 开发经验
- 掌握多服务应用的故障排查技巧
- 了解直接测试 HTTP API 与通过服务抽象层测试的区别
- 建立使用专业开发工具的信心

## Ollama 测试（第 1 周简化版）

Notebook 将 Ollama 测试拆分为多个目标明确的单元格：

- **测试 3A**：检查可用模型
- **测试 3B**：简单模型测试（如果已安装模型）
- **测试 3C**：性能分析
- **测试 3D**：学习笔记与搭建命令

### 轻松安装模型（第 1 周可选）

```bash
# 使用 Makefile（推荐）
make ollama-pull MODEL=llama3.2:1b
make ollama-test MODEL=llama3.2:1b

# 直接调用 HTTP 接口以便学习
curl -X POST http://localhost:11434/api/pull -d '{"name":"llama3.2:1b"}'
curl -X POST http://localhost:11434/api/generate -d '{"model":"llama3.2:1b","prompt":"Hello","stream":false}'
```

### 课程推荐模型

- **llama3.2:1b**（1.2GB）——速度快，适合测试
- **llama3.2:3b**（2.0GB）——速度与质量较均衡
- **llama3.1:8b**（4.7GB）——质量更好，但速度较慢

**注意**：第 1 周不要求安装任何模型；没有模型也可以完成服务健康检查。

## 目标读者

本资料适合：

- 希望学习现代软件基础设施的**初学者**
- 希望了解真实应用如何构建的**学生**
- 正在转向软件开发或 DevOps 的**专业人士**
- 对构建自己的 AI 科研工具感兴趣的**任何人**

## 时间投入

- **环境搭建**：2～3 小时（包括软件安装和下载）
- **完成 Notebook**：1 小时
- **总计**：2～4 小时

## 📖 补充资源

**第 1 周博客文章：** [为 RAG 系统提供动力的基础设施](https://jamwithai.substack.com/p/the-infrastructure-that-powers-rag)

- 深入了解各项基础设施组件
- 生产环境部署注意事项
- 架构决策说明

## 支持资源

如果遇到问题：

1. 查看 Notebook 中的故障排查章节
2. 阅读常见问题及解决方案
3. 确认所有前置条件都已正确安装
4. 按照分步验证流程操作
5. 在 Jam With AI 的 Substack 聊天频道提问

## 后续步骤

完成第 1 周后，你将能够：

- 理解各项服务对整个系统的作用
- 根据需要修改和扩展基础设施
- 继续学习第 2 周：arXiv 集成与 PDF 处理
- 更有信心地使用专业开发环境
