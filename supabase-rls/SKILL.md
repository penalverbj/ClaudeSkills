---
name: supabase-rls
description: >
  Supabase Row Level Security (RLS) patterns for HireFlow's multi-tenant Postgres database.
  Use this skill whenever creating or modifying database tables, writing SQL migrations,
  adding new org-scoped queries, auditing data access patterns, debugging unexpected
  data visibility, or any time the words "RLS", "row level security", "policy",
  "tenant isolation", or "org_id" come up. Also use when adding new Supabase queries
  in API routes or server components — every new table needs RLS policies before data
  is inserted. If you're ever unsure whether a query is properly scoped, consult this skill.
---

# Supabase RLS Patterns — HireFlow

## The Core Rule

Every table that contains org data MUST have RLS enabled and policies written before any data is inserted. An org must never be able to read, write, or delete another org's data — even if the application code has a bug.

RLS is the **last line of defence**, not the only one. Application code also filters by `org_id`. Both must be correct.

---

## Enabling RLS

Every new table gets these two statements in its migration:

```sql
-- Always enable immediately after CREATE TABLE
ALTER TABLE your_table ENABLE ROW LEVEL SECURITY;
ALTER TABLE your_table FORCE ROW LEVEL SECURITY;  -- Applies to table owner too
```

**Never skip `FORCE ROW LEVEL SECURITY`.** Without it, the Supabase service role (used by your API) bypasses all policies silently.

---

## Auth Helper

All policies use this helper to get the authenticated user's org:

```sql
-- Add this function once, used in all policies
CREATE OR REPLACE FUNCTION auth.org_id() RETURNS uuid AS $$
  SELECT (auth.jwt() -> 'user_metadata' ->> 'org_id')::uuid
$$ LANGUAGE sql STABLE;
```

In policies, reference it as `auth.org_id()`. This is derived from the JWT — the client cannot forge it.

---

## Standard Policy Templates

### Read-only table (SELECT only from client)

```sql
CREATE POLICY "org members can read their own rows"
ON your_table FOR SELECT
USING (org_id = auth.org_id());
```

### Full CRUD table

```sql
-- SELECT: members can read their org's rows
CREATE POLICY "org members can read"
ON your_table FOR SELECT
USING (org_id = auth.org_id());

-- INSERT: members can insert into their own org only
CREATE POLICY "org members can insert"
ON your_table FOR INSERT
WITH CHECK (org_id = auth.org_id());

-- UPDATE: members can only update their own org's rows
CREATE POLICY "org members can update"
ON your_table FOR UPDATE
USING (org_id = auth.org_id())
WITH CHECK (org_id = auth.org_id());

-- DELETE / SOFT DELETE: use with caution — prefer soft deletes
-- In HireFlow we use deleted_at, so hard DELETE is rarely needed
CREATE POLICY "org members can delete"
ON your_table FOR DELETE
USING (org_id = auth.org_id());
```

### Role-gated operations (owner/admin only)

```sql
-- Only owners can perform destructive actions
CREATE POLICY "only owners can archive jobs"
ON jobs FOR UPDATE
USING (
  org_id = auth.org_id()
  AND EXISTS (
    SELECT 1 FROM users
    WHERE users.id = auth.uid()
    AND users.org_id = auth.org_id()
    AND users.role = 'owner'
  )
)
WITH CHECK (org_id = auth.org_id());
```

### Public read (candidate application form — no auth)

```sql
-- /apply/:token — unauthenticated candidate can read job info
CREATE POLICY "public can read active jobs by token"
ON jobs FOR SELECT
USING (
  status = 'active'
  AND apply_link_enabled = true
  -- token matching is handled in the application layer, not RLS
  -- RLS just ensures only active jobs are readable publicly
);
```

### Append-only tables (stage_history, billing_events)

```sql
-- INSERT only — no UPDATE or DELETE ever
CREATE POLICY "org members can insert stage history"
ON stage_history FOR INSERT
WITH CHECK (
  org_id = auth.org_id()
  AND moved_by = auth.uid()
);

CREATE POLICY "org members can read stage history"
ON stage_history FOR SELECT
USING (org_id = auth.org_id());

-- No UPDATE policy — intentionally omitted
-- No DELETE policy — intentionally omitted
```

---

## HireFlow Table Policy Checklist

Run through this for every table in the schema:

| Table | RLS | SELECT | INSERT | UPDATE | DELETE | Notes |
|---|---|---|---|---|---|---|
| organizations | ✓ | own org | service role only | owner only | never | Billing fields updated via service role in webhook handler |
| users | ✓ | own org | service role only | own row | owner only | — |
| jobs | ✓ | own org | admin/owner | admin/owner | soft-delete only | Public read for active+token |
| applications | ✓ | own org | public (anon) for apply form | own org | soft-delete only | Anon insert for candidates |
| ai_scores | ✓ | own org | service role only | never | never | Written only by Trigger.dev job |
| stage_history | ✓ | own org | own org | never | never | Append-only |
| billing_events | ✓ | own org | service role only | never | never | Webhook handler uses service role |
| invitations | ✓ | own org | admin/owner | service role | admin/owner | — |

---

## Service Role Usage

The Supabase **service role** (using `SUPABASE_SERVICE_ROLE_KEY`) bypasses RLS entirely. Use it only in:

- Stripe webhook handler (updating `organizations.plan` and inserting `billing_events`)
- Trigger.dev resume pipeline (inserting `ai_scores`)
- Admin migration scripts

**Never use the service role in route handlers that serve end-user requests.** Use the session-scoped client instead.

```typescript
// lib/supabase/server.ts — session-scoped (respects RLS)
import { createServerClient } from '@supabase/ssr'
export function createClient() { /* reads cookies */ }

// lib/supabase/admin.ts — bypasses RLS (service role)
import { createClient } from '@supabase/supabase-js'
export const adminClient = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!  // Server only — never expose
)
```

---

## Testing RLS Policies

Always test policies in the Supabase SQL editor before shipping a migration.

### Test as a specific user

```sql
-- Impersonate a user to test their policy access
SET LOCAL role TO authenticated;
SET LOCAL request.jwt.claims TO '{"sub": "user-uuid-here", "user_metadata": {"org_id": "org-uuid-here"}}';

-- Now this query should only return rows for that org
SELECT * FROM jobs;

-- This should return 0 rows (different org's data)
SELECT * FROM jobs WHERE org_id = 'different-org-uuid';
```

### Test as anon (candidate applying)

```sql
SET LOCAL role TO anon;
-- Should be able to read active jobs
SELECT id, title FROM jobs WHERE apply_token = 'token-here';
-- Should NOT be able to read other orgs' candidates
SELECT * FROM applications;  -- Should return 0 rows
```

### Common RLS bugs to check

```sql
-- 1. Is RLS actually enabled?
SELECT tablename, rowsecurity FROM pg_tables
WHERE schemaname = 'public' AND tablename = 'your_table';
-- rowsecurity should be TRUE

-- 2. What policies exist?
SELECT * FROM pg_policies WHERE tablename = 'your_table';

-- 3. Are there rows a user shouldn't see?
-- Run a SELECT as that user and compare with service role SELECT
```

---

## Migration Template

Every new table migration follows this structure:

```sql
-- migrations/YYYYMMDD_add_your_table.sql

CREATE TABLE your_table (
  id      uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id  uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  -- ... other columns
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now(),
  deleted_at timestamptz  -- soft delete
);

-- Indexes (always index org_id on tenant-scoped tables)
CREATE INDEX idx_your_table_org_id ON your_table(org_id);
CREATE INDEX idx_your_table_org_deleted ON your_table(org_id) WHERE deleted_at IS NULL;

-- RLS (always immediately after table creation)
ALTER TABLE your_table ENABLE ROW LEVEL SECURITY;
ALTER TABLE your_table FORCE ROW LEVEL SECURITY;

-- Policies
CREATE POLICY "org members can read"
ON your_table FOR SELECT
USING (org_id = auth.org_id() AND deleted_at IS NULL);

CREATE POLICY "org members can insert"
ON your_table FOR INSERT
WITH CHECK (org_id = auth.org_id());

CREATE POLICY "org members can update"
ON your_table FOR UPDATE
USING (org_id = auth.org_id() AND deleted_at IS NULL)
WITH CHECK (org_id = auth.org_id());
```

---

## Common Mistakes

```sql
-- ❌ WRONG: Policy without org check leaks all rows
CREATE POLICY "users can read jobs"
ON jobs FOR SELECT
USING (true);  -- Every user can see every org's jobs!

-- ✅ CORRECT
CREATE POLICY "users can read own org jobs"
ON jobs FOR SELECT
USING (org_id = auth.org_id());

-- ❌ WRONG: INSERT policy missing WITH CHECK
CREATE POLICY "users can insert"
ON jobs FOR INSERT
USING (org_id = auth.org_id());  -- USING on INSERT does nothing

-- ✅ CORRECT: INSERT uses WITH CHECK
CREATE POLICY "users can insert"
ON jobs FOR INSERT
WITH CHECK (org_id = auth.org_id());

-- ❌ WRONG: Forgetting deleted_at in SELECT policy
CREATE POLICY "users can read"
ON jobs FOR SELECT
USING (org_id = auth.org_id());  -- Returns soft-deleted rows!

-- ✅ CORRECT: Exclude soft-deleted rows
CREATE POLICY "users can read"
ON jobs FOR SELECT
USING (org_id = auth.org_id() AND deleted_at IS NULL);
```

---

## Application Layer (Double Defence)

Even with RLS, always filter by org_id in queries:

```typescript
// ✅ Belt and suspenders — both RLS and app filter
const { data } = await supabase
  .from('jobs')
  .select('*')
  .eq('org_id', user.org_id)    // App layer
  .is('deleted_at', null)
  // RLS also enforces org_id automatically

// ❌ Relying on RLS alone (acceptable but bad habit)
const { data } = await supabase
  .from('jobs')
  .select('*')
  // RLS will filter, but explicit is better
```
