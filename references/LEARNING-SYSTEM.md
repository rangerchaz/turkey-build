# Learning System

**Role:** Cross-Project Intelligence Layer
**Purpose:** Enable agents to learn from past builds and improve over time

---

## Overview

The learning system gives agents memory that persists across projects. Agents record what works, what fails, and use that knowledge to make better decisions on future builds.

**Key Capabilities:**
- **Agent Learning Patterns** - Per-agent success/failure patterns with confidence scoring
- **Cross-Project Learning** - Find similar projects, predict issues, use historical benchmarks
- **False Memory Detection** - Identify and filter out incorrect learnings
- **Performance Tracking** - Track agent success rates and trends over time

---

## aimem Key Structure

All learning data uses hierarchical keys with the `turkeycode:learning:` namespace.

### Key Patterns

| Key Pattern | Purpose | Example Query |
|------------|---------|---------------|
| `turkeycode:learning:agent_pattern:{agent}:{id}` | Agent-specific patterns | `aimem_query("turkeycode learning agent_pattern backend")` |
| `turkeycode:learning:conductor_pattern:{key}` | Pattern success/failure tracking | `aimem_query("turkeycode learning conductor_pattern")` |
| `turkeycode:learning:project:{hash}` | Project outcome records | `aimem_query("turkeycode learning project")` |
| `turkeycode:learning:agent_performance:{agent}:{hash}` | Per-project agent performance | `aimem_query("turkeycode learning agent_performance frontend")` |
| `turkeycode:learning:vault:benchmarks` | Quality score percentiles | `aimem_query("turkeycode learning vault benchmarks")` |
| `turkeycode:learning:vault:failure_patterns` | Aggregated failure patterns | `aimem_query("turkeycode learning vault failure_patterns")` |

---

## Data Schemas

### AgentLearningPattern

Individual patterns learned by agents during builds.

```yaml
# Key: turkeycode:learning:agent_pattern:{agent}:{id}
# Example: turkeycode:learning:agent_pattern:backend:a1b2c3d4

type: agent_learning_pattern
version: "1.0"
id: a1b2c3d4

# Core fields
agent_name: backend
pattern: |
  When building session detection, use heartbeat.timestamp for gap
  calculation, not session.updated_at
outcome: |
  Correctly detected session boundaries with zero false positives
success: true
frequency: 7                    # Times this pattern has occurred

# Context
project_id: clipboard-manager-a1b2
created_at: "2025-01-15T10:30:00Z"
updated_at: "2025-01-17T14:20:00Z"

# False memory tracking
false_memory_flag: false
false_memory_reason: null

# For queryability
tags:
  - session-detection
  - heartbeat
  - timing
```

### ConductorPattern

Tracks success/failure rates for pattern types (design styles, platforms, etc.).

```yaml
# Key: turkeycode:learning:conductor_pattern:{key}
# Example: turkeycode:learning:conductor_pattern:dark_theme

type: conductor_pattern
version: "1.0"
pattern_key: dark_theme

# Success/failure tracking
successes: 12
failures: 3
total: 15
failure_rate: 0.2

# Common issues when this pattern fails
common_issues:
  - issue: "Contrast ratio insufficient for accessibility"
    frequency: 2
    typical_fix: "Use WCAG AA compliant color tokens"
  - issue: "Dark mode toggle doesn't persist"
    frequency: 1
    typical_fix: "Store preference in localStorage"

# Which agents typically affected
agents_typically_affected:
  - designer
  - frontend

# High failure flag (set when failure_rate > 0.3)
high_failure_scope: false

created_at: "2025-01-10T08:00:00Z"
updated_at: "2025-01-17T14:20:00Z"
```

### ConductorProject

Records project outcomes for cross-project learning.

```yaml
# Key: turkeycode:learning:project:{hash}
# Example: turkeycode:learning:project:a1b2c3d4e5f6

type: conductor_project
version: "1.0"
project_hash: a1b2c3d4e5f6

# Project identification
project_name: clipboard-manager
description: Dark-themed clipboard history manager with SQLite backend
design_style: dark
platform: web

# Quality scores
quality_scores:
  overall: 0.98
  functionality: 0.95
  semantic_compliance: 0.97
  e2e_coverage: 1.0
  visual_qa: 0.95
  data_flow: 1.0
  integration_quality: 0.96
  code_quality: 0.92
  database_safety: 1.0
  accessibility: 0.88

# Issues found
violations:
  - dimension: accessibility
    issue: "Missing focus states on secondary buttons"
    severity: minor
    fixed: true

# Build metrics
iterations_needed: 2
success: true
build_duration_minutes: 45

# Per-agent performance
agent_performance:
  designer: { success: true, iterations: 1 }
  backend: { success: true, iterations: 1 }
  frontend: { success: true, iterations: 2, issues: ["wrong class names"] }
  qa: { success: true, iterations: 1 }

created_at: "2025-01-15T10:00:00Z"
completed_at: "2025-01-15T10:45:00Z"
```

### AgentPerformanceRecord

Per-project performance record for each agent.

```yaml
# Key: turkeycode:learning:agent_performance:{agent}:{hash}
# Example: turkeycode:learning:agent_performance:frontend:a1b2c3d4e5f6

type: agent_performance_record
version: "1.0"
id: frontend:a1b2c3d4e5f6

agent_name: frontend
success: true
design_style: dark
platform: web
project_hash: a1b2c3d4e5f6

iterations_needed: 2
issues_encountered:
  - "Used .button instead of .btn-primary"
  - "Forgot loading states on API calls"
issues_resolved: true
duration_minutes: 18

created_at: "2025-01-15T10:20:00Z"
```

### Vault: Benchmarks

Aggregated quality benchmarks across all projects.

```yaml
# Key: turkeycode:learning:vault:benchmarks

type: vault_benchmarks
version: "1.0"

# Quality score percentiles
quality_scores:
  p50: 0.92
  p75: 0.95
  p90: 0.98
  mean: 0.93
  count: 47

# Iterations needed percentiles
iterations_needed:
  p50: 1
  p75: 2
  p90: 3
  mean: 1.6

# Per-agent success rates
agent_success_rates:
  discovery: 0.98
  designer: 0.94
  backend: 0.91
  frontend: 0.88
  devops: 0.95
  qa: 0.97
  security: 0.99
  code_review: 0.96
  performance: 0.94

updated_at: "2025-01-17T14:00:00Z"
```

### Vault: Failure Patterns

Common failure patterns aggregated from all projects.

```yaml
# Key: turkeycode:learning:vault:failure_patterns

type: vault_failure_patterns
version: "1.0"

patterns:
  - pattern: semantic_class_mismatch
    frequency: 42
    typical_agent: frontend
    typical_fix: "Read semantic-registry.yaml before building components"

  - pattern: api_path_mismatch
    frequency: 23
    typical_agents: [frontend, backend]
    typical_fix: "Verify api-contracts.yaml matches implementation"

  - pattern: missing_loading_states
    frequency: 18
    typical_agent: frontend
    typical_fix: "Add loading/error/empty states to all data components"

updated_at: "2025-01-17T14:00:00Z"
```

---

## Confidence Scoring

Not all learned patterns are equally reliable. Confidence scoring filters out low-quality learnings.

### Calculation Algorithm

```
Base confidence: 100

Deductions:
  - frequency < 3:           -20  (not enough evidence)
  - age > 3 months:          -15  (may be outdated)
  - programming error terms: -30  (likely not a pattern issue)
  - false_memory_flag:       -50  (known incorrect)

Final: max(0, calculated_score)
```

### Programming Error Indicators

Patterns containing these terms in outcome get -30 confidence:
- `undefined`, `null pointer`, `null reference`
- `syntax error`, `TypeError`, `AttributeError`, `KeyError`
- `failed to compile`, `build failed`, `file not found`

These indicate programming mistakes, not agent learning patterns.

### Confidence Thresholds

| Confidence | Range | Usage |
|------------|-------|-------|
| High | 80-100 | Apply pattern automatically |
| Medium | 50-79 | Suggest with caveats |
| Low | 20-49 | Mention as possibility only |
| Unreliable | 0-19 | Do not use |

### Example

```yaml
# High confidence pattern (85)
agent_name: backend
pattern: "Use includes() for nested associations to avoid N+1"
success: true
frequency: 8          # No deduction
updated_at: "2025-01-10"  # Recent, no deduction
false_memory_flag: false
# Confidence: 100 - 0 = 100 (high)

# Low confidence pattern (35)
agent_name: frontend
pattern: "Always use inline styles for animations"
success: true
frequency: 2          # -20 (low frequency)
updated_at: "2024-08-15"  # -15 (old)
outcome: "Worked but build failed later"  # -30 (error indicator)
# Confidence: 100 - 20 - 15 - 30 = 35 (low)
```

---

## False Memory Detection

False memories are patterns incorrectly recorded as learnings.

### Detection Triggers

1. **Contradiction** - Same pattern, opposite outcomes across projects
2. **Programming errors** - Outcome contains error indicators (syntax, undefined, etc.)
3. **Low frequency + failure** - Pattern failed and only occurred 1-2 times

### Marking False Memories

When detected, patterns are marked:

```yaml
false_memory_flag: true
false_memory_reason: "[FALSE MEMORY: Contradicted by 3 later successes]"
frequency: 5  # Reduced by 2 from original 7
```

### Cleanup Rules

Patterns are candidates for removal when:
- `false_memory_flag: true` AND `frequency < 3`
- `confidence < 20` AND `age > 6 months`
- Contradicted by 3+ high-confidence patterns

---

## Agent Integration

### Pre-Work: Query Learning

Every agent queries learning before starting work.

```javascript
// Agent pre-work hook
async function queryLearningBeforeWork(agentName, projectContext) {
  // 1. Get agent-specific patterns (high confidence only)
  const patterns = await aimem_query(
    `turkeycode learning agent_pattern ${agentName}`,
    type="structures",
    limit=20
  );

  const highConfidence = patterns.filter(p =>
    calculateConfidence(p) >= 50
  );

  // 2. Get conductor warnings for this project type
  const conductorPatterns = await aimem_query(
    "turkeycode learning conductor_pattern",
    type="structures"
  );

  const warnings = conductorPatterns
    .filter(p => p.high_failure_scope &&
                 p.agents_typically_affected.includes(agentName))
    .map(p => ({
      pattern: p.pattern_key,
      failure_rate: p.failure_rate,
      common_fixes: p.common_issues.slice(0, 2)
    }));

  // 3. Get similar project insights
  const insights = await getProjectInsights(
    projectContext.design_style,
    projectContext.platform
  );

  return { patterns: highConfidence, warnings, insights };
}
```

### Post-Work: Record Learning

Every agent records learnings after completing work.

```javascript
// Agent post-work hook
async function recordLearningAfterWork(
  agentName,
  projectContext,
  success,
  patternDescription,
  outcomeDescription,
  issuesEncountered = []
) {
  const patternId = generateUUID().slice(0, 8);
  const key = `turkeycode:learning:agent_pattern:${agentName}:${patternId}`;

  // Check for similar existing pattern
  const existing = await findSimilarPattern(agentName, patternDescription);

  if (existing) {
    // Update existing pattern
    existing.frequency += 1;
    existing.updated_at = new Date().toISOString();

    // Check for contradiction (potential false memory)
    if (existing.success !== success) {
      await checkAndMarkFalseMemory(existing, success);
    }

    await aimem_store({ key: existing.key, value: existing });
  } else {
    // Create new pattern
    await aimem_store({
      key,
      value: {
        type: 'agent_learning_pattern',
        version: '1.0',
        id: patternId,
        agent_name: agentName,
        pattern: patternDescription,
        outcome: outcomeDescription,
        success,
        frequency: 1,
        project_id: projectContext.project_hash,
        created_at: new Date().toISOString(),
        updated_at: new Date().toISOString(),
        false_memory_flag: false,
        false_memory_reason: null,
        tags: extractTags(patternDescription)
      }
    });
  }

  // Record performance record
  await aimem_store({
    key: `turkeycode:learning:agent_performance:${agentName}:${projectContext.project_hash}`,
    value: {
      type: 'agent_performance_record',
      version: '1.0',
      agent_name: agentName,
      success,
      design_style: projectContext.design_style,
      platform: projectContext.platform,
      project_hash: projectContext.project_hash,
      issues_encountered: issuesEncountered,
      created_at: new Date().toISOString()
    }
  });
}
```

### Learning Summary for Prompts

Generate learning summary to include in agent prompts.

```javascript
function generateLearningSummary(patterns) {
  const successes = patterns.filter(p => p.success);
  const failures = patterns.filter(p => !p.success);

  let summary = "";

  if (successes.length > 0) {
    summary += "### KEY SUCCESS PATTERNS:\n";
    for (const p of successes.slice(0, 5)) {
      const conf = calculateConfidence(p);
      summary += `[${conf}%] ${p.pattern.slice(0, 100)}...\n`;
    }
    summary += "\n";
  }

  if (failures.length > 0) {
    summary += "### AVOID THESE MISTAKES:\n";
    for (const p of failures.slice(0, 3)) {
      summary += `${p.pattern.slice(0, 80)} -> ${p.outcome.slice(0, 50)}\n`;
    }
  }

  return summary;
}
```

---

## Cross-Project Learning

### Finding Similar Projects

```javascript
async function findSimilarProjects(designStyle, platform, limit = 5) {
  const allProjects = await aimem_query(
    "turkeycode learning project",
    type="structures"
  );

  // Score by similarity
  const scored = allProjects.map(project => {
    let score = 0;
    if (project.design_style === designStyle) score += 50;
    if (project.platform === platform) score += 50;
    return { score, project };
  });

  // Return top matches
  return scored
    .filter(s => s.score > 0)
    .sort((a, b) => b.score - a.score)
    .slice(0, limit)
    .map(s => s.project);
}
```

### Getting Project Insights

```javascript
async function getProjectInsights(designStyle, platform) {
  const similar = await findSimilarProjects(designStyle, platform);

  if (similar.length === 0) {
    return {
      predicted_issues: [],
      recommended_validations: ["Standard semantic compliance"],
      confidence: 'low',
      predicted_iterations: 2
    };
  }

  // Aggregate insights
  const allIssues = similar.flatMap(p =>
    (p.violations || []).map(v => v.issue)
  );

  // Count issue frequency
  const issueCounts = {};
  for (const issue of allIssues) {
    issueCounts[issue] = (issueCounts[issue] || 0) + 1;
  }

  // Common issues (appear in >30% of similar projects)
  const commonIssues = Object.entries(issueCounts)
    .filter(([_, count]) => count / similar.length > 0.3)
    .map(([issue]) => issue);

  // Average iterations
  const avgIterations = similar.reduce((sum, p) =>
    sum + p.iterations_needed, 0) / similar.length;

  return {
    predicted_issues: commonIssues,
    recommended_validations: generateValidations(commonIssues),
    confidence: similar.length >= 3 ? 'high' : 'medium',
    predicted_iterations: Math.round(avgIterations)
  };
}
```

### Conductor Post-Build Recording

```javascript
async function conductorRecordBuild(
  projectContext,
  qualityScores,
  violations,
  iterationsNeeded,
  agentPerformance
) {
  // 1. Record project
  await aimem_store({
    key: `turkeycode:learning:project:${projectContext.project_hash}`,
    value: {
      type: 'conductor_project',
      version: '1.0',
      project_hash: projectContext.project_hash,
      project_name: projectContext.project_name,
      design_style: projectContext.design_style,
      platform: projectContext.platform,
      quality_scores: qualityScores,
      violations,
      iterations_needed: iterationsNeeded,
      success: qualityScores.overall >= 0.98,
      agent_performance: agentPerformance,
      created_at: new Date().toISOString()
    }
  });

  // 2. Update conductor patterns
  await updateConductorPatterns(projectContext, violations);

  // 3. Update vault benchmarks
  await updateVaultBenchmarks();
}
```

---

## Fallback Mode (No aimem)

When aimem is unavailable, use file-based storage.

### File Structure

```
.turkeycode/learning/
├── agent_patterns/
│   ├── backend.yaml
│   ├── frontend.yaml
│   ├── designer.yaml
│   └── ...
├── conductor_patterns.yaml
├── projects/
│   ├── a1b2c3d4.yaml
│   └── ...
└── vault/
    ├── benchmarks.yaml
    └── failure_patterns.yaml
```

### Fallback Detection

```javascript
async function learningStore(key, value) {
  if (await aimemAvailable()) {
    await aimem_store({ key, value });
  } else {
    await fallbackStore(key, value);
  }
}

async function learningQuery(query, options = {}) {
  if (await aimemAvailable()) {
    return await aimem_query(query, options);
  } else {
    return await fallbackQuery(query, options);
  }
}
```

### Fallback Warning

When running without aimem, PM outputs:

```
Running in fallback mode (aimem unavailable)

- Agent coordination: File-based
- Cross-project learning: Limited to current project
- Pattern reuse: Disabled

Build will complete but without full learning features.
```

---

## Best Practices

### For All Agents

1. **Query before building** - Always check learning for relevant patterns
2. **Record meaningful patterns** - Don't record trivial or obvious learnings
3. **Be specific** - "Use includes() for User.posts" not "avoid N+1"
4. **Include context** - What project/feature was this for?

### For Conductor

1. **Update benchmarks after every build** - Keep percentiles current
2. **Record all violations** - Even minor ones help future predictions
3. **Track per-agent performance** - Identify struggling agents
4. **Prune false memories** - Run cleanup periodically

### Pattern Quality

**Good Patterns:**
- Specific, actionable technique
- Clear outcome description
- Reproducible across projects

**Bad Patterns (likely false memories):**
- "This worked" (too vague)
- "File was missing" (programming error, not pattern)
- One-time workarounds

---

## Success Criteria

The learning system is working when:

- [ ] Agents query learning before starting work
- [ ] Agents record patterns after completing work
- [ ] Confidence scoring filters low-quality patterns
- [ ] False memories are detected and marked
- [ ] Conductor records project outcomes
- [ ] Similar project insights improve predictions
- [ ] Benchmarks reflect actual quality distribution
- [ ] Fallback mode works when aimem unavailable
