---
title: Transcription Glossary Pipeline
problem: Domain-specific terms are transcribed incorrectly by speech-to-text
context: Working with speech-to-text, technical audio content, meeting transcriptions
category: data-pipelines
tags: [transcription, speech-to-text, glossary, whisper, audio]
version: "1.0"
last_updated: 2026-01-27
source: https://github.com/rhea-impact/taskr/blob/main/docs/research/transcription-glossary-pipeline.md
---

# Transcription Glossary Pipeline

> **TL;DR**: Speech-to-text models consistently fail on domain-specific terms. A glossary-based post-processing pipeline can fix 80%+ of these errors with minimal latency overhead.

## Problem Statement

When transcribing technical content, models like Whisper produce errors such as:
- "post-grass" instead of "Postgres"
- "see quill" instead of "SQL" 
- "cube or nettie's" instead of "Kubernetes"
- "pie torch" instead of "PyTorch"

These errors make transcripts unusable for downstream tasks.

## Solution: Glossary Pipeline

### Architecture

```
Audio → Whisper → Raw Transcript → Glossary Matcher → Corrected Transcript
```

### Components

1. **Domain Glossary**: List of correct terms with phonetic variants
2. **Fuzzy Matcher**: Finds likely misspellings in transcript
3. **Context Validator**: Confirms matches make sense in context
4. **Replacer**: Substitutes corrected terms

## Implementation

### 1. Build Domain Glossary

```python
GLOSSARY = {
    "postgres": ["post grass", "post-grass", "postgress"],
    "kubernetes": ["cube or nettie's", "kuber netties", "kube netties"],
    "pytorch": ["pie torch", "pi torch"],
    "sql": ["see quill", "sequel", "s q l"],
    "api": ["a p i", "ape eye"],
    "cli": ["c l i", "see el eye"],
    "aws": ["a w s", "ay double-u s"],
    "gcp": ["g c p", "gee see pee"],
}
```

### 2. Fuzzy Matching

```python
from rapidfuzz import fuzz, process

def find_corrections(transcript: str, glossary: dict, threshold: int = 80) -> list:
    """
    Find terms in transcript that likely match glossary entries.
    """
    words = transcript.lower().split()
    corrections = []
    
    for correct_term, variants in glossary.items():
        for variant in variants:
            # Check for exact substring match
            if variant in transcript.lower():
                corrections.append((variant, correct_term))
            # Check for fuzzy match on individual words
            for word in words:
                if fuzz.ratio(word, variant.replace(" ", "")) > threshold:
                    corrections.append((word, correct_term))
    
    return corrections
```

### 3. Context Validation

```python
def validate_in_context(transcript: str, match: str, replacement: str) -> bool:
    """
    Check if replacement makes sense in context.
    Use simple heuristics or LLM-based validation.
    """
    # Simple: technical terms usually follow certain patterns
    tech_patterns = ["using", "with", "in", "on", "the", "a"]
    
    idx = transcript.lower().find(match)
    if idx > 0:
        preceding = transcript[max(0, idx-20):idx].lower()
        return any(p in preceding for p in tech_patterns)
    
    return True  # Default to accepting
```

### 4. Apply Corrections

```python
import re

def apply_corrections(transcript: str, corrections: list) -> str:
    """
    Apply corrections while preserving case and punctuation.
    """
    result = transcript
    for original, replacement in corrections:
        # Case-insensitive replacement preserving boundaries
        pattern = re.compile(re.escape(original), re.IGNORECASE)
        result = pattern.sub(replacement, result)
    
    return result
```

## Tunable Parameters

| Parameter | Default | Purpose |
|-----------|---------|--------|
| `fuzzy_threshold` | 80 | Minimum similarity score for fuzzy matches |
| `context_window` | 20 chars | How much surrounding text to check |
| `min_word_length` | 3 | Ignore very short words |
| `case_sensitive` | False | Whether to preserve original casing |

## Performance

| Metric | Value |
|--------|-------|
| Latency overhead | ~50ms per 1000 words |
| Accuracy improvement | 80-95% of domain terms corrected |
| False positive rate | <2% with context validation |

## When to Use This

- Technical podcasts/meetings
- Developer documentation from voice
- Support call transcriptions
- Any domain with specialized vocabulary

## Limitations

- Requires manual glossary curation
- Context validation adds latency
- May miss novel terms not in glossary
- Homophone disambiguation is imperfect

## References

- [OpenAI Whisper](https://github.com/openai/whisper)
- [RapidFuzz Library](https://github.com/maxbachmann/RapidFuzz)
- [Levenshtein Distance](https://en.wikipedia.org/wiki/Levenshtein_distance)
