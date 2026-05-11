# Spec-Driven Development for Dyad: Recommendation & Implementation Plan

> Status: Proposal / Research
> Author: Claude (research & synthesis)
> Date: 2026-05-11
> Branch: `claude/research-spec-driven-development-gKCTX`

---

## 1. Executive Summary

**Recommendation:** Adopt a **"Kiro-lite" three-document spec system** — `requirements.md`, `design.md`, `tasks.md` — persisted under `.dyad/specs/<feature-slug>/` inside the user's app, layered on top of dyad's existing Plan Mode.

**Reject:**
- **GitHub Spec Kit wholesale** — too ceremonious (7 documents per feature, slash-command driven, branch-scoped) for dyad's vibe-coder audience.
- **Dyad's current single-plan approach** as-is — the plan lives only in chat history, evaporates after compaction, can't be re-read by the implementer agent, and can't be hand-edited by the user.
- **Pure Lovable-style chat-only planning** — same fidelity loss as above; no living spec.

**Why Kiro-lite:**
1. It maps 1:1 onto dyad's existing Plan Mode phases (Discovery → Plan → Refinement → Implementation), so the migration is incremental.
2. Three files is the upper bound of ceremony dyad's audience will tolerate.
3. EARS notation (Easy Approach to Requirements Syntax) makes specs *verifiable* — a verifier agent can later re-check each clause against the implemented code.
4. Persisting specs as files in the user's app turns them into a real artifact the user owns, commits, and edits.
5. Kiro is Claude-native (Sonnet 4.5 + Bedrock) — the prompt patterns transfer cleanly to dyad.

---

## 2. Landscape Analysis

### 2.1 The four main spec-driven development approaches in 2026

| Approach | Output artifacts per feature | Audience fit for dyad | Ceremony |
|---|---|---|---|
| **GitHub Spec Kit** (90k★, v0.8.7) | `spec.md`, `plan.md`, `tasks.md`, `data-model.md`, `contracts/`, `research.md`, `quickstart.md`; branch-per-feature | Enterprise/greenfield. Too heavy. | High |
| **AWS Kiro** (built on Claude) | `requirements.md` (EARS), `design.md`, `tasks.md`. Mode classifier routes chat / spec / do. | Tech-savvy product devs. Closest fit. | Medium |
| **Lovable Plan Mode** (Feb 2026) | Inline plan in chat → approve → implement. | Vibe coders. Same audience as dyad. Same fidelity gap. | Low |
| **Traycer AI** | Plan Mode + Phase Mode (two-mode split). | Mid-market. Useful as a secondary reference. | Medium |
| **Dyad today** | One markdown plan via `write_plan` tool, only in chat. | Vibe coders. | Low |

### 2.2 What dyad already has (file references)

- `src/prompts/plan_mode_prompt.ts` — 137-line Plan Mode system prompt with three phases (Discovery, Plan Creation, Refinement).
- `src/prompts/local_agent_prompt.ts` — main agent loop with 39 tools.
- `src/pro/main/ipc/handlers/local_agent/local_agent_handler.ts` — agent orchestration.
- `src/pro/main/ipc/handlers/local_agent/tool_definitions.ts` — tool registry including `write_plan`, `exit_plan`, `planning_questionnaire`.
- `plans/` — 19 past implementation plans showing dyad's team already practices spec-first internally for its own development.

**Gap:** the same discipline isn't surfaced to the *user's* app. The user's plan vanishes after the chat turn.

### 2.3 Why not Spec Kit

- Spec Kit's own docs say it's "strongest for greenfield builds and larger feature work rather than small bug fixes." Most dyad sessions are iterative edits, not greenfield.
- Seven documents per feature collapses conversion. Dyad's target user wants a working preview in minutes.
- Slash commands (`/speckit.specify`, `/speckit.plan`, `/speckit.tasks`) feel like IDE tax in a chat-first product.
- The "constitution" + phase-gate model assumes engineering review culture that dyad's individual builders don't have.

### 2.4 Why not pure Lovable / status quo

- Plan lives only in chat → lost across sessions, lost on compaction.
- Implementer can't re-verify against the plan during build.
- User can't hand-edit a plan that doesn't exist as a file.
- No structured acceptance criteria → no automated verifier loop.
- No way to import a spec from a PRD or Figma export.

### 2.5 Why Kiro-lite hits the sweet spot

- Three documents only — same upper bound Kiro found through user testing.
- Maps directly onto dyad's existing 3-phase Plan Mode.
- EARS notation is the *single* feature that unlocks a verifier sub-agent.
- Files inside the user's project means they get committed to user's git, travel with the app, and become a real user-owned artifact.
- Backwards compatible — `write_plan` stays as the fallback for one-shot edits.

---

## 3. The Proposed Dyad Spec System

### 3.1 File layout

```
<user-app-root>/
  .dyad/
    specs/
      <feature-slug>/
        requirements.md   # WHAT and WHY  — user stories + EARS acceptance criteria
        design.md         # HOW           — UI flows, components, data shape, integrations
        tasks.md          # WORK          — ordered, checkable, parallelizable
        status.json       # { phase, current_task_id, last_updated, owner }
```

- `.dyad/` is gitignored by default in scaffolds, but specs are explicitly tracked: `!.dyad/specs/`.
- Feature slug is derived from the requirements title (e.g. `email-login`, `stripe-checkout`).
- `status.json` is optional metadata so the IDE chip ("Spec: email-login • 4/9 tasks done") doesn't have to parse markdown.

### 3.2 Phase mapping

| Phase | Current dyad | Proposed Kiro-lite |
|---|---|---|
| 1. Discovery | `planning_questionnaire` | Same, but persist answers into `requirements.md` as they are confirmed |
| 2. Plan creation | `write_plan` (single doc) | Three calls: `write_requirements` → `write_design` → `write_tasks` |
| 3. Refinement | Re-call `write_plan` | Re-call any of the three; user can also hand-edit any file on disk |
| 4. Implementation | `exit_plan` then local agent guesses | `exit_plan` then implementer reads `tasks.md` and calls `mark_task_complete` after each |
| 5. Verification (new) | none | Verifier sub-agent re-reads `requirements.md` and grades each EARS clause |
| 6. Reconciliation (new) | none | If scope shifted mid-build, agent proposes an update to `requirements.md` |

### 3.3 Document formats

#### `requirements.md` — uses EARS notation

```markdown
# Feature: Email login

## Problem & Context
Returning users currently lose their saved projects because we only support magic-link auth, which has 40% delivery failure. Adding email/password gives a deterministic path back in.

## Out of scope
- Social login (covered by spec `social-login`)
- Password reset (separate spec)

## User stories

### Story 1: Sign in with email
As a returning user, I want to sign in with email + password so I can access my saved projects without waiting for an email.

### Acceptance criteria (EARS)
- **Ubiquitous:** THE SYSTEM SHALL store passwords using argon2id with per-user salt.
- **Event-driven:** WHEN the user submits a valid email/password THE SYSTEM SHALL create a session and route to `/dashboard`.
- **Unwanted-behavior:** IF the password is wrong THEN THE SYSTEM SHALL display "Invalid credentials" without revealing which field failed.
- **State-driven:** WHILE a sign-in request is in flight THE SYSTEM SHALL disable the submit button and show a spinner.
- **Optional:** WHERE rate-limiting is enabled, THE SYSTEM SHALL block more than 5 failed attempts per IP per minute.
```

**Why EARS:** every clause becomes a testable assertion. A verifier agent can later check each WHEN/IF/WHILE clause against the implemented code or a generated Playwright test.

#### `design.md` — UI/UX-first, technical-last (matches dyad's existing plan template)

```markdown
# Design: Email login

## UI/UX
- Add a new route `/login` with `EmailForm` component.
- Tabbed UI: "Email" (default) and "Magic link" tabs.
- Inline validation: email format on blur, password length on submit.
- Error region above the submit button, ARIA-live=polite.

## Component tree
- `pages/login.tsx`
  - `<AuthShell>`
    - `<Tabs>`
      - `<EmailForm onSubmit={signInWithEmail} />`
      - `<MagicLinkForm onSubmit={sendMagicLink} />`

## Data shape
- Add `password_hash: string | null` to `users` table.
- Add `auth_attempts` table for rate-limiting.

## Integration points
- `src/lib/auth.ts` — add `signInWithEmail()` next to existing `sendMagicLink()`.
- `src/lib/db.ts` — extend the users query.
- `src/components/auth/` — new directory.

## Considerations
- Argon2id has a native dep — check Vite worker config supports it.
- Existing magic-link users won't have a password_hash; flow must handle that gracefully.
```

#### `tasks.md` — checklist with file scope + parallel markers

```markdown
# Tasks: Email login

- [ ] **T1** [P] Create `src/components/auth/EmailForm.tsx` with controlled inputs and inline validation.
- [ ] **T2** [P] Add `signInWithEmail()` in `src/lib/auth.ts` (delegates to existing session helper).
- [ ] **T3** [P] Add `users.password_hash` migration in `drizzle/`.
- [ ] **T4** Wire T1 → T2 in `src/pages/login.tsx`.
- [ ] **T5** Add Playwright test stubs from each EARS clause in `requirements.md` → `e2e-tests/login.spec.ts`.
- [ ] **T6** Manual verification: run dev server, sign in with seeded user, observe `/dashboard` route.
```

`[P]` markers (lifted from Spec Kit) tell the implementer which tasks have no inter-dependencies and can fan out via dyad's existing parallel tool calls.

### 3.4 New tools to register in `tool_definitions.ts`

| Tool | Args | Behavior |
|---|---|---|
| `write_requirements` | `feature_slug`, `content` | Create or overwrite `.dyad/specs/<slug>/requirements.md`. Auto-create the dir. |
| `write_design` | `feature_slug`, `content` | Same for `design.md`. Refuses if `requirements.md` doesn't exist (forces ordering). |
| `write_tasks` | `feature_slug`, `content` | Same for `tasks.md`. Refuses if `design.md` doesn't exist. |
| `read_spec` | `feature_slug` | Returns all three files concatenated for the implementer. |
| `list_specs` | (none) | Returns directory listing of `.dyad/specs/` with status.json summaries. |
| `mark_task_complete` | `feature_slug`, `task_id` | Flips `[ ]` → `[x]` in `tasks.md`; updates `status.json`. |
| `propose_spec_update` | `feature_slug`, `file`, `diff`, `reason` | During implementation, if scope drifts, agent surfaces a proposed edit for user approval. |

Keep existing: `planning_questionnaire`, `write_plan` (deprecate but keep for one-shot), `exit_plan`.

### 3.5 Prompt changes

**`src/prompts/plan_mode_prompt.ts`** — replace Phase 2 with:

> ## Phase 2: Spec Authoring
>
> Write three documents in order using `write_requirements`, then `write_design`, then `write_tasks`. Each file lives under `.dyad/specs/<feature-slug>/` in the user's app.
>
> **Before writing**, check if the feature slug already has a spec dir. If yes, read existing files and update them — do not start from scratch. Slugs are kebab-case derived from the requirements title.
>
> **`requirements.md`** must use EARS notation for every acceptance criterion. EARS clauses take one of five shapes:
> - Ubiquitous: `THE SYSTEM SHALL ...`
> - Event-driven: `WHEN <trigger> THE SYSTEM SHALL ...`
> - State-driven: `WHILE <state> THE SYSTEM SHALL ...`
> - Unwanted-behavior: `IF <condition> THEN THE SYSTEM SHALL ...`
> - Optional: `WHERE <feature flag> THE SYSTEM SHALL ...`
>
> **`design.md`** must lead with UI/UX, then component tree, then data shape, then integration points (every existing file the change will touch), then considerations.
>
> **`tasks.md`** is an ordered checklist. Mark a task `[P]` only when it has no dependency on any earlier unmarked task in the list. Each task names the file(s) it touches.

**`src/prompts/local_agent_prompt.ts`** — update step 1 of `<development_workflow>`:

> 1. **Understand:** If a `.dyad/specs/<feature>/` directory exists relevant to the user's request, read all three files first using `read_spec`. Treat `tasks.md` as the work plan. After each task completes (file written and verified), call `mark_task_complete` with the task ID. If you encounter scope drift, call `propose_spec_update` rather than silently widening scope.

### 3.6 Backwards compatibility

- `write_plan` continues to work for users who don't want the spec-mode overhead.
- Auto-classifier (see §4.1) decides whether to suggest spec mode vs single-plan mode based on request scope.
- Existing chats with single plans are not migrated — only new Plan Mode sessions use the new tools.

---

## 4. App Builder Improvements (Beyond Spec Mode)

Ranked by impact. Each item draws from the system-prompts research (Kiro, Lovable, Cursor, Spec Kit, Augment Code).

### 4.1 Mode auto-classifier *(highest leverage)*

**What:** A tiny Haiku-tier upstream call that classifies every user message as `{chat, do, spec}` and routes accordingly.

**Reference:** Kiro's `Mode_Clasifier_Prompt.txt` returns `{"chat": 0.1, "do": 0.8, "spec": 0.1}` for a casual greeting; routes to "do mode" as the default fallback. Lovable shipped the same pattern in February 2026 with documented rework reduction.

**Dyad implementation:**
- New file `src/prompts/mode_classifier_prompt.ts`.
- New IPC step in `local_agent_handler.ts` that runs the classifier before dispatching.
- If `spec > 0.5` and no spec exists for the inferred feature → suggest entering Plan Mode with a one-click chip.
- If `spec > 0.5` and a spec exists → auto-route to the implementer with `read_spec` pre-called.
- Respect the user's manual mode toggle as an override.

**Why now:** dyad already requires the user to toggle Plan Mode manually, which means most users skip it and end up with unstructured edits on complex features.

### 4.2 Living spec sync (reconciliation pass)

**What:** After the implementer finishes a `tasks.md` checklist, run a "spec reconciliation" agent that compares the final code against `requirements.md` and updates the spec to match what shipped.

**Why:** Without this, specs rot — the documented failure mode of every static-spec tool (per Augment Code's SDD survey). A stale spec is worse than no spec because users trust it.

**Dyad implementation:**
- Add a `reconcile_spec` tool that diffs final code vs spec and proposes updates.
- Trigger automatically when `mark_task_complete` is called on the last task.
- Surface diffs in the chat as an approval prompt.

### 4.3 Agent Hooks (Kiro's headline feature)

**What:** Per-project `.dyad/hooks/*.yaml` files that fire agent actions on file events.

**Example:**
```yaml
# .dyad/hooks/run-tests-on-save.yaml
trigger: on_file_save
match: "src/**/*.tsx"
action: |
  Run `npm test` for the test file matching the saved component.
  If it fails, propose a fix using search_replace.
```

**Why dyad is well-positioned:** the desktop app already controls a file watcher (`scripts/`, electron main). The infra is half-built.

**Reference:** Kiro Agent Hooks docs.

### 4.4 Constitutional rules per project

**What:** Promote dyad's globally-enforced rules (e.g. "use design tokens, not raw hex") to a user-editable per-project file at `.dyad/constitution.md`.

**Reference:** Spec Kit's "constitution" concept.

**Why:** Vibe coders accumulate house style over time. Letting them codify it in a file the agent must respect on every turn raises consistency without per-prompt repetition.

**Dyad implementation:**
- New section in the system prompt: `[[PROJECT_CONSTITUTION]]` placeholder.
- IPC handler reads `.dyad/constitution.md` if present and substitutes.
- Pre-built templates: "React + Tailwind + shadcn", "Next.js + Convex", "Vite + Supabase".

### 4.5 Parallel task execution from `[P]` markers

**What:** When `tasks.md` marks tasks `[P]`, the implementer fans them out via dyad's existing parallel tool calls.

**Why:** Dyad already runs tools in parallel opportunistically. Explicit `[P]` markers make it deterministic and let the agent commit to a parallel plan.

**Dyad implementation:**
- Implementer reads `tasks.md`, groups consecutive `[P]` tasks, issues parallel `write_file` / `search_replace` calls.
- Verify all writes succeeded before marking the group complete.

### 4.6 Spec-derived test generation

**What:** Each EARS clause in `requirements.md` becomes a Playwright test stub during implementation.

**Reference:** Augment Code: SDD "catches architectural violations and API contract drift that unit tests structurally cannot."

**Dyad implementation:**
- New tool `generate_tests_from_spec(feature_slug)` that maps each EARS clause to a test scaffold.
- Adds tests to `e2e-tests/` matching the existing Playwright setup.
- Agent fills in the test bodies during implementation.

### 4.7 Verifier sub-agent (Coordinator / Implementor / Verifier pattern)

**What:** After implementation, spawn a small Verifier sub-agent that re-reads `requirements.md` and grades each EARS criterion as `PASS / FAIL / UNKNOWN` against the implemented code.

**Reference:** Augment Code's multi-agent SDD pattern.

**Dyad implementation:**
- Dyad already has sub-agent plumbing (`local_agent_handler.ts`) and a `security_review_prompt.ts` of the same shape.
- New prompt `src/prompts/spec_verifier_prompt.ts`.
- New tool `verify_spec(feature_slug)` that runs the sub-agent and returns the grade matrix.
- Surface results in chat as a checklist; failing criteria become new tasks.

### 4.8 Spec import / export

**What:** Let users drop a PRD, Figma export, GitHub issue, or existing `requirements.md` into chat and have Plan Mode bootstrap the three docs from it.

**Why:** Removes the cold-start friction that kills SDD adoption. Most users don't write specs from scratch — they have a doc somewhere.

**Dyad implementation:**
- Extend the file-drop handler in the chat input.
- New tool `bootstrap_spec_from_document(content)` that produces an initial `requirements.md` draft.
- User reviews and the existing Plan Mode flow takes over.

### 4.9 Per-project model routing

**What:** Allow `.dyad/models.json` to specify which Claude model handles which phase (e.g. Opus for spec authoring, Sonnet for implementation, Haiku for classification).

**Reference:** Kiro routes between Sonnet for reasoning-heavy specs and Nova for high-throughput generation via Bedrock.

**Why:** Different phases have very different cost/latency profiles. Per-project routing lets power users tune.

### 4.10 Spec versioning & changelog

**What:** Every `write_requirements` / `write_design` / `write_tasks` call creates a numbered version in `.dyad/specs/<slug>/.history/` and appends a one-line entry to `CHANGELOG.md`.

**Why:** Spec drift is the #1 SDD failure mode. A version trail lets users (and the verifier agent) ask "when did this acceptance criterion change and why?"

---

## 5. What NOT to Do

- **Don't adopt slash commands** like `/speckit.specify`. Dyad is chat-first; commands feel like IDE tax to vibe coders.
- **Don't make spec mode mandatory.** Keep a one-shot "vibe" path for small edits. Kiro's own 2026 reviews flag mandatory specs as the #1 friction point.
- **Don't store specs in dyad's app data dir** (`~/Library/Application Support/dyad/...`). They belong to the user's project, in their git, in their backups.
- **Don't ship Spec Kit's extras** (`research.md`, `quickstart.md`, `contracts/`). Three files is already the upper limit dyad's audience will tolerate.
- **Don't auto-rewrite existing specs without approval.** Spec edits during implementation must surface as a proposed diff for the user to accept.
- **Don't conflate dyad's internal `plans/` with user-facing `.dyad/specs/`.** They serve different audiences (dyad maintainers vs end users).

---

## 6. Rollout Plan

### Phase 1 — Spec scaffolding (1–2 weeks)
- Add `.dyad/specs/` layout convention.
- Implement `write_requirements`, `write_design`, `write_tasks`, `read_spec`, `list_specs`, `mark_task_complete` tools.
- Behind feature flag `enable_spec_mode_v2`. Existing `write_plan` remains default.

### Phase 2 — Prompt migration (1 week)
- Update `plan_mode_prompt.ts` with EARS notation and three-doc workflow.
- Update `local_agent_prompt.ts` step 1 to consume specs.
- Add `propose_spec_update` for scope-drift handling.

### Phase 3 — Mode classifier (1 week)
- Add `mode_classifier_prompt.ts` and pre-router in `local_agent_handler.ts`.
- Auto-suggest spec mode on large requests.
- Telemetry: track classifier accuracy vs user override.

### Phase 4 — Verifier loop (2 weeks)
- Add `spec_verifier_prompt.ts` sub-agent.
- Add `verify_spec` tool.
- Surface grade matrix in chat UI.

### Phase 5 — Spec ergonomics (2 weeks)
- Spec import from PRD/Figma/issue.
- Spec versioning + changelog.
- Constitutional rules per project.

### Phase 6 — Agent Hooks (open-ended)
- Per-project `.dyad/hooks/*.yaml` with on_save / on_create triggers.

### Phase 7 — Deprecate single-plan path
- Once telemetry shows spec mode adoption stable, remove `write_plan`.
- Migrate any cached single plans into spec-format on next session open.

---

## 7. Open Questions

1. **Slug collisions** — what happens when two features want the same slug? (Proposal: agent prompts for disambiguation.)
2. **Spec ownership in multi-user scenarios** — dyad doesn't currently support multi-user editing, but if it does later, spec edits need a merge story.
3. **Mobile / scaffold compatibility** — do all scaffold types (`scaffold/`) have a sensible `.dyad/specs/` location? (Probably yes since they're all git repos.)
4. **Token budget** — reading three spec files on every implementer turn costs tokens. Should `read_spec` cache and only re-read on `mark_task_complete`?
5. **Translation** — dyad replies in the user's language. Are EARS clauses required to stay English (more verifiable) or translated (more readable)? Proposal: English EARS + translated story prose.

---

## 8. Sources

### Spec-driven development methodology
- [GitHub Spec Kit](https://github.com/github/spec-kit) — 90k★ open-source toolkit
- [Spec-driven development methodology (spec-kit)](https://github.com/github/spec-kit/blob/main/spec-driven.md)
- [GitHub Blog: Spec-driven development with AI toolkit](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/)
- [Microsoft Developer Blog: Diving Into Spec-Driven Development with Spec Kit](https://developer.microsoft.com/blog/spec-driven-development-spec-kit)
- [Microsoft Learn: Implement SDD using GitHub Spec Kit](https://learn.microsoft.com/en-us/training/modules/spec-driven-development-github-spec-kit-enterprise-developers/)
- [DevOps.com: GitHub's Spec Kit Puts the Spec Back in Software Development](https://devops.com/githubs-spec-kit-puts-the-spec-back-in-software-development/)

### Kiro (AWS) — primary inspiration
- [Kiro: Bring engineering rigor to agentic development](https://kiro.dev/)
- [Kiro IDE GitHub](https://github.com/kirodotdev/Kiro)
- [Amazon Kiro 2026 Review: Spec-Driven IDE vs Cursor](https://weavai.app/blog/en/2026/04/29/amazon-kiro-2026-review-spec-driven-ide-vs-cursor/)
- [Kiro Review: Amazon's Spec-Driven IDE Powered by Claude (OpenAIToolsHub)](https://www.openaitoolshub.org/en/blog/kiro-review-amazon-ide)
- [Using Spec Driven Development with AWS Kiro (geshan.com.np)](https://geshan.com.np/blog/2026/05/aws-kiro/)
- [Amazon Kiro AWS Agentic IDE: Complete 2026 Developer Guide](https://www.digitalapplied.com/blog/amazon-kiro-aws-agentic-ide-complete-guide)

### Lovable (closest peer to dyad)
- [Lovable: Better planning, better apps — Chat mode & Follow-up questions](https://lovable.dev/blog/chat-mode-and-questions)
- [Lovable Best Practices](https://docs.lovable.dev/tips-tricks/best-practice)
- [Lovable for Designers: Complete Guide 2026](https://muz.li/blog/lovable-for-designers-the-complete-guide-to-building-apps-with-ai-2026/)

### Comparative reviews & guides
- [Spec-Driven Development with AI: Complete Guide (2026)](https://prommer.net/en/tech/guides/spec-driven-development/)
- [Augment Code: 6 Best Spec-Driven Development Tools for AI Coding in 2026](https://www.augmentcode.com/tools/best-spec-driven-development-tools)
- [Augment Code: What Is Spec-Driven Development?](https://www.augmentcode.com/guides/what-is-spec-driven-development)
- [Addy Osmani: How to write a good spec for AI agents](https://addyosmani.com/blog/good-spec/)
- [I Tested Three Spec-Driven AI Tools — Honest Take](https://ranthebuilder.cloud/blog/i-tested-three-spec-driven-ai-tools-here-s-my-honest-take/)
- [Spec-Driven Development with AI Coding Agents: The Definitive Guide (Medium)](https://medium.com/predict/spec-driven-development-with-ai-coding-agents-the-definitive-guide-453fba1baf39)
- [Java Code Geeks: Spec-Driven Development with AI](https://www.javacodegeeks.com/2026/05/spec-driven-development-with-ai-write-the-spec-first-then-prompt-the-implementation.html)

### System prompt corpus (jdoit18950 fork of x1xhlol)
- [jdoit18950/system-prompts-and-models-of-ai-tools](https://github.com/jdoit18950/system-prompts-and-models-of-ai-tools)
- [x1xhlol/system-prompts-and-models-of-ai-tools (upstream)](https://github.com/x1xhlol/system-prompts-and-models-of-ai-tools)
- [Kiro folder (Mode/Spec/Vibe prompts)](https://github.com/jdoit18950/system-prompts-and-models-of-ai-tools/tree/main/Kiro)
- [Lovable folder](https://github.com/jdoit18950/system-prompts-and-models-of-ai-tools/tree/main/Lovable)
- [Cursor Prompts folder](https://github.com/jdoit18950/system-prompts-and-models-of-ai-tools/tree/main/Cursor%20Prompts)
- [Traycer AI folder](https://github.com/jdoit18950/system-prompts-and-models-of-ai-tools/tree/main/Traycer%20AI)
- [v0 Prompts and Tools folder](https://github.com/jdoit18950/system-prompts-and-models-of-ai-tools/tree/main/v0%20Prompts%20and%20Tools)
