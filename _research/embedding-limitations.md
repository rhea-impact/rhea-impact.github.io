---
title: Theoretical Limitations of Embedding-Based Retrieval
problem: Dense vector embeddings fail at scale due to mathematical constraints
context: Building vector search, RAG, or semantic retrieval with >100k documents
category: search-retrieval
tags: [embeddings, vector-search, rag, hybrid-search, retrieval]
version: "1.0"
last_updated: 2026-01-27
---

# Theoretical Limitations of Embedding-Based Retrieval

> Why dense vectors fundamentally cannot work at scale - and what to do about it

## TL;DR

Dense vector embeddings have **mathematical limitations** that cannot be overcome with better training or larger models. At web scale, there exist document combinations that **no query can retrieve** under the single-vector paradigm. This is why hybrid search (BM25 + vectors) is essential.

## Key Papers

### 1. On the Theoretical Limitations of Embedding-Based Retrieval

**Authors**: Orion Weller, Michael Boratko, Iftekhar Naim, Jinhyuk Lee (Google DeepMind)
**arXiv**: [2508.21038](https://arxiv.org/abs/2508.21038) (August 2025)
**Code/Data**: [google-deepmind/limit](https://github.com/google-deepmind/limit)

#### Abstract

> Vector embeddings have been tasked with an ever-increasing set of retrieval tasks over the years, with a nascent rise in using them for reasoning, instruction-following, coding, and more. These new benchmarks push embeddings to work for any query and any notion of relevance that could be given. While prior works have pointed out theoretical limitations of vector embeddings, there is a common assumption that these difficulties are exclusively due to unrealistic queries, and those that are not can be overcome with better training data and larger models. In this work, we demonstrate that we may encounter these theoretical limitations in realistic settings with extremely simple queries.

#### Key Finding: The Embedding Dimension Bottleneck

The number of **top-k subsets** of documents that can be returned by any query is **fundamentally limited by embedding dimension**.

This holds true even when:
- k = 2 (just returning top 2 documents)
- You directly optimize on the test set
- You use free parameterized embeddings (no model constraints)

#### Critical-n Values by Embedding Dimension

The paper provides extrapolated "critical-n" values - the corpus size at which embeddings break down:

| Embedding Dim | Critical Corpus Size |
|---------------|---------------------|
| 512 | ~500,000 documents |
| 768 | ~1.7 million |
| 1024 | ~4 million |
| 3072 | ~107 million |
| 4096 | ~250 million |

**Implication**: For web-scale search (billions of documents), even 4096-dimensional embeddings with ideal optimization cannot model all needed combinations.

#### The LIMIT Dataset

Google DeepMind created the **LIMIT dataset** to stress-test this:
- 50,000 documents, 1,000 queries
- State-of-the-art models fail on simple retrieval tasks
- Demonstrates limitations are not theoretical edge cases

### 2. The Curse of Dense Low-Dimensional Information Retrieval

**Authors**: Nils Reimers, Iryna Gurevych
**Venue**: ACL 2021
**arXiv**: [2012.14210](https://arxiv.org/abs/2012.14210)

#### Key Finding: False Positive Rate Increases with Index Size

> The performance for dense representations decreases quicker than sparse representations for increasing index sizes. In extreme cases, this can even lead to a tipping point where at a certain index size sparse representations outperform dense representations.

#### The Mechanism

As corpus size grows:
1. More documents exist in the embedding space
2. Documents cluster and overlap in limited dimensions
3. **False positives increase** - irrelevant documents become "nearest neighbors"
4. At some corpus size, BM25 beats dense retrieval

#### Why Lower Dimensions Are Worse

> The lower the dimension, the higher the chance for false positives, i.e., returning irrelevant documents.

This is why 768-dim embeddings that work great on MS MARCO can fail badly on larger real-world corpora.

## Why This Matters

### The Problem

If you have:
- 100k+ documents
- Complex queries ("find decisions about auth that led to bugs")
- Need for precise retrieval

Dense vectors **will fail** on some queries - not because of bad embeddings, but because of **mathematical impossibility**.

### The Solution: Hybrid Search

This is exactly why we need **RRF hybrid search** (see [Hybrid Search with RRF](/research/hybrid-search-rrf/)):

1. **BM25/Lexical** catches exact matches dense vectors miss
2. **Dense vectors** catch semantic similarity BM25 misses
3. **RRF fusion** combines both, surfacing docs that rank well in either

### Practical Implications

| Approach | Failure Mode |
|----------|--------------|
| Dense only | Misses obvious keyword matches; false positives at scale |
| BM25 only | Misses semantic similarity ("auth" vs "authentication") |
| Hybrid RRF | Catches both; degrades gracefully |

## The Math (Simplified)

### Why Embeddings Have Limited "Capacity"

In a d-dimensional space, there are only so many ways to partition documents into "closer" and "farther" from a query point.

Specifically, the number of possible **dichotomies** (ways to separate n points into two groups) is bounded by:

```
# of dichotomies ≤ 2 * (n choose 0) + (n choose 1) + ... + (n choose d)
```

For n >> d, this grows polynomially in n, not exponentially. But the number of possible **top-k rankings** grows combinatorially.

**Result**: There exist top-k combinations that no embedding geometry can produce.

### Concrete Example

With 1 million documents and 768-dim embeddings:
- Possible top-10 combinations: C(1M, 10) ≈ 10^53
- Achievable by any embedding: << 10^53

Many valid query results are **literally impossible** to retrieve.

## Recommendations

### For Search Implementation

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

- [On the Theoretical Limitations of Embedding-Based Retrieval](https://arxiv.org/abs/2508.21038) - Weller et al., 2025
- [The Curse of Dense Low-Dimensional IR](https://arxiv.org/abs/2012.14210) - Reimers & Gurevych, ACL 2021
- [LIMIT Dataset & Code](https://github.com/google-deepmind/limit) - Google DeepMind
- [ACL Anthology: Curse of Dense IR](https://aclanthology.org/2021.acl-short.77/)

## See Also

- [Hybrid Search with RRF](/research/hybrid-search-rrf/) - Implementation guide for hybrid search in Postgres
