# QA Scoring Guide

**Used by:** QA Agent during Review Wave
**Purpose:** Evaluate build quality before Conductor final review
**Relationship:** This is the QA agent's rubric. Conductor uses a different rubric (see CONDUCTOR-AGENT.md) for final release decisions.

---

Score every build against this 100-point rubric. Minimum passing score: 98/100.

## Scoring Rubric

### Functionality (40 points)

| Score | Criteria |
|-------|----------|
| 40 | All features work perfectly, edge cases handled, error states graceful |
| 35 | All features work, most edge cases handled |
| 30 | Core features work, some edge cases missing |
| 25 | Core features work with bugs |
| 20 | Some features broken |
| <20 | Major functionality missing |

**Check:**
- Every specified feature implemented
- Input validation on all user inputs
- Error messages are helpful
- Edge cases don't crash the app
- Loading states exist where needed

### Code Quality (25 points)

| Score | Criteria |
|-------|----------|
| 25 | Clean, consistent, follows conventions, well-organized |
| 20 | Good quality, minor inconsistencies |
| 15 | Acceptable, some code smells |
| 10 | Messy but functional |
| <10 | Spaghetti code |

**Check:**
- Consistent naming conventions
- No god classes (>200 lines)
- No god methods (>30 lines)
- DRY - no duplicate code blocks
- Single responsibility principle
- Proper separation of concerns

### Security (15 points)

| Score | Criteria |
|-------|----------|
| 15 | No vulnerabilities found |
| 12 | Low severity issues only |
| 8 | Medium severity issues |
| 4 | High severity issues |
| 0 | Critical vulnerabilities |

**Check:**
- No SQL injection vectors
- No XSS vulnerabilities
- No hardcoded secrets
- Auth/authz properly implemented
- Input sanitization
- CSRF protection (if applicable)

### Testing (10 points)

| Score | Criteria |
|-------|----------|
| 10 | Comprehensive tests, all pass, good coverage |
| 8 | Core paths tested, all pass |
| 6 | Some tests exist, all pass |
| 4 | Tests exist but some fail |
| 2 | Minimal tests |
| 0 | No tests |

**Check:**
- Unit tests for business logic
- Integration tests for API endpoints
- Tests actually run and pass
- Critical paths covered

### Documentation (10 points)

| Score | Criteria |
|-------|----------|
| 10 | Full README, API docs, inline comments where needed |
| 8 | Good README, some API docs |
| 6 | Basic README with setup instructions |
| 4 | Minimal README |
| 2 | No README but some comments |
| 0 | No documentation |

**Check:**
- README with setup instructions
- Environment variables documented
- API endpoints documented
- Complex logic has comments
- Architecture decisions noted

## Scoring Process

1. Build completes
2. Run through each category
3. Assign points with justification
4. Total the score
5. If < 98, identify gaps
6. Fix gaps
7. Re-score
8. Repeat until ≥ 98

## Example Scoring

```
ADHD Hyperfocus Tracker - QA Score

Functionality: 39/40
  ✓ Heartbeat tracking works
  ✓ Session detection works
  ✓ Dashboard displays data
  ✓ Analytics charts render
  - Minor: No loading spinner on dashboard

Code Quality: 24/25
  ✓ Consistent naming
  ✓ Clean separation (models/controllers/services)
  ✓ No duplicate code
  - Minor: One method slightly long (35 lines)

Security: 15/15
  ✓ No SQL injection (using parameterized queries)
  ✓ No XSS (Rails escapes by default)
  ✓ No hardcoded secrets
  ✓ API validates input

Testing: 8/10
  ✓ Model tests exist
  ✓ Controller tests exist
  - Missing: Extension unit tests

Documentation: 12/10 (bonus for thoroughness)
  ✓ Full README with setup
  ✓ API documented
  ✓ Architecture explained
  ✓ VS Code extension usage documented

TOTAL: 98/100 ✓ PASS
```

## Automatic Fixes

When score < 98, address in this order:

1. Security issues (any score < 15)
2. Functionality gaps (any feature not working)
3. Code quality issues (if < 20)
4. Testing gaps (if < 6)
5. Documentation gaps (if < 6)

Don't ship with security issues. Ever.

---

## See Also

- **CONDUCTOR-AGENT.md** - Final release scoring (different rubric, focuses on integration and semantic compliance)
- **RUNTIME-VERIFICATION.md** - Proves the app actually runs (required before Conductor scoring)
