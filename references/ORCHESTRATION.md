# Orchestration v8

**Feature branches, not agent branches. Build like a real team.**

## Overview

The PM Agent scopes features from requirements. Each feature gets a branch. Multiple agents collaborate on each feature. This works for greenfield AND iteration.

## Git Branching Strategy

```
main (release)
  │
  └── develop (integration)
        │
        │   PM scopes features from requirements
        │
        ├── feature/clipboard-monitor ──────────────────► merge to develop
        │   └── Backend + DevOps collaborate
        │
        ├── feature/web-dashboard ──────────────────────► merge to develop
        │   └── Designer + Frontend + Docs collaborate
        │
        ├── feature/search-and-filter ──────────────────► merge to develop
        │   └── Backend + Frontend collaborate
        │
        │   Review Wave (QA, Security, CodeReview, Perf)
        │
        │   Runtime Verification
        │       │
        │       ▼
        ├── bugfix/empty-response ──────────────────────► merge to develop
        ├── bugfix/missing-index ───────────────────────► merge to develop
        │
        │   E2E Browser Testing
        │       │
        │       ▼
        ├── bugfix/e2e-missing-testid ─────────────────► merge to develop
        ├── bugfix/e2e-form-validation ────────────────► merge to develop
        │
        │   Conductor approval (98/100)
        │
        └──────────────────────────────────────────────► merge to main
                                                         tag v1.0.0
```

### Branch Types

| Prefix | Purpose | Example |
|--------|---------|---------|
| `feature/*` | New functionality | `feature/user-auth` |
| `bugfix/*` | Runtime verification failures | `bugfix/empty-dashboard` |
| `develop` | Integration branch | Always deployable |
| `main` | Production releases | Tagged versions only |

### Feature Branch Workflow

```bash
# PM creates feature branch
git checkout develop
git checkout -b feature/search-and-filter

# Multiple agents work on same branch
# Backend adds API endpoints
# Frontend adds UI components
# Designer provides component specs

# Commits show agent attribution
git commit -m "feat(backend): search endpoint with date filtering"
git commit -m "feat(frontend): filter dropdown component"
git commit -m "feat(frontend): date range picker"

# Merge when feature complete
git checkout develop
git merge feature/search-and-filter --no-edit

# POST-MERGE SMOKE TEST (required)
# Immediately verify the merge didn't break anything
```

### Post-Merge Smoke Test

**After EVERY feature merge to develop, run a quick verification.**

Don't wait until "Runtime Verification" phase - catch breakage immediately.

```
procedure post_merge_smoke_test():

  timeout: 2 minutes max

  # 1. Build compiles
  run(build_command)  # npm run build, cargo build, etc.
  if failed: STOP, fix before continuing

  # 2. Server starts
  start_server()
  wait_for_ready(timeout=30s)
  if failed: STOP, fix before continuing

  # 3. Health check
  GET /health (or /api/health)
  if status != 200: STOP, fix before continuing

  # 4. Auth still works (if project has auth)
  POST /login with invalid credentials
  if status == 500: STOP, database schema likely broken
  if status == 401: PASS (auth code runs)

  # 5. New feature endpoint responds
  hit_new_endpoint()
  if status == 500: STOP, new code has bug

  stop_server()

on_smoke_test_failure:
  - Do NOT merge more features
  - Create bugfix/* branch immediately
  - Fix the issue
  - Re-run smoke test
  - Then continue with next feature
```

This catches the "migration didn't run" bug immediately, not after 5 more features are merged.

## Execution Flow

```
PM Agent
    │
    ├── 1. Read requirements (or existing code + new request)
    │
    ├── 2. Output scope.yaml with features list:
    │       features:
    │         - name: clipboard-monitor
    │           agents: [backend, devops]
    │           dependencies: []
    │         - name: web-dashboard  
    │           agents: [designer, frontend, docs]
    │           dependencies: [clipboard-monitor]
    │         - name: search-filter
    │           agents: [backend, frontend]
    │           dependencies: [web-dashboard]
    │
    ├── 3. For each feature (in dependency order):
    │       - Create feature/* branch
    │       - Dispatch assigned agents
    │       - Agents collaborate on same branch
    │       - Merge to develop when complete
    │
    ├── 4. Run Review Wave on develop:
    │       - QA, Security, CodeReview, Performance (parallel)
    │
    ├── 5. Runtime Verification:
    │       - Start server, hit endpoints
    │       - Failures → bugfix/* branches
    │       - Loop until passes
    │
    ├── 5b. E2E Browser Testing:
    │       - Setup Playwright, run browser tests
    │       - Test all user flows (auth, CRUD, forms, nav)
    │       - Failures → bugfix/* branches
    │       - Loop until passes
    │
    └── 6. Conductor Score:
            - < 98%: create fix branches, iterate
            - ≥ 98%: merge develop → main, tag release
```

## PM Agent Responsibilities

The PM is the orchestrator. It runs first and last.

### Greenfield Build

```yaml
# PM reads: User prompt
# PM outputs: scope.yaml

project:
  name: clipboard-manager
  type: greenfield
  
features:
  - name: core-daemon
    description: Background clipboard monitor
    agents: [backend]
    branch: feature/core-daemon
    dependencies: []
    
  - name: storage-layer
    description: SQLite database and queries
    agents: [backend]
    branch: feature/storage-layer
    dependencies: [core-daemon]
    
  - name: api-server
    description: REST API endpoints
    agents: [backend, devops]
    branch: feature/api-server
    dependencies: [storage-layer]
    
  - name: web-dashboard
    description: Dark-themed UI
    agents: [designer, frontend]
    branch: feature/web-dashboard
    dependencies: [api-server]
    
  - name: search-capability
    description: Filter and search clips
    agents: [backend, frontend]
    branch: feature/search-capability
    dependencies: [web-dashboard]
    
  - name: documentation
    description: README, API docs, CLAUDE.md
    agents: [docs]
    branch: feature/documentation
    dependencies: []  # Can run parallel

review_wave:
  agents: [qa, security, code-review, performance]
  runs_after: all features merged
```

### Iteration Build

```yaml
# PM reads: Existing code + new request
# PM outputs: scope.yaml (just new features)

project:
  name: clipboard-manager
  type: iteration
  version: 1.0.0 → 1.1.0

existing_code:
  analyzed: true
  tests: 55
  coverage: 78%

# CRITICAL: Iteration mode must verify existing features still work
regression_check:
  required: true
  after_each_feature_merge: true

features:
  - name: date-range-filter
    description: Filter clips by date
    agents: [backend, frontend]
    branch: feature/date-range-filter
    touches:
      - src/server.js (add query params)
      - public/app.js (add filter UI)
      - public/style.css (filter styles)
    dependencies: []
    
  - name: export-to-json
    description: Export selected clips
    agents: [backend, frontend]
    branch: feature/export-to-json
    touches:
      - src/server.js (add export endpoint)
      - public/app.js (add export button + logic)
    dependencies: []
    
  - name: keyboard-shortcuts
    description: j/k navigation, / search, enter copy
    agents: [frontend]
    branch: feature/keyboard-shortcuts
    touches:
      - public/app.js (keyboard event handlers)
    dependencies: []

review_wave:
  agents: [qa, security]  # Lighter review for iteration
  runs_after: all features merged
```

### Iteration Regression Check

**In iteration mode, new features can break existing functionality.**

This is especially true when:
- Modifying shared models (User, Session, etc.)
- Adding columns to existing tables
- Changing API response shapes
- Modifying auth flows

```
procedure iteration_regression_check():

  # Run after ALL new features are merged, before Review Wave

  # 1. All existing tests must pass
  run_existing_tests()
  if any_failed:
    create_bugfix_branch()
    fix_regression()
    merge_and_retry()

  # 2. Core user flows still work
  core_flows = [
    "user can register",
    "user can login",
    "user can perform main CRUD operations",
    "user can logout"
  ]

  for flow in core_flows:
    test_flow(flow)
    if failed:
      # This is a regression - new code broke existing feature
      create_bugfix_branch("regression-{flow}")
      fix()
      merge_and_retry()

  # 3. New + old features integrate correctly
  if new_feature_touches_existing_model:
    verify_existing_queries_still_work()
    verify_existing_api_responses_unchanged()

  # 4. Database queries work with new schema
  if migrations_added:
    run_migrations()
    test_all_model_queries()  # Not just new ones
```

### Why This Matters for Iteration

| Greenfield | Iteration |
|------------|-----------|
| No existing code to break | Existing code can break |
| Fresh database | Existing data + new columns |
| No users | Real users affected |
| Full test coverage built | May miss edge cases |

**The auth/login flow is the most common regression.** Adding a column to the User model will break login if the migration doesn't run.

## Agent Collaboration on Feature Branches

Multiple agents work on the same feature branch:

```bash
# PM creates branch
git checkout develop
git checkout -b feature/web-dashboard

# Designer works first (creates specs)
# Commits: feat(design): dashboard component specs
# Commits: feat(design): color tokens and typography

# Frontend works next (implements specs)  
# Commits: feat(frontend): dashboard layout
# Commits: feat(frontend): clip card component
# Commits: feat(frontend): search bar

# All on same branch, sequential within feature
git checkout develop
git merge feature/web-dashboard --no-edit
```

### Commit Message Convention

```
feat(agent): description     # New functionality
fix(agent): description      # Bug fix
docs(agent): description     # Documentation
test(agent): description     # Tests
refactor(agent): description # Code restructure
```

Examples:
```
feat(backend): add clips search endpoint with date filtering
feat(frontend): implement filter dropdown component
fix(backend): handle empty clipboard gracefully
test(qa): add integration tests for search API
docs(docs): update API documentation for filters
```

## Bugfix Branches

Runtime verification failures get bugfix branches. **CRITICAL: PM dispatches BUGFIX-AGENT, not generic backend/frontend agents.**

```bash
# Runtime verification fails: "GET /api/clips returns 500"

# PM creates bugfix branch
git checkout develop
git checkout -b bugfix/clips-endpoint-500

# PM dispatches BUGFIX-AGENT (not backend agent)
# Bugfix agent follows the protocol:
#   1. REPRODUCE - Document exact failure
#   2. TRACE - Follow data DB → Service → API → UI
#   3. ISOLATE - Find exact root cause line
#   4. FIX - Minimal surgical change
#   5. VERIFY - Prove fix works end-to-end

git commit -m "fix(backend): handle null content_hash in clips query

Root cause: clips_query() returned None for content_hash field,
but serializer expected string type.

Trace: DB ✓ → Service ✓ → API ✗ (serializer type error)"

# Merge fix
git checkout develop
git merge bugfix/clips-endpoint-500 --no-edit

# Re-run verification
```

### Why BUGFIX-AGENT, Not Generic Agents?

| Generic Agent Approach | BUGFIX-AGENT Approach |
|------------------------|----------------------|
| "Add error handling" | "Trace to find root cause first" |
| "Try changing X" | "Follow data through all layers" |
| Guess-and-check loop | Systematic diagnosis |
| May create new bugs | Minimal, targeted fix |
| "Works on my machine" | Verified end-to-end |

**The bugfix agent's job is to UNDERSTAND before FIXING.**

### Bugfix vs Feature

| Type | When | Branch | Agent |
|------|------|--------|-------|
| Feature | New functionality | `feature/*` | Backend/Frontend/etc |
| Bugfix | Runtime failure | `bugfix/*` | **BUGFIX-AGENT** |
| Bugfix | Conductor failure | `bugfix/*` | **BUGFIX-AGENT** |
| Bugfix | QA test failure | `bugfix/*` | **BUGFIX-AGENT** |
| Bugfix | E2E test failure | `bugfix/*` | **BUGFIX-AGENT** |

## Parallel Execution Mechanics

### ⚡ ALWAYS MAXIMIZE PARALLELISM

**PM MUST dispatch agents in parallel whenever possible. Sequential execution is slower and wasteful.**

```
WRONG (sequential):
  feature/auth → wait → feature/api → wait → feature/ui → wait → feature/docs
  Total: 4 × agent_time

RIGHT (parallel waves):
  Wave 1: feature/auth + feature/docs (parallel, no deps)
  Wave 2: feature/api (needs auth)
  Wave 3: feature/ui (needs api)
  Total: 3 × agent_time (25% faster)
```

### Parallel vs Sequential - Quick Reference

| Phase | Parallel? | Why |
|-------|-----------|-----|
| **Feature Build (no deps)** | ✅ YES | Independent features can build simultaneously |
| **Feature Build (has deps)** | ❌ NO | Must wait for dependencies |
| **Review Wave** | ✅ YES | QA, Security, CodeReview, Perf all run parallel |
| **Runtime Verification** | ❌ NO | Needs merged code from reviews |
| **E2E Testing** | ❌ NO | Needs running server |
| **Visual QA** | ❌ NO | Needs screenshots from E2E |
| **Conductor** | ❌ NO | Needs all reports |

### How to Dispatch Parallel Agents

**Use a single message with multiple tool calls:**

```yaml
# CORRECT - Parallel dispatch in ONE message
dispatch_parallel:
  - agent: backend, feature: auth
  - agent: docs, feature: documentation
  - agent: devops, feature: ci-cd

# WRONG - Sequential dispatch across multiple messages
message_1: dispatch backend for auth
message_2: dispatch docs for documentation  # WASTED TIME
message_3: dispatch devops for ci-cd        # WASTED TIME
```

Features with no dependencies can and SHOULD run in parallel. PM dispatches multiple agents concurrently using parallel tool calls.

### Dependency Graph Analysis

```
Before execution, PM builds a dependency graph:

features:
  core-daemon:      deps: []           → Wave 1
  documentation:    deps: []           → Wave 1 (parallel)
  storage-layer:    deps: [core-daemon] → Wave 2
  api-server:       deps: [storage-layer] → Wave 3
  web-dashboard:    deps: [api-server] → Wave 4
  search-capability: deps: [api-server] → Wave 4 (parallel)
```

### Parallel Dispatch Pattern

```yaml
# PM dispatches Wave 1 (no dependencies) in parallel:
parallel_dispatch:
  - agent: backend
    feature: core-daemon
    branch: feature/core-daemon
  - agent: docs
    feature: documentation
    branch: feature/documentation

# Wait for Wave 1 completion, then dispatch Wave 2, etc.
```

### Visualization

```
feature/core-daemon ────────────────────► merge
feature/documentation ──────────────────► merge  (parallel, no deps)

     ↓ both merged

feature/storage-layer ──────────────────► merge

     ↓

feature/api-server ─────────────────────► merge

     ↓

feature/web-dashboard ──────────────────► merge
feature/search-capability ──────────────► merge  (parallel after dashboard)
```

## Review Wave

After all features merged, run reviews on develop:

```bash
git checkout develop

# All review agents work on develop branch for analysis
# They do NOT create feature/* branches for reviews
```

### Review Agent Behavior

| Agent | On develop | If fixes needed |
|-------|------------|-----------------|
| QA | Runs tests, reports results | Creates `bugfix/qa-*` for test fixes |
| Security | Scans for vulnerabilities | Creates `bugfix/security-*` for fixes |
| CodeReview | Analyzes code quality | Creates `bugfix/refactor-*` for fixes |
| Performance | Profiles performance | Creates `bugfix/perf-*` for optimizations |

### Review Outcomes

1. **Pass** → Recommendations logged, continue to Runtime Verification
2. **Fail** → Create targeted `bugfix/*` branch, fix, merge, re-review

## Runtime Verification Loop

```
┌─────────────────────────────────────────┐
│  1. Start server                        │
│  2. Hit each endpoint                   │
│  3. Verify response matches contract    │
└─────────────────────────────────────────┘
              │
        Pass? │
              │
    ┌─────────┴─────────┐
    │                   │
   Yes                  No
    │                   │
    ▼                   ▼
  E2E            Create bugfix/*
Testing          Fix, merge, retry
    │                   │
    └───────────────────┘
```

## E2E Browser Testing Loop

### ⛔ BLOCKING GATE - E2E TESTING IS MANDATORY

**E2E testing is NOT optional. The build CANNOT proceed to Conductor without:**
1. E2E tests actually EXECUTED (not just files created)
2. Minimum test count met (see formula below)
3. All tests passing OR bugfix branches created

**If you skip E2E testing, the app is NOT deploy-ready. Period.**

```
┌─────────────────────────────────────────┐
│  1. Setup Playwright (if not done)      │
│  2. Open browser, test user flows       │
│  3. Verify: auth, CRUD, forms, nav      │
│  4. Check data-testid coverage          │
│  5. VERIFY TEST COUNT MEETS MINIMUM     │
└─────────────────────────────────────────┘
              │
        Pass? │
              │
    ┌─────────┴─────────┐
    │                   │
   Yes                  No
    │                   │
    ▼                   ▼
Conductor         Create bugfix/*
  Score           Fix, merge, retry
    │                   │
    └───────────────────┘
```

### ⛔ MINIMUM TEST COUNT REQUIREMENTS (ENFORCED)

**These are HARD requirements, not suggestions:**

| Feature Area | Minimum Tests | What to Test |
|--------------|---------------|--------------|
| Authentication | 15+ | Login UI, flow, logout, register, session, validation errors |
| Each CRUD Page | 10+ | List, empty state, create, edit, delete, pagination |
| Each Form | 8+ | Fields editable, validation, submit success, submit failure |
| Navigation | 5+ | Links, back button, deep links, breadcrumbs |
| Data Content | 5+ per page | Verify REAL data displayed (not "Unknown" placeholders) |

**Formula:** `minimum_tests = (auth_pages × 15) + (crud_pages × 10) + (static_pages × 5) + (forms × 8)`

### Test Count Gates

| App Complexity | Minimum Tests | If Below |
|----------------|---------------|----------|
| Simple (1-2 pages) | 30 tests | E2E NOT COMPLETE |
| Standard (3-5 pages) | 50 tests | E2E NOT COMPLETE |
| Full app (6+ pages) | 75+ tests | E2E NOT COMPLETE |

**⛔ If test count is under the minimum, E2E phase has NOT passed. Do NOT proceed to Conductor.**

### E2E Completion Checklist (ALL required)

- [ ] Playwright installed and configured
- [ ] Tests actually EXECUTED (show output with pass/fail counts)
- [ ] Test count meets minimum for app complexity
- [ ] Authentication flows fully tested (15+ tests)
- [ ] Every CRUD page tested (10+ tests each)
- [ ] Every form tested (8+ tests each)
- [ ] Data content verified (no "Unknown" in UI)
- [ ] All tests passing OR bugfix branches created for failures
- [ ] **Screenshots captured for Visual QA (see below)**

---

## Visual QA Analysis (Screenshot Review)

### BLOCKING GATE - Visual QA is Mandatory

**After E2E tests pass, Visual QA analyzes screenshots using Claude's vision capabilities.**

```
┌─────────────────────────────────────────┐
│  1. E2E Agent captures screenshots      │
│     - 3 viewports per page              │
│     - All states (loaded, empty, error) │
│  2. Visual QA Agent analyzes each       │
│  3. Demo Agent integrates Visual QA     │
│  4. Critical issues = blocking          │
└─────────────────────────────────────────┘
              │
        Pass? │
              │
    ┌─────────┴─────────┐
    │                   │
   Yes                  No
    │                   │
    ▼                   ▼
Conductor         Create bugfix/*
  Score           Fix CSS, re-capture
    │                   │
    └───────────────────┘
```

### Screenshot Requirements

| Viewport | Resolution | Required |
|----------|------------|----------|
| Desktop | 1280x720 | All pages |
| Tablet | 768x1024 | All pages |
| Mobile | 375x667 | All pages |

### Minimum Screenshot Counts

| App Complexity | Pages | Min Screenshots |
|----------------|-------|-----------------|
| Simple (1-2 pages) | 2 | 18 screenshots |
| Standard (3-5 pages) | 5 | 45 screenshots |
| Full app (6+ pages) | 8 | 72+ screenshots |

### Visual QA Checks

Visual QA Agent analyzes each screenshot for:

- **Layout**: Overlapping elements, overflow, alignment
- **Typography**: Readability, contrast, truncation
- **Styling**: Buttons look clickable, inputs have borders, consistent colors
- **Components**: Forms have labels, cards styled, navigation clear
- **States**: Loading spinners, empty messages, error styling

### Severity Levels

| Severity | Definition | Action |
|----------|------------|--------|
| **Critical** | Blocks usability, looks broken | **BLOCKING** - must fix |
| **Major** | Noticeable, affects UX | Should fix before release |
| **Minor** | Polish issue | Can document for later |

### Visual QA Completion Checklist

- [ ] All screenshots captured (minimum count met)
- [ ] All 3 viewports analyzed per page
- [ ] No critical visual issues remaining
- [ ] Major issues documented with fix suggestions
- [ ] Demo Agent integrated Visual QA into critique report

**If critical visual issues exist, do NOT proceed to Conductor.**

---

## Conductor Scoring

```yaml
score: 98/100

breakdown:
  functionality: 24/25
  code_quality: 20/20
  security: 18/20      # -2: missing rate limiting
  testing: 15/15
  documentation: 10/10
  performance: 9/10    # -1: unoptimized query

action:
  status: PASS  # ≥ 98
  # If FAIL: create bugfix/conductor-security, bugfix/conductor-performance
```

---

## Escalation Protocol

When automated fixes fail repeatedly, escalate to user rather than looping forever.

### Retry Limits

| Phase | Max Retries | Then |
|-------|-------------|------|
| Feature build | 3 | Escalate |
| Runtime verification | 5 | Escalate |
| E2E browser testing | 5 | Escalate |
| Conductor fixes | 3 | Escalate |
| Bugfix attempts | 3 | Escalate |

### Escalation Procedure

When retry limit exceeded:

```yaml
escalation:
  phase: runtime_verification
  attempts: 5
  last_error: "GET /api/clips returns 500 - null pointer in ClipsController"

  context_dump:
    branch: develop
    last_commit: abc123
    failing_endpoint: GET /api/clips
    error_log: |
      NoMethodError: undefined method 'content' for nil:NilClass
      app/controllers/api/clips_controller.rb:15

    attempted_fixes:
      - "Added nil check on line 15" → still failed
      - "Changed query to use find_by" → still failed
      - "Added default empty array" → still failed

  action: PAUSE_AND_ASK_USER
  message: |
    I've tried 5 times to fix this runtime error but it keeps failing.

    **Error:** GET /api/clips returns 500
    **Root cause:** Null pointer in ClipsController line 15

    **Attempted fixes:**
    1. Added nil check → still failed
    2. Changed query method → still failed
    3. Added default value → still failed

    **Options:**
    1. I can try a different approach (describe what you'd like)
    2. You can fix it manually and tell me to continue
    3. We can skip this endpoint and proceed

    What would you like to do?
```

### Recovery After User Input

After user provides guidance:

```yaml
recovery:
  user_choice: "Try using .where instead of .find"
  action: create bugfix/user-guided-fix
  resume_from: runtime_verification
```

## Complete Flow Example

```
User: "Build a clipboard history manager"

🦃 PM Agent:
  → Calls Discovery

🦃 Discovery Agent:
  → "A few questions..."
  → User answers
  → scope.yaml with 5 features

🦃 feature/core-daemon (Backend)
  → feat(backend): clipboard polling daemon
  → merge to develop ✓

🦃 feature/storage-layer (Backend)
  → feat(backend): SQLite schema and queries
  → merge to develop ✓

🦃 feature/api-server (Backend + DevOps)
  → feat(backend): REST API endpoints
  → feat(devops): Dockerfile and docker-compose
  → merge to develop ✓

🦃 feature/web-dashboard (Designer + Frontend)
  → feat(design): component specs and tokens
  → feat(frontend): dashboard implementation
  → merge to develop ✓

🦃 feature/documentation (Docs)
  → docs(docs): README and CLAUDE.md
  → merge to develop ✓

🦃 Review Wave:
  → QA: 55 tests, all pass ✓
  → Security: PASS ✓
  → CodeReview: PASS ✓
  → Performance: PASS ✓

🦃 Runtime Verification:
  → GET /api/health ✓
  → GET /api/clips ✓
  → POST /api/clips ✓
  → GET / (dashboard) ✓

🦃 E2E Browser Testing:
  → Playwright configured ✓
  → 75 tests across 6 files
  → auth.spec.ts: 15/15 ✓
  → clips.spec.ts: 20/20 ✓
  → forms.spec.ts: 18/18 ✓
  → navigation.spec.ts: 8/8 ✓
  → data-testid coverage: 100% ✓

🦃 Conductor: 98/100 ✓

→ merge develop to main
→ tag v1.0.0
→ Done! 🦃🦃🦃
```

## Iteration Example

```
User: "Add date filtering and export to the clipboard manager"

🦃 PM Agent:
  → Reads existing code
  → scope.yaml with 2 features (not full rebuild)

🦃 feature/date-range-filter (Backend + Frontend)
  → feat(backend): date query params
  → feat(frontend): filter UI component
  → merge to develop ✓

🦃 feature/export-to-json (Backend + Frontend)
  → feat(backend): export endpoint
  → feat(frontend): export button
  → merge to develop ✓

🦃 Review Wave (light):
  → QA: 12 new tests + 55 existing pass ✓
  → Security: PASS ✓

🦃 Runtime Verification:
  → All existing endpoints ✓
  → New filter endpoint ✓
  → New export endpoint ✓

🦃 E2E Browser Testing:
  → 12 new tests added
  → filter.spec.ts: 6/6 ✓
  → export.spec.ts: 6/6 ✓
  → All 87 tests pass ✓

🦃 Conductor: 98/100 ✓

→ merge develop to main
→ tag v1.1.0
→ Done! 🦃🦃🦃
```

## Success Criteria

- [ ] PM scopes features, not agent tasks
- [ ] Each feature gets `feature/*` branch
- [ ] Multiple agents collaborate on same feature branch
- [ ] Features merge to develop in dependency order
- [ ] Runtime failures → `bugfix/*` branches
- [ ] E2E failures → `bugfix/*` branches
- [ ] Conductor failures → `bugfix/*` branches
- [ ] E2E tests cover all user flows (50+ tests minimum)
- [ ] All components have data-testid attributes
- [ ] Iteration uses same workflow (fewer features)
- [ ] develop always deployable
- [ ] main only receives tagged releases

## Git Log on Success

```
* abc1234 (HEAD -> main, tag: v1.0.0) Merge develop
|\
| * def5678 (develop) fix(backend): handle edge case
| * ghi9012 docs(docs): README and CLAUDE.md
| * jkl3456 feat(frontend): dashboard implementation
| * mno7890 feat(design): component specs
| * pqr1234 feat(devops): Docker configuration
| * stu5678 feat(backend): REST API
| * vwx9012 feat(backend): storage layer
| * yza3456 feat(backend): clipboard daemon
|/
* klm9012 Initial commit
``` 
