---
title: Artemis - AI Call Tracking Middleware
problem: No visibility into AI usage across apps, users, and models
context: Running multiple AI-powered applications, need to track costs and usage
category: infrastructure
tags: [llm, observability, tokens, middleware, openrouter, anthropic]
version: "1.0"
last_updated: 2026-01-28
---

# Artemis: AI Call Tracking Middleware

> An odometer for your AI. Track every token, every call, every app.

## The Problem

When running multiple AI-powered applications, you lose visibility:

| Question | Without Tracking |
|----------|------------------|
| How much did we spend on AI this month? | Check each provider dashboard manually |
| Which app is the most expensive? | No idea |
| Which user is generating the most tokens? | No idea |
| Are we using the right model for each task? | Gut feeling |
| Did that prompt change reduce costs? | Compare invoices month-over-month |

## The Solution: Centralized AI Middleware

Route all AI calls through a single middleware that logs everything:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  App 1      │────▶│             │────▶│  Anthropic  │
├─────────────┤     │   ARTEMIS   │     ├─────────────┤
│  App 2      │────▶│  Middleware │────▶│  OpenAI     │
├─────────────┤     │             │     ├─────────────┤
│  App 3      │────▶│  - Log call │────▶│  OpenRouter │
└─────────────┘     │  - Track $  │     └─────────────┘
                    │  - Route    │
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  ai_calls   │
                    │  (Supabase) │
                    └─────────────┘
```

## Architecture

### Core Components

| Component | Purpose |
|-----------|---------|
| **Proxy Endpoint** | Single URL all apps call instead of providers directly |
| **Router** | Maps model aliases to actual provider endpoints |
| **Logger** | Records every call with metadata |
| **Dashboard** | Visualize usage by app, user, model |

### Data Model

```sql
CREATE TABLE ai_calls (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- What
  model TEXT NOT NULL,           -- e.g., "claude-3-haiku", "gpt-4o"
  provider TEXT NOT NULL,        -- e.g., "anthropic", "openai", "openrouter"

  -- Who
  app_id TEXT NOT NULL,          -- Which application
  user_id TEXT,                  -- Which user (optional)
  session_id TEXT,               -- Conversation/session grouping

  -- Tokens
  input_tokens INT NOT NULL,
  output_tokens INT NOT NULL,
  total_tokens INT GENERATED ALWAYS AS (input_tokens + output_tokens) STORED,

  -- Cost (computed from token counts + pricing table)
  cost_usd DECIMAL(10, 6),

  -- Timing
  latency_ms INT,                -- Time to first token or completion
  created_at TIMESTAMPTZ DEFAULT now(),

  -- Debug (optional, can be large)
  prompt_preview TEXT,           -- First 500 chars of prompt
  response_preview TEXT,         -- First 500 chars of response
  metadata JSONB                 -- Custom fields per app
);

-- Indexes for common queries
CREATE INDEX idx_ai_calls_app_created ON ai_calls(app_id, created_at DESC);
CREATE INDEX idx_ai_calls_model_created ON ai_calls(model, created_at DESC);
CREATE INDEX idx_ai_calls_user ON ai_calls(user_id) WHERE user_id IS NOT NULL;
```

### Pricing Table

```sql
CREATE TABLE ai_pricing (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  input_price_per_1m DECIMAL(10, 4),   -- $ per 1M input tokens
  output_price_per_1m DECIMAL(10, 4),  -- $ per 1M output tokens
  effective_date DATE NOT NULL,
  UNIQUE(provider, model, effective_date)
);

-- Current pricing (as of Jan 2026)
INSERT INTO ai_pricing (provider, model, input_price_per_1m, output_price_per_1m, effective_date) VALUES
  ('anthropic', 'claude-3-5-sonnet', 3.00, 15.00, '2025-01-01'),
  ('anthropic', 'claude-3-haiku', 0.25, 1.25, '2025-01-01'),
  ('anthropic', 'claude-3-opus', 15.00, 75.00, '2025-01-01'),
  ('openai', 'gpt-4o', 2.50, 10.00, '2025-01-01'),
  ('openai', 'gpt-4o-mini', 0.15, 0.60, '2025-01-01'),
  ('openai', 'o1', 15.00, 60.00, '2025-01-01');
```

## Implementation

### Proxy Endpoint (Edge Function)

```typescript
// supabase/functions/artemis/index.ts
import { serve } from "https://deno.land/std@0.168.0/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

const PROVIDERS = {
  anthropic: "https://api.anthropic.com/v1/messages",
  openai: "https://api.openai.com/v1/chat/completions",
  openrouter: "https://openrouter.ai/api/v1/chat/completions",
};

serve(async (req) => {
  const startTime = Date.now();
  const supabase = createClient(/*...*/);

  // 1. Parse request
  const body = await req.json();
  const { model, messages, app_id, user_id, session_id, ...rest } = body;

  // 2. Route to provider
  const provider = getProvider(model);
  const providerUrl = PROVIDERS[provider];
  const apiKey = getApiKey(provider);

  // 3. Make the actual API call
  const response = await fetch(providerUrl, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${apiKey}`,
      // Provider-specific headers
      ...(provider === "anthropic" && {
        "x-api-key": apiKey,
        "anthropic-version": "2024-01-01",
      }),
    },
    body: JSON.stringify({
      model: mapModelName(model, provider),
      messages,
      ...rest,
    }),
  });

  const result = await response.json();
  const latencyMs = Date.now() - startTime;

  // 4. Extract token counts
  const usage = extractUsage(result, provider);

  // 5. Log to database (async, don't block response)
  logCall(supabase, {
    model,
    provider,
    app_id: app_id || "unknown",
    user_id,
    session_id,
    input_tokens: usage.input,
    output_tokens: usage.output,
    latency_ms: latencyMs,
    prompt_preview: messages[0]?.content?.substring(0, 500),
    response_preview: extractResponse(result, provider)?.substring(0, 500),
  });

  // 6. Return response to caller
  return new Response(JSON.stringify(result), {
    status: response.status,
    headers: { "Content-Type": "application/json" },
  });
});

function getProvider(model: string): string {
  if (model.startsWith("claude")) return "anthropic";
  if (model.startsWith("gpt") || model.startsWith("o1")) return "openai";
  return "openrouter";  // Default fallback
}

function extractUsage(result: any, provider: string) {
  if (provider === "anthropic") {
    return {
      input: result.usage?.input_tokens || 0,
      output: result.usage?.output_tokens || 0,
    };
  }
  // OpenAI / OpenRouter format
  return {
    input: result.usage?.prompt_tokens || 0,
    output: result.usage?.completion_tokens || 0,
  };
}

async function logCall(supabase: any, data: any) {
  // Fire and forget - don't block the response
  supabase.from("ai_calls").insert(data).then(({ error }) => {
    if (error) console.error("Failed to log AI call:", error);
  });
}
```

### Client SDK

```typescript
// lib/artemis-client.ts
export class ArtemisClient {
  private endpoint: string;
  private appId: string;

  constructor(config: { endpoint: string; appId: string }) {
    this.endpoint = config.endpoint;
    this.appId = config.appId;
  }

  async chat(params: {
    model: string;
    messages: Array<{ role: string; content: string }>;
    userId?: string;
    sessionId?: string;
    [key: string]: any;
  }) {
    const response = await fetch(this.endpoint, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        ...params,
        app_id: this.appId,
        user_id: params.userId,
        session_id: params.sessionId,
      }),
    });

    return response.json();
  }
}

// Usage
const artemis = new ArtemisClient({
  endpoint: "https://your-project.supabase.co/functions/v1/artemis",
  appId: "watts-ai",
});

const response = await artemis.chat({
  model: "claude-3-haiku",
  messages: [{ role: "user", content: "Hello!" }],
  userId: "user_123",
});
```

## Dashboards & Queries

### Daily Cost by App

```sql
SELECT
  app_id,
  DATE(created_at) as date,
  SUM(cost_usd) as daily_cost,
  SUM(total_tokens) as total_tokens,
  COUNT(*) as call_count
FROM ai_calls
WHERE created_at > now() - INTERVAL '30 days'
GROUP BY app_id, DATE(created_at)
ORDER BY date DESC, daily_cost DESC;
```

### Cost by Model

```sql
SELECT
  model,
  COUNT(*) as calls,
  SUM(input_tokens) as input_tokens,
  SUM(output_tokens) as output_tokens,
  SUM(cost_usd) as total_cost,
  AVG(latency_ms) as avg_latency_ms
FROM ai_calls
WHERE created_at > now() - INTERVAL '7 days'
GROUP BY model
ORDER BY total_cost DESC;
```

### Top Users by Cost

```sql
SELECT
  user_id,
  COUNT(*) as calls,
  SUM(total_tokens) as tokens,
  SUM(cost_usd) as cost
FROM ai_calls
WHERE user_id IS NOT NULL
  AND created_at > now() - INTERVAL '30 days'
GROUP BY user_id
ORDER BY cost DESC
LIMIT 20;
```

### Latency Percentiles

```sql
SELECT
  model,
  PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY latency_ms) as p50,
  PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY latency_ms) as p90,
  PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY latency_ms) as p99
FROM ai_calls
WHERE created_at > now() - INTERVAL '24 hours'
GROUP BY model;
```

## Advanced Features

### Model Aliasing

Map friendly names to actual models, enabling easy switching:

```typescript
const MODEL_ALIASES = {
  "fast": "claude-3-haiku",
  "smart": "claude-3-5-sonnet",
  "genius": "claude-3-opus",
  "cheap": "gpt-4o-mini",
  "default": "claude-3-haiku",
};

function resolveModel(alias: string): string {
  return MODEL_ALIASES[alias] || alias;
}
```

### Rate Limiting

```sql
-- Check if user is over quota
CREATE FUNCTION check_user_quota(p_user_id TEXT, p_daily_limit INT)
RETURNS BOOLEAN AS $$
  SELECT COALESCE(SUM(total_tokens), 0) < p_daily_limit
  FROM ai_calls
  WHERE user_id = p_user_id
    AND created_at > now() - INTERVAL '24 hours';
$$ LANGUAGE SQL;
```

### Cost Alerts

```sql
-- Daily cost anomaly detection
WITH daily_costs AS (
  SELECT
    app_id,
    DATE(created_at) as date,
    SUM(cost_usd) as cost
  FROM ai_calls
  WHERE created_at > now() - INTERVAL '14 days'
  GROUP BY app_id, DATE(created_at)
),
stats AS (
  SELECT
    app_id,
    AVG(cost) as avg_cost,
    STDDEV(cost) as stddev_cost
  FROM daily_costs
  WHERE date < CURRENT_DATE  -- Exclude today
  GROUP BY app_id
)
SELECT
  d.app_id,
  d.cost as today_cost,
  s.avg_cost,
  (d.cost - s.avg_cost) / NULLIF(s.stddev_cost, 0) as z_score
FROM daily_costs d
JOIN stats s ON d.app_id = s.app_id
WHERE d.date = CURRENT_DATE
  AND (d.cost - s.avg_cost) / NULLIF(s.stddev_cost, 0) > 2;  -- 2 std devs
```

## Why Not Use Provider Dashboards?

| Feature | Provider Dashboard | Artemis |
|---------|-------------------|---------|
| See all providers in one place | ❌ | ✅ |
| Track by app/user/session | ❌ | ✅ |
| Custom cost allocation | ❌ | ✅ |
| Query raw data | ❌ | ✅ |
| Rate limiting | ❌ | ✅ |
| Model aliasing | ❌ | ✅ |
| Latency tracking | Limited | ✅ |
| Prompt/response logging | ❌ | ✅ |

## Alternatives

| Tool | Best For | Limitation |
|------|----------|------------|
| **LangSmith** | Full LLM observability | Complex, LangChain-focused |
| **Helicone** | Drop-in proxy | SaaS only |
| **Portkey** | Enterprise AI gateway | Complex setup |
| **OpenRouter** | Model routing | No per-app tracking |
| **Artemis** | Simple, self-hosted, Supabase-native | Build it yourself |

## References

- [OpenRouter API](https://openrouter.ai/docs)
- [Anthropic API](https://docs.anthropic.com/en/api)
- [OpenAI API](https://platform.openai.com/docs/api-reference)
- [Supabase Edge Functions](https://supabase.com/docs/guides/functions)

## See Also

- [Webhook Catcher's Mitt](/research/webhook-catchers-mitt/) - Reliable ingestion patterns
- [Hybrid Search with RRF](/research/hybrid-search-rrf/) - Using AI responses in search
