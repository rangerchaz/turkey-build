# Auto App Builder

**Build like a team, not a hackathon.**

PM scopes features, creates branches, assigns agents. Multiple agents collaborate on each feature. Same workflow for greenfield and iteration.

Not built for speed. Built for reliable, well-built code.

## How It Works

```
🦃 PM Agent → scope.yaml with features
      ↓
┌─────────────────────────────────────┐
│  🦃 feature/core-storage            │
│  Backend agent                      │
└─────────────────────────────────────┘
      ↓ merge to develop
┌─────────────────────────────────────┐
│  🦃 feature/api-server              │
│  Backend + DevOps agents            │
└─────────────────────────────────────┘
      ↓ merge to develop
┌─────────────────────────────────────┐
│  🦃 feature/web-dashboard           │
│  Designer + Frontend agents         │
└─────────────────────────────────────┘
      ↓ merge to develop

🦃 Review Wave: QA + Security + CodeReview + Perf
      ↓
🦃 Runtime Verification (failures → bugfix/*)
      ↓
🦃 Conductor (98/100 gate)
      ↓
🦃🦃🦃 merge develop → main → v1.0.0
```

Feature branches. Multiple agents per feature. Bugfixes in bugfix branches. Clean git history.

## Two Modes

### Greenfield
```
"Build a clipboard history manager"
→ PM scopes 5 features → Builds each → Ships v1.0.0
```

### Iteration  
```
"Add date filtering and export"
→ PM reads existing code → Scopes 2 features → Ships v1.1.0
```

Same workflow. PM just scopes fewer features for iteration.

## The Agents

| Agent | Role |
|-------|------|
| **PM** | Orchestrator - scopes features, assigns agents |
| Designer | Design tokens, component specs |
| Backend | API, database, business logic |
| Frontend | UI components |
| Docs | README, API docs, CLAUDE.md |
| Report | Status and analytics reports |
| DevOps | Docker, CI/CD, deployment |
| QA | Tests |
| Security | Vulnerability scan |
| CodeReview | Quality analysis |
| Performance | Optimization |
| Demo | User perspective critique |
| Conductor | Quality gate (98/100 required) |

## Installation

### Claude Code
```bash
mkdir -p ~/.claude/skills
cp -r auto-app-builder ~/.claude/skills/
```

### Cursor
```bash
mkdir -p ~/.cursor/skills
cp -r auto-app-builder ~/.cursor/skills/
```

## Usage

```
Use the auto-app-builder skill to build a clipboard history manager

Use the auto-app-builder skill to add search filters to the existing app
```

## What You Get

After 40-60 minutes:

- **Working application** (runtime verified)
- **Full test suite** (50+ tests typical)
- **Security review**
- **Docker + CI/CD**
- **Documentation**
- **Clean git history** with feature branches

## Git Flow

```
main (releases only)
  │
  └── develop (integration)
        │
        ├── feature/user-auth ──────► merge
        ├── feature/dashboard ──────► merge
        ├── feature/search ─────────► merge
        │
        │   (runtime fails)
        ├── bugfix/null-response ───► merge
        │
        │   (conductor approved)
        └───────────────────────────► merge to main
                                      tag v1.0.0
```

## Quality Gate

| Category | Points |
|----------|--------|
| Functionality | 25 |
| Code Quality | 20 |
| Security | 20 |
| Testing | 15 |
| Documentation | 10 |
| Performance | 10 |

**98/100 required.** Failures → bugfix branches → re-score.

## Example Build

```
🦃 Building: Clipboard History Manager
   "Let's talk turkey."

[6:09 PM] 🦃 PM scopes 5 features
[6:12 PM] 🦃 feature/core-daemon (Backend) → merged ✓
[6:18 PM] 🦃 feature/storage-layer (Backend) → merged ✓
[6:25 PM] 🦃 feature/api-server (Backend, DevOps) → merged ✓
[6:38 PM] 🦃 feature/web-dashboard (Designer, Frontend) → merged ✓
[6:42 PM] 🦃 feature/documentation (Docs) → merged ✓
          "The nest is built."
[6:55 PM] 🦃 Review Wave complete ✓
          "Green across the board. Gobble approved."
[7:00 PM] 🦃 Runtime verification: ALL PASS ✓
          "Not a cold turkey - this one runs hot!"
[7:05 PM] 🦃 Conductor: 98/100 ✓
          "This turkey is fully baked."
[7:07 PM] 🦃🦃🦃 v1.0.0 tagged
          "Gobble gobble - dinner is served!"

📁 Delivered:
  src/         Application code
  tests/       55 automated tests
  public/      Dark-themed dashboard
  Dockerfile   Ready to deploy
```

## Example Iteration

```
🦃 Adding features to: Clipboard History Manager

[7:30 PM] 🦃 PM reads existing code
[7:32 PM] 🦃 PM scopes 2 new features
[7:35 PM] 🦃 feature/date-filter (Backend, Frontend) → merged ✓
[7:42 PM] 🦃 feature/export-json (Backend, Frontend) → merged ✓
[7:48 PM] 🦃 Review Wave (light) ✓
[7:52 PM] 🦃 Runtime verification: ALL PASS ✓
[7:55 PM] 🦃 Conductor: 98/100 ✓
[7:57 PM] 🦃🦃🦃 v1.1.0 tagged

📁 Changes:
  src/server.js   +45 lines (filter params, export endpoint)
  public/app.js   +120 lines (filter UI, export button)
  tests/          +12 new tests (67 total)
```

## Files

```
auto-app-builder/
├── SKILL.md              # Entry point
├── README.md             # This file
├── LICENSE               # MIT
└── references/
    ├── ORCHESTRATION.md  # Execution flow
    ├── PM-AGENT.md       # Orchestrator spec (includes turkey humor)
    ├── *-AGENT.md        # Agent specifications
    ├── QA-SCORING.md     # 100-point rubric
    └── RUNTIME-VERIFICATION.md
```

## License

MIT

---

Built by [Chad Cox](https://linkedin.com/in/chadcox) • [TurkeyCode.ai](https://turkeycode.ai)
