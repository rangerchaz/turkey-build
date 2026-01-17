# Turkey Build

**Build like a team, not a hackathon.**

PM scopes features, creates branches, assigns agents. Multiple agents collaborate on each feature. Same workflow for greenfield and iteration.

Not built for speed. Built for reliable, well-built code (hopefully).

## Requirements

- **Claude Max or Pro subscription** (recommended) - This skill uses significant tokens. A full build spawns multiple agents, each reading reference docs, writing code, and running verification.
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

**Quick Start:** `/turkey-build build me a todo app with add, complete, and delete`

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

When aimem MCP server is available, agents learn from past builds and get smarter over time.

### What Gets Learned

| Data Type | Purpose | Example |
|-----------|---------|---------|
| **Agent Patterns** | Per-agent success/failure patterns | "Session detection works better with heartbeat.timestamp" |
| **Project History** | Outcomes and quality scores | "Project X scored 97% with 1 iteration" |
| **Conductor Patterns** | Success rates by pattern type | "Dark themes fail 20% due to contrast issues" |
| **Vault Benchmarks** | Quality percentiles across all projects | p50: 92%, p75: 95%, p90: 98% |

### Confidence Scoring

Not all patterns are equal. Each pattern gets a confidence score (0-100):

```
Base score: 100
- Frequency < 3 occurrences:  -20
- Pattern older than 3 months: -15
- Contains error indicators:   -30
- Marked as false memory:      -50
```

Only patterns with confidence >= 50 are used in prompts.

### False Memory Detection

Agents can learn wrong things. The system detects and filters these:

- **Contradictory patterns** - Same pattern with opposite outcomes gets flagged
- **Error indicators** - Patterns containing "didn't work", "reverted", "broke" get demoted
- **Low frequency** - Patterns seen only once are treated with skepticism

### How Agents Use It

**Before building:**
```yaml
# Designer queries for patterns
"What design tokens worked for similar apps?"
"What accessibility issues were found in past dark themes?"

# Backend queries for approaches
"What session detection implementation scored highest?"
"What API patterns caused issues?"
```

**After building:**
```yaml
# Conductor records outcomes
project: clipboard-manager
quality_score: 97
iterations: 1
issues_found: ["semantic class mismatch"]
fix_applied: "stricter class name validation"
```

### Key Structure

All learning data stored with `turkeycode:learning:` namespace:

```
turkeycode:learning:
├── agent_pattern:{agent}:{id}      # Per-agent patterns
├── conductor_pattern:{key}         # Success/failure by pattern type
├── project:{hash}                  # Project outcomes
├── agent_performance:{agent}:{hash} # Per-project agent stats
└── vault:
    ├── benchmarks                  # Quality percentiles
    └── failure_patterns            # Common failures
```

### Fallback Mode

Without aimem, the system falls back to file-based storage:

```
.turkeycode/learning/
├── agent_patterns/{agent}.yaml
├── conductor_patterns.yaml
├── projects/{hash}.yaml
└── vault/
    ├── benchmarks.yaml
    └── failure_patterns.yaml
```

Agents work independently without cross-project learning, but still function.

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
