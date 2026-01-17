# Turkey Build

**Build like a team, not a hackathon.**

PM scopes features, creates branches, assigns agents. Multiple agents collaborate on each feature. Same workflow for greenfield and iteration.

Not built for speed. Built for reliable, well-built code (hopefully).

## Requirements

- **Claude Code** (CLI) or compatible AI coding assistant
- **aimem MCP server** (optional but recommended) - Enables cross-project learning and agent memory coordination

## How It Works

```
PM Agent → scope.yaml with features
      ↓
┌─────────────────────────────────────┐
│  feature/core-storage               │
│  Backend agent                      │
└─────────────────────────────────────┘
      ↓ merge to develop
┌─────────────────────────────────────┐
│  feature/api-server                 │
│  Backend + DevOps agents            │
└─────────────────────────────────────┘
      ↓ merge to develop
┌─────────────────────────────────────┐
│  feature/web-dashboard              │
│  Designer + Frontend agents         │
└─────────────────────────────────────┘
      ↓ merge to develop

Review Wave: QA + Security + CodeReview + Performance
      ↓
Runtime Verification (failures → bugfix/*)
      ↓
Data Flow Verification (schema sync, no placeholders)
      ↓
E2E Browser Testing (Playwright, 3 viewports)
      ↓
Visual QA (Claude vision screenshot analysis)
      ↓
Conductor (98/100 gate)
      ↓
merge develop → main → v1.0.0
```

Feature branches. Multiple agents per feature. Bugfixes in bugfix branches. Clean git history.

## Seven Modes

### Greenfield (New Build)
```
"Build a clipboard history manager"
→ PM scopes 5 features → Builds each → Ships v1.0.0
```

### Iteration (Add Features)
```
"Add date filtering and export"
→ PM reads existing code → Scopes 2 features → Ships v1.1.0
```

### Bugfix (Fix Issues)
```
"Cards not showing in battle view"
→ PM creates bugfix branch → Trace → Fix → Verify
```

### Refactor (Restructure Code)
```
"Clean up the auth code" / "Split the god file"
→ Analyze → Plan → Restructure → Verify behavior unchanged
```

### UI Polish (Visual Cleanup)
```
"Make the UI look better" / "Fix the messy CSS"
→ Visual QA scan → Designer review → CSS cleanup → Responsive fixes
```

### Migration (Upgrade Dependencies)
```
"Upgrade to React 19" / "Move from Express to Hono"
→ Audit → Plan migration → Update incrementally → Test each step
```

### Audit (Analysis Only)
```
"Review security" / "Check performance" / "Analyze code quality"
→ Run relevant agents → Produce report → No code changes unless requested
```

## The Agents

| Agent | Role |
|-------|------|
| **PM** | Orchestrator - scopes features, assigns agents |
| Discovery | Requirements → scope.yaml |
| Designer | Design tokens, semantic registry, component specs |
| Backend | API, database, business logic |
| Frontend | UI components using semantic classes |
| Docs | README, API docs, CLAUDE.md |
| DevOps | Docker, CI/CD, deployment |
| QA | Unit/integration tests |
| Security | Vulnerability scanning |
| CodeReview | Quality analysis |
| Performance | Optimization |
| Demo | User perspective critique |
| **E2E** | Playwright browser testing, screenshot capture |
| **Visual QA** | Claude vision screenshot analysis |
| **Data Flow** | Schema sync, placeholder detection |
| **Bugfix** | Systematic debugging (reproduce → trace → isolate → fix → verify) |
| Conductor | Quality gate (98/100 required) |

## Learning System (aimem)

When aimem MCP server is available, agents learn from past builds:

- **Agent Patterns** - What worked/failed for each agent type
- **Project History** - Outcomes and quality scores across projects
- **Confidence Scoring** - Patterns rated 0-100 based on frequency and success
- **False Memory Detection** - Contradictory patterns identified and demoted

```yaml
# Agents query before building
"What design patterns worked for similar apps?"
"What API issues were found in past projects?"

# Agents record after building
"This session detection approach scored 97%"
"Schema mismatch was the main issue"
```

Fallback: Without aimem, agents work independently (no cross-project learning).

## Installation

### Claude Code
```bash
mkdir -p ~/.claude/skills
git clone https://github.com/rangerchaz/turkey-build.git ~/.claude/skills/turkey-build
```

### With aimem (recommended)
Configure aimem MCP server in your Claude Code settings for cross-project learning.

## Usage

```
/turkey-build build a clipboard history manager

/turkey-build add search filters to the existing app

/turkey-build fix the bug where cards don't display
```

## What You Get

- **Working application** (runtime verified, E2E tested)
- **Full test suite** (unit + E2E, 50-150+ tests typical)
- **Visual QA** (screenshots at 3 viewports)
- **Security review**
- **Docker + CI/CD**
- **Documentation**
- **Clean git history** with feature branches

## Quality Gate

| Dimension | Weight |
|-----------|--------|
| Functionality | 25% |
| Code Quality | 25% |
| Test Coverage | 20% |
| Design System | 15% |
| Documentation | 15% |

**98% required.** Failures → bugfix branches → re-score.

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
        │   (runtime/e2e fails)
        ├── bugfix/null-response ───► merge
        │
        │   (conductor approved)
        └───────────────────────────► merge to main
                                      tag v1.0.0
```

## Files

```
turkey-build/
├── SKILL.md                    # Entry point
├── README.md                   # This file
├── LICENSE                     # MIT
└── references/
    ├── ORCHESTRATION.md        # Execution flow
    ├── PM-AGENT.md             # Orchestrator spec
    ├── AIMEM-INTEGRATION.md    # Memory coordination
    ├── MEMORY-COORDINATION.md  # Team memory system
    ├── LEARNING-SYSTEM.md      # Cross-project learning
    ├── DISCOVERY-AGENT.md
    ├── DESIGNER-AGENT.md
    ├── BACKEND-AGENT.md
    ├── FRONTEND-AGENT.md
    ├── QA-AGENT.md
    ├── DEVOPS-AGENT.md
    ├── SECURITY-AGENT.md
    ├── CODE-REVIEW-AGENT.md
    ├── PERFORMANCE-AGENT.md
    ├── DEMO-AGENT.md
    ├── DOCS-AGENT.md
    ├── E2E-AGENT.md            # Browser testing
    ├── VISUAL-QA-AGENT.md      # Screenshot analysis
    ├── DATA-FLOW-VERIFICATION.md
    ├── BUGFIX-AGENT.md         # Systematic debugging
    ├── CONDUCTOR-AGENT.md      # Quality gate
    ├── QA-SCORING.md
    └── RUNTIME-VERIFICATION.md
```

## License

MIT

---

Built by [Chad Cox](https://linkedin.com/in/chadcox1) • [TurkeyCode.ai](https://turkeycode.ai)
