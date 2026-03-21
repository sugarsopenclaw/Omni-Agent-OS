# 09 - Document Ingestion Pipeline 架构决策

**技术点**: Document Ingestion Pipeline

**分析日期**: 2026-03-21

---

## 1. 已有实现

### 1.1 IronClaw (Rust) — 多格式文档提取

**位置**: `reference-source-code/personal-general-ai/ironclaw/src/document_extraction/extractors.rs`

**核心设计**:

```rust
/// 根据 MIME 类型提取文本
pub fn extract_text(data: &[u8], mime: &str, filename: Option<&str>) -> Result<String, String> {
    let base_mime = mime.split(';').next().unwrap_or(mime).trim();

    match base_mime {
        // PDF
        "application/pdf" => extract_pdf(data),

        // Office XML
        "application/vnd.openxmlformats-officedocument.wordprocessingml.document" => extract_docx(data),
        "application/vnd.openxmlformats-officedocument.presentationml.presentation" => extract_pptx(data),
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet" => extract_xlsx(data),

        // 纯文本
        "text/plain" | "text/markdown" | "text/html" | "text/x-python" => extract_utf8(data),

        // RTF
        "application/rtf" | "text/rtf" => extract_rtf(data),

        // 回退：按文件扩展名
        _ => try_extract_by_extension(data, filename),
    }
}

fn extract_pdf(data: &[u8]) -> Result<String, String> {
    pdf_extract::extract_text_from_mem(data)
        .map(|t| t.trim().to_string())
}

fn extract_docx(data: &[u8]) -> Result<String, String> {
    extract_office_xml(data, "word/document.xml")
}

fn extract_office_xml(data: &[u8], content_path: &str) -> Result<String, String> {
    let cursor = std::io::Cursor::new(data);
    let mut archive = zip::ZipArchive::new(cursor)?;
    let mut file = archive.by_name(content_path)?;
    let mut xml = String::new();
    file.read_to_string(&mut xml)?;
    Ok(strip_xml_tags(&xml))
}
```

**支持格式**:
- PDF (pdf_extract)
- DOCX/PPTX/XLSX (Office XML)
- 纯文本 (Markdown, HTML, 代码)
- RTF

---

### 1.2 IronClaw (Rust) — 文档分块

**位置**: `reference-source-code/personal-general-ai/ironclaw/src/workspace/document.rs`

**核心设计**:

```rust
/// 记忆文档
pub struct MemoryDocument {
    pub id: Uuid,
    pub user_id: String,
    pub path: String,
    pub content: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

/// 文档分块
pub struct MemoryChunk {
    pub id: Uuid,
    pub document_id: Uuid,
    pub chunk_index: i32,
    pub content: String,
    pub embedding: Option<Vec<f32>>,
}

impl MemoryChunk {
    pub fn new(document_id: Uuid, chunk_index: i32, content: impl Into<String>) -> Self {
        Self {
            id: Uuid::new_v4(),
            document_id,
            chunk_index,
            content: content.into(),
            embedding: None,
            created_at: Utc::now(),
        }
    }
}
```

**标准文档路径**:
```rust
pub mod paths {
    pub const MEMORY: &str = "MEMORY.md";
    pub const IDENTITY: &str = "IDENTITY.md";
    pub const SOUL: &str = "SOUL.md";
    pub const AGENTS: &str = "AGENTS.md";
    pub const USER: &str = "USER.md";
    pub const HEARTBEAT: &str = "HEARTBEAT.md";
    pub const DAILY_DIR: &str = "daily/";
}
```

---

### 1.3 CoPaw (Python) — ReMe 集成

**位置**: `reference-source-code/personal-general-ai/CoPaw/src/copaw/agents/memory/memory_manager.py`

**核心设计**:

```python
class MemoryManager:
    """Memory management wrapper around ReMe library."""

    def __init__(self):
        self.vector_enabled = self._check_vector_config()
        self.memory_backend = self._select_backend()

    def _select_backend(self) -> str:
        """Select storage backend based on platform."""
        if platform.system() == "Windows":
            return "local"
        return "chroma"

    async def add_memory(self, content: str, metadata: Dict = None) -> bool:
        """Add content to memory with optional embedding."""
        return await self._reme.add(
            content=content,
            metadata=metadata or {},
            vectorize=self.vector_enabled,
        )

    async def search(self, query: str, top_k: int = 5) -> List[MemoryResult]:
        """Search memory with hybrid approach."""
        if self.vector_enabled:
            return await self._reme.hybrid_search(
                query=query,
                top_k=top_k,
                vector_weight=0.7,
                text_weight=0.3,
            )
        else:
            return await self._reme.fts_search(query=query, top_k=top_k)
```

**配置**:
```python
# Chunking 配置
chunk_tokens = 400          # 每块 token 数
chunk_overlap = 80          # 重叠 20%

# Embedding 配置
embedding_model = "text-embedding-v3"
embedding_dimensions = 1024
```

---

## 2. 值得直接复用

### 2.1 IronClaw 文档提取器 ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- 多格式支持 (PDF/Office/文本)
- MIME 类型检测
- 回退策略

**复用方式**:
```rust
// 直接使用 extract_text 函数
let text = extract_text(&data, mime, Some(filename))?;
```

---

### 2.2 标准文档路径 ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- 约定优于配置
- 清晰的语义

---

### 2.3 Token-based 分块 ⭐⭐⭐⭐

**推荐指数**: 4/5

**理由**:
- 400 tokens/块
- 20% 重叠
- 平衡粒度

---

## 3. 需要垂直特化

### 3.1 领域特定解析

- 医疗: DICOM, HL7
- 法律: 法律 XML
- 金融: XBRL

### 3.2 结构化提取

- 表格 → 结构化数据
- 图表 → 描述文本
- 代码 → AST

---

## 4. 推荐接口

```rust
pub trait DocumentExtractor: Send + Sync {
    fn extract(&self, data: &[u8], mime: &str) -> Result<String>;
    fn supported_formats(&self) -> Vec<String>;
}

pub trait DocumentChunker: Send + Sync {
    fn chunk(&self, text: &str) -> Vec<Chunk>;
}

pub struct Chunk {
    pub content: String,
    pub index: usize,
    pub metadata: ChunkMetadata,
}
```

---

## 5. MVP 方案

```python
class SimpleDocumentProcessor:
    def __init__(self):
        self.extractors = {
            "text/plain": self.extract_text,
            "text/markdown": self.extract_text,
            "application/pdf": self.extract_pdf,
        }

    def process(self, file_path: Path) -> List[Chunk]:
        # 1. 提取文本
        mime = self.detect_mime(file_path)
        text = self.extractors[mime](file_path)

        # 2. 分块
        chunks = self.chunk(text)

        return chunks

    def chunk(self, text: str, chunk_size: int = 400) -> List[Chunk]:
        words = text.split()
        chunks = []
        for i in range(0, len(words), chunk_size):
            content = " ".join(words[i:i+chunk_size])
            chunks.append(Chunk(content=content, index=len(chunks)))
        return chunks
```

---

## 6. 扩展路线

1. 多格式提取
2. 智能分块
3. 结构化提取
4. OCR 集成

---

*文档版本：1.0*
*最后更新：2026-03-21*
