---
title: Hybrid Search with pgvector + RRF
problem: Pure vector or BM25 search alone is insufficient for production quality
context: Choosing search architecture, seeing poor recall in retrieval systems
category: search-retrieval
tags: [hybrid-search, rrf, pgvector, bm25, postgres, retrieval]
version: "1.0"
last_updated: 2026-01-27
source: https://github.com/rhea-impact/taskr/blob/main/docs/research/hybrid-search-pgvector-rrf.md
---

# Hybrid Search: pgvector + RRF

> **TL;DR**: Weaviate "felt" good because of tuned HNSW and first-class hybrid search. You can achieve comparable or better quality with **Postgres + pgvector + RRF** if you replicate those pieces carefully.

## Problem Statement

A naive pgvector `ORDER BY embedding <-> query_vec LIMIT k` over an un-tuned index with no hybrid scoring will always feel worse than dedicated vector databases.

**Why?** You're missing:
1. Tuned HNSW parameters
2. Lexical matching for exact terms
3. Rank fusion to combine signals

## Solution: RRF Hybrid Search

### Why RRF Works

- **Rank-based fusion**: Works on ranks, not raw scores - no painful score normalization
- **Best of both worlds**: Documents ranking well in both lexical AND semantic lists bubble to top
- **Extensible**: Add recency, user-weights, source boosts without changing storage layer

### RRF Formula

```
RRF_score(d) = 1/(k + rank_vec(d)) + 1/(k + rank_bm25(d))
```

Where `k` is typically **60** (dampening factor).

### SQL Implementation

```sql
-- Hybrid search with RRF fusion
WITH vector_search AS (
  SELECT id, ROW_NUMBER() OVER (ORDER BY embedding <-> $1) AS rank_vec
  FROM documents
  WHERE deleted_at IS NULL
  ORDER BY embedding <-> $1
  LIMIT 100
),
lexical_search AS (
  SELECT id, ROW_NUMBER() OVER (ORDER BY ts_rank_cd(search_vector, query) DESC) AS rank_bm25
  FROM documents, plainto_tsquery('english', $2) query
  WHERE search_vector @@ query AND deleted_at IS NULL
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
SELECT d.*, (rrf_vec + rrf_bm25) AS rrf_score
FROM rrf_scores r
JOIN documents d ON d.id = r.id
ORDER BY rrf_score DESC
LIMIT $3;
```

## HNSW Index Configuration

```sql
-- Create HNSW index (Postgres 15+ with pgvector 0.5+)
CREATE INDEX documents_embedding_hnsw_idx
ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Set ef for queries (higher = better recall, slower)
SET hnsw.ef_search = 100;
```

### Recommended Parameters by Dataset Size

| Dataset Size | m | ef_construction | ef_search |
|-------------|---|-----------------|----------|
| < 10k | 16 | 64 | 40 |
| 10k - 100k | 16 | 100 | 100 |
| 100k - 1M | 24 | 200 | 200 |
| > 1M | 32 | 256 | 256 |

## Tunable Parameters

| Parameter | Default | Purpose |
|-----------|---------|--------|
| `rrf_k` | 60 | Dampening factor - higher = more weight to lower ranks |
| `vector_weight` | 5 | Multiplier for semantic similarity |
| `bm25_weight` | 3 | Multiplier for lexical matches |
| `recency_halflife` | 30 days | How fast recency decays |
| `limit_per_source` | 100 | How many candidates from each search |

## Trade-offs vs Dedicated Vector DBs

| Aspect | Weaviate | Postgres + pgvector |
|--------|----------|--------------------|
| ANN Performance | Heavily optimized | Improving, may lag at very large scale |
| Ergonomics | Batteries included | DIY hybrid search |
| Consistency | Eventually consistent | ACID transactions |
| Infra | Separate service | Same database |
| Ranking Control | Limited | Full transparency |

**For workloads with tens-hundreds of thousands of chunks, moderate QPS, and need for tunable ranking**: Postgres+pgvector+RRF can be strictly better on quality and control.

## References

- [ParadeDB: Hybrid Search in PostgreSQL](https://www.paradedb.com/blog/hybrid-search-in-postgresql-the-missing-manual)
- [Jonathan Katz: Hybrid Search with Postgres](https://jkatz.github.io/post/postgres/hybrid-search-postgres-pgvector/)
- [TigerData: True BM25 in Postgres](https://www.tigerdata.com/blog/introducing-pg_textsearch-true-bm25-ranking-hybrid-retrieval-postgres)
