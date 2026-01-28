---
title: Transcription with Custom Glossaries
problem: Domain terms transcribed incorrectly in speech-to-text
context: Working with speech-to-text, technical content, domain-specific audio
category: data-pipelines
tags: [transcription, glossary, speech-to-text, amazon-transcribe, nlp]
version: "1.0"
last_updated: 2026-01-27
---

# Transcription with Custom Glossaries: Multi-Pass AI Pipeline

> Building domain-aware speech-to-text with Amazon Transcribe and agentic glossary creation

## Overview

For accurate transcription of domain-specific audio (meetings, support calls, technical discussions), you need:

1. **Custom vocabulary** - Tell the STT engine about your jargon
2. **Multi-pass glossary creation** - Build that vocabulary systematically with AI
3. **Continuous refinement** - Update as terminology evolves

## From Production: Entity Glossary for LLM Correction

> Real pattern from AIC Holdings call processing pipeline

### The Problem

Speech-to-text engines produce errors that are **obvious to domain experts** but invisible to generic models:

| Audio | Raw Transcript | Correct |
|-------|---------------|---------|
| "SPY is down 2%" | "spy is down 2%" | "SPY (S&P 500 ETF) is down 2%" |
| "Check the BTIG note" | "check the bee tig note" | "Check the BTIG note" |
| "Hedgeye's quad 4" | "hedge eyes quad four" | "Hedgeye's Quad 4" |
| "Ask Colin about it" | "ask colin about it" | "Ask Colin Bird about it" |

### The Solution: Entity Glossary + LLM Post-Processing

Instead of (or in addition to) custom STT vocabulary, pass an **entity glossary** to an LLM for post-transcription correction:

```typescript
// Entity glossary structure
const GLOSSARY = {
  tickers: [
    { term: "SPY", expansion: "S&P 500 ETF", sounds_like: ["spy"] },
    { term: "QQQ", expansion: "Nasdaq 100 ETF", sounds_like: ["q q q", "triple q"] },
    { term: "HEI", expansion: "HEICO Corporation", sounds_like: ["hey", "h e i"] },
  ],
  companies: [
    { term: "BTIG", expansion: "BTIG LLC (broker)", sounds_like: ["bee tig", "b tig"] },
    { term: "Hedgeye", expansion: "Hedgeye Risk Management", sounds_like: ["hedge eye", "hedge eyes"] },
  ],
  people: [
    { term: "Colin Bird", role: "CIO, AIC Holdings", sounds_like: ["colin", "collin"] },
    { term: "Keith McCullough", role: "CEO, Hedgeye", sounds_like: ["keith", "mccullough"] },
  ],
  concepts: [
    { term: "Quad 4", definition: "Hedgeye macro regime: growth slowing, inflation slowing" },
    { term: "RRF", expansion: "Reciprocal Rank Fusion", sounds_like: ["r r f"] },
  ]
};
```

### LLM Correction Prompt

```typescript
const CORRECTION_PROMPT = `
You are correcting a meeting transcript. Fix obvious transcription errors using the glossary below.

RULES:
1. Fix clear mishearings (e.g., "bee tig" → "BTIG")
2. Expand acronyms on first use (e.g., "SPY" → "SPY (S&P 500 ETF)")
3. Fix speaker names (e.g., "colin" → "Colin Bird")
4. Preserve speaker attributions and timestamps exactly
5. Do NOT change meaning or add information
6. If unsure, leave unchanged

GLOSSARY:
${JSON.stringify(GLOSSARY, null, 2)}

TRANSCRIPT:
${rawTranscript}

Return corrected transcript with same structure.
`;
```

### Architecture: Post-Processing Pipeline

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Fireflies  │────▶│  Raw JSON   │────▶│  Glossary   │
│  Webhook    │     │  (storage)  │     │  Lookup     │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
                                               ▼
                    ┌─────────────┐     ┌─────────────┐
                    │  Corrected  │◀────│  LLM Pass   │
                    │  Transcript │     │  (Claude)   │
                    └─────────────┘     └─────────────┘
```

**Key insight**: Run correction **after** raw ingestion. Store both versions:
- `raw_transcript` - Immutable source of truth
- `corrected_transcript` - LLM-enhanced version

This allows re-running correction as glossary improves without losing original data.

### Glossary Maintenance

The glossary itself needs a governance workflow:

```sql
CREATE TABLE entity_glossary (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  term TEXT NOT NULL,
  category TEXT NOT NULL,  -- ticker, company, person, concept
  expansion TEXT,
  sounds_like TEXT[],
  definition TEXT,
  metadata JSONB,
  status TEXT DEFAULT 'active',  -- active, deprecated, review
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Example entries
INSERT INTO entity_glossary (term, category, expansion, sounds_like) VALUES
  ('SPY', 'ticker', 'S&P 500 ETF', ARRAY['spy']),
  ('BTIG', 'company', 'BTIG LLC (broker)', ARRAY['bee tig', 'b tig']),
  ('Colin Bird', 'person', 'CIO, AIC Holdings', ARRAY['colin', 'collin']);
```

### Results

| Metric | Before Glossary | After Glossary |
|--------|-----------------|----------------|
| Ticker accuracy | ~60% | ~95% |
| Name accuracy | ~70% | ~90% |
| Acronym expansion | 0% | 100% (first use) |
| Processing cost | - | ~$0.02/transcript (Haiku) |

### When to Use This vs Custom STT Vocabulary

| Approach | Best For | Limitation |
|----------|----------|------------|
| **STT Custom Vocabulary** | Known terms before transcription | Limited to ~50k terms, no context |
| **LLM Post-Processing** | Context-aware correction, expansions | Adds latency and cost |
| **Both** | Production systems | More complexity |

**Recommendation**: Start with LLM post-processing (faster to iterate), add STT vocabulary for high-frequency terms once patterns stabilize.

## Amazon Transcribe with Custom Vocabulary

### Basic Flow

```
MP3 → S3 → Transcribe Job (+ Custom Vocab) → JSON Transcript
```

### CLI Example

```bash
aws transcribe start-transcription-job \
  --region us-west-2 \
  --transcription-job-name my-mp3-job \
  --media MediaFileUri=s3://my-bucket/audio/my-call.mp3 \
  --output-bucket-name my-bucket \
  --language-code en-US \
  --settings VocabularyName=my-domain-vocab
```

### Custom Vocabulary Format

Create a CSV/text file with terms Transcribe should recognize:

| Phrase | IPA | SoundsLike | DisplayAs |
|--------|-----|------------|-----------|
| taskr | | task er | Taskr |
| pgvector | | P G vector | pgvector |
| devlog | | dev log | devlog |

### Additional Options

| Feature | Purpose |
|---------|---------|
| **Custom Vocabulary** | Domain terms, jargon, proper nouns |
| **Vocabulary Filters** | Mask/replace sensitive words in output |
| **Custom Language Models** | Train on your corpus for better accuracy |

## Multi-Pass Glossary Creation Pipeline

Pros treat glossary building as a **governance system** with multi-step agentic workflows.

### Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    GLOSSARY PIPELINE                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │  Ingest  │───▶│ Extract  │───▶│ Cluster  │              │
│  │  Corpus  │    │  Terms   │    │ & Dedup  │              │
│  └──────────┘    └──────────┘    └──────────┘              │
│                                        │                    │
│                                        ▼                    │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │  Deploy  │◀───│  Review  │◀───│ Generate │              │
│  │ to Tools │    │  & QA    │    │  Defs    │              │
│  └──────────┘    └──────────┘    └──────────┘              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Pass 1: Domain Term Discovery

**Goal**: High recall, low precision extraction from raw text

```python
# Prompt pattern for term extraction
EXTRACT_PROMPT = """
Extract domain-specific terms ACTUALLY PRESENT in this text.
Do NOT invent terms. Only extract what appears.

Categories to look for:
- People and organizations
- Products and features
- Technical terms and acronyms
- Metrics and units

Output JSON: [{"term": "...", "category": "...", "context": "..."}]
"""
```

**Tips**:
- Chunk input, process in parallel
- Use multiple narrow prompts per category vs one catch-all
- Store with source_id and example_snippet for traceability

### Pass 2: Clustering and Canonicalization

**Goal**: Turn noisy candidates into stable term identities

```python
# Input: raw terms with counts and contexts
# Output: canonical terms with variants

CLUSTER_PROMPT = """
Group these terms into clusters of the same concept:
- Singular/plural variants
- Spelling variations
- Abbreviations and expansions

For each cluster, output:
{
  "canonical_term": "preferred form",
  "synonyms": ["variant1", "variant2"],
  "type": "person|product|technical|metric",
  "representative_example": "sentence showing usage"
}
"""
```

**Pre-processing heuristics** (reduce LLM tokens):
- Case folding
- Levenshtein distance < 2
- Acronym expansion matching

### Pass 3: Definition Generation

**Goal**: Consistent, structured glossary entries

```python
DEFINITION_SCHEMA = {
    "term": str,           # Canonical form
    "synonyms": list,      # All variants
    "category": str,       # Classification
    "definition": str,     # 1-2 sentences
    "when_to_use": str,    # Usage guidance
    "when_not_to_use": str,
    "example_sentence": str,
    "related_terms": list
}

DEFINITION_PROMPT = """
Generate glossary entries matching this exact schema.
Style: concise, for internal engineers.

Here are 2 examples of good entries:
{examples}

Now generate entries for these terms:
{terms}
"""
```

### Pass 4: Conflict Detection and QA

**Goal**: Catch errors before freezing the glossary

```python
QA_PROMPT = """
Review this glossary and flag issues:

1. CONFLICTING: Terms with contradictory definitions
2. OVERLAPPING: Terms that should be merged
3. DUPLICATES: Same concept, different entries
4. TOO_GENERIC: Terms that are too broad
5. HALLUCINATED: Terms not found in any source

For each issue, explain and suggest fix.
"""
```

**Validation rules**:
- Every term must appear in at least one source snippet
- Drop terms introduced only in later passes
- Sample for human review on high-impact domains

### Pass 5: Deployment

Export glossary to downstream systems:

| System | Export Format |
|--------|---------------|
| Amazon Transcribe | Custom vocabulary CSV |
| Search (Elasticsearch) | Synonyms file |
| Translation (SDL, Phrase) | TBX termbase |
| LLM prompts | Style guide / RAG chunks |
| UI linters | Term dictionary |

## Agentic Implementation with Claude

The Claude Agent SDK is ideal for this pipeline - it handles "gather context → take action → verify" loops.

### Agent Architecture

```python
# Pseudo-code for agentic glossary builder

class GlossaryAgent:
    tools = [
        "read_corpus",      # Scan docs/transcripts
        "write_json",       # Persist intermediate results
        "run_extraction",   # LLM term extraction
        "run_clustering",   # LLM dedup/normalize
        "run_definition",   # LLM generate definitions
        "run_qa",           # LLM conflict detection
        "export_vocab",     # Write Transcribe format
        "open_pr"           # Git PR for review
    ]

    async def build_glossary(self, corpus_path: str):
        # Step 1: Extract candidates
        raw_terms = await self.run_extraction(corpus_path)
        await self.write_json("candidates.json", raw_terms)

        # Step 2: Cluster and dedupe
        clusters = await self.run_clustering(raw_terms)
        await self.write_json("clusters.json", clusters)

        # Step 3: Generate definitions
        glossary = await self.run_definition(clusters)
        await self.write_json("glossary.json", glossary)

        # Step 4: QA pass
        issues = await self.run_qa(glossary)
        if issues:
            glossary = await self.fix_issues(glossary, issues)

        # Step 5: Deploy
        await self.export_vocab(glossary, "transcribe_vocab.csv")
        await self.open_pr("glossary/", "Update domain glossary")
```

### Verification Loop

```python
async def verify_glossary(self, glossary, corpus):
    """Ensure every term exists in source material"""
    for term in glossary:
        found = await self.search_corpus(corpus, term["canonical"])
        if not found:
            # Check synonyms
            found = any(
                await self.search_corpus(corpus, syn)
                for syn in term["synonyms"]
            )
        if not found:
            term["status"] = "unverified"
            term["action"] = "review_or_remove"
    return glossary
```

## Enterprise Glossary Best Practices

From business/translation terminology platforms:

### Governance Model

| Field | Purpose |
|-------|---------|
| `owner` | Who maintains this term |
| `status` | draft → review → approved → deprecated |
| `review_date` | When to revisit |
| `change_history` | Audit trail |

### Rich Metadata

```json
{
  "term": "RRF",
  "canonical": "Reciprocal Rank Fusion",
  "synonyms": ["RRF", "rank fusion"],
  "domain": "search",
  "definition": "Algorithm combining multiple ranked lists...",
  "do_use": "When describing hybrid search ranking",
  "dont_use": "Don't abbreviate in user-facing docs",
  "related": ["BM25", "hybrid search", "pgvector"],
  "system_of_record": "docs/research/hybrid-search-pgvector-rrf.md",
  "owner": "search-team",
  "status": "approved",
  "tags": ["search", "ranking", "postgres"]
}
```

### Integration Points

| System | Integration |
|--------|-------------|
| Data catalog | Link terms to tables/columns |
| Authoring tools | Autocomplete, linting |
| Translation | Termbase sync |
| Search | Synonym expansion |
| LLM/RAG | System prompt, retrieval |

## Recent Research (2024-2025)

### Syntactic Retrieval for Term Extraction (ACL 2025)

**Paper**: [Enhancing Automatic Term Extraction with Large Language Models via Syntactic Retrieval](https://arxiv.org/abs/2506.21222)

Key insight: For few-shot term extraction, select demonstration examples by **syntactic similarity** (not semantic). This is domain-agnostic and provides better guidance for term boundary detection.

> "Syntactic retrieval improves F1-score across three specialized ATE benchmarks."

**Why it matters**: When building prompts for Pass 1 (term discovery), retrieve examples that have similar grammatical structure to the input, not just similar meaning.

### Whisper Fine-Tuning for Domain Speech

For audio with heavy domain jargon, consider fine-tuning Whisper instead of (or in addition to) custom vocabularies:

| Approach | WER Improvement | Effort |
|----------|-----------------|--------|
| Custom vocabulary only | 10-20% | Low |
| LoRA fine-tuning | 60%+ | Medium |
| Full fine-tuning | 70%+ | High |

**Example**: Cockpit speech recognition achieved WER reduction from 68% to 26% using LoRA fine-tuning ([Whisper Cockpit Study](https://arxiv.org/html/2506.21990v1)).

**When to fine-tune**:
- Heavy domain jargon (medical, legal, aviation)
- Accents or speech patterns not in training data
- Custom vocabulary alone insufficient

### Multi-Step LLM Chain Best Practices

From [Deepchecks research](https://www.deepchecks.com/orchestrating-multi-step-llm-chains-best-practices/):

1. **Modular Design**: Break chains into reusable, independent components
2. **Fallback Logic**: Anticipate failures with retry/rephrase mechanisms
3. **Optimization**: Cache frequent queries, parallelize independent steps
4. **Cost-Aware**: Use token-efficient prompts, cheaper models for simple passes

### Enterprise Glossary Platforms

Modern data catalogs integrate glossaries with AI:

| Platform | Key Feature |
|----------|-------------|
| [SAS Information Catalog](https://communities.sas.com/t5/SAS-Communities-Library/The-Power-of-Clarity-How-the-Business-Glossary-Enhances-Your/ta-p/934008) | Glossary REST API for automation |
| [Informatica EDC](https://www.informatica.com/blogs/business-glossary-vs-data-catalog.html) | AI/ML auto-association of terms to metadata |
| [Alation](https://www.alation.com/) | ML-suggested terms, crowdsourced definitions |
| [Ataccama](https://www.ataccama.com/) | Auto-sync glossary to data catalog |

**Pattern**: All use ML for term suggestion + human governance workflow.

### Claude Agent SDK for Glossary Pipelines

The [Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk) provides:

- **File-aware processing**: Read/write corpus and glossary files
- **Tool orchestration**: Chain extraction → clustering → definition passes
- **Verification loops**: Iterate until quality thresholds met
- **MCP integration**: Custom tools for domain-specific validation

```python
# Agent SDK pattern for glossary building
from anthropic.agent import Agent

agent = Agent(
    tools=[
        "read_files",      # Corpus access
        "write_files",     # Persist glossary
        "search_corpus",   # Verify terms exist
        "run_validation",  # QA checks
    ]
)

# Agent autonomously runs multi-pass pipeline
result = agent.run(
    "Build a glossary from docs/*.md with governance workflow"
)
```

## Alternative: Vosk with Custom Language Models

For on-premise or offline transcription, [Vosk](https://alphacephei.com/vosk/) supports custom language models:

> "Custom language models demonstrate clear and consistent advantages over default models... reducing recognition errors such as mispronounced words, missing terms, or incorrect substitutions."

Source: [Improving Speech Recognition Accuracy Using Custom Language Models with Vosk](https://arxiv.org/html/2503.21025v1)

## References

- [Amazon Transcribe Custom Vocabularies](https://docs.aws.amazon.com/transcribe/latest/dg/custom-vocabulary.html)
- [Amazon Transcribe Custom Language Models](https://docs.aws.amazon.com/transcribe/latest/dg/custom-language-models.html)
- [Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)
- [Syntactic Retrieval for ATE (ACL 2025)](https://arxiv.org/abs/2506.21222)
- [Fine-Tuning Whisper for Domain Speech](https://huggingface.co/blog/fine-tune-whisper)
- [Whisper LoRA for Cockpit Speech](https://arxiv.org/html/2506.21990v1)
- [Vosk Custom Language Models](https://arxiv.org/html/2503.21025v1)
- [Multi-Step LLM Chain Best Practices](https://www.deepchecks.com/orchestrating-multi-step-llm-chains-best-practices/)
- [SAS Information Catalog Glossary](https://communities.sas.com/t5/SAS-Communities-Library/The-Power-of-Clarity-How-the-Business-Glossary-Enhances-Your/ta-p/934008)
- [Informatica Business Glossary vs Data Catalog](https://www.informatica.com/blogs/business-glossary-vs-data-catalog.html)

## See Also

- [Hybrid Search with RRF](/research/hybrid-search-rrf/) - Using glossary for search synonym expansion
- [Embedding Limitations](/research/embedding-limitations/) - Why lexical/glossary matching still matters
