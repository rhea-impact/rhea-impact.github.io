---
title: Effective Embeddings - Learning Through Assignment
problem: Embedding-based categorization doesn't improve from human feedback
context: Building semantic matching for classification, tagging, or organization systems
category: embeddings
tags: [embeddings, machine-learning, categorization, semantic-search, human-in-the-loop]
version: "1.0"
last_updated: 2026-01-29
---

# Effective Embeddings: Learning Through Assignment

> Make categories learn what they contain by absorbing the embeddings of their members

## TL;DR

Instead of building complex feedback loops to "train" a categorization system, make each category's embedding the **centroid of everything assigned to it**. When a human assigns an item to a category, that item's embedding gets averaged into the category's effective embedding. The category literally *becomes* the semantic center of its contents. No ML training. No feedback tables. Instant learning.

## The Problem

You have:
- A set of **categories** (areas, tags, folders, labels)
- A stream of **items** to categorize (tasks, documents, emails)
- **Embeddings** for both

The naive approach: compute cosine similarity between item and category embeddings, suggest the closest match.

**Why it fails**: Category names/descriptions are often abstract ("Health", "Work", "Personal Development"). The embedding of the word "Health" doesn't capture that YOUR health category contains gym schedules, meal prep, and therapy notes.

### Traditional "Solutions"

| Approach | Problem |
|----------|---------|
| Feedback tables | Slow learning, requires many examples |
| Fine-tuning embeddings | Expensive, requires training pipeline |
| Keyword boosting | Doesn't generalize semantically |
| Manual rules | Doesn't scale, requires maintenance |

## The Insight

A category's meaning isn't its name - it's **what you put in it**.

If you assign 5 fitness-related projects to "Health", then "Health" should now match fitness-like things better. Not eventually. **Immediately.**

## The Solution: Effective Embeddings

Instead of using the static embedding of a category's name/description, compute an **effective embedding** that incorporates all assigned items:

```python
def compute_effective_embedding(category_id: int, base_embedding: list[float]) -> list[float]:
    """
    Category embedding = centroid of (base + all assigned items).
    """
    # Get embeddings of all items assigned to this category
    assigned_embeddings = get_assigned_item_embeddings(category_id)

    if not assigned_embeddings:
        return base_embedding  # No items yet, use base

    # Average all embeddings: base + assigned items
    all_embeddings = [base_embedding] + assigned_embeddings

    dim = len(base_embedding)
    effective = [0.0] * dim
    for emb in all_embeddings:
        for i in range(dim):
            effective[i] += emb[i]
    for i in range(dim):
        effective[i] /= len(all_embeddings)

    return effective
```

### How It Works

1. **Initial state**: Category "Health" has embedding of just the word
2. **User assigns** "Gym membership renewal" to Health
3. **Effective embedding** = average(Health_base, gym_task)
4. **Next match**: Items similar to gym stuff now match Health better
5. **User assigns** "Meal prep for the week" to Health
6. **Effective embedding** = average(Health_base, gym_task, meal_task)
7. **Category keeps learning** with each assignment

### Why This Works

- **No delayed feedback**: Learning happens on assignment, not after
- **No training pipeline**: Just vector averaging
- **No separate storage**: Compute on-the-fly or cache
- **Graceful start**: Base embedding works until items are assigned
- **Self-correcting**: If you mis-assign, just reassign - the wrong item leaves the average

## Implementation

### Basic Version (Compute On-The-Fly)

```python
def find_best_category(item_embedding: list[float], categories: list) -> list:
    """Find best matching categories using effective embeddings."""
    matches = []

    for cat in categories:
        # Get base embedding
        base_emb = get_embedding(cat.id)

        # Compute effective embedding (absorbs assigned items)
        effective_emb = compute_effective_embedding(cat.id, base_emb)

        # Compare item against effective embedding
        similarity = cosine_similarity(item_embedding, effective_emb)
        matches.append({
            "category": cat,
            "similarity": similarity
        })

    return sorted(matches, key=lambda x: x["similarity"], reverse=True)
```

### With Weighting

You may want the base embedding to have more weight than individual items:

```python
def compute_effective_embedding_weighted(
    category_id: int,
    base_embedding: list[float],
    base_weight: float = 2.0  # Base counts as 2 items
) -> list[float]:
    assigned = get_assigned_item_embeddings(category_id)

    if not assigned:
        return base_embedding

    dim = len(base_embedding)
    effective = [x * base_weight for x in base_embedding]
    total_weight = base_weight

    for emb in assigned:
        for i in range(dim):
            effective[i] += emb[i]
        total_weight += 1

    return [x / total_weight for x in effective]
```

### Cached Version (For Performance)

If you have many items per category, cache the effective embedding:

```sql
ALTER TABLE categories ADD COLUMN effective_embedding TEXT;
ALTER TABLE categories ADD COLUMN effective_embedding_count INTEGER DEFAULT 0;
```

Update on assignment:

```python
def update_category_effective_embedding(category_id: int):
    base = get_base_embedding(category_id)
    effective = compute_effective_embedding(category_id, base)
    count = get_assigned_count(category_id)

    save_effective_embedding(category_id, effective, count)
```

## Real-World Example: Area Assignment

In a task management system using PARA methodology:

```python
def find_unassigned_area_matches():
    """Match orphaned items to areas using effective embeddings."""

    # Get all areas with their effective embeddings
    areas = []
    for area in get_all_areas():
        base_emb = get_area_embedding(area.id)
        effective_emb = compute_effective_embedding(area.id, base_emb)
        areas.append((area, effective_emb))

    # Match unassigned projects
    matches = []
    for project in get_unassigned_projects():
        # Use weighted average of project's task embeddings
        project_emb = compute_project_embedding_from_tasks(project.id)

        # Find best area matches
        for area, area_emb in areas:
            sim = cosine_similarity(project_emb, area_emb)
            matches.append({
                "project": project,
                "area": area,
                "similarity": sim
            })

    return sorted(matches, key=lambda x: x["similarity"], reverse=True)
```

**Result**: As you assign projects to areas, the areas learn what belongs to them. A "Health" area that started abstract becomes specifically tuned to YOUR health-related content.

## Comparison to Alternatives

| Approach | Learning Speed | Compute Cost | Accuracy Over Time |
|----------|---------------|--------------|-------------------|
| Static embeddings | None | Low | Constant |
| Feedback + retraining | Slow (batch) | High | Improves slowly |
| Effective embeddings | Instant | Low | Improves immediately |

## Edge Cases

### Empty Categories

When a category has no assigned items, fall back to base embedding. The category works from day one.

### One-Item Categories

Works fine - effective embedding is average of base + one item. Slightly biased toward that item, which is correct behavior.

### Mis-Assignments

If user assigns wrong item, then reassigns elsewhere:
- Wrong category: item leaves average, embedding shifts back
- Correct category: item joins average, embedding shifts toward it
- Self-correcting without explicit "undo"

### Category Drift

Over time, category may drift from original meaning as items accumulate. Options:
- Weight base embedding higher (e.g., 2x or 3x)
- Cap number of items in average (rolling window)
- Periodically re-anchor to base

## When NOT to Use This

- **Strict taxonomy**: If categories have formal definitions that shouldn't drift
- **Adversarial inputs**: Users could intentionally poison categories
- **Audit requirements**: Need to explain exactly why something matched

## Performance Notes

For a typical system:
- 10 categories, 100 items each
- 1024-dim embeddings
- Compute effective embedding: ~1ms per category
- Total matching: ~10ms

Compute on-the-fly is fine up to ~1000 items per category. Beyond that, cache effective embeddings and update incrementally.

## Summary

Stop building complex feedback systems. Make categories absorb what you put in them:

```
effective_embedding = mean(base_embedding, *assigned_item_embeddings)
```

The category literally becomes the centroid of its contents. That's real learning - not a feedback loop, just direct representation.

## See Also

- [Theoretical Limitations of Embedding-Based Retrieval](/research/embedding-limitations/) - Why pure embedding search fails at scale
- [Hybrid Search with RRF](/research/hybrid-search-rrf/) - Combining embeddings with lexical search
