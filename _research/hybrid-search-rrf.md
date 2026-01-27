---
title: Hybrid Search with pgvector + RRF
problem: Pure vector or BM25 search alone is insufficient for production quality
context: Choosing search architecture, seeing poor recall in retrieval systems
category: search-retrieval
tags: [hybrid-search, rrf, pgvector, bm25, postgres, retrieval]
version: "1.0"
last_updated: 2026-01-27
---

# Hybrid Search: pgvector + RRF vs Weaviate

> Building production-quality search with Postgres

## Summary

Weaviate "felt" good because of (a) a tuned HNSW index and (b) first-class hybrid search with ranking out of the box. We can achieve comparable or better quality with **Postgres + pgvector + RRF** if we replicate those pieces carefully.

## Why Weaviate Search Felt Strong

Under the hood, Weaviate isn't magic on the embedding side - its edge is the retrieval stack.

### Key Factors

1. **HNSW vector index tuned for recall/latency**
   - Configurable `ef`, `efConstruction`, `maxConnections`, distance metric
   - Dynamic `ef` for high recall at low latency without manual tuning
   - [Weaviate Vector Index Docs](https://docs.weaviate.io/weaviate/concepts/vector-index)

2. **Inverted index + vector index for hybrid search**
   - Combines BM25-like inverted index with HNSW vector index
   - Lexical matches AND semantic neighbors rank high together
   - [Performance Analysis](https://myscale.com/blog/weaviate-vs-chroma-performance-analysis-vector-databases/)

3. **Dynamic index behavior and caching**
   - Flat index for small sets, auto-switch to HNSW when collection grows
   - Vector caching smooths performance at scale

4. **Good defaults**
   - HNSW parameters, distance metric, and query behavior tuned for solid recall
   - "Just works" out of the box

**The gap**: A naive pgvector `ORDER BY embedding <-> query_vec LIMIT k` over an un-tuned index with no hybrid scoring will always feel worse.

## Can RRF + pgvector Beat Weaviate?

**Yes.** From a ranking-quality perspective, proper hybrid search (BM25 + vectors) with RRF fusion directly in Postgres can beat Weaviate's default behavior.

### Why RRF Helps

- **Rank-based fusion**: Works on ranks, not raw scores - no painful score normalization
- **Best of both worlds**: Documents ranking well in both lexical AND semantic lists bubble to top
- **Extensible**: Add recency, user-weights, source boosts without changing storage layer

### The Trade-offs

| Aspect | Weaviate | Postgres + pgvector |
|--------|----------|---------------------|
| ANN Performance | Heavily optimized | Improving, may lag at very large scale |
| Ergonomics | Batteries included | DIY hybrid search |
| Consistency | Eventually consistent | ACID transactions |
| Infra | Separate service | Same database |
| Ranking Control | Limited | Full transparency |

**For most workloads** (tens-hundreds of thousands of chunks, moderate QPS, need for tunable ranking): Postgres+pgvector+RRF can be strictly better on quality and "control surface," and good enough on latency.

## RRF Implementation

### Baseline RRF Formula

Standard RRF with two rankings (lexical and vector):

```
RRF_score(d) = 1/(k + rank_vec(d)) + 1/(k + rank_bm25(d))
```

Where `k` is typically **60** (dampening factor).

### SQL Pattern: pgvector + BM25

```sql
-- Hybrid search with RRF fusion
WITH vector_search AS (
    SELECT
        id,
        ROW_NUMBER() OVER (ORDER BY embedding <-> $1) AS rank_vec
    FROM documents
    WHERE deleted_at IS NULL
    ORDER BY embedding <-> $1
    LIMIT 100
),
lexical_search AS (
    SELECT
        id,
        ROW_NUMBER() OVER (ORDER BY ts_rank_cd(search_vector, query) DESC) AS rank_bm25
    FROM documents, plainto_tsquery('english', $2) query
    WHERE search_vector @@ query
      AND deleted_at IS NULL
    ORDER BY ts_rank_cd(search_vector, query) DESC
    LIMIT 100
),
rrf_scores AS (
    SELECT
        COALESCE(v.id, l.id) AS id,
        COALESCE(1.0 / (60 + v.rank_vec), 0.0) AS rrf_vec,
        COALESCE(1.0 / (60 + l.rank_bm25), 0.0) AS rrf_bm25
    FROM vector_search v
    FULL OUTER JOIN lexical_search l ON v.id = l.id
)
SELECT
    d.*,
    (rrf_vec + rrf_bm25) AS rrf_score
FROM rrf_scores r
JOIN documents d ON d.id = r.id
ORDER BY rrf_score DESC
LIMIT $3;
```

### Adding More Signals (Beyond Weaviate)

Once you own the ranking query, you can extend:

```sql
-- Extended RRF with recency and source weights
WITH ... AS (
    -- vector and lexical CTEs
),
rrf_scores AS (
    SELECT
        id,
        -- Base RRF
        COALESCE(1.0 / (60 + rank_vec), 0.0) AS rrf_vec,
        COALESCE(1.0 / (60 + rank_bm25), 0.0) AS rrf_bm25,
        -- Recency boost (exponential decay over 30 days)
        EXP(-EXTRACT(EPOCH FROM (NOW() - created_at)) / (30 * 86400)) AS recency,
        -- Source weight (decisions worth more than incidents)
        CASE category
            WHEN 'decision' THEN 1.5
            WHEN 'pattern' THEN 1.3
            ELSE 1.0
        END AS category_boost
    FROM ...
)
SELECT *,
    (rrf_vec * 5 + rrf_bm25 * 3 + recency * 0.2) * category_boost AS final_score
FROM rrf_scores
ORDER BY final_score DESC;
```

### Tunable Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `rrf_k` | 60 | Dampening factor - higher = more weight to lower ranks |
| `vector_weight` | 5 | Multiplier for semantic similarity |
| `bm25_weight` | 3 | Multiplier for lexical matches |
| `recency_halflife` | 30 days | How fast recency decays |
| `limit_per_source` | 100 | How many candidates from each search |

## Index Configuration

### HNSW Index for pgvector

```sql
-- Create HNSW index (Postgres 15+ with pgvector 0.5+)
CREATE INDEX documents_embedding_hnsw_idx ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Set ef for queries (higher = better recall, slower)
SET hnsw.ef_search = 100;
```

### Recommended HNSW Parameters

| Dataset Size | m | ef_construction | ef_search |
|-------------|---|-----------------|-----------|
| < 10k | 16 | 64 | 40 |
| 10k - 100k | 16 | 100 | 100 |
| 100k - 1M | 24 | 200 | 200 |
| > 1M | 32 | 256 | 256 |

## References

- [ParadeDB: Hybrid Search in PostgreSQL](https://www.paradedb.com/blog/hybrid-search-in-postgresql-the-missing-manual)
- [Jonathan Katz: Hybrid Search with Postgres](https://jkatz.github.io/post/postgres/hybrid-search-postgres-pgvector/)
- [TigerData: True BM25 in Postgres](https://www.tigerdata.com/blog/introducing-pg_textsearch-true-bm25-ranking-hybrid-retrieval-postgres)
- [Weaviate Vector Index Concepts](https://docs.weaviate.io/weaviate/concepts/vector-index)
- [DBI Services: RAG Hybrid Search with Re-ranking](https://www.dbi-services.com/blog/rag-series-hybrid-search-with-re-ranking/)

## See Also

- [Embedding Limitations](/research/embedding-limitations/) - Why hybrid search is essential
