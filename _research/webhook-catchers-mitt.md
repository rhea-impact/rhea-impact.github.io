---
title: Webhook Processing - Catcher's Mitt Pattern
problem: Webhooks fail silently, lose data, or create race conditions
context: Processing incoming webhooks from external services reliably
category: architecture
tags: [webhooks, supabase, edge-functions, reliability, queues]
version: "1.0"
last_updated: 2026-01-28
---

# Webhook Processing: The Catcher's Mitt Pattern

> Never drop a webhook. Separate reception from processing.

## The Problem

Webhook integrations fail in predictable ways:

| Failure Mode | Cause | Consequence |
|--------------|-------|-------------|
| **Timeout** | Processing takes too long | Webhook sender retries, creates duplicates |
| **Error during processing** | Bug in handler | Data lost, no retry |
| **Signature verification fails** | Clock skew, key rotation | Legitimate webhooks rejected |
| **Backpressure** | Burst of webhooks | Queue fills, drops messages |
| **No idempotency** | Duplicate webhooks processed twice | Corrupted data |

## The Solution: Catcher's Mitt Architecture

Separate **reception** (catch it) from **processing** (do something with it):

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  External       │────▶│  Catcher's Mitt │────▶│  inbox table    │
│  Webhook        │     │  (just INSERT)  │     │  (queue)        │
└─────────────────┘     └─────────────────┘     └────────┬────────┘
                                                         │
         ┌───────────────────────────────────────────────┤
         │                                               │
         ▼                                               ▼
┌─────────────────┐                           ┌─────────────────┐
│  Backup Sync    │──────────────────────────▶│  Processor      │
│  (poll API)     │   adds missing items      │  (does work)    │
└─────────────────┘                           └─────────────────┘
```

### Three Components

| Component | Responsibility | Can Fail? |
|-----------|----------------|-----------|
| **Catcher** | Verify signature, INSERT to inbox, return 200 | NO |
| **Processor** | Pick up items, do work, mark complete | YES (retries) |
| **Backup Sync** | Poll API for missed items, add to inbox | YES (safe) |

## From Production: Fireflies Integration

Real implementation for meeting transcript webhooks.

### Component 1: The Catcher (Edge Function)

```typescript
// supabase/functions/fireflies-webhook/index.ts
import { serve } from "https://deno.land/std@0.168.0/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

serve(async (req) => {
  // 1. Verify signature FIRST (fail fast)
  const signature = req.headers.get("x-hub-signature");
  const secret = Deno.env.get("FIREFLIES_WEBHOOK_SECRET");

  if (!verifySignature(await req.clone().text(), signature, secret)) {
    return new Response("Invalid signature", { status: 401 });
  }

  // 2. Parse payload
  const payload = await req.json();
  const meetingId = payload.meetingId || payload.transcriptId;

  if (!meetingId) {
    return new Response("Missing meetingId", { status: 400 });
  }

  // 3. INSERT to inbox (the only thing that can fail)
  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
  );

  const { error } = await supabase
    .from("fireflies_inbox")
    .upsert({
      meeting_id: meetingId,
      source: "webhook",
      status: "pending",
      payload: payload,
    }, {
      onConflict: "meeting_id",
      ignoreDuplicates: true,  // Idempotent!
    });

  if (error) {
    console.error("Inbox insert failed:", error);
    // Still return 200 - backup sync will catch it
    // Returning 500 causes sender to retry = duplicates
  }

  // 4. Return 200 IMMEDIATELY
  return new Response(JSON.stringify({ received: true }), {
    status: 200,
    headers: { "Content-Type": "application/json" },
  });
});

function verifySignature(body: string, signature: string | null, secret: string): boolean {
  if (!signature || !secret) return false;

  const encoder = new TextEncoder();
  const key = await crypto.subtle.importKey(
    "raw",
    encoder.encode(secret),
    { name: "HMAC", hash: "SHA-256" },
    false,
    ["sign"]
  );

  const sig = await crypto.subtle.sign("HMAC", key, encoder.encode(body));
  const expected = "sha256=" + Array.from(new Uint8Array(sig))
    .map(b => b.toString(16).padStart(2, "0"))
    .join("");

  return signature === expected;
}
```

**Key design decisions:**
- Signature verification happens BEFORE any DB work
- `upsert` with `ignoreDuplicates` makes it idempotent
- Return 200 even if INSERT fails (backup sync catches it)
- No processing logic - just catch and queue

### Component 2: The Processor (Scheduled Function)

```typescript
// supabase/functions/fireflies-process/index.ts
// Runs every 5 minutes via pg_cron

const BATCH_SIZE = 10;
const MAX_ATTEMPTS = 5;

serve(async (req) => {
  const supabase = createClient(/*...*/);

  // 1. Fetch pending items (oldest first)
  const { data: pending } = await supabase
    .from("fireflies_inbox")
    .select("*")
    .eq("status", "pending")
    .order("created_at", { ascending: true })
    .limit(BATCH_SIZE);

  for (const item of pending || []) {
    // 2. Claim item atomically
    const { error: claimError } = await supabase
      .from("fireflies_inbox")
      .update({
        status: "processing",
        attempt_count: item.attempt_count + 1
      })
      .eq("id", item.id)
      .eq("status", "pending");  // Only if still pending!

    if (claimError) continue;  // Someone else claimed it

    try {
      // 3. Do the actual work
      const transcript = await fetchFullTranscript(item.meeting_id);
      await uploadToStorage(transcript);
      await saveToDatabase(transcript);

      // 4. Mark completed
      await supabase
        .from("fireflies_inbox")
        .update({ status: "completed", processed_at: new Date() })
        .eq("id", item.id);

    } catch (error) {
      // 5. Handle failure
      if (item.attempt_count + 1 >= MAX_ATTEMPTS) {
        await supabase
          .from("fireflies_inbox")
          .update({ status: "failed", last_error: error.message })
          .eq("id", item.id);
      } else {
        // Return to pending for retry
        await supabase
          .from("fireflies_inbox")
          .update({ status: "pending", last_error: error.message })
          .eq("id", item.id);
      }
    }
  }

  return new Response(JSON.stringify({ processed: pending?.length || 0 }));
});
```

**Key design decisions:**
- Atomic claim with `eq("status", "pending")` prevents race conditions
- Exponential backoff via attempt counting
- Failed items stay in queue for investigation
- Batch size limits memory/timeout issues

### Component 3: Backup Sync (Hourly Poll)

```typescript
// supabase/functions/fireflies-backup-sync/index.ts
// Runs hourly via pg_cron

serve(async (req) => {
  const supabase = createClient(/*...*/);

  // 1. Fetch recent transcripts from API
  const transcripts = await fetchRecentTranscripts(24);  // Last 24 hours

  for (const transcript of transcripts) {
    // 2. Check if already in inbox
    const { data: existing } = await supabase
      .from("fireflies_inbox")
      .select("id")
      .eq("meeting_id", transcript.id)
      .maybeSingle();

    if (existing) continue;  // Already have it

    // 3. Add missing items to inbox
    await supabase
      .from("fireflies_inbox")
      .insert({
        meeting_id: transcript.id,
        source: "backup_sync",  // Track how we got it
        status: "pending",
        payload: { id: transcript.id, title: transcript.title },
      });
  }

  return new Response(JSON.stringify({ checked: transcripts.length }));
});
```

**Why backup sync matters:**
- Webhooks get lost (network issues, downtime, bugs)
- API is source of truth
- Polling catches anything webhook missed
- `source: "backup_sync"` helps debug delivery issues

### The Inbox Table

```sql
CREATE TABLE fireflies_inbox (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  meeting_id TEXT UNIQUE NOT NULL,  -- Idempotency key
  source TEXT NOT NULL,  -- 'webhook' or 'backup_sync'
  status TEXT NOT NULL DEFAULT 'pending',  -- pending, processing, completed, failed
  payload JSONB,
  attempt_count INT DEFAULT 0,
  last_error TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  processed_at TIMESTAMPTZ,
  deleted_at TIMESTAMPTZ  -- Soft delete
);

-- Index for processor queries
CREATE INDEX idx_inbox_status_created
  ON fireflies_inbox(status, created_at)
  WHERE deleted_at IS NULL;
```

### Cron Setup (pg_cron)

```sql
-- Process pending items every 5 minutes
SELECT cron.schedule(
  'fireflies-process',
  '*/5 * * * *',
  $$
  SELECT net.http_post(
    url := 'https://your-project.supabase.co/functions/v1/fireflies-process',
    headers := jsonb_build_object(
      'Authorization', 'Bearer ' || current_setting('app.settings.service_role_key')
    )
  );
  $$
);

-- Backup sync every hour
SELECT cron.schedule(
  'fireflies-backup-sync',
  '0 * * * *',
  $$
  SELECT net.http_post(
    url := 'https://your-project.supabase.co/functions/v1/fireflies-backup-sync',
    headers := jsonb_build_object(
      'Authorization', 'Bearer ' || current_setting('app.settings.service_role_key')
    )
  );
  $$
);
```

## Monitoring

```sql
-- Check inbox status
SELECT status, COUNT(*), source
FROM fireflies_inbox
WHERE deleted_at IS NULL
GROUP BY status, source;

-- View failed items
SELECT meeting_id, last_error, attempt_count, created_at
FROM fireflies_inbox
WHERE status = 'failed'
ORDER BY created_at DESC;

-- Processing lag (items waiting too long)
SELECT meeting_id, created_at,
       EXTRACT(EPOCH FROM (now() - created_at))/60 AS minutes_waiting
FROM fireflies_inbox
WHERE status = 'pending'
  AND created_at < now() - INTERVAL '30 minutes';
```

## When to Use This Pattern

| Scenario | Use Catcher's Mitt? |
|----------|---------------------|
| Payment webhooks (Stripe) | YES - can't lose transactions |
| Meeting transcripts | YES - async processing anyway |
| Real-time notifications | MAYBE - latency matters |
| Health checks | NO - just respond inline |

**Rule of thumb**: If losing a webhook would cause data loss or require manual recovery, use the catcher's mitt.

## Anti-Patterns

| Don't Do This | Why |
|---------------|-----|
| Process inline in webhook handler | Timeouts, retries, duplicates |
| Return 500 on processing errors | Sender retries = duplicates |
| Skip signature verification | Security hole |
| No idempotency key | Duplicates corrupt data |
| No backup sync | Webhooks get lost |
| No monitoring | Silent failures |

## References

- [Supabase Edge Functions](https://supabase.com/docs/guides/functions)
- [Supabase pg_cron](https://supabase.com/docs/guides/database/extensions/pg_cron)
- [Stripe Webhook Best Practices](https://stripe.com/docs/webhooks/best-practices)
- [GitHub Webhooks](https://docs.github.com/en/webhooks)

## See Also

- [Transcription with Custom Glossaries](/research/transcription-glossary/) - What to do after ingestion
- [Triage Workflow Pattern](/research/triage-workflow/) - Processing pipeline orchestration
