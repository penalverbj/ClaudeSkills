---
name: triggerdev-async
description: >
  Trigger.dev background job patterns for HireFlow's async AI resume pipeline.
  Use this skill whenever writing or modifying Trigger.dev jobs, handling the
  resume processing pipeline, enqueueing background work from API routes, dealing
  with job retries or failure states, updating UI-facing status from background jobs,
  sending completion notifications, or any time the words "background job", "async
  pipeline", "Trigger.dev", "resume processing", "scoring queue", "job task", or
  "enqueue" come up. Always use this skill before touching the trigger/ directory
  or any code that calls triggerClient.sendEvent().
---

# Trigger.dev Async Job Patterns — HireFlow

## Why Async?

Resume AI scoring takes 10–15 seconds. Vercel serverless functions time out at 10s (hobby) or 60s (pro). Running the full pipeline synchronously in an API route would timeout, block the request, and leave the UI hanging. Trigger.dev offloads the work to a background runner that can run for minutes, retries on failure, and notifies the UI when complete.

---

## File Structure

```
trigger/
├── index.ts               # TriggerClient setup + job exports
├── resume-pipeline.ts     # Main resume processing job
└── notifications.ts       # In-app notification helpers

lib/
└── trigger/
    └── client.ts          # Shared client for enqueueing from API routes
```

---

## Client Setup

```typescript
// trigger/index.ts
import { TriggerClient } from '@trigger.dev/sdk'

export const client = new TriggerClient({
  id: 'hireflow',
  apiKey: process.env.TRIGGER_SECRET_KEY!,
  apiUrl: process.env.TRIGGER_API_URL,  // Only needed for self-hosted
})

// Export all jobs so Trigger.dev discovers them
export { resumePipelineJob } from './resume-pipeline'
```

```typescript
// lib/trigger/client.ts — used in API routes to enqueue jobs
import { client } from '@/trigger'

export async function enqueueResumePipeline(payload: ResumePipelinePayload) {
  return client.sendEvent({
    name: 'resume.uploaded',
    payload,
  })
}
```

---

## The Resume Pipeline Job

```typescript
// trigger/resume-pipeline.ts
import { client } from './index'
import { eventTrigger } from '@trigger.dev/sdk'
import { z } from 'zod'
import { adminClient } from '@/lib/supabase/admin'
import { extractTextFromPdf } from '@/lib/ai/pdf-extract'
import { parseResumeStructure } from '@/lib/ai/parse-resume'
import { generateEmbedding } from '@/lib/ai/embeddings'
import { scoreCandidate } from '@/lib/ai/score-resume'

// Payload schema — validate everything coming in
const payloadSchema = z.object({
  applicationId: z.string().uuid(),
  jobId:         z.string().uuid(),
  orgId:         z.string().uuid(),
  resumePath:    z.string(),  // Supabase storage path
})

export const resumePipelineJob = client.defineJob({
  id:          'resume-pipeline',
  name:        'Resume Processing Pipeline',
  version:     '1.0.0',
  trigger:     eventTrigger({ name: 'resume.uploaded' }),
  // Retry config — resume processing can fail on transient AI errors
  retryConfig: {
    maxAttempts: 3,
    factor:      2,
    minTimeoutInMs: 2000,    // 2s initial wait
    maxTimeoutInMs: 30000,   // 30s max wait between retries
    randomize:    true,
  },

  run: async (payload, io, ctx) => {
    // Validate payload
    const { applicationId, jobId, orgId, resumePath } =
      payloadSchema.parse(payload)

    // Fetch job description for scoring context
    const { data: job } = await adminClient
      .from('jobs')
      .select('title, description')
      .eq('id', jobId)
      .single()

    if (!job) throw new Error(`Job ${jobId} not found`)

    // ── Step 1: Extract text from PDF ─────────────────────────────────
    const resumeText = await io.runTask('extract-pdf-text', async () => {
      const { data: fileData } = await adminClient.storage
        .from('resumes')
        .download(resumePath)
      if (!fileData) throw new Error(`Resume not found at ${resumePath}`)
      return extractTextFromPdf(fileData)
    })

    // ── Step 2: Parse structure with Claude Haiku ──────────────────────
    const parsedResume = await io.runTask('parse-resume-structure', async () => {
      return parseResumeStructure(resumeText)
      // Returns: { name, skills[], experience[], education[] }
    })

    // ── Step 3: Generate embedding with Voyage AI ──────────────────────
    const embedding = await io.runTask('generate-embedding', async () => {
      return generateEmbedding(resumeText)
      // Returns: number[] (1024 dimensions for Voyage 3.5)
    })

    // ── Step 4: Score candidate against job description ────────────────
    const scores = await io.runTask('score-candidate', async () => {
      return scoreCandidate({
        resumeText,
        parsedResume,
        jobTitle:       job.title,
        jobDescription: job.description,
      })
      // Returns: { overall, skills, experience, requirements, summary }
    })

    // ── Step 5: Write results to database ─────────────────────────────
    await io.runTask('save-scores', async () => {
      const { error } = await adminClient
        .from('ai_scores')
        .insert({
          application_id:     applicationId,
          job_id:             jobId,
          overall_score:      scores.overall,
          skills_score:       scores.skills,
          experience_score:   scores.experience,
          requirements_score: scores.requirements,
          summary:            scores.summary,
          extracted_skills:   parsedResume.skills,
          model_used:         'claude-haiku-4-5',
          processed_at:       new Date().toISOString(),
          processing_ms:      ctx.run.durationMs,
        })
      if (error) throw error
    })

    // ── Step 6: Notify hiring manager ─────────────────────────────────
    await io.runTask('notify-hiring-manager', async () => {
      await notifyOrgOfNewScore({ orgId, applicationId, jobId, scores })
    })

    return { success: true, applicationId, score: scores.overall }
  },
})
```

---

## Enqueueing from an API Route

```typescript
// app/api/apply/[token]/route.ts — after saving application + resume
import { enqueueResumePipeline } from '@/lib/trigger/client'

export async function POST(req: Request, { params }: { params: { token: string } }) {
  // ... validate token, save application, upload resume ...

  // Enqueue async scoring — fire and forget
  // The UI will poll for ai_scores to appear
  await enqueueResumePipeline({
    applicationId: newApplication.id,
    jobId:         job.id,
    orgId:         job.org_id,
    resumePath:    uploadedResumePath,
  })

  // Return immediately — don't wait for scoring
  return Response.json({ data: { applicationId: newApplication.id }, error: null })
}
```

---

## UI Polling Pattern

The frontend polls for score completion. There is no websocket — polling is simpler and sufficient.

```typescript
// hooks/useApplicationScore.ts
import { useEffect, useState } from 'react'

export function useApplicationScore(applicationId: string) {
  const [score, setScore] = useState<AiScore | null>(null)
  const [polling, setPolling] = useState(true)

  useEffect(() => {
    if (!polling) return

    const interval = setInterval(async () => {
      const res = await fetch(`/api/applications/${applicationId}/score`)
      const { data } = await res.json()

      if (data) {
        setScore(data)
        setPolling(false)  // Stop polling once score arrives
      }
    }, 3000)  // Poll every 3 seconds

    // Timeout after 2 minutes — something went wrong
    const timeout = setTimeout(() => {
      setPolling(false)
    }, 120_000)

    return () => {
      clearInterval(interval)
      clearTimeout(timeout)
    }
  }, [applicationId, polling])

  return { score, pending: polling && !score }
}

// Usage in component:
// const { score, pending } = useApplicationScore(application.id)
// pending ? <ScoringSpinner /> : <ScoreCard score={score} />
```

---

## Task Wrapping with `io.runTask`

Every step in a job must be wrapped in `io.runTask`. This is critical:

- Tasks are **checkpointed** — if the job crashes and retries, completed tasks are skipped
- Tasks appear in the **Trigger.dev dashboard** for debugging
- Tasks have **individual timeouts** configurable per step

```typescript
// ✅ CORRECT — each step is a named, checkpointed task
const text = await io.runTask('extract-pdf', async () => extractPdf(path))
const scores = await io.runTask('score-resume', async () => scoreIt(text))

// ❌ WRONG — side-by-side work without task wrapper
// If this crashes at step 3, the whole job retries from the top
// Step 1 and 2 run again, wasting AI credits
const text = await extractPdf(path)     // Runs again on retry
const parsed = await parseText(text)    // Runs again on retry
const scores = await scoreIt(parsed)    // Runs again on retry
```

---

## Error Handling

```typescript
// In a task: throw to trigger retry
await io.runTask('score-candidate', async () => {
  const result = await anthropic.messages.create({ ... })
  if (!result.content[0]) throw new Error('Empty response from Claude')  // Will retry
  return result
})

// Distinguish retryable vs permanent errors
await io.runTask('parse-resume', async () => {
  try {
    return await parseWithClaude(text)
  } catch (err) {
    if (err instanceof AnthropicRateLimitError) {
      throw err  // Retryable — Trigger.dev will wait and retry
    }
    if (err instanceof InvalidPdfError) {
      // Non-retryable — update application with error state and return
      await adminClient.from('applications').update({
        notes: 'Resume could not be parsed — invalid PDF format'
      }).eq('id', applicationId)
      return null  // Return null to skip scoring gracefully
    }
    throw err
  }
})
```

---

## Checking Job Status

```typescript
// In the Trigger.dev dashboard: https://app.trigger.dev
// Filter by job ID, event name, or application ID in payload

// Programmatically check a specific run:
const run = await client.getRun(runId)
console.log(run.status)  // 'SUCCESS' | 'FAILURE' | 'RUNNING' | 'PENDING'

// In the API route, you can return the run ID to the client
// for debugging purposes (not required for production flow)
const event = await enqueueResumePipeline(payload)
// event.id is the Trigger.dev run ID
```

---

## Local Development

```bash
# In one terminal: run your Next.js dev server
npm run dev

# In another terminal: connect to Trigger.dev cloud with local runner
npx trigger dev

# The local runner forwards jobs from Trigger.dev cloud to your local process
# This means you need a Trigger.dev account even for local dev
# But it also means you can debug jobs with real cloud events
```

---

## Checklist for New Jobs

- [ ] Payload validated with Zod at the start of `run`
- [ ] Every async step wrapped in `io.runTask` with a descriptive name
- [ ] `retryConfig` set with sensible backoff values
- [ ] Non-retryable errors caught and handled without re-throwing
- [ ] Database writes use `adminClient` (service role)
- [ ] Job exported from `trigger/index.ts` so Trigger.dev discovers it
- [ ] Tested locally with `npx trigger dev` + manual event send
- [ ] UI has a loading/pending state for the duration of job execution
- [ ] Timeout handled in the UI (stop polling after N seconds)
