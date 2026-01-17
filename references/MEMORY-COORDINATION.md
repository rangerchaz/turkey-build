# Memory & Coordination System

**The Brain That Connects All Agents** 🧠

## Overview

Without shared memory, agents are isolated and make inconsistent decisions. With memory, they're a coordinated team sharing one brain.

## Memory Layers

### 1. Team Memory (Current Project)

Shared state for the current build:

```yaml
team_memory:
  # Published by Designer
  semantic_registry:
    buttons:
      btn-primary: ".btn-primary"
      btn-secondary: ".btn-secondary"
    cards:
      session-card: ".session-card"
    layouts:
      dashboard-layout: ".dashboard-layout"
      
  design_tokens:
    colors:
      primary: "#7c3aed"
      background: "#0a0a0a"
    spacing:
      base: "1rem"
      
  # Published by Backend
  api_contracts:
    heartbeats:
      create:
        method: POST
        path: /api/heartbeats
        body: {project_name, branch, file_type}
        response: {session_id, heartbeat_id}
        
    sessions:
      list:
        method: GET
        path: /api/sessions
        response: [{id, project_name, duration_minutes}]
        
  # Published by PM
  build_status:
    current_phase: 3
    agents_complete: [discovery, designer, backend]
    agents_running: [frontend]
    agents_pending: [qa, conductor]
```

### 2. Agent Memory (Individual Learning)

Each agent remembers its patterns:

```yaml
agent_memory:
  designer:
    successful_patterns:
      - type: "dark_theme"
        tokens: {...}
        quality_score: 0.97
        
    mistakes_made:
      - issue: "forgot focus states"
        fix: "always include :focus in components"
        
  backend:
    successful_patterns:
      - type: "session_detection"
        implementation: {...}
        quality_score: 0.95
        
    common_issues:
      - issue: "N+1 queries on dashboard"
        fix: "use includes() for nested data"
```

### 3. Conductor Vault (Cross-Project)

Memory across ALL projects:

```yaml
conductor_vault:
  quality_benchmarks:
    all_projects:
      p50: 0.92
      p75: 0.95
      p90: 0.98

  failure_patterns:
    - pattern: "semantic_class_mismatch"
      frequency: 42
      typical_fix: "Use exact registry names"

    - pattern: "api_path_mismatch"
      frequency: 23
      typical_fix: "Verify contracts before building"

  successful_implementations:
    - feature: "auth_system"
      approach: "jwt_with_refresh"
      quality_score: 0.97
      reusable_code: {...}
```

### 4. Learning Layer (Intelligence)

The Learning System adds intelligence to memory. See [LEARNING-SYSTEM.md](./LEARNING-SYSTEM.md) for full documentation.

```
┌─────────────────────────────────────────────────────────────────┐
│                     LEARNING LAYER                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐    ┌─────────────────┐    ┌──────────────┐ │
│  │ Agent Patterns  │    │ Conductor       │    │ Vault        │ │
│  │ (per agent)     │    │ Patterns        │    │ Aggregations │ │
│  │                 │    │ (per type)      │    │ (global)     │ │
│  │ • Success/fail  │    │ • Success rate  │    │ • Benchmarks │ │
│  │ • Confidence    │    │ • Common issues │    │ • Failure    │ │
│  │ • Frequency     │    │ • Agents affect │    │   patterns   │ │
│  └─────────────────┘    └─────────────────┘    └──────────────┘ │
│           │                     │                     │         │
│           └─────────────────────┼─────────────────────┘         │
│                                 │                               │
│                    ┌────────────┴────────────┐                  │
│                    │   Project History       │                  │
│                    │   (outcomes, scores,    │                  │
│                    │    iterations, agents)  │                  │
│                    └─────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

**Key Capabilities:**

| Capability | How It Works |
|------------|--------------|
| **Confidence Scoring** | Patterns rated 0-100 based on frequency, age, error indicators |
| **False Memory Detection** | Contradictory patterns marked and demoted |
| **Cross-Project Learning** | Similar projects inform predictions |
| **Quality Benchmarks** | p50/p75/p90 from actual project outcomes |

**aimem Key Structure:**

```
turkeycode:learning:
├── agent_pattern:{agent}:{id}     # Per-agent learned patterns
├── conductor_pattern:{key}        # Success/failure by pattern type
├── project:{hash}                 # Project outcome records
├── agent_performance:{agent}:{hash}  # Per-project agent performance
└── vault:
    ├── benchmarks                 # Quality score percentiles
    └── failure_patterns           # Aggregated failure patterns
```

**Learning Flow:**

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Pre-Work    │────▶│   Execution  │────▶│  Post-Work   │
│              │     │              │     │              │
│ • Query      │     │ • Use high   │     │ • Record     │
│   patterns   │     │   confidence │     │   pattern    │
│ • Get        │     │   patterns   │     │ • Update     │
│   warnings   │     │ • Avoid      │     │   frequency  │
│ • Check      │     │   false      │     │ • Check for  │
│   insights   │     │   memories   │     │   contradictions │
└──────────────┘     └──────────────┘     └──────────────┘
```

## Query Patterns

### Before Building

Every agent should query memory before starting:

```
# Designer querying for patterns
"What design tokens worked for similar apps?"
"What accessibility issues were found in past dark themes?"

# Backend querying for patterns
"What's a good session detection implementation?"
"What API patterns scored highest?"

# Frontend querying for context
"What's in the semantic registry?" (REQUIRED)
"What are the API contracts?" (REQUIRED)
```

### During Building

Query to stay consistent:

```
# Frontend building a component
"What's the exact class name for primary buttons?"
→ team_memory.semantic_registry.buttons.btn-primary

# Frontend integrating API
"What's the heartbeat endpoint format?"
→ team_memory.api_contracts.heartbeats.create
```

### After Building

Record outcomes for future learning:

```
# Conductor recording
"This project scored 94% with 1 iteration"
"Semantic compliance was the main issue"
"Fixed by stricter class name validation"
```

## Integration with aimem

When using aimem MCP server:

### Query Functions

```javascript
// Search for patterns
aimem_query("session detection service", type="structures")
// Returns: matching code structures

// Check if something exists
aimem_verify("SessionDetectionService", type="structure")
// Returns: true/false with location

// Search conversations for decisions
aimem_conversations("why did we use JWT?")
// Returns: relevant past discussions
```

### Store Functions

```javascript
// Store important decisions
aimem_store({
  type: "decision",
  topic: "auth_approach",
  choice: "JWT with refresh tokens",
  reasoning: "Stateless, works with mobile extension"
})

// Store successful patterns
aimem_store({
  type: "pattern",
  name: "heartbeat_service",
  code: "...",
  quality_score: 0.97
})
```

## Coordination Flows

### Designer → Frontend Handoff

```yaml
# Designer publishes
publish:
  to: team_memory.semantic_registry
  content:
    buttons:
      btn-primary: ".btn-primary"
      btn-secondary: ".btn-secondary"

# Frontend reads
read:
  from: team_memory.semantic_registry
  
# Frontend MUST use exact names
component: |
  <button className="btn-primary">  // ✅ From registry
  <button className="primary-btn">  // ❌ Made up
```

### Backend → Frontend Handoff

```yaml
# Backend publishes
publish:
  to: team_memory.api_contracts
  content:
    sessions:
      path: "/api/sessions"
      method: "GET"

# Frontend reads
read:
  from: team_memory.api_contracts
  
# Frontend MUST match exactly
api_call: |
  fetch('/api/sessions')   // ✅ From contract
  fetch('/api/session')    // ❌ Wrong path
```

### Conductor → PM Iteration

```yaml
# Conductor publishes iteration request
publish:
  to: team_memory.iteration_request
  content:
    agents: [frontend, demo]
    fixes:
      - type: semantic_compliance
        issues: ["wrong class names"]
      - type: api_integration
        issues: ["wrong endpoint path"]

# PM reads and coordinates
read:
  from: team_memory.iteration_request
  
# PM creates targeted fix prompts
action: |
  Only rerun frontend and demo
  Only fix specified issues
  Don't rebuild everything
```

## Memory-Aware Prompts

### Designer Prompt Addition

```markdown
## Memory Integration

Before designing:
- Query: "successful design patterns for [project_type]"
- Query: "common accessibility issues to avoid"

After designing:
- Publish: semantic_registry to team_memory
- Publish: design_tokens to team_memory
- Store: successful patterns to agent_memory
```

### Frontend Prompt Addition

```markdown
## Memory Integration (REQUIRED)

Before building:
- READ: team_memory.semantic_registry (use exact class names)
- READ: team_memory.api_contracts (use exact paths)
- Query: "frontend patterns that scored well"

During building:
- Use ONLY class names from semantic_registry
- Use ONLY API paths from api_contracts
- Query memory when unsure about naming
```

### Conductor Prompt Addition

```markdown
## Memory Integration

For review:
- Read: conductor_vault.quality_benchmarks (set thresholds)
- Read: conductor_vault.failure_patterns (know what to check)

For decision:
- Compare: current score vs historical benchmarks
- Check: is this a known pattern?

After decision:
- Store: project outcome to conductor_vault
- Store: any new patterns discovered
- Update: benchmarks if needed
```

## Memory Schema

### Team Memory (per project)

```typescript
interface TeamMemory {
  project_id: string;
  
  semantic_registry: {
    buttons: Record<string, string>;
    cards: Record<string, string>;
    layouts: Record<string, string>;
    typography: Record<string, string>;
  };
  
  design_tokens: {
    colors: Record<string, string>;
    spacing: Record<string, string>;
    typography: Record<string, string>;
  };
  
  api_contracts: {
    [resource: string]: {
      [action: string]: {
        method: string;
        path: string;
        body?: Record<string, string>;
        response: Record<string, any>;
      };
    };
  };
  
  build_status: {
    current_phase: number;
    agents_complete: string[];
    agents_running: string[];
    agents_pending: string[];
  };
}
```

### Conductor Vault (global)

```typescript
interface ConductorVault {
  quality_benchmarks: {
    p50: number;
    p75: number;
    p90: number;
  };
  
  failure_patterns: Array<{
    pattern: string;
    frequency: number;
    typical_fix: string;
  }>;
  
  successful_implementations: Array<{
    feature: string;
    approach: string;
    quality_score: number;
    reusable_code?: string;
  }>;
  
  project_history: Array<{
    project_id: string;
    quality_score: number;
    iterations: number;
    issues_found: string[];
  }>;
}
```

## Anti-Patterns

**DON'T:**
- Build without reading team_memory
- Make up class names without checking registry
- Guess API paths without checking contracts
- Ignore historical patterns
- Forget to publish outputs

**DO:**
- Always query memory before building
- Use exact names from published registries
- Learn from past project outcomes
- Publish outputs for other agents
- Record patterns for future builds

## Success Criteria

Memory integration works when:
- [ ] All agents query before building
- [ ] Semantic registry is the single source of truth
- [ ] API contracts prevent mismatches
- [ ] Historical patterns inform decisions
- [ ] Outcomes are recorded for learning
