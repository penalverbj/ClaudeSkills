---
name: ai-prompt-hireflow
description: >
  AI prompt engineering patterns for HireFlow's Claude API integrations. Use this
  skill whenever writing or modifying prompts, adding new AI features, choosing
  between Claude Haiku and Sonnet, handling structured JSON output from Claude,
  building the job description generator, resume parser, candidate scorer, or
  comparison feature. Also use when debugging unexpected AI output, adding prompt
  templates to lib/ai/prompts.ts, or any time the words "prompt", "Claude API",
  "AI scoring", "job description generation", "resume parsing", "structured output",
  "model selection", or "token cost" come up. Always consult this skill before
  writing any string that will be sent to the Anthropic API.
---

# AI Prompt Engineering — HireFlow

## Core Principles

1. **All prompts live in `lib/ai/prompts.ts`.** Never inline a prompt in a route handler, component, or Trigger.dev job. One file, all prompts, always.
2. **Always request JSON output explicitly.** Tell Claude the exact shape you expect. Validate the response with Zod before using it.
3. **System prompt = persona + constraints. User prompt = data.** Keep them separate.
4. **Choose the right model.** Haiku for extraction and scoring. Sonnet for reasoning and comparison. Never use Opus in production paths.
5. **Handle malformed output gracefully.** Claude occasionally produces slightly off JSON. Always wrap parsing in try/catch with a fallback.

---

## Model Selection Guide

| Task | Model | Reason |
|---|---|---|
| Resume text parsing / extraction | `claude-haiku-4-5` | Structured extraction — no deep reasoning needed |
| Job description generation | `claude-haiku-4-5` | Template-like output, fast, cheap |
| Candidate scoring (standard) | `claude-haiku-4-5` | Scoring rubric is well-defined |
| Candidate scoring (deep review) | `claude-sonnet-4-5` | More nuanced analysis, Pro plan only |
| Candidate comparison (Pro) | `claude-sonnet-4-5` | Multi-candidate reasoning requires stronger model |
| Any real-time streamed response | `claude-haiku-4-5` | Lower latency to first token |

**Never use `claude-opus-*` in any production code path.** Cost is prohibitive at scale.

---

## File Structure

```
lib/ai/
├── prompts.ts          # ALL prompt templates — the only place they live
├── client.ts           # Anthropic SDK instance
├── generate-description.ts
├── parse-resume.ts
├── score-resume.ts
├── compare-candidates.ts
└── embeddings.ts       # Voyage AI calls
```

---

## Client Setup

```typescript
// lib/ai/client.ts
import Anthropic from '@anthropic-ai/sdk'

export const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY!,
})

// Model constants — change here, updates everywhere
export const MODELS = {
  fast:     'claude-haiku-4-5',
  standard: 'claude-haiku-4-5',
  deep:     'claude-sonnet-4-5',
} as const
```

---

## Prompt Templates

```typescript
// lib/ai/prompts.ts

// ── Job Description Generation ────────────────────────────────────────────

export const JOB_DESCRIPTION_SYSTEM = `
You are an expert HR writer who creates clear, compelling job descriptions for
small businesses. Your descriptions are:
- Direct and specific — no corporate fluff or buzzwords
- Inclusive in language — avoid gendered terms
- Realistic about requirements — don't overqualify for the role
- Honest about what the job actually involves day to day

You must respond with ONLY a valid JSON object matching the schema provided.
No preamble, no explanation, no markdown fences. Only the JSON object.
`.trim()

export function jobDescriptionUserPrompt(input: JobDescriptionInput): string {
  return `
Generate a job description for this role:

Job Title: ${input.jobTitle}
Employment Type: ${input.employmentType}
Work Setting: ${input.workSetting}
${input.payMin ? `Pay Range: $${input.payMin}–$${input.payMax} ${input.payPeriod}` : ''}
${input.companyDescription ? `About the Business: ${input.companyDescription}` : ''}
${input.keyResponsibilities ? `Key Responsibilities (user's words): ${input.keyResponsibilities}` : ''}
${input.mustHaveSkills ? `Must-Have Requirements: ${input.mustHaveSkills}` : ''}
${input.niceToHaveSkills ? `Nice-to-Have: ${input.niceToHaveSkills}` : ''}
Tone: ${input.tone ?? 'professional but approachable'}

Respond with this exact JSON structure:
{
  "title": "string — final job title",
  "summary": "string — 2-3 sentence elevator pitch for the role",
  "responsibilities": ["string", "string", ...],
  "requirements": ["string", "string", ...],
  "niceToHave": ["string", "string", ...],
  "salaryText": "string — natural language pay description, or empty string if not provided"
}
`.trim()
}


// ── Resume Parsing ────────────────────────────────────────────────────────

export const RESUME_PARSE_SYSTEM = `
You are a resume parser. Extract structured information from the resume text.
Be conservative — only extract information that is clearly stated.
Do not infer, embellish, or fill in gaps.

You must respond with ONLY a valid JSON object. No preamble, no markdown.
`.trim()

export function resumeParseUserPrompt(resumeText: string): string {
  return `
Extract structured data from this resume:

---
${resumeText.slice(0, 6000)}  // Cap at ~1500 tokens
---

Respond with this exact JSON structure:
{
  "fullName": "string",
  "email": "string or null",
  "phone": "string or null",
  "location": "string or null",
  "skills": ["string", ...],
  "experience": [
    {
      "company": "string",
      "title": "string",
      "startDate": "string or null",
      "endDate": "string or null (use 'Present' if current)",
      "description": "string — 1-2 sentence summary"
    }
  ],
  "education": [
    {
      "institution": "string",
      "degree": "string or null",
      "field": "string or null",
      "graduationYear": "string or null"
    }
  ],
  "totalYearsExperience": "number or null — your best estimate"
}
`.trim()
}


// ── Candidate Scoring ─────────────────────────────────────────────────────

export const CANDIDATE_SCORE_SYSTEM = `
You are an objective hiring assistant evaluating candidates for small businesses.
Score candidates honestly — a 60 is a good candidate, not a failure.
Do not inflate scores. Be specific in your summary about strengths and gaps.

Scoring rubric:
- 85-100: Exceptional match. Exceeds most requirements.
- 70-84:  Strong match. Meets all key requirements with some gaps.
- 55-69:  Adequate match. Meets core requirements, notable gaps.
- 40-54:  Partial match. Missing key requirements but has transferable value.
- 0-39:   Poor match. Significant gaps across requirements.

You must respond with ONLY a valid JSON object. No preamble, no markdown.
`.trim()

export function candidateScoreUserPrompt(input: CandidateScoreInput): string {
  return `
Score this candidate for the following role.

JOB: ${input.jobTitle}
DESCRIPTION:
${input.jobDescription?.slice(0, 2000) ?? 'No description provided'}

CANDIDATE RESUME SUMMARY:
${input.resumeText.slice(0, 3000)}

PARSED SKILLS: ${input.parsedSkills?.join(', ') ?? 'Not extracted'}
ESTIMATED EXPERIENCE: ${input.yearsExperience ?? 'Unknown'} years

Respond with this exact JSON structure:
{
  "overall": number (0-100),
  "skills": number (0-100),
  "experience": number (0-100),
  "requirements": number (0-100),
  "summary": "string — 2-3 sentences. Lead with their strongest quality, then the most important gap.",
  "strengths": ["string", "string"],
  "gaps": ["string", "string"],
  "redFlags": ["string"] or []
}
`.trim()
}


// ── Candidate Comparison (Pro) ────────────────────────────────────────────

export const CANDIDATE_COMPARE_SYSTEM = `
You are a hiring advisor helping a small business owner make a final hiring decision.
Be direct and practical. The owner is not an HR professional — avoid jargon.
Give a clear recommendation with reasoning.

You must respond with ONLY a valid JSON object. No preamble, no markdown.
`.trim()

export function candidateCompareUserPrompt(input: CandidateCompareInput): string {
  const candidateList = input.candidates.map((c, i) =>
    `CANDIDATE ${i + 1}: ${c.name}\nScore: ${c.overallScore}/100\nSummary: ${c.summary}\nStrengths: ${c.strengths.join(', ')}\nGaps: ${c.gaps.join(', ')}`
  ).join('\n\n')

  return `
Help me choose between these ${input.candidates.length} candidates for: ${input.jobTitle}

${candidateList}

Respond with this exact JSON structure:
{
  "ranking": [
    {
      "candidateName": "string",
      "rank": number,
      "justification": "string — 1-2 sentences"
    }
  ],
  "topPickName": "string — name of recommended candidate",
  "topPickReason": "string — 2-3 sentence plain-english explanation for a business owner",
  "keyDifferentiator": "string — the single most important factor in this decision",
  "questionsToAsk": ["string", "string"] — 2-3 interview questions to validate top pick
}
`.trim()
}
```

---

## Calling the API

### Standard (non-streaming)

```typescript
// lib/ai/score-resume.ts
import { anthropic, MODELS } from './client'
import { CANDIDATE_SCORE_SYSTEM, candidateScoreUserPrompt } from './prompts'
import { z } from 'zod'

const scoreSchema = z.object({
  overall:      z.number().min(0).max(100),
  skills:       z.number().min(0).max(100),
  experience:   z.number().min(0).max(100),
  requirements: z.number().min(0).max(100),
  summary:      z.string(),
  strengths:    z.array(z.string()),
  gaps:         z.array(z.string()),
  redFlags:     z.array(z.string()),
})

export async function scoreCandidate(input: CandidateScoreInput) {
  const response = await anthropic.messages.create({
    model:      MODELS.standard,
    max_tokens: 1000,
    system:     CANDIDATE_SCORE_SYSTEM,
    messages: [
      { role: 'user', content: candidateScoreUserPrompt(input) }
    ],
  })

  const raw = response.content[0].type === 'text'
    ? response.content[0].text
    : ''

  return parseJsonResponse(raw, scoreSchema, defaultScore())
}

// Shared JSON parser with fallback
function parseJsonResponse<T>(raw: string, schema: z.ZodSchema<T>, fallback: T): T {
  try {
    // Strip markdown fences if Claude accidentally adds them
    const cleaned = raw.replace(/```json\n?|\n?```/g, '').trim()
    const parsed = JSON.parse(cleaned)
    return schema.parse(parsed)
  } catch (err) {
    console.error('Failed to parse Claude JSON response:', raw, err)
    return fallback
  }
}

function defaultScore() {
  return {
    overall: 0, skills: 0, experience: 0, requirements: 0,
    summary: 'Score could not be generated.',
    strengths: [], gaps: [], redFlags: [],
  }
}
```

### Streaming (job description generation)

```typescript
// lib/ai/generate-description.ts
import { anthropic, MODELS } from './client'
import { JOB_DESCRIPTION_SYSTEM, jobDescriptionUserPrompt } from './prompts'

export async function streamJobDescription(input: JobDescriptionInput) {
  // Returns a stream — the route handler pipes this to the response
  return anthropic.messages.stream({
    model:      MODELS.fast,
    max_tokens: 1500,
    system:     JOB_DESCRIPTION_SYSTEM,
    messages: [
      { role: 'user', content: jobDescriptionUserPrompt(input) }
    ],
  })
}

// app/api/ai/generate-description/route.ts
import { streamJobDescription } from '@/lib/ai/generate-description'

export async function POST(req: Request) {
  // ... auth + validation ...
  const stream = await streamJobDescription(body)

  // Stream the response directly to the client
  return new Response(stream.toReadableStream(), {
    headers: { 'Content-Type': 'text/event-stream' },
  })
}
```

---

## Voyage AI Embeddings

```typescript
// lib/ai/embeddings.ts
// Voyage AI doesn't have an official npm SDK — use fetch directly

export async function generateEmbedding(text: string): Promise<number[]> {
  const response = await fetch('https://api.voyageai.com/v1/embeddings', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.VOYAGE_API_KEY}`,
      'Content-Type':  'application/json',
    },
    body: JSON.stringify({
      model: 'voyage-3.5',
      input: text.slice(0, 32000),  // Voyage 3.5 context limit
    }),
  })

  const data = await response.json()
  return data.data[0].embedding  // number[] — 1024 dimensions
}

// Storing in Supabase pgvector:
await adminClient.from('ai_scores').insert({
  // ... other fields ...
  embedding: JSON.stringify(embedding),  // pgvector accepts JSON array
})

// Querying by similarity (in a SQL function or RPC):
// SELECT *, 1 - (embedding <=> query_embedding) AS similarity
// FROM ai_scores
// ORDER BY embedding <=> query_embedding
// LIMIT 10;
```

---

## Token Budget Reference

Keep prompts under these limits to avoid unexpected costs:

| Prompt Type | Input Tokens | Output Tokens | Cost (Haiku) |
|---|---|---|---|
| Job description gen | ~800 | ~600 | ~$0.0000038 |
| Resume parsing | ~800 | ~400 | ~$0.0000028 |
| Candidate scoring | ~1,600 | ~400 | ~$0.0000152 |
| Candidate comparison (3) | ~2,500 | ~600 | ~$0.000028 (Sonnet) |

Cap resume text at 3,000 characters (~750 tokens) and job descriptions at 2,000 characters (~500 tokens) in all prompts. Beyond this, returns diminish.

---

## Prompt Quality Rules

```typescript
// ❌ WRONG: Vague persona, no output constraints
system: "You are a helpful assistant."
user: "Score this resume."

// ✅ CORRECT: Specific persona, explicit JSON contract
system: CANDIDATE_SCORE_SYSTEM   // Defines scorer persona + rubric + JSON instruction
user: candidateScoreUserPrompt(input)  // Provides data in consistent structure


// ❌ WRONG: Trusting Claude's JSON without validation
const text = response.content[0].text
const data = JSON.parse(text)  // Throws if Claude adds a preamble

// ✅ CORRECT: Strip fences, parse, validate schema, use fallback
const data = parseJsonResponse(text, scoreSchema, defaultScore())


// ❌ WRONG: Sending entire resume text uncapped
user: `Score this resume: ${resumeText}`  // Could be 50,000 tokens for a long CV

// ✅ CORRECT: Cap at a safe limit
user: `Score this resume: ${resumeText.slice(0, 3000)}`


// ❌ WRONG: Inline prompt string in a route handler
const response = await anthropic.messages.create({
  system: "You are an HR assistant...",  // Hard to find, test, or update
  messages: [{ role: 'user', content: `Score ${resumeText}` }]
})

// ✅ CORRECT: Import from prompts.ts
import { CANDIDATE_SCORE_SYSTEM, candidateScoreUserPrompt } from '@/lib/ai/prompts'
```

---

## Adding a New AI Feature Checklist

- [ ] System prompt added to `lib/ai/prompts.ts` as a named export
- [ ] User prompt added as a function that takes typed input
- [ ] Model chosen from `MODELS` constant — not a hardcoded string
- [ ] Response parsed with Zod schema
- [ ] Fallback value defined for when parsing fails
- [ ] Input text capped to prevent token overruns
- [ ] Called from a Trigger.dev job if latency > 5s, otherwise from route handler
- [ ] `model_used` stored in the database if scores are persisted
- [ ] Feature gated to correct plan tier (comparison = Pro only)
