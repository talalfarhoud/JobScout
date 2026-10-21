# JobPilot

JobPilot is a Next.js application for running and tracking a technical job search in one place. It collects LinkedIn listings through an authenticated Browserbase session, compares each listing with a saved candidate profile using an OpenAI model, and stores the results in an InsForge-backed inventory. From that inventory you can review matches, follow the original listing or apply link, generate a role-specific resume, and optionally try the project's experimental browser-automation paths.

> **Project status:** The discovery, scoring, inventory, and resume workflow is the stable part of the project. Automated application submission is experimental and is not presented as a reliable replacement for applying manually. There is currently no license file in the repository.

## Capabilities

### Job discovery and ranking

- Search LinkedIn by job title and location using a reusable Browserbase context that keeps the LinkedIn session authenticated between runs. The context ID and connection flag are read from the `profiles` table.
- Extract the title, company, location, LinkedIn URL, external apply URL, and Easy Apply flag from each listing.
- Score listings against the saved profile with GPT-4o and store the score plus a short rationale.
- Treat scores at or above the threshold in `lib/utils.ts` (`MATCH_THRESHOLD`, currently 70) as strong matches; lower scores remain available for review.

### Job inventory and review

- Filterable, sortable job table with match scores, apply type, source links, and application status.
- Job detail view with the full description, match breakdown, original LinkedIn link, external apply link, and resume controls.
- Dashboard with run statistics, a review queue, recent agent logs streamed through InsForge Realtime, and links to saved Browserbase recordings.

### Resume and cover-letter assistance

- Profile editor covering contact details, target role, experience, skills, work history, projects, education, achievements, and application preferences.
- Base resume generation from saved profile evidence as a PDF, with the result stored in InsForge Storage.
- Per-job resume tailoring based on the job description.
- Cover-letter drafting when an apply attempt needs one, using the tone selected in the profile.

### Experimental apply automation

- LinkedIn Easy Apply path using Stagehand DOM mode and the saved LinkedIn context.
- External ATS path using Stagehand hybrid mode and an applicant-variable prompt.
- Session recordings saved for each attempt so the run can be inspected afterwards.
- Post-fill review extraction that reports empty or likely misfilled fields and blocks submission when the form is not ready.

Both apply paths are marked experimental in the UI. The external ATS path in particular has unreliable field targeting and can stop before submitting. Review every attempt and use the original application link when automation is not appropriate.

## Technology

| Area | Used in this repository |
| --- | --- |
| Framework | Next.js 16.2.6 (App Router, Server Actions, Route Handlers) |
| UI | React 19, TypeScript 5, Tailwind CSS 4, shadcn/ui (`radix-nova` style), Radix UI, Lucide icons |
| Backend | InsForge (PostgreSQL, authentication, storage, realtime) |
| Browser automation | Browserbase and Stagehand |
| Orchestration | AgentSpan SDK |
| Language models | OpenAI GPT-4o |
| PDF generation | `@react-pdf/renderer` |
| Validation | Zod |

Version numbers above come from `package.json` and `package-lock.json`. In particular, the project uses Next.js 16 and Tailwind CSS 4, not the Next.js 15 / Tailwind 3.4 combination mentioned in earlier project notes.

## Prerequisites

- Node.js 20.9.0 or newer, as required by Next.js 16.2.6.
- npm, which is the package manager used by the checked-in lockfile.
- An InsForge project with authentication, a PostgreSQL database, and a storage bucket named `resumes`.
- Google and/or GitHub OAuth configured in InsForge for sign-in.
- A Browserbase account with an API key and project ID.
- An OpenAI API key with access to GPT-4o.
- A LinkedIn account for the browser session used during discovery.
- Optional: an AgentSpan server if you want durable apply runs. The code defaults to `http://localhost:6767/api`.

The repository does not include database migrations or a schema definition. You must create the tables expected by the application (`profiles`, `jobs`, `agent_runs`, and `agent_logs`) and the `resumes` storage bucket in your InsForge project before the app can store data.

## Installation

```bash
git clone <your-repository-url>
cd JobPilot
npm install
```

Create `.env.local` in the project root and fill in the values described below. Then start the development server:

```bash
npm run dev
```

Open `http://localhost:3000`.

Other available scripts:

```bash
npm run build   # production build
npm run start   # serve the production build
npm run lint    # run ESLint
```

## Configuration

The application reads the following environment variables.

### Required

| Variable | Purpose |
| --- | --- |
| `NEXT_PUBLIC_INSFORGE_URL` | Base URL of your InsForge project. |
| `NEXT_PUBLIC_INSFORGE_ANON_KEY` | Public anon key used by the browser client. |
| `BROWSERBASE_API_KEY` | Browserbase API key used to create sessions. |
| `BROWSERBASE_PROJECT_ID` | Browserbase project that owns the sessions. |
| `OPENAI_API_KEY` | Used for matching, cover letters, resume tailoring, and Stagehand. |

### Optional

| Variable | Purpose |
| --- | --- |
| `INSFORGE_ANON_KEY` | Optional server-side alternative to the public anon key. |
| `NEXT_PUBLIC_APP_URL` | Public URL of the app. Defaults to `VERCEL_URL` when deployed on Vercel, otherwise `http://localhost:3000`. |
| `BROWSERBASE_ADVANCED_STEALTH` | Set to `true` to enable Browserbase advanced stealth mode. |
| `AGENTSPAN_SERVER_URL` | AgentSpan server endpoint. Defaults to `http://localhost:6767/api`. |

Example `.env.local` using placeholders only:

```env
NEXT_PUBLIC_INSFORGE_URL=https://YOUR_PROJECT.REGION.insforge.app
NEXT_PUBLIC_INSFORGE_ANON_KEY=YOUR_INSFORGE_ANON_KEY
BROWSERBASE_API_KEY=YOUR_BROWSERBASE_API_KEY
BROWSERBASE_PROJECT_ID=YOUR_BROWSERBASE_PROJECT_ID
OPENAI_API_KEY=YOUR_OPENAI_API_KEY
NEXT_PUBLIC_APP_URL=http://localhost:3000

# Optional
# BROWSERBASE_ADVANCED_STEALTH=true
# AGENTSPAN_SERVER_URL=http://localhost:6767/api
```

The current source code does not reference `INSFORGE_SERVICE_ROLE_KEY`, `AGENTSPAN_API_KEY`, or PostHog variables. Earlier documentation listed those names; they are omitted here because nothing in this version reads them.

For OAuth, register the callback route `/api/auth/callback` with InsForge and add the matching redirect URLs to `insforge.toml`. Keep separate entries for local development and each deployed environment.

## Using the application

1. **Sign in.** Use Google or GitHub through the InsForge OAuth flow.
2. **Complete the profile.** Add the details used for matching and application forms, then generate the base resume PDF from the saved profile evidence.
3. **Provide a LinkedIn context.** Discovery expects `profiles.linkedin_context_id` and `profiles.linkedin_connected = true` for the signed-in user. The current UI does not include a LinkedIn connection workflow, so these values must be created and saved through your own InsForge setup before discovery or Easy Apply can run.
4. **Run a search.** On the dashboard, enter a job title and location. The agent opens LinkedIn, extracts listings, saves them, and scores them against the profile.
5. **Review results.** Open the job inventory to filter and sort matches. Each job page shows the score, rationale, description, and application links.
6. **Prepare materials.** Generate a tailored resume for a role, then preview or download it. A cover letter is drafted automatically when an apply attempt needs one.
7. **Apply.** Use the LinkedIn or external application link directly. If you choose an experimental automation path, inspect the saved recording and the post-fill review before treating the attempt as complete.

## Project layout

```text
app/            Pages, layouts, and API route handlers
actions/        Server Actions for authentication, profile, and jobs
agent/          LinkedIn discovery, matching, cover letters, apply automation, recordings
components/     UI components, including shadcn/ui primitives
lib/            Auth, InsForge clients, Browserbase/Stagehand setup, utilities
types/          Shared TypeScript types
proxy.ts        Session refresh and route protection
insforge.toml   InsForge authentication redirect configuration
```

## Operational notes and troubleshooting

- **Startup fails with a missing Browserbase variable.** `BROWSERBASE_API_KEY` and `BROWSERBASE_PROJECT_ID` are required before any session can be created.
- **LinkedIn discovery asks you to connect LinkedIn.** Check that `profiles.linkedin_context_id` is populated and `profiles.linkedin_connected` is `true` for the user. If both are set, the stored Browserbase context has probably expired and a new context must be created outside the current UI.
- **OAuth redirects fail.** Confirm that the callback URL registered with InsForge matches the URL used to open the app, including protocol and port.
- **Queued auto-apply runs fail immediately.** AgentSpan defaults to a local server at `http://localhost:6767/api`. Start that server or set `AGENTSPAN_SERVER_URL`. The per-job manual apply endpoint does not use AgentSpan.
- **The apply agent stops without submitting.** This is expected behavior when the review step finds missing or uncertain fields. Open the original application link and finish manually.
- **Database errors after setup.** Verify that the required tables and the `resumes` bucket exist in InsForge; the repository does not create them automatically.

## Security and responsible use

- Never commit `.env.local` or real API keys. The repository ignores `.env*` files, but confirm this before publishing a fork.
- Profiles and generated resumes contain personal data. Browserbase recordings and agent logs may also contain personal information from forms and job listings.
- Automated browsing and form submission can conflict with the terms of service of LinkedIn or an employer's application system. Use automation only with accounts and forms you are authorized to access, and review submissions before sending them.
- This project is not a compliance-certified recruiting product.

## License

No license file is present in this repository. Do not assume that the code may be copied, modified, or redistributed. Add an appropriate license before publishing the project if you intend to grant those permissions.
