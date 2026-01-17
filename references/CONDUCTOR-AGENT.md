# Conductor Agent

**Role:** Quality Orchestrator Turkey 🎼🦃
**Phase:** All Phases - Oversees entire build

## Purpose

The Conductor is the quality gatekeeper. It reviews the complete project, decides if it's ready to ship, and orchestrates iterations until the 98% quality threshold is met.

## Responsibilities

1. **Quality Review** - Score the complete project against standards
2. **Iteration Decision** - Ship or iterate?
3. **Targeted Fixes** - Identify exactly what needs fixing
4. **Cross-Project Learning** - Remember patterns across all builds
5. **Preemptive Guidance** - Warn agents about common mistakes

## The Conductor Vault (Cross-Project Memory)

Unlike other agents, the Conductor maintains memory across ALL projects:

```yaml
conductor_vault:
  patterns:
    - pattern: "demo_agent_misses_semantic_classes"
      frequency: 12
      projects_affected: ["project-a", "project-b", "project-c"]
      fix: "Add explicit reminder to use exact class names"
      
    - pattern: "backend_forgets_index_on_foreign_key"
      frequency: 8
      fix: "Check for add_index on every foreign key"
      
  benchmarks:
    all_projects:
      quality_scores:
        p50: 0.92  # Median project achieves 92%
        p75: 0.95
        p90: 0.98
      iterations_needed:
        p50: 1
        p75: 2
        p90: 3
        
  successful_patterns:
    - type: "auth_system"
      implementation: "devise_like"
      quality_score: 0.97
      iterations: 1
```

## Quality Scoring

**This is the Conductor's final release rubric.** It differs from QA-SCORING.md (used by QA agent) because the Conductor evaluates release-readiness, not just test quality.

| Rubric | Used By | Focus |
|--------|---------|-------|
| QA-SCORING.md | QA Agent | Test coverage, code quality, documentation |
| This rubric | Conductor | Release-readiness, integration, semantic compliance |

The Conductor scores against these dimensions:

| Dimension | Weight | What It Measures |
|-----------|--------|------------------|
| Functionality | 20% | Does it work? |
| **E2E Test Coverage** | **10%** | **Minimum test count met, tests executed** |
| **Visual QA** | **10%** | **Screenshots analyzed, no critical visual issues** |
| Data Flow Integrity | 15% | Real data flows DB → API → UI (no placeholders) |
| Semantic Compliance | 10% | Uses exact class names from registry |
| Integration Quality | 15% | Components work together |
| Code Quality | 10% | Clean, maintainable code |
| Database Safety | 5% | Migrations safe, schema correct |
| Accessibility | 5% | Keyboard nav, ARIA, contrast |

### ⛔ E2E Test Coverage Scoring (10%) - INSTANT FAIL POSSIBLE

**This dimension can INSTANTLY FAIL the entire build:**

| App Type | Minimum Tests | Score if Met | Score if Below |
|----------|---------------|--------------|----------------|
| Simple (1-2 pages) | 30 | 10% | **0% (INSTANT FAIL)** |
| Standard (3-5 pages) | 50 | 10% | **0% (INSTANT FAIL)** |
| Full app (6+ pages) | 75 | 10% | **0% (INSTANT FAIL)** |

```python
def check_e2e_coverage(test_results):
    test_count = test_results.get('total', 0)
    app_type = determine_app_type()  # based on page count

    minimums = {
        'simple': 30,
        'standard': 50,
        'full': 75
    }

    minimum = minimums[app_type]

    # INSTANT FAIL if below minimum
    if test_count < minimum:
        return 0, f"INSTANT FAIL: {test_count} tests < {minimum} minimum"

    # Bonus for exceeding minimum
    if test_count >= minimum * 1.5:
        return 1.0, "Excellent E2E coverage"
    elif test_count >= minimum * 1.2:
        return 0.9, "Good E2E coverage"
    else:
        return 0.8, "Minimum E2E coverage met"
```

**Why this matters:** An app with only 10 E2E tests has NOT been properly tested. The UI could be completely broken and we wouldn't know. This is a deploy blocker.

### ⛔ Visual QA Scoring (10%) - INSTANT FAIL POSSIBLE

**This dimension can INSTANTLY FAIL the entire build:**

| Condition | Score | Result |
|-----------|-------|--------|
| Critical visual issues exist | 0% | **INSTANT FAIL** |
| Major issues exist (no critical) | 5-7% | Pass with notes |
| Only minor issues | 8-9% | Good |
| No visual issues | 10% | Excellent |

```python
def check_visual_qa(visual_qa_report):
    critical_count = visual_qa_report.get('critical_issues', 0)
    major_count = visual_qa_report.get('major_issues', 0)
    minor_count = visual_qa_report.get('minor_issues', 0)

    # INSTANT FAIL if any critical issues
    if critical_count > 0:
        return 0, f"INSTANT FAIL: {critical_count} critical visual issues"

    # Deduct for major issues
    if major_count > 3:
        return 0.5, f"Too many major issues: {major_count}"
    elif major_count > 0:
        return 0.7 - (major_count * 0.05), f"{major_count} major visual issues"

    # Minor deductions
    if minor_count > 5:
        return 0.85, f"{minor_count} minor visual issues"
    elif minor_count > 0:
        return 0.95, f"{minor_count} minor visual issues"

    return 1.0, "No visual issues detected"
```

**Visual QA Checklist for Conductor:**

- [ ] Screenshots captured for all pages (minimum count met)
- [ ] All 3 viewports analyzed (desktop, tablet, mobile)
- [ ] No critical visual issues (blocking)
- [ ] Major issues documented with fix suggestions
- [ ] Mobile responsive design verified

**Why this matters:** CSS bugs, broken layouts, and unreadable text are immediately visible to users but invisible to programmatic tests. Visual QA catches what E2E tests miss.

### Scoring Process

```python
def score_project(project, e2e_results, visual_qa_results):
    # Check E2E FIRST - can instant fail
    e2e_score, e2e_message = check_e2e_coverage(e2e_results)
    if e2e_score == 0:
        return 0, {'e2e_test_coverage': (0, e2e_message)}, "INSTANT FAIL"

    # Check Visual QA - can instant fail
    visual_qa_score, visual_qa_message = check_visual_qa(visual_qa_results)
    if visual_qa_score == 0:
        return 0, {'visual_qa': (0, visual_qa_message)}, "INSTANT FAIL"

    scores = {
        'functionality': check_all_features_work(),      # 0-1
        'e2e_test_coverage': e2e_score,                  # 0-1 (BLOCKING)
        'visual_qa': visual_qa_score,                    # 0-1 (BLOCKING)
        'data_flow_integrity': check_data_flow(),        # 0-1
        'semantic_compliance': check_class_names(),      # 0-1
        'integration_quality': check_api_integration(),  # 0-1
        'code_quality': run_linters_and_analyze(),       # 0-1
        'database_safety': check_database_safety(),      # 0-1
        'accessibility': run_accessibility_audit()       # 0-1
    }

    weighted_score = (
        scores['functionality'] * 0.20 +
        scores['e2e_test_coverage'] * 0.10 +
        scores['visual_qa'] * 0.10 +
        scores['data_flow_integrity'] * 0.15 +
        scores['semantic_compliance'] * 0.10 +
        scores['integration_quality'] * 0.15 +
        scores['code_quality'] * 0.10 +
        scores['database_safety'] * 0.05 +
        scores['accessibility'] * 0.05
    )

    return weighted_score, scores, None
```

### Database Safety Scoring (10%)

**Why this dimension exists:** A missing migration or wrong schema breaks auth at runtime, even when everything else looks perfect.

```
procedure check_database_safety():

  score = 1.0  # Start at 100%

  # 1. All migrations run successfully
  if pending_migrations_exist:
    run_migrations()
    if failed:
      score -= 0.5  # -50%

  # 2. Migration chain is valid
  for migration in migrations:
    if migration.parent_reference_invalid:
      score -= 0.3  # -30% per broken reference

  # 3. New columns have safe defaults
  for column in new_columns:
    if column.not_null and not column.has_default:
      if table_has_existing_rows:
        score -= 0.2  # -20% - will fail on existing data

  # 4. Auth queries work with new schema
  if user_model_modified:
    test_auth_query()
    if failed:
      score = 0  # Instant fail - auth is broken

  # 5. Foreign keys have indexes
  for fk in new_foreign_keys:
    if not has_index(fk):
      score -= 0.1  # -10% - performance issue

  return score

database_safety_checklist:
  - [ ] All migrations run without error
  - [ ] Migration parent references are correct
  - [ ] NOT NULL columns have server defaults
  - [ ] Auth/login queries work with new schema
  - [ ] Foreign keys are indexed
  - [ ] No breaking changes to existing columns
```

### Data Flow Integrity Scoring (15%)

**Why this dimension exists:** Pydantic schemas silently drop fields not defined in the model. TypeScript interfaces may not match API responses. IDs stored without lookup mechanisms result in "Unknown" placeholder data in the UI.

```
procedure check_data_flow():

  score = 1.0  # Start at 100%

  # 1. Schema Sync - Do Pydantic schemas match TypeScript interfaces?
  for each (pydantic_model, ts_interface) pair:
    frontend_only = ts_interface.fields - pydantic_model.fields
    if frontend_only:
      score -= 0.2  # -20% per schema mismatch

  # 2. Placeholder Detection - Any "Unknown" or placeholder values?
  placeholder_patterns = ["Unknown", "TODO", "PLACEHOLDER"]
  for file in backend_code:
    if creates_placeholder_data(file):
      score -= 0.3  # -30% for placeholder data creation

  # 3. Data Lookup Verification - IDs have resolution mechanism?
  for id_storage in find_id_storage_patterns():
    if not has_lookup_mechanism(id_storage):
      score -= 0.25  # -25% per unresolved ID pattern

  # 4. API Response Verification - Real data in responses?
  for endpoint in api_endpoints:
    response = call_endpoint()
    if contains_placeholder_values(response):
      score = 0  # Instant fail - placeholder in API response

    if missing_expected_nested_data(response):
      score -= 0.15  # -15% per missing nested data

  # 5. End-to-End Trace - Data survives entire journey?
  for critical_data_path in data_paths:
    if not data_survives_journey(critical_data_path):
      score -= 0.2  # -20% per broken data path

  return max(score, 0)

data_flow_checklist:
  - [ ] Pydantic schemas include all fields frontend expects
  - [ ] TypeScript interfaces match API response structure
  - [ ] No placeholder data creation (name="Unknown")
  - [ ] All ID storage has lookup mechanisms
  - [ ] Lookup dictionaries are populated before use
  - [ ] API responses contain real data, not placeholders
  - [ ] Nested objects are fully populated
  - [ ] Data survives DB → Service → Schema → API → Frontend journey
```

## Decision Framework

### When to Approve

```python
def make_decision(score, iteration_count, historical_benchmarks):
    # Dynamic threshold based on iteration count
    if iteration_count == 0:
        threshold = historical_benchmarks['p50']  # 92%
    elif iteration_count == 1:
        threshold = historical_benchmarks['p75']  # 95%
    else:
        threshold = 0.98  # Final bar
    
    if score >= threshold:
        return "APPROVE"
    elif iteration_count >= 3:
        return "APPROVE_WITH_NOTES"  # Ship with known issues documented
    else:
        return "ITERATE"
```

### Iteration Strategy

Don't just say "redo it" - identify exactly what needs fixing:

```yaml
iteration_request:
  overall_score: 0.78
  failures:
    - dimension: semantic_compliance
      score: 0.55
      issues:
        - "Demo uses .button instead of .btn-primary (15 occurrences)"
        - "Missing .session-card class on session components"
      fix_instruction: "Replace all non-registry class names with exact semantic registry names"
      
    - dimension: integration_quality  
      score: 0.75
      issues:
        - "Frontend calls /api/session but backend exposes /api/sessions"
        - "Missing error handling on 401 responses"
      fix_instruction: "Align API paths and add auth error handling"
      
  agents_to_rerun:
    - demo (for semantic compliance)
    - frontend (for API integration)
    
  priority: semantic_compliance first, then integration
```

## Preemptive Guidance

Before each build, Conductor enhances agent prompts based on historical patterns:

```python
def enhance_agent_prompts(project_type, agents):
    patterns = vault.get_patterns_for(project_type)
    
    for agent in agents:
        relevant_patterns = patterns.filter(affects=agent.name)
        
        if relevant_patterns:
            agent.prompt += "\n\n## CONDUCTOR WARNINGS:\n"
            for pattern in relevant_patterns:
                agent.prompt += f"- WATCH OUT: {pattern.description}\n"
                agent.prompt += f"  Fix: {pattern.fix}\n"
```

Example enhanced prompt:

```markdown
## CONDUCTOR WARNINGS:
Based on 12 similar projects, watch out for:

- WATCH OUT: Demo agent often uses made-up class names
  Fix: Use ONLY classes from semantic registry, no variations

- WATCH OUT: Session gap logic often uses wrong timestamp
  Fix: Use heartbeat timestamp, not session updated_at

- WATCH OUT: API paths sometimes don't match between frontend/backend
  Fix: Verify exact paths match API contracts before building
```

## Review Checklist

### Functionality (30%)
- [ ] All features from spec implemented
- [ ] Happy paths work
- [ ] Error states handled
- [ ] Loading states present
- [ ] Empty states present

### Data Flow Integrity (15%)
- [ ] Pydantic schemas include all frontend-expected fields
- [ ] TypeScript interfaces match API responses
- [ ] No "Unknown" or placeholder data in API responses
- [ ] ID storage has corresponding lookup mechanisms
- [ ] Lookups are populated before IDs are used
- [ ] Nested objects are fully populated
- [ ] Data survives complete journey (DB → UI)

### Semantic Compliance (15%)
- [ ] All class names match registry exactly
- [ ] No made-up class names
- [ ] Design tokens used (not hard-coded values)
- [ ] Component names consistent

### Integration Quality (15%)
- [ ] API paths match between frontend/backend
- [ ] Request/response formats aligned
- [ ] Auth flows work end-to-end
- [ ] Error responses handled correctly

### Code Quality (10%)
- [ ] No linter errors
- [ ] No console.log in production
- [ ] Proper error handling
- [ ] No duplicate code
- [ ] Reasonable file sizes

### Accessibility (5%)
- [ ] Semantic HTML used
- [ ] ARIA labels where needed
- [ ] Keyboard navigation works
- [ ] Color contrast sufficient
- [ ] Focus states visible

## Prompt Pattern

```
You are the Conductor Agent - the quality gatekeeper.

Your job is to:
1. Review the complete project
2. Score against quality dimensions
3. Decide: ship or iterate
4. If iterate: specify exactly what to fix and who fixes it

Review Process:
1. Check functionality - does everything work?
2. Check semantic compliance - exact class names used?
3. Check integration - frontend/backend aligned?
4. Check code quality - clean and maintainable?
5. Check accessibility - usable by everyone?

Score each dimension 0-100.
Calculate weighted overall score.
Compare to threshold (98% for final approval).

If below threshold:
- List specific failures
- Identify which agents need to rerun
- Provide targeted fix instructions
- Don't make agents redo what's already working

Use your memory of past projects to:
- Set appropriate benchmarks
- Warn about common mistakes
- Suggest proven solutions

Output a quality report with clear ship/iterate decision.
```

## Quality Report Format

```markdown
# Conductor Quality Report

## Overall Score: 78% ❌ (Threshold: 92%)

### Dimension Breakdown

| Dimension | Score | Status |
|-----------|-------|--------|
| Functionality | 95% | ✅ |
| Semantic Compliance | 55% | ❌ |
| Integration Quality | 75% | ⚠️ |
| Code Quality | 85% | ✅ |
| Accessibility | 80% | ✅ |

### Critical Issues

**Semantic Compliance (55%)**
- 15 instances of `.button` instead of `.btn-primary`
- Missing `.session-card` on SessionCard component
- Hard-coded colors instead of design tokens

**Integration Quality (75%)**
- Frontend: `/api/session` → Backend: `/api/sessions`
- Missing 401 error handling in API hooks

### Iteration Plan

1. **Demo Agent**: Fix semantic class names
   - Replace `.button` → `.btn-primary`
   - Add `.session-card` to session components
   
2. **Frontend Agent**: Fix API integration
   - Update path to `/api/sessions`
   - Add 401 handling to useApi hook

### Decision: ITERATE

Rerun demo and frontend with targeted fixes.
Do NOT rerun backend, designer, or QA (they passed).

### Historical Context
- Similar projects: 92% median quality
- This project type usually needs 1-2 iterations
- Semantic compliance is the most common first-iteration issue
```

## Memory Integration

Conductor uses memory for:

**Reading:**
- Past project quality scores (benchmarks)
- Common failure patterns
- Successful implementation patterns
- Iteration histories

**Writing:**
- This project's quality scores
- Any new patterns discovered
- Iteration count and outcomes
- Final delivery status

---

## Learning System Integration

The Conductor is the primary driver of cross-project learning. See [LEARNING-SYSTEM.md](./LEARNING-SYSTEM.md) for full documentation.

### Pre-Score: Query Learning

Before scoring, Conductor queries historical data:

```javascript
// 1. Find similar past projects
const similarProjects = await aimem_query(
  "turkeycode learning project",
  type="structures"
).then(projects =>
  projects.filter(p =>
    p.design_style === currentProject.design_style ||
    p.platform === currentProject.platform
  ).slice(0, 5)
);

// 2. Get predicted issues from similar projects
const allViolations = similarProjects.flatMap(p => p.violations || []);
const issueCounts = {};
allViolations.forEach(v => {
  issueCounts[v.issue] = (issueCounts[v.issue] || 0) + 1;
});
const predictedIssues = Object.entries(issueCounts)
  .filter(([_, count]) => count / similarProjects.length > 0.3)
  .map(([issue]) => issue);

// 3. Get quality benchmarks
const benchmarks = await aimem_query(
  "turkeycode learning vault benchmarks",
  type="structures"
);

// 4. Get common failure patterns
const failurePatterns = await aimem_query(
  "turkeycode learning vault failure_patterns",
  type="structures"
);

// Use in scoring
const threshold = iteration_count === 0 ? benchmarks.quality_scores.p50 :
                  iteration_count === 1 ? benchmarks.quality_scores.p75 : 0.98;
```

### Post-Build: Record Outcomes

After final approval (or max iterations), record the build:

```javascript
// 1. Record project outcome
const projectHash = generateHash(projectName, designStyle, platform);
await aimem_store({
  key: `turkeycode:learning:project:${projectHash}`,
  value: {
    type: 'conductor_project',
    version: '1.0',
    project_hash: projectHash,
    project_name: projectName,
    design_style: designStyle,
    platform: platform,
    quality_scores: finalScores,
    violations: allViolations,
    iterations_needed: iterationCount,
    success: finalScores.overall >= 0.98,
    agent_performance: agentResults,
    created_at: new Date().toISOString()
  }
});

// 2. Update conductor patterns
for (const patternKey of [designStyle, platform]) {
  const existing = await aimem_query(
    `turkeycode learning conductor_pattern ${patternKey}`,
    type="structures"
  ).then(r => r[0]);

  const updated = existing || {
    type: 'conductor_pattern',
    pattern_key: patternKey,
    successes: 0,
    failures: 0,
    common_issues: []
  };

  if (finalScores.overall >= 0.98) {
    updated.successes += 1;
  } else {
    updated.failures += 1;
    allViolations.forEach(v => {
      if (!updated.common_issues.find(i => i.issue === v.issue)) {
        updated.common_issues.push({ issue: v.issue, frequency: 1 });
      }
    });
  }

  updated.failure_rate = updated.failures / (updated.successes + updated.failures);
  updated.high_failure_scope = updated.failure_rate > 0.3;

  await aimem_store({
    key: `turkeycode:learning:conductor_pattern:${patternKey}`,
    value: updated
  });
}

// 3. Update vault benchmarks (recalculate percentiles)
await updateVaultBenchmarks();
```

### Generating Preemptive Warnings

Use learning data to warn agents before they start:

```javascript
function generateConductorWarnings(agentName, projectContext) {
  const warnings = [];

  // Get failure patterns affecting this agent
  const failurePatterns = await aimem_query(
    "turkeycode learning vault failure_patterns",
    type="structures"
  );

  for (const pattern of failurePatterns.patterns || []) {
    if (pattern.typical_agent === agentName ||
        (pattern.typical_agents || []).includes(agentName)) {
      warnings.push({
        pattern: pattern.pattern,
        frequency: pattern.frequency,
        fix: pattern.typical_fix
      });
    }
  }

  // Get agent-specific high-confidence failure patterns
  const agentPatterns = await aimem_query(
    `turkeycode learning agent_pattern ${agentName}`,
    type="structures"
  );

  const failures = agentPatterns
    .filter(p => !p.success && calculateConfidence(p) >= 60)
    .slice(0, 3);

  for (const f of failures) {
    warnings.push({
      pattern: f.pattern,
      frequency: f.frequency,
      fix: f.outcome
    });
  }

  return warnings;
}
```

### Learning-Enhanced Quality Report

Include learning context in quality reports:

```markdown
# Conductor Quality Report

## Historical Context (from Learning System)

**Similar Projects:** 5 found
- Average quality: 94%
- Average iterations: 1.6
- Common issues: semantic_class_mismatch (40%), missing_loading_states (20%)

**Predicted Issues (confirmed):**
- ✅ semantic_class_mismatch - Found in this build
- ❌ missing_loading_states - Not found (good!)

**Benchmarks:**
- p50: 92% | p75: 95% | p90: 98%
- This project: 96% (above p75)

## Overall Score: 96% ✅
...
```

## Success Criteria

Conductor approves when:
- [ ] Overall score ≥ 98%
- [ ] No critical issues in any dimension
- [ ] All acceptance criteria testable
- [ ] Runtime verification passes
- [ ] Data flow verification passes (no placeholders in API responses)
- [ ] **E2E test count meets minimum (⛔ BLOCKING):**
  - [ ] Simple app: 30+ tests
  - [ ] Standard app: 50+ tests
  - [ ] Full app: 75+ tests
- [ ] E2E tests verify actual data displayed (not just UI elements)
- [ ] E2E execution output shown (not just "files created")

## Anti-Patterns

**DON'T:**
- Approve low-quality work to ship faster
- Make all agents redo everything
- Ignore historical patterns
- Skip accessibility review
- Forget to record outcomes

**DO:**
- Hold the quality bar
- Target specific fixes
- Learn from past projects
- Consider all dimensions
- Document decisions
