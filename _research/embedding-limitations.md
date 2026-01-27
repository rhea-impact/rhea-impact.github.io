---
title: Theoretical Limitations of Embedding-Based Retrieval
problem: Dense vector embeddings fail at scale due to mathematical constraints
context: Building vector search, RAG, or semantic retrieval with >100k documents
category: search-retrieval
tags: [embeddings, vector-search, rag, hybrid-search, retrieval]
version: "1.0"
last_updated: 2026-01-27
source: https://github.com/rhea-impact/taskr/blob/main/docs/research/embedding-limitations-theory.md
---

# Theoretical Limitations of Embedding-Based Retrieval

> **TL;DR**: Dense vector embeddings have mathematical limitations that cannot be overcome with better training or larger models. At web scale, there exist document combinations that no query can retrieve under the single-vector paradigm. This is why hybrid search (BM25 + vectors) is essential.

## Problem Statement

If you're building:
- Vector search over >100k documents
- RAG systems at scale  
- Semantic retrieval for production use

Dense vectors **will fail** on some queries - not because of bad embeddings, but because of **mathematical impossibility**.

## Key Finding: The Embedding Dimension Bottleneck

The number of top-k subsets of documents that can be returned by any query is **fundamentally limited by embedding dimension**.

### Critical Corpus Sizes by Dimension

| Embedding Dim | Critical Corpus Size |
|---------------|---------------------|
| 512 | ~500,000 documents |
| 768 | ~1.7 million |
| 1024 | ~4 million |
| 3072 | ~107 million |
| 4096 | ~250 million |

**Implication**: For web-scale search (billions of documents), even 4096-dimensional embeddings with ideal optimization cannot model all needed combinations.

## The Math (Simplified)

In a d-dimensional space, there are only so many ways to partition documents into "closer" and "farther" from a query point.

The number of possible dichotomies is bounded by:
```
# of dichotomies ≤ 2 * (n choose 0) + (n choose 1) + ... + (n choose d)
```

For n >> d, this grows polynomially in n, not exponentially. But the number of possible top-k rankings grows combinatorially.

**Result**: There exist top-k combinations that no embedding geometry can produce.

## Solution: Hybrid Search

| Approach | Failure Mode |
|----------|-------------|
| Dense only | Misses obvious keyword matches; false positives at scale |
| BM25 only | Misses semantic similarity ("auth" vs "authentication") |
| Hybrid RRF | Catches both; degrades gracefully |

### Implementation

1. **BM25/Lexical** catches exact matches dense vectors miss
2. **Dense vectors** catch semantic similarity BM25 misses  
3. **RRF fusion** combines both, surfacing docs that rank well in either

See: [Hybrid Search with RRF](/research/hybrid-search-rrf/)

## Recommendations

### When Building Search

1. **Never rely on dense vectors alone** for search
2. **Always implement hybrid** (BM25 + vector + RRF)
3. **Tune weights per use case**:
   - Log search (exact IDs, errors): weight BM25 higher
   - Semantic search (concepts): weight vectors higher
4. **Monitor for degradation** as corpus grows
5. **Consider re-ranking** for high-precision needs

### When Dense Vectors ARE Appropriate

- Small corpora (< 100k documents)
- Semantic similarity (not exact retrieval)
- Combined with lexical search
- As a candidate generator, not final ranker

## References

- [On the Theoretical Limitations of Embedding-Based Retrieval](https://arxiv.org/abs/2508.21038) - Weller et al., Google DeepMind, 2025
- [The Curse of Dense Low-Dimensional IR](https://arxiv.org/abs/2012.14210) - Reimers & Gurevych, ACL 2021
- [LIMIT Dataset & Code](https://github.com/google-deepmind/limit) - Google DeepMind
