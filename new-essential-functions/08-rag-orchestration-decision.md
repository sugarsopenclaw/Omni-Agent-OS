# 08 - RAG Orchestration 与 Citation 机制 架构决策

**技术点**: RAG Orchestration 与 Citation 机制

**分析日期**: 2026-03-21

---

## 1. 已有实现

### 1.1 CoPaw (Python) — ReMe Hybrid Search

**位置**: `reference-source-code/personal-general-ai/CoPaw/src/copaw/agents/tools/memory_search.py`

**核心设计**:

```python
# Hybrid Search (Vector + BM25)
search_config = {
    "vector_weight": 0.7,
    "bm25_weight": 0.3,
    "candidate_multiplier": 3,
    "max_candidates": 200,
}

async def memory_search(
    query: str,
    max_results: int = 5,
    min_score: float = 0.1,
) -> ToolResponse:
    """Search MEMORY.md and memory/*.md files semantically.
    Uses hybrid search (vector + BM25) with weighted fusion.
    """
    results = await reme.search(
        query=query,
        top_k=max_results,
        min_score=min_score,
    )
    return ToolResponse(results=results)
```

**特点**: Vector + BM25 混合搜索、加权融合

---

### 1.2 IronClaw (Rust) — RRF Fusion + MMR

**位置**: `reference-source-code/personal-general-ai/ironclaw/src/workspace/search.rs`

**核心设计**:

```rust
pub struct HybridSearchConfig {
    pub vector_weight: f64,     // 0.6 default
    pub fts_weight: f64,        // 0.4 default
    pub use_rrf: bool,          // RRF vs weighted fusion
    pub mmr_lambda: f64,        // 0.5-0.7 for diversity
    pub temporal_decay: TemporalDecayConfig,
}

pub async fn hybrid_search(
    &self,
    query: &str,
    limit: usize,
    config: &HybridSearchConfig,
) -> Result<Vec<SearchResult>> {
    // Parallel vector + FTS search
    let (vector_results, fts_results) = tokio::join!(
        self.vector_search(query, limit * 2),
        self.fts_search(query, limit * 2),
    );

    // Fuse results
    let fused = if config.use_rrf {
        rrf_fusion(vector_results?, fts_results?, 60.0)
    } else {
        weighted_fusion(vector_results?, fts_results?, config.vector_weight, config.fts_weight)
    };

    // Rerank with MMR
    if config.mmr_lambda > 0.0 {
        mmr_rerank(fused, config.mmr_lambda, limit)
    } else {
        Ok(fused.into_iter().take(limit).collect())
    }
}
```

**特点**: RRF 融合、MMR 重排序、时间衰减

---

### 1.3 OpenClaw (TypeScript) — 最复杂的 RAG

**位置**: `reference-source-code/personal-general-ai/openclaw/docs/concepts/memory.md`

**核心设计**:

```typescript
export type ResolvedMemorySearchConfig = {
  enabled: boolean;
  sources: Array<"memory" | "sessions">;
  provider: "openai" | "local" | "gemini" | "voyage" | "auto";
  model: string;

  // Chunking
  chunkTokens: number;           // 400 default
  chunkOverlapTokens: number;    // 80 default (20%)

  // Hybrid search
  vectorWeight: number;          // 0.7
  textWeight: number;            // 0.3
  candidateMultiplier: number;   // 4x

  // MMR Reranking
  mmr: {
    enabled: boolean;
    lambda: number;              // 0.7 diversity vs relevance
    diversity: number;           // 3 candidates
  };

  // Temporal decay
  temporalDecay: {
    enabled: boolean;
    halfLifeDays: number;        // 30 days
  };
};
```

**搜索管线**:
```
Query → Vector Search (0.7) ─┐
      → BM25/FTS5 (0.3) ────┤→ Fusion → MMR → Temporal Decay → Results
```

---

## 2. 值得直接复用

### 2.1 Hybrid Search (Vector + BM25) ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- 向量搜索 (语义)
- BM25 (关键词)
- 互补优势

**复用方式**:
```python
# Vector + BM25 并行搜索
vector_results = vector_search(query)
bm25_results = bm25_search(query)

# 加权融合
fused = weighted_fusion(vector_results, bm25_results, 0.7, 0.3)
```

---

### 2.2 RRF Fusion ⭐⭐⭐⭐⭐

**推荐指数**: 5/5

**理由**:
- 无需调参
- 效果稳定

```python
def rrf_fusion(results_list, k=60):
    scores = {}
    for results in results_list:
        for rank, doc in enumerate(results):
            scores[doc.id] = scores.get(doc.id, 0) + 1 / (k + rank)
    return sorted(scores.items(), key=lambda x: -x[1])
```

---

### 2.3 MMR Reranking ⭐⭐⭐⭐

**推荐指数**: 4/5

**理由**:
- 平衡相关性
- 增加多样性

---

### 2.4 Temporal Decay ⭐⭐⭐⭐

**推荐指数**: 4/5

**理由**:
- 时间敏感
- 近期优先

---

## 3. 需要垂直特化

### 3.1 Citation 机制

**通用**: 返回来源

**垂直特化**:
- 医疗: 引用临床指南
- 法律: 引用法条/判例
- 学术: 引用论文

### 3.2 领域特定 Chunking

- 医疗: 按病历结构
- 法律: 按条款
- 代码: 按函数

---

## 4. 推荐接口

```rust
pub trait RAGPipeline: Send + Sync {
    async fn search(&self, query: &str) -> Result<Vec<SearchResult>>;
    fn chunk(&self, doc: &Document) -> Vec<Chunk>;
    async fn embed(&self, chunks: &[Chunk]) -> Result<Vec<Vector>>;
}

pub struct SearchResult {
    pub content: String,
    pub source: String,
    pub score: f64,
    pub citation: Citation,
}
```

---

## 5. MVP 方案

```python
class SimpleRAG:
    def __init__(self, embedder, vector_store):
        self.embedder = embedder
        self.store = vector_store

    def add_document(self, doc: str):
        chunks = self.chunk(doc)
        vectors = self.embedder.encode(chunks)
        self.store.add(vectors, chunks)

    def search(self, query: str, k: int = 5) -> List[str]:
        query_vec = self.embedder.encode([query])[0]
        return self.store.search(query_vec, k)

    def chunk(self, text: str) -> List[str]:
        # Simple chunking
        words = text.split()
        chunks = []
        for i in range(0, len(words), 100):
            chunks.append(" ".join(words[i:i+100]))
        return chunks
```

---

## 6. 扩展路线

1. Hybrid Search
2. RRF Fusion
3. MMR Rerank
4. Citation 机制

---

*文档版本：1.0*
*最后更新：2026-03-21*
