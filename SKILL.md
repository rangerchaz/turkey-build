---
name: turkey-build
description: Multi-agent app builder with 7 modes - greenfield, iteration, bugfix, refactor, UI polish, migration, and audit. PM orchestrates specialized agents with feature branches. 98%+ quality gate, runtime verified, visual QA.
license: MIT
metadata:
  author: turkeycode
  version: "9.1.0"
  website: https://turkeycode.ai
allowed-tools: Bash, Read, Write, Edit
requires:
  mcp-servers:
    - aimem  # Required for agent coordination and cross-project learning
---

# Turkey Build

## STOP. READ THESE FILES FIRST.

Before writing ANY code, you MUST read these files in order:

```
1. references/ORCHESTRATION.md      ← HOW to run the build (features, branches, flow)
2. references/PM-AGENT.md           ← PM orchestrates everything
3. references/AIMEM-INTEGRATION.md  ← HOW agents coordinate through memory
```

**DO NOT** use the summaries below as your guide. They are overviews only.
**DO** read the full reference files - they contain the actual implementation details.

---

## What This Skill Does

Say "build me X" → Get production-ready, runtime-verified code.

**Key Differentiators:**
- **Feature branches** - Branches per feature, not per agent
- **PM orchestration** - PM scopes features, assigns agents, delivers release
- **Greenfield + iteration** - Same workflow for new builds and adding features
- **aimem coordination** - Agents signal completion, PM coordinates
- **98% quality gate** - Conductor won't ship below threshold
- **Runtime verification** - Actually starts the app, hits endpoints, verifies
- **Bugfix branches** - Runtime/Conductor failures get targeted fixes

---

## Execution Flow

```
PM Agent reads requirements (or existing code + request)
         │
         ▼
PM outputs scope.yaml with features:
  - feature/core-daemon (backend)
  - feature/api-server (backend, devops)
  - feature/web-dashboard (designer, frontend)
  - feature/documentation (docs)
         │
         ▼
For each feature (in dependency order):
┌─────────────────────────────────────┐
│  Create feature/* branch            │
│  Dispatch assigned agents           │
│  Agents collaborate on same branch  │
│  Merge to develop when complete     │
└─────────────────────────────────────┘
         │
         ▼
Review Wave (parallel on develop):
  QA, Security, CodeReview, Performance
         │
         ▼
Runtime Verification:
  Start server → Hit endpoints → Verify responses
  Failures → BUGFIX-AGENT → Trace → Fix → Re-verify
         │
         ▼
Data Flow Verification:
  Schema sync → Placeholder detection → Data trace
  Verify real data flows DB → API → UI (no "Unknown")
  Failures → BUGFIX-AGENT → Trace → Fix → Re-verify
         │
         ▼
E2E Browser Testing:
  Open browser → Click through UI → Verify user flows
  Verify ACTUAL DATA displayed (not just elements exist)
  Capture screenshots (3 viewports × all pages × states)
  Failures → BUGFIX-AGENT → Trace → Fix → Re-test
         │
         ▼
Visual QA Analysis:
  Demo Agent calls Visual QA Agent
  Analyze screenshots using Claude vision
  Detect CSS bugs, layout issues, responsive problems
  Critical issues → BUGFIX-AGENT → Fix CSS → Re-capture
         │
         ▼
Conductor Score:
  < 98% → BUGFIX-AGENT → Trace → Fix → Re-score
  ≥ 98% → merge develop → main → tag release
```

---

## Branch Types

| Prefix | Purpose | Example |
|--------|---------|---------|
| `feature/*` | New functionality | `feature/user-auth` |
| `bugfix/*` | Runtime/Conductor failures | `bugfix/empty-dashboard` |
| `develop` | Integration branch | Always deployable |
| `main` | Production releases | Tagged versions only |

---

## Seven Modes

### Greenfield (New Build)
```
User: "Build a clipboard history manager"
PM: Scopes 5 features → Builds each → Review → Ship v1.0.0
```

### Iteration (Add Features)
```
User: "Add date filtering and export to the clipboard manager"
PM: Reads existing code → Scopes 2 new features → Builds each → Review → Ship v1.1.0
```

### Bugfix (Fix Issues)
```
User: "Cards not showing in battle view"
PM: Creates bugfix branch → BUGFIX-AGENT traces → Finds root cause → Fixes → Verifies
```

### Refactor (Restructure Code)
```
User: "Clean up the auth code" / "Split the god file"
PM: Analyzes structure → Plans refactor → Restructures → Verifies behavior unchanged
```

### UI Polish (Visual Cleanup)
```
User: "Make the UI look better" / "Fix the messy CSS"
PM: Visual QA scan → Designer reviews → CSS cleanup → Component polish → Responsive fixes
```

### Migration (Upgrade Dependencies)
```
User: "Upgrade to React 19" / "Move from Express to Hono"
PM: Audit current → Plan migration → Update incrementally → Test each step → Verify
```

### Audit (Analysis Only)
```
User: "Review security" / "Check performance" / "Analyze code quality"
PM: Runs relevant agents → Produces report → No code changes unless requested
```

Same workflow. PM routes to different agents based on task type.

---

## Agent Reference Files

**PM reads all agent files. Other agents read their own file.**

| Agent | File | Role |
|-------|------|------|
| **PM** | `PM-AGENT.md` | **Orchestrator** - Scopes features, assigns agents |
| Discovery | `DISCOVERY-AGENT.md` | Requirements → scope.yaml (called by PM) |
| Designer | `DESIGNER-AGENT.md` | Design tokens, component specs |
| Backend | `BACKEND-AGENT.md` | API, database, services |
| Frontend | `FRONTEND-AGENT.md` | UI components |
| Docs | `DOCS-AGENT.md` | README, API docs, CLAUDE.md |
| DevOps | `DEVOPS-AGENT.md` | Docker, CI/CD |
| QA | `QA-AGENT.md` | Unit/integration tests |
| Security | `SECURITY-AGENT.md` | Vulnerability scanning |
| Code Review | `CODE-REVIEW-AGENT.md` | Quality analysis |
| Performance | `PERFORMANCE-AGENT.md` | Optimization |
| Demo | `DEMO-AGENT.md` | User perspective critique (calls Visual QA) |
| **E2E** | `E2E-AGENT.md` | **Browser testing** - Playwright, real user flows, screenshot capture |
| **Visual QA** | `VISUAL-QA-AGENT.md` | **Screenshot analysis** - Claude vision, CSS bugs, layout issues |
| **Data Flow** | `DATA-FLOW-VERIFICATION.md` | **Schema sync, placeholder detection, data trace** |
| **Bugfix** | `BUGFIX-AGENT.md` | **Systematic debugging** - Reproduce, trace, isolate, fix, verify |
| Conductor | `CONDUCTOR-AGENT.md` | Quality gate (98/100 with visual QA dimension) |

All files in `references/` directory.

---

## Agent Tool Requirements

Each agent requires specific tools based on their responsibilities:

| Agent | Required Tools | Purpose |
|-------|---------------|---------|
| **PM** | Bash, Read, Write, Edit | Git operations, file management, orchestration |
| **Discovery** | Read | Analyze existing code for iteration mode |
| **Designer** | Write, Edit | Create design tokens, semantic registry |
| **Backend** | Bash, Read, Write, Edit | Run migrations, create APIs, database setup |
| **Frontend** | Read, Write, Edit | Read registry/contracts, build UI components |
| **Docs** | Read, Write, Edit | Create README, API docs, CLAUDE.md |
| **DevOps** | Bash, Write, Edit | Docker setup, CI/CD configuration |
| **QA** | Bash, Read, Write, Edit | Run tests, create test files |
| **Security** | Bash, Read | Run security scans, analyze code |
| **Code Review** | Read, Edit | Analyze and refactor code |
| **Performance** | Bash, Read, Edit | Profile code, optimize queries |
| **Demo** | Read | Analyze from user perspective |
| **E2E** | Bash, Read, Write, Edit | Playwright setup, browser tests, add data-testid |
| **Bugfix** | Bash, Read, Write, Edit | Trace data flow, diagnose root cause, apply fixes |
| **Conductor** | Read | Score quality, make ship decisions |

---

## Critical Rules

1. **PM runs first and last** - PM orchestrates the entire build
2. **Feature branches, not agent branches** - `feature/user-auth` not `feature/backend`
3. **Multiple agents per feature** - Designer + Frontend collaborate on same branch
4. **Bugfix branches for failures** - Don't rebuild, fix targeted issues
5. **Iteration reads existing code** - PM understands what exists before adding
6. **98% threshold** - Conductor iterates until quality passes
7. **Runtime verification** - "Compiles" ≠ "Works"
8. **Data flow verification** - No "Unknown" or placeholder data in UI

---

## File Structure

```
auto-app-builder/
├── SKILL.md                          # This file (entry point)
├── README.md
├── LICENSE
└── references/
    ├── ORCHESTRATION.md              # ← READ FIRST
    ├── PM-AGENT.md                   # ← PM orchestrates
    ├── AIMEM-INTEGRATION.md          # ← Agent coordination
    ├── DISCOVERY-AGENT.md
    ├── DESIGNER-AGENT.md
    ├── BACKEND-AGENT.md
    ├── FRONTEND-AGENT.md
    ├── QA-AGENT.md                   # ← Includes schema sync checks
    ├── DEVOPS-AGENT.md
    ├── DEMO-AGENT.md                 # ← Calls Visual QA for screenshot analysis
    ├── E2E-AGENT.md                  # ← Browser testing + screenshot capture
    ├── VISUAL-QA-AGENT.md            # ← NEW: Claude vision screenshot analysis
    ├── DATA-FLOW-VERIFICATION.md     # ← Schema sync, placeholder detection
    ├── BUGFIX-AGENT.md               # ← Systematic debugging methodology
    ├── CONDUCTOR-AGENT.md            # ← Quality gate with Visual QA dimension
    ├── SECURITY-AGENT.md
    ├── CODE-REVIEW-AGENT.md
    ├── PERFORMANCE-AGENT.md
    ├── DOCS-AGENT.md
    ├── MEMORY-COORDINATION.md
    ├── QA-SCORING.md
    └── RUNTIME-VERIFICATION.md
```

---

## License

MIT • [TurkeyCode.ai](https://turkeycode.ai)
