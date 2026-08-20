<!-- 本文件由对应 Jupyter Notebook 转换并翻译生成；代码单元保持原样。 -->

# 第 2 周：arXiv API 集成和 PDF 处理

**我们本周要构建的内容：**

第 2 周重点关注实现核心数据摄取流水线，该流水线将自动获取、处理和存储 arXiv 论文。这是为我们的 RAG 系统提供最新学术内容的基础。

## 第 2 周重点领域

### 🎯 核心目标
- **arXiv API 集成**：构建具有速率限制和重试逻辑的强大客户端
- **PDF 处理管道**：通过结构化内容提取下载和解析科学 PDF
- **数据库存储**：在 PostgreSQL 中保留论文元数据和内容
- **错误处理**：实施全面的错误处理和优雅的降级
- **自动化就绪**：为 Airflow 编排准备组件

### 🔧 我们将在此笔记本中测试什么
1. **arXiv API 客户端** - 获取具有适当速率限制的 CS.AI 论文
2. **PDF下载系统** - 下载并缓存PDF并进行错误处理  
3. **Docling PDF Parser** - 提取结构化内容（部分、表格、图形）
4. **数据库集成** - 从 PostgreSQL 存储和检索论文
5. **完整的管道** - 从 arXiv 到数据库的端到端处理
6. **生产就绪情况** - 错误处理、日志记录和性能指标


### 📊 成功指标
- arXiv API 调用在适当的速率限制下成功
- PDF下载和缓存工作可靠  
- Docling 从科学 PDF 中提取结构化内容
- 数据库存储完整的论文元数据
- 管道优雅地处理错误并继续处理
- 所有组件均已准备好通过 Airflow 实现自动化（第 2 周及以后）

---

## 第 2 周组件状态
|组件|目的|状态 |
|------------|---------|--------|
| **arXiv API 客户端** |限速获取 CS.AI 论文 | ✅ 完整 |
| **PDF 下载器** |在本地下载并缓存 PDF | ✅ 完整 |
| **Docling 解析器** |从 PDF 中提取结构化内容 | ✅ 完整 |
| **元数据获取器** |编排完整的管道 | ✅ 完整 |
| **数据库存储** |在 PostgreSQL 中存储论文 | ⚠️ 需要音量刷新 |
| **Airflow DAG** |自动化日常摄取 | ⚠️ 需要容器更新 |

## ⚠️ 重要提示：第​​ 2 周数据库架构更新

**新用户或架构冲突**：如果您刚刚开始第 2 周或遇到数据库架构冲突，请使用以下全新开始方法：

### 全新开始（建议第 2 周）
```bash
# Complete clean slate - removes all data but ensures correct schema
docker compose down -v

# Build fresh containers with latest code
docker compose up --build -d
```

**何时使用这个：**
- 首次运行第 2 周内容 
- 架构错误或列缺失错误
- 想要从干净的数据库开始
- 前一周 1 的数据并不重要

**注意**：这会破坏现有数据，但可确保您拥有正确的第 2 周架构以及用于 PDF 处理和 arXiv 元数据的所有新列。

---

## 先决条件检查

**开始之前：**
1. 第一周基础设施完成
2. 紫外线环境激活
3. Docker桌面运行

**为什么要使用新容器？** 第 2 周包括新的 Airflow 依赖项和代码更改，需要重建图像而不是使用缓存层。

```python
# Check if Fresh Containers are Built and All Services Healthy
import subprocess
import requests
from pathlib import Path

print("WEEK 2 CONTAINER & SERVICE HEALTH CHECK")
print("=" * 50)

# Find project root
current_dir = Path.cwd()
if current_dir.name == "week2" and current_dir.parent.name == "notebooks":
    project_root = current_dir.parent.parent
elif (current_dir / "compose.yml").exists():
    project_root = current_dir
else:
    print("✗ Could not find project root")
    exit()

print(f"Project root: {project_root}")

# Step 1: Check if containers are built and running
print("\n1. Checking container status...")
try:
    result = subprocess.run(
        ["docker", "compose", "ps", "--format", "table"],
        cwd=str(project_root),
        capture_output=True,
        text=True,
        timeout=10
    )
    
    if result.returncode == 0 and result.stdout.strip():
        print("✓ Containers are running:")
        for line in result.stdout.strip().split('\n'):
            print(f"   {line}")
    else:
        print("✗ No containers running or docker compose failed")
        print("Please run the build commands from the markdown cell above")
        exit()
        
except Exception as e:
    print(f"✗ Error checking containers: {e}")
    print("Please run the build commands from the markdown cell above")
    exit()

# Step 2: Check all service health (corrected endpoints)
print("\n2. Checking service health...")
services_to_test = {
    "FastAPI": "http://localhost:8000/api/v1/health",
    "PostgreSQL (via API)": "http://localhost:8000/api/v1/health", 
    "Ollama": "http://localhost:11434/api/version",
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

print("\n" + "=" * 50)
if all_healthy:
    print("✓ ALL SERVICES HEALTHY! Ready for Week 2 development.")
else:
    print("✗ Some services need attention.")
    print("If you just rebuilt containers, wait 1-2 minutes and run this cell again.")
    print("Airflow and OpenSearch take longest to start up.")
```

```python
# Environment Check
import sys
from pathlib import Path

print(f"Python Version: {sys.version_info.major}.{sys.version_info.minor}.{sys.version_info.micro}")
print(f"Environment: {sys.executable}")

# Find project root
current_dir = Path.cwd()
if current_dir.name == "week2" and current_dir.parent.name == "notebooks":
    project_root = current_dir.parent.parent
elif (current_dir / "compose.yml").exists():
    project_root = current_dir
else:
    project_root = None

if project_root and (project_root / "compose.yml").exists():
    print(f"✓ Project root: {project_root}")
    # Add project to Python path
    sys.path.insert(0, str(project_root))
else:
    print("✗ Missing compose.yml - check directory")
    exit()
```

## 服务健康验证

确保第一周的所有服务仍然正常运行：

### 🔗 服务接入点
- **FastAPI**：http://localhost:8000/docs（API 文档）
- **PostgreSQL**：通过 API 或 `docker exec -it rag-postgres psql -U rag_user -d rag_db`
- **OpenSearch**：http://localhost:9200/_cluster/health
- **Ollama**：http://localhost:11434（LLM服务）
- **Airflow**：http://localhost:8080（用户名：`admin`，密码：`admin`）

```python
# Test Service Connectivity
import requests
import subprocess
import json

services_to_test = {
    "FastAPI": "http://localhost:8000/api/v1/health",
    "PostgreSQL (via API)": "http://localhost:8000/api/v1/health", 
    "Ollama": "http://localhost:11434/api/version",
    "OpenSearch": "http://localhost:9200/_cluster/health",
    "Airflow": "http://localhost:8080/health"  
}

print("WEEK 2 PREREQUISITE CHECK")
print("=" * 50)

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
    print("All services healthy! Ready for Week 2 development.")
else:
    print("Some services need attention. Check Week 1 notebook.")
```

## 1. arXiv API 客户端测试

使用速率限制和重试逻辑测试 arXiv API 客户端：

```python
# Test arXiv API Client
import asyncio
from datetime import datetime, timedelta

# Import our arXiv client
from src.services.arxiv.factory import make_arxiv_client

print("TESTING ARXIV API CLIENT")
print("=" * 40)

# Create client
arxiv_client = make_arxiv_client()
print(f"✓ Client created: {arxiv_client.base_url}")
print(f"   Rate limit: {arxiv_client.rate_limit_delay}s")
print(f"   Max results: {arxiv_client.max_results}")
print(f"   Category: {arxiv_client.search_category}")
print()
```

```python
# Test Paper Fetching
async def test_paper_fetching():
    """Test fetching papers from arXiv with rate limiting."""
    
    print("Test 1: Fetch Recent CS.AI Papers")
    try:
        papers = await arxiv_client.fetch_papers(
            max_results=2, 
            sort_by="submittedDate",
            sort_order="descending"
        )
        
        print(f"✓ Fetched {len(papers)} papers")
        
        if papers:
            for i, paper in enumerate(papers[:2], 1):
                print(f"   {i}. [{paper.arxiv_id}] {paper.title[:60]}...")
                print(f"      Authors: {', '.join(paper.authors[:2])}{'...' if len(paper.authors) > 2 else ''}")
                print(f"      Categories: {', '.join(paper.categories)}")
                print(f"      Published: {paper.published_date}")
                print()
        
        return papers
        
    except Exception as e:
        print(f"✗ Error fetching papers: {e}")
        if "503" in str(e):
            print("   arXiv API temporarily unavailable (normal)")
            print("   Rate limiting and error handling working correctly")
        return []

# Run the test
papers = await test_paper_fetching()
```

```python
# Test Date Filtering
async def test_date_filtering():
    """Test date range filtering functionality."""
    
    print("Test 2: Date Range Filtering")
    
    # Use specific dates: 
    from_date = "20250808"  
    to_date = "20250809"    
    try:
        date_papers = await arxiv_client.fetch_papers(
            max_results=5,
            from_date=from_date,
            to_date=to_date
        )
        
        print(f"✓ Date filtering test: {len(date_papers)} papers from {from_date}-{to_date}")
        
        if date_papers:
            for i, paper in enumerate(date_papers, 1):
                print(f"   {i}. [{paper.arxiv_id}] {paper.title[:60]}...")
                print(f"      Authors: {', '.join(paper.authors[:2])}{'...' if len(paper.authors) > 2 else ''}")
                print(f"      Categories: {', '.join(paper.categories)}")
                print(f"      Published: {paper.published_date}")
                print()
        
        return date_papers
        
    except Exception as e:
        print(f"✗ Date filtering error: {e}")
        return []

# Run date filtering test
date_papers = await test_date_filtering()
```

## 2. PDF下载和缓存

使用缓存测试 PDF 下载功能：

```python
# Test PDF Download
async def test_pdf_download(test_papers):
    """Test PDF downloading with caching."""

    print("Test 3: PDF Download & Caching")
    
    if not test_papers:
        print("No papers available for PDF download test")
        return None
    
    # Test with first paper
    test_paper = test_papers[0]
    print(f"Testing PDF download for: {test_paper.arxiv_id}")
    print(f"Title: {test_paper.title[:60]}...")
    
    try:
        # Download PDF 
        pdf_path = await arxiv_client.download_pdf(test_paper)
        
        if pdf_path and pdf_path.exists():
            size_mb = pdf_path.stat().st_size / (1024 * 1024)
            print(f"✓ PDF downloaded: {pdf_path.name} ({size_mb:.2f} MB)")
            
            return pdf_path
        else:
            print("✗ PDF download failed")
            return None
            
    except Exception as e:
        print(f"✗ PDF download error: {e}")
        return None

# Run PDF download test 
pdf_path = await test_pdf_download(date_papers[:1])
```

## 3. 文档 PDF 处理

使用 Docling 测试 PDF 解析以进行结构化内容提取：

```python
# Test PDF Parsing with Docling
from src.services.pdf_parser.factory import make_pdf_parser_service
from src.config import get_settings
from pathlib import Path

print("Test 4: PDF Parsing with Docling")
print("=" * 40)

# Create PDF parser
pdf_parser = make_pdf_parser_service()
settings = get_settings()
print("PDF parser service created")
print(f"Config: {settings.pdf_parser.max_pages} pages, {settings.pdf_parser.max_file_size_mb}MB")

# Test parsing with actual PDF files
cache_dir = Path("data/arxiv_pdfs")
if cache_dir.exists():
    pdf_files = list(cache_dir.glob("*.pdf"))
    print(f"\nFound {len(pdf_files)} PDF files to test parsing")
    
    if pdf_files:
        # Test parsing the first PDF
        test_pdf = pdf_files[0]
        print(f"Testing PDF parsing with: {test_pdf.name}")
        
        try:
            pdf_content = await pdf_parser.parse_pdf(test_pdf)
            
            if pdf_content:
                print(f"✓ PDF parsing successful!")
                print(f"  Sections: {len(pdf_content.sections)}")
                print(f"  Raw text length: {len(pdf_content.raw_text)} characters")
                print(f"  Parser used: {pdf_content.parser_used}")
                
                # Show first section as example
                if pdf_content.sections:
                    first_section = pdf_content.sections[0]
                    print(f"  First section: '{first_section.title}' ({len(first_section.content)} chars)")
            else:
                print("✗ PDF parsing failed (Docling compatibility issue)")
                print("This is expected - not all PDFs work with Docling")
                
        except Exception as e:
            print(f"✗ PDF parsing error: {e}")
            print("This demonstrates the error handling in action")
    else:
        print("No PDF files available for parsing test")
else:
    print("No PDF cache directory found")
```

## 4. 数据库存储测试

测试将论文存储在 PostgreSQL 数据库中：

```python
# Test Database Storage
from src.db.factory import make_database
from src.repositories.paper import PaperRepository
from src.schemas.arxiv.paper import PaperCreate
from dateutil import parser as date_parser

print("Test 5: Database Storage")
print("=" * 40)

# Create database connection
database = make_database()
print("✓ Database connection created")

if papers:
    test_paper = papers[0]
    print(f"Storing paper: {test_paper.arxiv_id}")
    
    try:
        with database.get_session() as session:
            paper_repo = PaperRepository(session)
            
            # Convert to database format
            published_date = date_parser.parse(test_paper.published_date) if isinstance(test_paper.published_date, str) else test_paper.published_date
            
            paper_create = PaperCreate(
                arxiv_id=test_paper.arxiv_id,
                title=test_paper.title,
                authors=test_paper.authors,
                abstract=test_paper.abstract,
                categories=test_paper.categories,
                published_date=published_date,
                pdf_url=test_paper.pdf_url
            )
            
            # Store paper (upsert to avoid duplicates)
            stored_paper = paper_repo.upsert(paper_create)
            
            if stored_paper:
                print(f"✓ Paper stored with ID: {stored_paper.id}")
                print(f"   Database ID: {stored_paper.id}")
                print(f"   arXiv ID: {stored_paper.arxiv_id}")
                print(f"   Title: {stored_paper.title[:50]}...")
                print(f"   Authors: {len(stored_paper.authors)} authors")
                print(f"   Categories: {', '.join(stored_paper.categories)}")
                
                # Test retrieval
                retrieved_paper = paper_repo.get_by_arxiv_id(test_paper.arxiv_id)
                if retrieved_paper:
                    print(f"✓ Paper retrieval test passed")
                else:
                    print(f"✗ Paper retrieval failed")
            else:
                print("✗ Paper storage failed")
                
    except Exception as e:
        print(f"✗ Database error: {e}")
else:
    print("No papers available for database storage test")
```

```python
# Test Complete Pipeline
from src.services.metadata_fetcher import make_metadata_fetcher

print("Test 6: Complete Metadata Fetcher Pipeline")
print("=" * 50)

# Create metadata fetcher
metadata_fetcher = make_metadata_fetcher(arxiv_client, pdf_parser)
print("✓ Metadata fetcher service created")

# Test with small batch
print("Running small batch test (2 papers, no PDF processing for speed)...")

try:
    with database.get_session() as session:
        results = await metadata_fetcher.fetch_and_process_papers(
            max_results=2,  
            process_pdfs=False,  
            store_to_db=True,
            db_session=session
        )
    
    print("\nPIPELINE RESULTS:")
    print(f"   Papers fetched: {results.get('papers_fetched', 0)}")
    print(f"   PDFs downloaded: {results.get('pdfs_downloaded', 0)}")
    print(f"   PDFs parsed: {results.get('pdfs_parsed', 0)}")
    print(f"   Papers stored: {results.get('papers_stored', 0)}")
    print(f"   Processing time: {results.get('processing_time', 0):.1f}s")
    print(f"   Errors: {len(results.get('errors', []))}")
    
    if results.get('errors'):
        print("\nErrors encountered:")
        for error in results.get('errors', [])[:3]:  # Show first 3 errors
            print(f"   - {error}")
    
    if results.get('papers_fetched', 0) > 0:
        print("\n✓ Pipeline test successful!")
    else:
        print("\nNo papers fetched - may be arXiv API unavailability")
        
except Exception as e:
    print(f"✗ Pipeline error: {e}")
```

```python
# Test Airflow DAGs
print("Test 7: Airflow DAG Status")
print("=" * 40)

print("  Airflow UI Access:")
print("   URL: http://localhost:8080")
print("   Username: admin")
print("   Password: admin")
print()

# Check DAG status using docker exec
try:
    result = subprocess.run(
        ["docker", "exec", "rag-airflow", "airflow", "dags", "list"],
        capture_output=True,
        text=True,
        timeout=10
    )
    
    if result.returncode == 0:
        lines = result.stdout.strip().split('\n')
        dag_lines = [line for line in lines if 'arxiv' in line.lower() or 'hello' in line.lower()]
        
        print("Available DAGs:")
        for line in dag_lines:
            if '|' in line:
                parts = [part.strip() for part in line.split('|')]
                if len(parts) >= 3:
                    dag_id = parts[0]
                    is_paused = parts[2]
                    status = "Active" if is_paused == "False" else "Paused"
                    print(f"   - {dag_id}: {status}")
        
        # Check for import errors
        error_result = subprocess.run(
            ["docker", "exec", "rag-airflow", "airflow", "dags", "list-import-errors"],
            capture_output=True,
            text=True,
            timeout=10
        )
        
        if "docling" in error_result.stderr:
            print("\nKnown Issue: Docling not installed in Airflow container")
            print("   - This is expected for Week 2")
            print("   - DAG structure is complete, runtime needs container fix")
            print("   - Solution: Add docling to Airflow container startup")
        elif error_result.returncode == 0:
            print("\n✓ No DAG import errors found")
        
    else:
        print(f"✗ Could not list DAGs: {result.stderr}")
        
except Exception as e:
    print(f"✗ Airflow test error: {e}")

print("\n  To view DAGs graphically:")
print("   1. Open http://localhost:8080 in your browser")
print("   2. Login with admin/admin")
print("   3. Click on 'arxiv_paper_ingestion' DAG to see the workflow")
```

```python
# Test Complete Pipeline with PDF Processing
print("Test 8: Complete Pipeline with PDF Processing")
print("=" * 50)

# Reuse metadata fetcher from Test 6
print("✓ Using metadata fetcher service from previous test")

# Test with small batch including PDF processing
print("Running enhanced test (3 papers with PDF processing)...")

try:
    with database.get_session() as session:
        results = await metadata_fetcher.fetch_and_process_papers(
            max_results=3,  # Small batch
            from_date="20250813",  # Recent date
            to_date="20250814",
            process_pdfs=True,  
            store_to_db=True,
            db_session=session
        )
    
    print("\nENHANCED PIPELINE RESULTS:")
    print(f"   Papers fetched: {results.get('papers_fetched', 0)}")
    print(f"   PDFs downloaded: {results.get('pdfs_downloaded', 0)}")
    print(f"   PDFs parsed: {results.get('pdfs_parsed', 0)}")
    print(f"   Papers stored: {results.get('papers_stored', 0)}")
    print(f"   Processing time: {results.get('processing_time', 0):.1f}s")
    print(f"   Errors: {len(results.get('errors', []))}")
    
    # Show success rates
    if results.get('papers_fetched', 0) > 0:
        download_rate = (results['pdfs_downloaded'] / results['papers_fetched']) * 100
        parse_rate = (results['pdfs_parsed'] / results['pdfs_downloaded']) * 100 if results.get('pdfs_downloaded', 0) > 0 else 0
        print(f"   Download success rate: {download_rate:.1f}%")
        print(f"   Parse success rate: {parse_rate:.1f}%")
    
    if results.get('errors'):
        print("\nErrors encountered (showing graceful error handling):")
        for error in results.get('errors', [])[:3]:  # Show first 3 errors
            print(f"   - {error}")
    
    if results.get('papers_fetched', 0) > 0:
        print("\n✓ Enhanced pipeline test successful!")
        if results.get('errors'):
            print("✓ System continued processing despite PDF failures")
    else:
        print("\n! No papers fetched - may be arXiv API unavailability")
        
except Exception as e:
    print(f"✗ Pipeline error: {e}")
```

```python

```
