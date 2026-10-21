# JobPilot Architecture

This document describes how JobPilot is put together: its runtime boundaries, authentication model, data flow, agent pipelines, and known gaps. It reflects the code in this repository, not an intended future design.

## Scope and status

JobPilot is a single Next.js App Router application backed by InsForge, Browserbase, Stagehand, and OpenAI. There is no separate backend service, worker process, or job queue. Browser automation runs inside ordinary HTTP requests handled by API routes.

The stable path is:

```text
profile → LinkedIn discovery → GPT-4o scoring → job inventory → resume generation/tailoring
```

The apply automation is experimental. The manual per-job flow is reachable from the UI. The queued auto-apply flow exists in code but is not wired to the current UI.

## System context

```text
┌─────────────────────────────────────────────────────────────────────┐
│ Browser                                                             │
│  ├─ Server-rendered pages                                           │
│  ├─ Client components (forms, tables, agent controls)               │
│  └─ InsForge Realtime subscription                                  │
└───────────────┬───────────────────────────────┬─────────────────────┘
                │ HTTP                          │ WebSocket
                ▼                               ▼
┌───────────────────────────────────────┐   ┌─────────────────────────┐
│ Next.js server                        │   │ InsForge Realtime       │
│  ├─ proxy.ts           session gate   │   └─────────────────────────┘
│  ├─ app/**/page.tsx    data loading   │
│  ├─ actions/*          mutations      │
│  ├─ app/api/**/route   operations     │
│  └─ agent/*            automation     │
└───────┬───────────────┬───────────────┬─────────────────────────────┘
        │               │               │
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────────────────────────┐
│ InsForge     │ │ Browserbase  │ │ OpenAI                           │
│ PostgreSQL   │ │ sessions     │ │ matching, writing, Stagehand,    │
│ auth         │ │ recordings   │ │ AgentSpan model                  │
│ storage      │ │ contexts     │ └──────────────────────────────────┘
└──────────────┘ └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │ Stagehand    │
                 │ browser agent│
                 └──────────────┘
```

## Runtime layers

### `proxy.ts`

The Next.js 16 proxy guards:

```text
/dashboard/:path*
/profile/:path*
/jobs/:path*
```

If an access token cookie exists, the request continues. Otherwise the proxy:

1. reads the refresh token cookie
2. calls the InsForge refresh endpoint with the anon key
3. rotates access, refresh, and CSRF cookies
4. redirects to `/login` if refresh fails

This is a session gate, not an authorization layer for individual rows. Route handlers and pages still verify the user and scope database queries.

### `app/**/page.tsx`

Server components load data directly from InsForge using the authenticated server client. Examples:

- `/dashboard` reads the latest runs and inventory statistics.
- `/jobs` reads jobs, runs, and profile defaults, then filters, sorts, and paginates in memory.
- `/jobs/[id]` reads one job and the profile’s resume and LinkedIn flags.
- `/profile` reads the full profile row.

### `actions/*`

Server Actions handle short mutations:

- start OAuth and sign out
- save profile
- dismiss a review job

They revalidate affected paths after writes. They do not call agent functions.

### `app/api/**/route.ts`

Route handlers own operations that are too long-running or too external for a page render:

| Route | Responsibility |
| --- | --- |
| `POST /api/agent/find` | LinkedIn discovery plus match scoring |
| `POST /api/agent/stop` | Mark the latest active run as stopped |
| `POST /api/agent/apply` | AgentSpan queued auto-apply |
| `POST /api/jobs/[id]/apply` | Manual per-job apply attempt |
| `POST /api/resume/generate` | Base resume generation |
| `POST /api/resume/tailor` | Per-job resume tailoring |
| `GET /api/resume/preview` | Base resume preview |
| `GET /api/resume/download` | Base resume download |
| `GET /api/resume/jobs/[id]/preview` | Tailored resume preview |
| `GET /api/resume/jobs/[id]/download` | Tailored resume download |
| `GET /api/auth/callback` | OAuth code exchange |
| `GET /api/auth/realtime-token` | Token for the browser realtime client |

These routes contain no UI and no page rendering logic. They authenticate, validate input, call agent or resume functions, and return JSON.

### `agent/*`

The agent modules are server-only TypeScript libraries:

| Module | Responsibility |
| --- | --- |
| `linkedin.ts` | Search, DOM extraction, LLM extraction, apply-mode detection, job persistence |
| `matcher.ts` | GPT-4o scoring and match persistence |
| `cover-letter.ts` | GPT-4o cover-letter drafting |
| `apply.ts` | Easy Apply, external ATS form filling, queue state transitions |
| `index.ts` | AgentSpan agent and tool definition |
| `recording.ts` | Persist Browserbase session and recording identifiers |

They import shared infrastructure from `lib/` but not components or Server Actions. Only API routes import agent modules.

### `lib/*`

| Module | Responsibility |
| --- | --- |
| `auth.ts` | Token/user lookup, refresh, cookie helpers, redirect safety |
| `auth-cookies.ts` | Cookie names |
| `insforge-server.ts` | User-scoped InsForge client for server code |
| `insforge-client.ts` | Public browser client |
| `browserbase.ts` | Session, context, recording, and release operations |
| `stagehand.ts` | Stagehand session creation and teardown |
| `resume.tsx` | Resume content generation, PDF rendering, storage paths |
| `utils.ts` | `MATCH_THRESHOLD` and the `cn` helper |

## Authentication and session model

```text
User
  │  click "Continue with Google/GitHub"
  ▼
Server Action: startOAuth()
  ├─ InsForge signInWithOAuth({ skipBrowserRedirect: true })
  ├─ set code-verifier cookie (10 min)
  ├─ set intended-destination cookie (10 min)
  └─ redirect to provider
        │
        ▼
Provider → /api/auth/callback?insforge_code=...
  ├─ read verifier cookie
  ├─ exchange code for tokens
  ├─ write access/refresh/CSRF cookies
  ├─ check profiles.is_complete
  └─ redirect to /profile or the intended path
```

Cookie behavior:

| Cookie | Lifetime | HttpOnly | Purpose |
| --- | --- | --- | --- |
| `insforge_access_token` | 15 minutes | Yes | API and database access |
| `insforge_refresh_token` | 7 days | Yes | Session refresh |
| `insforge_csrf_token` | 7 days | Yes | Present, but not validated by app code |
| `insforge_code_verifier` | 10 minutes | Yes | OAuth PKCE exchange |
| `jobpilot_oauth_next` | 10 minutes | Yes | Post-login destination |

The server client passes the user’s access token to InsForge:

```ts
createClient({
  baseUrl: process.env.NEXT_PUBLIC_INSFORGE_URL,
  anonKey: process.env.INSFORGE_ANON_KEY ?? process.env.NEXT_PUBLIC_INSFORGE_ANON_KEY,
  edgeFunctionToken: accessToken,
  isServerMode: true,
});
```

No service-role key is used in application code. Multi-user isolation therefore depends on InsForge row-level security. The repository contains no schema or policy definitions, so those policies must be configured and verified in the InsForge project.

## Data model

The application expects four database tables and one storage bucket. The exact column types and constraints are not defined in the repository.

### `profiles`

One row per authenticated user, keyed by the InsForge user ID.

```text
identity        full_name, email, phone, location
targeting       job_title, experience_level, years_experience, skills, remote_preference
links           linkedin_url, portfolio_url
resume evidence resume_work_experience, resume_projects,
                resume_education, resume_achievements
application     work_authorization, sponsorship_requirement, salary_expectation,
                start_availability, relocation_preference, background_check_consent,
                gender_identity, hispanic_latino_identity, veteran_status,
                disability_status, application_notes
state           is_complete
files           resume_pdf_url
browser         linkedin_context_id, linkedin_connected
```

### `jobs`

The central work item.

```text
identity        id, run_id, user_id
listing         title, company, location, description
links           linkedin_url, external_apply_url
scoring         match_score, match_reason
workflow        status, error_message
materials       cover_letter, resume_url, is_tailored
timestamps      found_at, applied_at
```

Observed statuses:

```text
found ──────► applying ──────► applied
  │                │
  │                └─────────► failed
  └──────────────► dismissed

queued ─────► applying ──────► applied / failed
needs_input   (present in UI types; not written by current code)
```

### `agent_runs`

One row per discovery or apply run.

```text
identity        id, user_id
status          finding | applying | completed | failed | stopped
search          job_title_searched, location_searched
counts          jobs_found, jobs_matched, jobs_applied,
                jobs_failed, jobs_in_review
timeline        started_at, completed_at
browser         browserbase_session_id, browserbase_recording_url
orchestration   agentspan_run_id
```

### `agent_logs`

Append-only activity records used by the dashboard feed.

```text
run_id, user_id, job_id, level, message, created_at
```

Levels in use: `info`, `success`, `warning`, `error`.

### Storage

Bucket: `resumes`

```text
resumes/{userId}/resume.pdf
resumes/{userId}/{jobId}.pdf
```

The first path is the base resume. The second is the tailored resume for one job. URLs are stored in `profiles.resume_pdf_url` and `jobs.resume_url`. Deterministic paths allow regeneration to replace the previous object safely.

## Discovery and matching

`POST /api/agent/find` performs a complete cycle in one HTTP request.

```text
authenticate
  │
  ├─ insert agent_runs row (status=finding)
  │
  ├─ discoverLinkedInJobs()
  │    ├─ load profiles.linkedin_context_id
  │    ├─ create Browserbase session (600 s, persisted context)
  │    ├─ navigate to LinkedIn search URL
  │    ├─ reject login/authwall pages
  │    ├─ extract job cards from DOM
  │    ├─ extract job cards with Stagehand + Zod schema
  │    ├─ merge and deduplicate
  │    ├─ visit detail pages to determine apply mode
  │    ├─ insert jobs
  │    └─ close session, return recording URL
  │
  ├─ matchDiscoveredJobs()
  │    ├─ load profile and found jobs
  │    ├─ one GPT-4o call per job
  │    ├─ update score, reason, status
  │    └─ update run counts
  │
  └─ return counts
```

### Dual extraction

LinkedIn cards are read twice:

1. A deterministic DOM scraper collects visible cards.
2. Stagehand `extract()` returns structured jobs validated by a Zod schema.

Results are normalized, merged, deduplicated by job identity, and capped by the requested limit. Detail pages are then visited to classify each listing as Easy Apply or external apply.

### Matching

`matcher.ts` builds a payload from the profile and a job, calls GPT-4o, validates the response, and writes:

```text
jobs.match_score
jobs.match_reason
jobs.status = "found"
```

`MATCH_THRESHOLD` in `lib/utils.ts` is currently `70`. It controls how runs and the UI classify strong matches. Jobs below the threshold are still persisted with status `found`; they are not filtered out of the database.

Matching is sequential and uses one model call per job. Cost and latency scale with the number of listings.

### Execution characteristics

- LinkedIn discovery and matching run inside the same request.
- The Browserbase session timeout is 600 seconds.
- No job queue or background worker is involved.
- If the request is terminated, the run can remain in an active state until reconciled.
- The discovery recording URL is returned to the caller but is not persisted by `discoverLinkedInJobs` itself.

## Resume generation and tailoring

```text
Base resume
  profiles row
    → buildResumeProfile()
    → generateResumeContent()           GPT-4o → structured content
    → renderResumePdf()                 @react-pdf/renderer
    → upload resumes/{userId}/resume.pdf
    → update profiles.resume_pdf_url

Tailored resume
  profiles row + job description + match context
    → generateTailoredResumeContent()   GPT-4o → structured content
    → renderResumePdf()
    → upload resumes/{userId}/{jobId}.pdf
    → update jobs.resume_url, jobs.is_tailored
```

The model does not generate a PDF directly. It returns structured content that React PDF components render, which keeps the layout consistent.

Validation gates:

- the user must be signed in
- the profile must exist and be complete
- the profile must contain enough work/project/achievement evidence
- tailoring additionally requires an existing base resume and a job with sufficient context

Preview and download routes read the storage object and verify ownership before streaming it.

There is no resume upload feature. The base resume is generated from profile evidence.

## Apply architecture

### Status guard

`applyManualJob()` only proceeds when the job is in `found` or `failed`. It rejects jobs that are already applied or in another state.

### Manual Easy Apply

```text
load profile and job
  → require linkedin_context_id and linkedin_connected
  → create Stagehand session with saved context
  → save recording URL to agent_runs
  → open linkedin_url
  → Stagehand DOM agent, max 18 steps
  → extract submission result
  → applied or failed
```

The Easy Apply path is gated on two profile fields that the current UI cannot write. Rows must be prepared externally.

### Manual external apply

```text
load profile and job
  → require external_apply_url
  → require base or tailored resume
  → generate cover letter if missing
  → create fresh Browserbase session (600 s, experimental)
  → save recording URL immediately
  → navigate to application page
  → build applicant variables
  → Stagehand hybrid agent, max 40 steps
  → on failure, run post-fill review extraction
  → close session and delete temp resume
  → update job status and logs
```

The post-fill review reports:

- filled fields
- empty required fields
- likely misfilled fields
- blockers
- whether submission should be allowed

Manual mode passes the public resume URL to the agent rather than relying on the temporary local file.

### Queued auto-apply with AgentSpan

```text
POST /api/agent/apply
  → beginApplyToQueuedJobs()
      → set run to applying
      → load profile
      → load jobs where status = queued
  → AgentRuntime.run(jobPilotApplyAgent)
      → tool: apply_queued_job
      → stepId = apply-{jobId}
      → idempotencyKey = same stepId
  → applySingleQueuedJob()
      → applyToJob({ mode: "auto" })
  → complete or fail run
```

Properties:

- one tool, one job per call
- deterministic step ID
- idempotency key
- AgentSpan run ID stored on `agent_runs`
- per-job timeout of 900 seconds and runtime timeout of 1,200 seconds

Differences from the manual path:

- 120-second browser session instead of 600
- no post-fill review extraction
- no experimental flag
- requires a job in `queued` status

Nothing in the current application sets a job to `queued`. This path is therefore dormant unless rows are inserted or updated externally.

### Stop behavior

`POST /api/agent/stop` finds the latest active run and marks it `stopped`. It does not abort the running request, close the Browserbase session, or cancel an AgentSpan execution. It is a database-level reconciliation action.

## Browser and model infrastructure

### Browserbase

`lib/browserbase.ts` centralizes:

- session creation with recording enabled
- CAPTCHA solving
- optional advanced stealth
- optional persistent context
- metadata sanitization
- session release
- deterministic recording URLs

Metadata values are sanitized to letters, digits, `_`, `.`, `:`, and `-`, truncated to 96 characters, because Browserbase rejects other characters.

### Stagehand

`lib/stagehand.ts` centralizes:

- connection to a Browserbase session
- the model configuration (`openai/gpt-4o`)
- CAPTCHA handling
- experimental mode
- forced close on initialization failure

### Model usage

`OPENAI_API_KEY` is used by:

1. direct chat completions for matching and cover letters
2. structured resume content generation
3. the Stagehand browser agent
4. the AgentSpan agent

Rotating this key affects every AI feature simultaneously.

## Logging, recordings, and realtime

Agent modules write progress to `agent_logs` using the user-scoped client. Apply attempts persist the session ID and recording URL to `agent_runs` before the browser work starts, so failed attempts remain reviewable.

The dashboard live feed:

1. requests `/api/auth/realtime-token`
2. receives the access token as JSON
3. creates a browser InsForge client
4. subscribes to log changes

This exposes the short-lived access token to client JavaScript, whereas the token is otherwise held in an HttpOnly cookie. The tradeoff is acceptable only if XSS risk is controlled.

## Configuration

| Variable | Used by |
| --- | --- |
| `NEXT_PUBLIC_INSFORGE_URL` | Server and browser clients |
| `NEXT_PUBLIC_INSFORGE_ANON_KEY` | Browser client |
| `INSFORGE_ANON_KEY` | Optional server-side anon key |
| `BROWSERBASE_API_KEY` | Session and Stagehand creation |
| `BROWSERBASE_PROJECT_ID` | Session and context creation |
| `BROWSERBASE_ADVANCED_STEALTH` | Optional stealth mode |
| `OPENAI_API_KEY` | Matching, writing, resume text, Stagehand, AgentSpan |
| `NEXT_PUBLIC_APP_URL` | OAuth redirect and secure-cookie decisions |
| `AGENTSPAN_SERVER_URL` | Optional AgentSpan runtime endpoint |

OAuth redirect URLs are configured in `insforge.toml`. Keep local and deployed values separate. Do not commit real keys or environment-specific callback URLs.

## Deployment characteristics

The code assumes Next.js hosting, remote InsForge, remote Browserbase, and OpenAI. `getAppUrl()` falls back to `VERCEL_URL` and then `http://localhost:3000`.

Operational constraints:

- discovery and manual apply can occupy an HTTP request for many minutes
- serverless function duration limits must accommodate the browser timeout
- there is no cron, queue, or worker
- temporary resume files use the OS temp directory
- there is no rate limiting on expensive authenticated endpoints
- there are no database migrations or RLS policy files
- no automated tests are present

## Security considerations

### Good properties

- Access and refresh tokens are HttpOnly.
- OAuth uses a code verifier and a validated internal `next` path.
- No service-role key is present in application code.
- Database access uses the user’s token.
- Most read paths scope by `user_id`; several job updates address a row by ID after an ownership-scoped read.
- Apply attempts save evidence before performing browser work.

### Risks to verify

- **Row-level security:** the app relies on InsForge policies that are not in the repository.
- **CSRF:** the CSRF cookie is stored but never checked by application code; `SameSite=Lax` is the effective protection.
- **Realtime token exposure:** the access token is returned to the browser for the realtime client.
- **XSS:** profile and job content are rendered in the UI; a successful XSS could use the realtime token.
- **No rate limiting:** authenticated users can repeatedly trigger paid OpenAI and Browserbase work.
- **Recordings:** Browserbase recordings may contain personal data from forms.
- **Temporary files:** resumes are written to the temp directory during apply and deleted afterward; a crash can leave them behind.
- **Missing write path for LinkedIn:** the current UI cannot establish the context required by discovery and Easy Apply.

## Known gaps

These are observable in the current code, not hypothetical future work:

1. **No LinkedIn connection endpoint or UI.** `profiles.linkedin_context_id` and `linkedin_connected` are read but never written.
2. **No transition to `queued`.** The AgentSpan auto-apply route cannot be reached through normal UI usage.
3. **Stop is not cancellation.** It updates the run row only.
4. **No formal schema.** Tables, columns, constraints, indexes, and policies must be created separately.
5. **No tests.** Status transitions, auth refresh, and apply branching are unguarded.
6. **The discovery recording is not persisted.** Only apply recordings are written to `agent_runs`.
7. **`needs_input` is unused.** Easy Apply review cases are saved as `failed` with a review message.
8. **`AGENTS.md` conflicts with the code.** It states that Easy Apply is never touched and that Tailwind 3.4 is used; the code implements Easy Apply and `package.json` uses Tailwind 4.

## Where to change what

| Change | Location |
| --- | --- |
| Add a page or route | `app/` |
| Add a simple mutation | `actions/` |
| Change discovery behavior | `agent/linkedin.ts` |
| Change scoring prompt or threshold behavior | `agent/matcher.ts`, `lib/utils.ts` |
| Change resume layout or prompts | `lib/resume.tsx` |
| Change apply behavior | `agent/apply.ts` |
| Change AgentSpan orchestration | `agent/index.ts` |
| Change session setup | `lib/browserbase.ts`, `lib/stagehand.ts` |
| Change auth or cookie behavior | `lib/auth.ts`, `proxy.ts`, `app/api/auth/` |
| Change data access | `lib/insforge-server.ts` and the calling page/route |

## Verification notes

This document was checked against:

- `package.json` and `package-lock.json`
- `proxy.ts`
- `lib/auth.ts`, `lib/auth-cookies.ts`
- `lib/insforge-server.ts`, `lib/insforge-client.ts`
- `lib/browserbase.ts`, `lib/stagehand.ts`
- `agent/linkedin.ts`, `agent/matcher.ts`, `agent/apply.ts`,
  `agent/index.ts`, `agent/cover-letter.ts`, `agent/recording.ts`
- `app/api/**/route.ts`
- `app/**/page.tsx`
- `actions/*`
- `insforge.toml`

It intentionally does not document credentials, real callback URLs, personal data, or contents of internal reports.
