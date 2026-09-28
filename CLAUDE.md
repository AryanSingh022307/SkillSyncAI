@AGENTS.md

# SkillSync AI

> "Align your skills with what the market actually needs."

Hackathon-ready AI career-intelligence platform (ASYNC 2026, team Psytech) for students, fresh graduates and early-career professionals.

**Core flow:** Resume/Profile → Skill Extraction → Market Requirements → Readiness Score → Gap Analysis → Priority Skills → 4-Week Roadmap → AI Mock Interview → Feedback → Improved Roadmap.

**Target roles (`RoleId`):** `software-engineer`, `frontend-developer`, `backend-developer`, `data-analyst`, `embedded-systems-engineer`, `ai-ml-engineer`.

## Stack

Next.js 16 (App Router, Turbopack) · React 19 · TypeScript (strict) · Tailwind CSS v4 (CSS-first `@theme` in `globals.css`, no tailwind.config) · `lucide-react` icons · `geist` (self-hosted fonts — Google Fonts is NOT used so builds work offline) · `unpdf` (server-side PDF text). OpenAI is called with plain `fetch` (no SDK); OpenAI + Supabase are optional, server-side only.

## Architecture

```
src/
  app/                    # Routes only. Pages stay thin: compose feature components.
    api/<name>/route.ts   # Server route handlers (the ONLY place AI/DB is called)
  components/
    ui/                   # Design-system primitives (Button, Card, Badge, ScoreRing…). No business logic.
    layout/               # Navbar, Footer, PageHeader, Container, background
    landing/              # Sections for "/"
    <feature>/            # analyze/, dashboard/, roadmap/, market/, interview/ (added per phase)
  lib/
    types.ts              # ALL shared data models — single source of truth
    constants.ts          # Routes, nav items, role list, storage keys
    utils.ts              # cn(), clamp(), formatters (pure)
    data/                 # skills.ts (taxonomy + aliases), roles.ts (requirements), market.ts (demo snapshot), sample-resumes.ts,
                          #   prerequisites.ts (ordering graph), learning-templates.ts (per-skill roadmap content + category fallback),
                          #   interview-questions.ts (local question bank: BEHAVIORAL_QUESTIONS + per-role TECHNICAL_QUESTIONS)
    engine/               # Pure logic, usable client+server:
                          #   resume-sections → extract-skills → assess (per-skill gap/priority)
                          #   → readiness (deterministic score) → insights (top gaps, heatmap, market insight)
                          #   skill-gap.ts = benchmarkSkills() orchestrator; validate.ts = input rules
                          #   roadmap.ts = generateRoadmap() — selects/orders/builds the 4-week plan from an assessment
                          #   interview.ts = slotType()/selectTopicSeed()/pickOneLocalQuestion()/selectLocalQuestions() —
                          #     deterministic, gap-aware question topic selection (AI, if used, may only reword — never re-select)
    market/provider.ts    # MarketDataProvider interface + curated provider; getMarketData() is the only entry point
    store/                # Client state: external store + localStorage persistence
    hooks/                # use-proctoring, use-webcam, use-answer-recorder, use-speech-transcript, use-elapsed-timer
                          #   (each degrades gracefully to a simulated/disabled state — never throws, never blocks the interview)
    server/               # `import "server-only"`: env, pdf (unpdf), ai-extract (OpenAI), analysis-service (pipeline),
                          #   ai-roadmap.ts (optional per-week content rewrite, never selection/order/numbers),
                          #   ai-interview.ts (optional per-question wording rewrite, never topic/category selection)
```

**Rule of the layers:** `app` → `components` → `lib/store` (client) / `app/api` → `lib/engine` + `lib/server` → `lib/data`. Components never import from `lib/server`. `lib/engine` is pure (no fetch, no env, no React).

## Routes

| Route | Purpose | Render |
|---|---|---|
| `/` | Landing: hero, 5-step flow, market radar, readiness heatmap, roadmap, interview showcases | Server (runs the real engine on a sample resume at build time) |
| `/analyze` | PDF upload / pasted text + role + level → live progress → skill breakdown | Client → `POST /api/analyze` |
| `/dashboard` | Stats, readiness score + “How this score is calculated”, top 3 gaps, market insight, gap heatmap, sortable skill inventory, role switcher | Client (reads store; role switch → `/api/skill-gap`) |
| `/roadmap` | 4-week personalised plan generated from real gaps, task progress | Client (reads store) → `POST /api/roadmap` |
| `/market` | Trending, fast-growing, skill demand, job demand by role, skills by role, emerging scatter | Server (`getMarketData()`, revalidate 1h) + client role tabs |
| `/mock-interview` | Proctored mock interview: setup → 5-question session (webcam panel, recording, live transcript, tab-switch/blur proctoring, auto-ends after >3 violations with no feedback) → completion summary with per-question, per-skill scored feedback | Client → `/api/mock-interview/question` (per-question), `/api/mock-interview/feedback` (on completion) |
| `/api/analyze` | multipart (`file` or `text`, `targetRole`, `experienceLevel`) → 4xx JSON error, or 200 NDJSON stream of `AnalyzeEvent` (`stage`… → `result` \| `error`) | Server |
| `/api/skill-gap` | POST JSON `{ targetRole, experienceLevel, skills: [{ skillId\|name, proficiency, evidence? }] }` → `SkillGapReport` | Server |
| `/api/market` | GET `MarketData` (`?role=` filters roles) | Server |
| `/api/roadmap` | POST JSON `{ targetRole, experienceLevel, skills: [{ skillId\|name, proficiency, evidence? }] }` → `Roadmap` (4 weeks). Gaps are always recomputed server-side from `skills`, never trusted from the client | Server |
| `/api/mock-interview/question` | POST JSON `{ targetRole, difficulty, interviewType, skills?, gaps?, slotIndex?, count?, excludeIds?, excludeCategories? }` → `{ question: InterviewQuestion }` (includes `category`, `difficulty`, `expectedTopics`). Called once per question (per `slotIndex`) so the client can accumulate a no-repeat session; topic/category is always chosen deterministically by `engine/interview.ts` from real gaps, AI (if configured) may only rewrite wording | Server |
| `/api/mock-interview/feedback` | POST JSON `{ session: InterviewSession }` → `InterviewFeedback` (overall/technical/behavioral/communication scores, `strengths`/`growthAreas` as `FeedbackArea[]`, `perQuestion: QuestionFeedback[]`). Scoring is always deterministic (`engine/interview-feedback.ts`, topic coverage + elaboration + specificity); AI (if configured) may only polish the top-level `summary` sentence. Rejected (400) if `session.endedReason === "violations"` | Server |
| `/api/status` | Reports which integrations are live (`ai`, `db`) | Server |

`/dashboard` serves as the results page — no separate `/results`.

## Persistence

- Primary: `lib/store/skillsync-store.ts` — a tiny external store (`useSyncExternalStore`, no provider needed) holding `AppState` (`analysis`, `interview`, `roadmapProgress: Record<taskId, TaskStatus>`), mirrored to `localStorage` key `STORAGE_KEY` (`skillsync:v5` — bump when `AppState` changes; bumped from v4 in Phase 6 for the new `InterviewSession` shape). Components use `useSkillSync()` → `{ analysis, interview, roadmapProgress, hydrated, actions }`; always check `hydrated` before rendering "no data" states. `actions.setRoadmap(roadmap)` attaches a generated roadmap and drops progress for any task id that no longer exists (regeneration-safe since task ids are stable — `week-skillId-index`); `actions.setTaskStatus(taskId, status)` cycles Not Started → In Progress → Completed. `actions.setInterview(session | null)` attaches/clears the in-progress or completed `InterviewSession`.
- Optional (later phase): Supabase `analyses` table, written only from API routes when env vars exist. The app must work fully without it.

## AI + fallback contract

- Every API route: try OpenAI (if `OPENAI_API_KEY` set) → validate output shape → else run the `lib/engine` equivalent.
- Every result carries `source: "ai" | "local"` (+ `fallbackReason` if AI failed); the UI shows it honestly. Never fake an AI call.
- AI only estimates proficiency + evidence. Market demand, required level, gap, priority and readiness are ALWAYS computed deterministically in `engine/assess.ts`.
- Progress UI shows only real server stage events; it may pace their reveal (min ~420ms/step) for readability, never invent steps.
- Market numbers are a curated demo snapshot (`data/market.ts`); always display `MARKET_SNAPSHOT.label/note` with them.
- Env access only through `lib/server/env.ts`. Never use `NEXT_PUBLIC_` for secrets.

## Key data models (`src/lib/types.ts`)

`AnalysisInput` → `ExtractedSkill[]` (proficiency + `SkillEvidence[]`) → benchmarked against `RoleProfile.requirements` → `SkillAssessment[]` = `{ name, currentProficiency, marketDemand, requiredLevel, gap, evidence, priority, priorityScore, importance ("core"|"important"|"nice-to-have"|"bonus"), detected }` sorted by priority → `ReadinessScore`. Bundled as `AnalysisResult` (with `stats`, `source`, `roadmap: Roadmap | null` until the roadmap phase).

**Mock interview**: `InterviewSession` = `{ id, targetRole, difficulty ("easy"|"medium"|"hard"), interviewType ("technical"|"behavioral"|"mixed"), source, fallbackReason?, questions: InterviewQuestion[], currentQuestion, answers: InterviewAnswer[], violations: ProctoringEvent[], startedAt, completedAt?, endedReason?: InterviewEndReason, feedback?: InterviewFeedback }`. `InterviewQuestion` = `{ id, prompt, category, type, difficulty, expectedTopics, skillId?, estimatedAnswerSeconds, source }`. `InterviewAnswer` = `{ questionId, answer, transcript, method ("typed"|"recorded"), durationSeconds, submittedAt }`. `ProctoringEvent` = `{ id, type ("tab-switch"|"window-blur"), at, message }`. `InterviewEndReason` = `"completed" | "violations"` — set alongside `completedAt`; more than `MAX_INTERVIEW_VIOLATIONS` (3, in `lib/constants.ts`) violations ends the session immediately as `"violations"`, discarding any in-progress answer, and no feedback is generated for it. For a normal finish (`endedReason: "completed"`), `POST /api/mock-interview/feedback` scores the session into `InterviewFeedback` = `{ source, fallbackReason?, generatedAt, overallScore, technicalScore, behavioralScore, communicationScore, summary, strengths: FeedbackArea[], growthAreas: FeedbackArea[], perQuestion: QuestionFeedback[] }`, deterministically derived per question from topic coverage, elaboration and specificity against `expectedTopics` (`engine/interview-feedback.ts`) and grouped into technical (by skill) and behavioral (by category) `FeedbackArea`s; AI, when configured, may only rewrite the `summary` wording.

**Readiness (deterministic, `engine/readiness.ts`)**: `overall = 0.25·skillCoverage + 0.30·proficiencyMatch + 0.30·marketAlignment + 0.15·(100 − priorityGapImpact)` over role requirements only (bonus skills excluded). Returns `{ overallScore, skillCoverage, proficiencyMatch, marketAlignment, priorityGapImpact, label, breakdown, explanation{summary, formula, factors[]} }`.
**Consistency rule**: dashboard, heatmap, top gaps and insight must all derive from `analysis.skills` via `engine/insights.ts` (`comparePriority`, `topPriorityGaps`, `topStrengths`, `severityOf`, `generateMarketInsight`). Never hand-write names or numbers in UI. Terminology: *Priority* (critical/high/medium/low) = urgency; *Severity* (strong/moderate/weak/critical) = heatmap level — say "high-priority" in copy to avoid mixing them.
Heatmap colors: `--color-sev-{strong,moderate,weak,critical}` (validated palette); always pair with icon + label.

Extraction scoring (heuristic): base 15 + section points (experience > projects > skills list > certs > education) + frequency + action verbs + measurable impact + years + position-aware self-rating ("Proficient:", "(learning)") + implied skills (Next.js → React). requiredLevel = role base ± experience offset (student −12 … intermediate +10). priorityScore = gap × demand × importance.

**Roadmap (`engine/roadmap.ts`, `Roadmap`/`RoadmapItem` in types.ts)**: selection is 100% deterministic — rank real gaps (`gap ≥ MIN_MEANINGFUL_GAP`) with the exact same `comparePriority` used by the dashboard, reorder with `data/prerequisites.ts` (an unmet prerequisite is pulled ahead of the skill that needs it), then map one skill to each of the 4 weeks in ranked order. Fewer than 4 real gaps → remaining weeks deepen the highest-demand remaining skill, or become a capstone/interview-prep week if nothing is left. Content (objectives/tasks/mini project/outcome) comes from `data/learning-templates.ts` (`SKILL_TEMPLATES[skillId]` or `categoryTemplate(category)` fallback); `stage: "intro"/"advanced"` tasks gate on `currentLevel`, and `TemplateCtx.stack()` references the candidate's own detected stack. AI (`server/ai-roadmap.ts`) may only rewrite that content per-week — it never sees or changes selection, order, or numeric fields — and any AI failure keeps the deterministic content for that week.

**Mock interview (`engine/interview.ts`, `data/interview-questions.ts`, `hooks/use-*`)**: selection is deterministic and slot-based — for interview type "technical"/"behavioral" every slot is that type; "mixed" leans technical (`slotType()` interleaves ~60% technical). `selectTopicSeed()` ranks the relevant question pool (role-specific `TECHNICAL_QUESTIONS[role]` or the shared `BEHAVIORAL_QUESTIONS`) by whether its `skillId` is one of the candidate's real gap skills, then by difficulty distance, and picks a topic not already used (`excludeIds`/`excludeCategories`) — never a generic or duplicate question, and never the same set for two different roles. `POST /api/mock-interview/question` calls this once per question (`slotIndex`), so the client accumulates a 5-question session with no repeats; if `OPENAI_API_KEY` is set, `server/ai-interview.ts` may rewrite only that slot's wording (question text, category label, expected topics) — it never re-picks the topic or skill — and falls back to the deterministic local wording on any failure (`source`/`fallbackReason`), same contract as the roadmap's AI rewrite. Proctoring (`hooks/use-proctoring.ts`) watches `document.visibilitychange`/`window.blur` while the session is active, de-duplicating same-action double-fires within a 600ms window, and is explicitly labelled as a browser-based demo — no facial recognition, identity verification, or recording upload. Webcam, answer recording (`MediaRecorder`) and live transcript (Web Speech API) are all opt-in/best-effort with a simulated-avatar / typed-answer fallback so the interview is always completable without camera, mic, or speech-API access.

## Conventions

- Server Components by default; add `"use client"` only for interactivity/state/browser APIs.
- Named exports for components; `export default` only for Next route files.
- Styling: Tailwind utilities + design tokens (`bg-paper`, `bg-surface`, `text-fg`, `text-muted`, `border-line`, `from-violet`, `text-cyan`…). Custom utilities in `globals.css`: `glass`, `glass-strong`, `gradient-border`, `gradient-text`, `brand-gradient`, `grid-pattern`, `glow-violet|cyan`, `skeleton`. `cn()` uses tailwind-merge — never name a custom utility with a Tailwind prefix (`bg-*`, `text-*`, `border-*`) or it gets merged away.
- **Theme (warm light, since Phase 7):** page base is `--color-paper` (warm cream, `#faf6ee`), cards are `.glass`/`bg-surface` (near-white with a soft shadow, not a translucent fill), text runs `--color-fg` (warm near-black) down through `soft/muted/faint`. The old `--color-ink-700..950` dark-navy scale is kept **only** for deliberate dark accents — icon chips (`bg-ink-800`/`850`), the analyze paste-textarea's terminal-style panel, step-number badges, native `<option>` backgrounds — never as a page or card background. Subtle surface tints use `bg-black/[0.0N]` (not `bg-white/[...]`, which was the old dark-theme convention and is now invisible on a light surface); never use a bare shorthand opacity like `bg-black/20` for anything holding body text — at that strength it reads as a flat grey block instead of a legible inset (use `bg-black/[0.03]` + `border-line`, or `bg-surface`, instead). Ambient background glows (`layout/background.tsx`) are intentionally very low-opacity (`/[0.05]`–`/[0.08]`) washes, not the old dark-mode neon glow. Any new gap-severity or chart color must be re-validated with the dataviz skill's `validate_palette.js --mode light` against `--color-surface`, not eyeballed. `gradient-text`/`brand-gradient` lead with `--color-blue` (Phase 7b), keeping violet/indigo/cyan as secondary stops. `layout/background.tsx` also carries a faded, blue-tinted photo (`public/backgrounds/hero-handshake.jpg`) behind the ambient glows, masked to fade out by the bottom of the hero — global via `app/layout.tsx`, so any new page inherits it automatically. Note: a `position: fixed` layer needs `z-0` (not a negative z-index) to render above the browser's canvas background — see Phase 7b in `docs/IMPLEMENTATION_PLAN.md` for why.
- Typographic/layout personality: mixed-weight headlines (a plain lede line + a bold oversized headline, sometimes with a `gradient-text` accent word), a pill-shaped active nav tab (`bg-surface` pill inside a recessed `bg-black/[0.025]` bar), floating white stat cards with soft shadows, generous whitespace. Avoid default admin-dashboard patterns (sidebars, dense tables, grey boxes).
- Mobile first; every page must work at 360px width.
- No new dependency without a clear need. Charts: hand-rolled SVG first.

## Constraints

No auth, no payments, no production infra. Prioritise a working browser demo. Don't over-engineer. See `docs/IMPLEMENTATION_PLAN.md` for phase status.

## Commands

`npm run dev` · `npm run build` · `npm run lint` · `npx tsc --noEmit` · `npm run test:engine` (determinism, gap ordering, insight/heatmap consistency across all samples × roles × levels)
