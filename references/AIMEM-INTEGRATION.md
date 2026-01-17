# aimem Integration

**Specific aimem MCP Calls for Each Build Phase**

## Overview

aimem provides persistent memory across conversations and projects. Every agent reads from and writes to aimem to maintain coordination.

---

## Startup Check (Required)

Before any build begins, PM MUST verify aimem is available.

### Availability Check

```javascript
// PM runs this before starting any build
async function checkAimemAvailable() {
  try {
    // Try a simple query to verify aimem is responding
    const result = await aimem_query("health_check", type="all", limit=1);
    return { available: true };
  } catch (error) {
    return {
      available: false,
      error: error.message
    };
  }
}
```

### On Startup

```yaml
# PM's first action
startup_check:
  action: aimem_query("turkeycode", type="all", limit=1)

  if_available:
    message: "aimem connected. Memory coordination enabled."
    continue: true

  if_unavailable:
    message: |
      aimem MCP server not available.

      **Impact:**
      - Cross-project learning disabled
      - Agent coordination will use file-based fallback
      - Conductor vault patterns unavailable

      **To enable full functionality:**
      Install aimem: https://github.com/aimem/aimem-mcp

      **Continue anyway?** Build will work but without memory features.

    options:
      - "Continue without aimem (reduced functionality)"
      - "Abort and install aimem first"
```

---

## Fallback Mode (No aimem)

When aimem is unavailable, use file-based coordination instead.

### File-Based State

```yaml
# Instead of aimem_store, write to .turkeycode/state.yaml
.turkeycode/
├── state.yaml           # Current build state
├── scope.yaml           # Discovery output
├── semantic-registry.yaml  # Designer output
├── api-contracts.yaml   # Backend output
└── decisions/
    └── *.yaml           # Decision records
```

### Fallback Coordination

```yaml
# Agent completion signals via git commits
# Instead of: aimem_store({ key: "agent:designer:status", ... })
# Do: Commit with message containing status

commit_message: |
  feat(design): complete design tokens and registry

  [TURKEYCODE:STATUS]
  agent: designer
  status: complete
  outputs: [semantic-registry.yaml, design-tokens.css]
  [/TURKEYCODE:STATUS]
```

### Fallback Queries

```yaml
# Instead of: aimem_query("agent:designer:status")
# Do: Parse git log for status blocks

git log --grep="TURKEYCODE:STATUS" --format="%B" | parse_status_blocks
```

### What's Lost Without aimem

| Feature | With aimem | Without aimem |
|---------|------------|---------------|
| Agent coordination | Real-time memory | Git commit parsing |
| Cross-project learning | Conductor vault queries | Not available |
| Pattern reuse | aimem_query for patterns | Not available |
| Decision memory | Queryable decisions | File-based only |
| Build history | Full queryable history | Git log only |

### Fallback Warning

When running in fallback mode, PM outputs warning at start:

```
⚠️ Running in fallback mode (aimem unavailable)

- Agent coordination: File-based (slower)
- Cross-project learning: Disabled
- Pattern reuse: Disabled

Build will complete but without memory optimization.
```

---

## aimem Tools Available

```javascript
// Search code, conversations, decisions, commits
aimem_query(query, type?, limit?)
// type: "all" | "structures" | "conversations" | "decisions" | "commits"

// Check if function, class, or file exists
aimem_verify(name, type?)
// type: "structure" | "file"

// Search past conversations
aimem_conversations(query?, id?, limit?)
```

## Phase-by-Phase aimem Usage

### Initial Setup

Before any agent starts, coordinator sets up:

```javascript
// Store project info
aimem_store({
  key: "project:init",
  value: {
    name: "adhd-focus-tracker",
    repo: "github.com/user/adhd-tracker",
    develop_branch: "develop",
    started_at: new Date().toISOString()
  }
})
```

### Phase 1: Discovery Agent

**Before starting:**
```javascript
// Check for similar past projects
aimem_query("scope document requirements", type="conversations", limit=5)

// Look for feature patterns we've built before
aimem_query("user authentication feature pattern", type="structures")
aimem_query("session tracking implementation", type="structures")
```

**After completing (on develop branch):**
```javascript
// Signal completion with git info
aimem_store({
  key: "agent:discovery:status",
  value: {
    status: "complete",
    branch: "develop",
    commit: "abc123",
    outputs: ["scope.yaml"]
  }
})
```

---

### Phase 2: Designer Agent (feature/design branch)

**Before starting:**
```javascript
// Wait for discovery
aimem_query("agent:discovery:status", type="structures")

// Find successful design patterns
aimem_query("design tokens CSS custom properties", type="structures")
aimem_query("semantic registry class names", type="conversations")
```

**After completing and merging:**
```javascript
// Signal merge to develop
aimem_store({
  key: "agent:designer:status",
  value: {
    status: "complete",
    branch: "feature/design",
    merged_to: "develop",
    commit: "def456",
    outputs: ["semantic-registry.yaml", "design-tokens.css"]
  }
})
```

---

### Phase 3: Backend Agent (feature/backend branch)

**Before starting:**
```javascript
// Wait for discovery
aimem_query("agent:discovery:status", type="structures")

// Check for existing implementations
aimem_query("SessionDetectionService", type="structures")
aimem_query("heartbeat API endpoint", type="structures")
```

**After completing and merging:**
```javascript
// Signal merge to develop
aimem_store({
  key: "agent:backend:status",
  value: {
    status: "complete",
    branch: "feature/backend",
    merged_to: "develop",
    commit: "ghi789",
    outputs: ["api-contracts.yaml", "app/controllers/", "app/models/"]
  }
})
```

---

### Phase 3: Frontend Agent (feature/frontend branch)

**Before starting (CRITICAL - must wait for merges):**
```javascript
// MUST verify Designer AND Backend have MERGED to develop
const designerStatus = aimem_query("agent:designer:status", type="structures")
const backendStatus = aimem_query("agent:backend:status", type="structures")

// Check for "merged_to: develop" - not just "complete"
if (designerStatus.merged_to !== "develop" || backendStatus.merged_to !== "develop") {
  // WAIT - poll every 30 seconds
  sleep(30)
  retry()
}

// Now safe to: git checkout develop && git pull
// semantic-registry.yaml and api-contracts.yaml are in the repo
```

**During build:**
```javascript
// Verify class names from the actual file in repo (not aimem)
// The file is now in develop branch after Designer merged
// Read: semantic-registry.yaml

// Verify API paths from the actual file
// Read: api-contracts.yaml
```

**After completing and merging:**
```javascript
aimem_store({
  key: "agent:frontend:status",
  value: {
    status: "complete",
    branch: "feature/frontend",
    merged_to: "develop",
    commit: "jkl012",
    outputs: ["src/components/", "src/pages/"]
  }
})
```

---

### Phase 4: QA Agent (feature/qa branch)

**Before starting:**
```javascript
// Wait for all build agents to merge
aimem_query("agent:frontend:status merged_to develop")
aimem_query("agent:backend:status merged_to develop")

// git checkout develop && git pull
```

**After completing:**
```javascript
aimem_store({
  key: "agent:qa:status",
  value: {
    status: "complete",
    branch: "feature/qa",
    merged_to: "develop",
    test_results: { passed: 45, failed: 0 }
  }
})
```

---

### Phase 5: Review Agents (parallel, feature branches)

**Security Agent (feature/security-review):**
```javascript
// After review and fixes
aimem_store({
  key: "agent:security:status",
  value: {
    status: "complete",
    branch: "feature/security-review",
    merged_to: "develop",
    issues_found: 2,
    issues_fixed: 2
  }
})
```

**Code Review Agent (feature/code-review):**
```javascript
aimem_store({
  key: "agent:code-review:status",
  value: {
    status: "complete",
    branch: "feature/code-review",
    merged_to: "develop",
    improvements: ["extracted service object", "added index"]
  }
})
```

**Performance Agent (feature/performance):**
```javascript
aimem_store({
  key: "agent:performance:status",
  value: {
    status: "complete",
    branch: "feature/performance",
    merged_to: "develop",
    optimizations: ["added caching", "fixed N+1"]
  }
})
```

**Demo Agent (works on develop, no separate branch):**
```javascript
// Demo critiques the current state of develop
aimem_store({
  key: "agent:demo:status",
  value: {
    status: "complete",
    semantic_compliance: 0.95,
    issues_found: ["missing loading state on analytics"],
    critique_report: "demo-critique.md"
  }
})
```

---

### Phase 6: Conductor Agent

**Before scoring:**
```javascript
// Verify all agents have merged
const agents = ["designer", "backend", "frontend", "devops", "qa", "security", "code-review", "performance"]
for (const agent of agents) {
  const status = aimem_query(`agent:${agent}:status`, type="structures")
  if (status.merged_to !== "develop") {
    throw new Error(`${agent} not merged yet`)
  }
}

// Get historical benchmarks from conductor vault
aimem_query("conductor:vault:benchmarks", type="structures")
aimem_query("conductor:vault:failure_patterns", type="structures")
```

**After scoring:**
```javascript
// Record this build's outcome
aimem_store({
  key: "conductor:score",
  value: {
    score: 0.94,
    iteration: 1,
    failures: [
      { agent: "frontend", issue: "loading states missing" }
    ],
    decision: "iterate"
  }
})

// If iterating, record what needs fixing
aimem_store({
  key: "conductor:iteration:1",
  value: {
    fixes_needed: [
      { branch: "fix/loading-states", agent: "frontend", issue: "..." }
    ]
  }
})
```

**After final approval:**
```javascript
// Record successful build in vault for future reference
aimem_store({
  key: "conductor:vault:project_history",
  value: {
    project: "adhd-focus-tracker",
    final_score: 0.98,
    iterations: 2,
    total_time: "42 minutes",
    merged_to_main: true,
    tag: "v1.0.0"
  }
})
```

---

### Phase 7: Runtime Verification

**Before starting:**
```javascript
// Verify Conductor approved
const conductorScore = aimem_query("conductor:score", type="structures")
if (conductorScore.score < 0.98) {
  throw new Error("Conductor has not approved - score too low")
}

// Verify we're on develop with all merges
// git checkout develop && git pull
```

**After verification passes:**
```javascript
aimem_store({
  key: "runtime:verification",
  value: {
    status: "passed",
    endpoints_tested: 12,
    all_passed: true,
    ready_for_main: true
  }
})

// Trigger merge to main
// git checkout main && git merge develop && git tag v1.0.0 && git push --tags
```

---

## Memory Patterns

### Storing Team State

Since aimem indexes git commits, store state in committed files:

```yaml
# .turkeycode/state.yaml (committed to repo)
project: adhd-focus-tracker
phase: frontend
status: in_progress

semantic_registry_ready: true
api_contracts_ready: true

agents:
  discovery: complete
  designer: complete
  backend: complete
  frontend: in_progress
  qa: pending
```

Query later:
```javascript
aimem_query("turkeycode state phase", type="structures")
```

### Storing Decisions

Commit decision records:

```yaml
# .turkeycode/decisions/001-auth-approach.yaml
decision: Use JWT with refresh tokens
date: 2024-01-01
reasoning: |
  - Stateless for API
  - Works with VS Code extension
  - Standard pattern
alternatives_considered:
  - Session cookies (rejected: doesn't work well with extension)
  - API keys (rejected: less secure for user auth)
```

Query later:
```javascript
aimem_query("decision auth JWT", type="structures")
aimem_conversations("why JWT authentication")
```

### Storing Patterns

When something works well, commit it as a pattern:

```yaml
# .turkeycode/patterns/session-detection.yaml
name: Session Detection Service
quality_score: 0.97
iterations: 1
description: |
  Detects session boundaries from heartbeat gaps.
  30-minute gap = new session.
  
key_code: |
  def find_or_create_session(heartbeat_data)
    recent = @user.sessions
                  .where(end_time: nil)
                  .where('updated_at > ?', 30.minutes.ago)
                  .first
    # ...
  end
  
learned:
  - Use heartbeat timestamp, not session updated_at
  - Close previous session before creating new
```

Query later:
```javascript
aimem_query("session detection pattern", type="structures")
```

---

## Conductor Vault via aimem

The Conductor's cross-project memory is stored in committed files:

```yaml
# .turkeycode/vault/benchmarks.yaml
quality_scores:
  p50: 0.92
  p75: 0.95
  p90: 0.98

common_failures:
  - pattern: semantic_class_mismatch
    frequency: 42
    typical_agent: frontend
    typical_fix: "Read registry before building"
    
  - pattern: api_path_mismatch
    frequency: 23
    typical_agents: [frontend, backend]
    typical_fix: "Verify contracts match"

project_history:
  - project: adhd-tracker
    score: 0.94
    iterations: 2
    issues: [semantic_compliance, loading_states]
    
  - project: invoice-app
    score: 0.98
    iterations: 1
    issues: []
```

Query patterns:
```javascript
// Before build - check common issues
aimem_query("conductor vault common failures", type="structures")

// After build - compare to history
aimem_query("conductor vault project history quality", type="structures")
```

---

## Learning System Keys

The learning system uses aimem for cross-project intelligence. See [LEARNING-SYSTEM.md](./LEARNING-SYSTEM.md) for full documentation.

### Key Namespace

All learning data uses the `turkeycode:learning:` prefix:

| Key Pattern | Purpose |
|------------|---------|
| `turkeycode:learning:agent_pattern:{agent}:{id}` | Agent-specific learned patterns |
| `turkeycode:learning:conductor_pattern:{key}` | Success/failure tracking by pattern type |
| `turkeycode:learning:project:{hash}` | Project outcome records |
| `turkeycode:learning:agent_performance:{agent}:{hash}` | Per-project agent performance |
| `turkeycode:learning:vault:benchmarks` | Quality score percentiles (p50/p75/p90) |
| `turkeycode:learning:vault:failure_patterns` | Common failure patterns |

### Agent Learning Hooks

**Before work** - query relevant patterns:
```javascript
// Get high-confidence patterns for this agent
const patterns = aimem_query(
  `turkeycode learning agent_pattern ${agentName}`,
  type="structures",
  limit=20
)

// Filter by confidence (see LEARNING-SYSTEM.md for calculation)
const highConfidence = patterns.filter(p => calculateConfidence(p) >= 50)
```

**After work** - record what was learned:
```javascript
aimem_store({
  key: `turkeycode:learning:agent_pattern:${agentName}:${uuid}`,
  value: {
    type: 'agent_learning_pattern',
    agent_name: agentName,
    pattern: "What was learned",
    outcome: "What happened",
    success: true,
    frequency: 1,
    created_at: new Date().toISOString()
  }
})
```

### Cross-Project Queries

```javascript
// Find similar past projects
aimem_query("turkeycode learning project", type="structures")

// Get predicted issues for this project type
aimem_query("turkeycode learning vault failure_patterns", type="structures")

// Get quality benchmarks
aimem_query("turkeycode learning vault benchmarks", type="structures")
```

### Confidence Scoring

Patterns have confidence scores (0-100):
- Base: 100
- Frequency < 3: **-20**
- Age > 3 months: **-15**
- Programming error terms: **-30**
- False memory flag: **-50**

Only use patterns with confidence >= 50 in agent prompts.

### Fallback Learning Storage

When aimem unavailable, learning data stored in:
```
.turkeycode/learning/
├── agent_patterns/{agent}.yaml
├── conductor_patterns.yaml
├── projects/{hash}.yaml
└── vault/
    ├── benchmarks.yaml
    └── failure_patterns.yaml
```

---

## Verification Queries

Always verify critical items exist before proceeding:

```javascript
// Before Frontend starts
aimem_verify("semantic-registry.yaml", type="file")  // MUST exist
aimem_verify("api-contracts.yaml", type="file")       // MUST exist

// Before QA starts
aimem_verify("HeartbeatsController", type="structure")
aimem_verify("SessionCard", type="structure")

// Before Conductor scores
aimem_verify("spec/", type="file")  // Tests must exist
```

---

## Recovery Queries

If build fails or needs to resume:

```javascript
// Find where we left off
aimem_query("turkeycode state phase status", type="structures")

// Find what was already built
aimem_verify("Session", type="structure")
aimem_verify("HeartbeatsController", type="structure")
aimem_verify("SessionCard", type="structure")

// Check past conversation for context
aimem_conversations("turkeycode build adhd tracker")
```

---

## Best Practices

1. **Commit frequently** - aimem indexes git, so committed files are searchable
2. **Use structured filenames** - `semantic-registry.yaml` is findable, `stuff.txt` is not
3. **Store decisions** - Future builds benefit from past reasoning
4. **Query before building** - Don't reinvent; find existing patterns
5. **Verify dependencies** - Frontend MUST verify registry exists before starting
6. **Record outcomes** - Conductor vault improves with every project

---

## Example Full Build Query Sequence

```javascript
// 1. Start - check for existing project
aimem_query("adhd focus tracker", type="all")

// 2. Discovery - find similar scopes
aimem_query("passive tracking scope requirements", type="conversations")

// 3. Design - find design patterns
aimem_query("dark theme design tokens", type="structures")

// 4. Backend - find API patterns
aimem_query("heartbeat session detection", type="structures")

// 5. Frontend - MUST read registry
aimem_query("semantic-registry", type="structures")
aimem_verify("semantic-registry.yaml", type="file")

// 6. QA - find test patterns
aimem_query("session service spec rspec", type="structures")

// 7. Review - check common issues
aimem_query("conductor vault common failures semantic", type="structures")

// 8. Conductor - compare to history
aimem_query("conductor vault quality benchmark", type="structures")

// 9. Runtime - find startup patterns
aimem_query("Rails server verification", type="conversations")
```
