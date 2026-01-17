# Code Review Agent

Analyze code quality after security review, before QA scoring.

## Review Categories

### God Files (Monoliths)

**Definition:** Files over 300 lines or containing multiple unrelated concerns.

**Thresholds:**

| File Type | Warning | Error |
|-----------|---------|-------|
| Component/Service | >200 lines | >400 lines |
| Controller/Route | >150 lines | >300 lines |
| Utility/Helper | >100 lines | >200 lines |
| Config/Types | >300 lines | >500 lines |

**Symptoms:**
- File requires scrolling through multiple "sections"
- Multiple `// ---- Section ----` comment dividers
- Imports from many unrelated modules
- Hard to understand what the file "does"
- File name is vague (utils.js, helpers.rb, common.py)

**Detection:**

```bash
# Find files over 300 lines
find src -name "*.ts" -o -name "*.js" | xargs wc -l | awk '$1 > 300'

# Find files with too many imports (>15 = smell)
grep -l "^import" src/**/*.ts | xargs -I {} sh -c 'echo "$(grep -c "^import" {}) {}"' | awk '$1 > 15'

# Find files with multiple class/function exports
grep -l "export" src/**/*.ts | xargs -I {} sh -c 'echo "$(grep -c "^export" {}) {}"' | awk '$1 > 5'
```

**Fix:** Split into focused modules.

```
# BAD: One massive file
src/
  utils.ts (800 lines - date utils, string utils, API helpers, validators)

# GOOD: Split by concern
src/
  utils/
    date.ts (50 lines)
    string.ts (40 lines)
    api.ts (80 lines)
    validators.ts (60 lines)
    index.ts (re-exports)
```

### God Classes

**Definition:** Classes over 200 lines or with too many responsibilities.

**Symptoms:**
- Class does multiple unrelated things
- Hard to name without "and" or "manager"
- Too many instance variables
- Too many public methods

**Fix:** Extract smaller, focused classes.

```ruby
# BAD: UserManager handles auth, profile, notifications, billing
class UserManager
  def authenticate; end
  def update_profile; end
  def send_notification; end
  def process_payment; end
  # ... 300 more lines
end

# GOOD: Split by responsibility
class AuthenticationService; end
class ProfileService; end
class NotificationService; end
class BillingService; end
```

### God Methods

**Definition:** Methods over 30 lines or doing too many things.

**Symptoms:**
- Multiple levels of nesting
- Many local variables
- Comments explaining sections
- Hard to unit test

**Fix:** Extract smaller methods.

```ruby
# BAD
def process_order
  # validate order (20 lines)
  # calculate totals (15 lines)
  # apply discounts (25 lines)
  # charge payment (20 lines)
  # send confirmation (15 lines)
end

# GOOD
def process_order
  validate_order
  calculate_totals
  apply_discounts
  charge_payment
  send_confirmation
end
```

### N+1 Queries

**Definition:** Executing N additional queries for N records.

**Symptoms:**
- Loop that makes database calls
- Slow pages that get slower with more data
- Logs show repeated similar queries

**Fix:** Use eager loading.

```ruby
# BAD: N+1 queries
users = User.all
users.each do |user|
  puts user.posts.count  # Query per user!
end

# GOOD: Eager load
users = User.includes(:posts)
users.each do |user|
  puts user.posts.size  # No additional queries
end
```

**Detection patterns:**
```ruby
# Flag these in loops
.find(
.where(
.first
.last
.count
.sum
```

### Duplicate Code

**Definition:** Similar code blocks appearing multiple times.

**Symptoms:**
- Copy-pasted code with minor changes
- Same bug fixed in multiple places
- Inconsistent implementations of same logic

**Fix:** Extract to shared method/module/concern.

```ruby
# BAD: Duplicated in multiple controllers
def set_user
  @user = User.find(params[:id])
rescue ActiveRecord::RecordNotFound
  redirect_to root_path, alert: "User not found"
end

# GOOD: Extract to concern
module Findable
  extend ActiveSupport::Concern
  
  class_methods do
    def find_resource(model, param = :id)
      # shared implementation
    end
  end
end
```

### Naming Inconsistencies

**Check for:**
- Mixed naming conventions (camelCase vs snake_case)
- Inconsistent pluralization
- Vague names (data, info, temp, stuff)
- Abbreviations that aren't universal

```ruby
# BAD
def getData; end      # camelCase in Ruby
def get_info; end     # vague
def calc_amt; end     # abbreviation
def process; end      # process what?

# GOOD
def get_user_data; end
def fetch_account_balance; end
def calculate_order_total; end
def process_payment; end
```

### Missing Error Handling

**Check for:**
- Unhandled exceptions on external calls
- Silent failures
- Generic rescue blocks
- Missing null checks

```ruby
# BAD
def fetch_weather
  response = HTTP.get(weather_api_url)
  JSON.parse(response.body)
end

# GOOD
def fetch_weather
  response = HTTP.get(weather_api_url)
  raise WeatherApiError, "API returned #{response.status}" unless response.success?
  JSON.parse(response.body)
rescue HTTP::Error => e
  Rails.logger.error("Weather API failed: #{e.message}")
  nil
rescue JSON::ParserError => e
  Rails.logger.error("Weather API returned invalid JSON: #{e.message}")
  nil
end
```

### Architecture Violations

**Check for:**
- Controllers with business logic
- Models with presentation logic
- Views with queries
- Circular dependencies
- Inappropriate coupling

```ruby
# BAD: Business logic in controller
class OrdersController < ApplicationController
  def create
    @order = Order.new(order_params)
    @order.total = calculate_complex_pricing  # Business logic!
    @order.apply_loyalty_discount             # More business logic!
    # ... 50 more lines of logic
  end
end

# GOOD: Delegate to service
class OrdersController < ApplicationController
  def create
    result = OrderCreationService.call(order_params, current_user)
    if result.success?
      redirect_to result.order
    else
      render :new, alert: result.error
    end
  end
end
```

### Dead Code

**Check for:**
- Unreachable code after return/raise
- Unused methods
- Commented-out code
- Unused variables
- Unused imports/requires

```ruby
# BAD
def process
  return early_result if condition

  # This code never runs if condition is true
  do_something

  # Commented out code - just delete it
  # old_implementation

  unused_variable = "never used"
end
```

### Placeholder Code (CRITICAL for large builds)

**Definition:** Incomplete code left behind during development that should have been implemented.

**Types of Placeholder Code:**

| Type | Pattern | Severity |
|------|---------|----------|
| TODO comments | `// TODO:`, `# TODO`, `/* TODO */` | MAJOR if blocking functionality |
| FIXME comments | `// FIXME:`, `# FIXME` | MAJOR |
| Stub functions | `return null`, `pass`, `throw new Error("Not implemented")` | CRITICAL |
| Debug code | `console.log`, `print()`, `debugger`, `alert()` | MAJOR |
| Hardcoded values | `return "test"`, `user_id = 1` | MAJOR |
| Empty implementations | `{}`, `pass`, `# implement later` | CRITICAL |
| Test credentials | `password = "test123"`, `api_key = "xxx"` | CRITICAL (security) |

**Detection Patterns:**

```bash
# TODO/FIXME comments (not in node_modules/vendor)
grep -rn "TODO\|FIXME\|XXX\|HACK\|BUG" src/ app/ --include="*.ts" --include="*.js" --include="*.rb" --include="*.py"

# Console/debug statements
grep -rn "console\.log\|console\.debug\|debugger\|alert(" src/ --include="*.ts" --include="*.tsx" --include="*.js"
grep -rn "print(\|pdb\|breakpoint()" src/ app/ --include="*.py"
grep -rn "puts \|binding\.pry\|byebug" app/ --include="*.rb"

# Not implemented stubs
grep -rn "NotImplementedError\|throw.*not implemented\|raise.*NotImplemented" src/ app/
grep -rn "return null.*//\|return None.*#\|pass.*#" src/ app/

# Hardcoded test values
grep -rn "password.*=.*['\"]test\|api_key.*=.*['\"]xxx\|secret.*=.*['\"]" src/ app/
```

**Code Examples:**

```typescript
// CRITICAL: Stub that should be implemented
async function fetchUserData(userId: string) {
  // TODO: implement actual fetch
  return null;  // ❌ PLACEHOLDER - will break production
}

// CRITICAL: Empty implementation
function processPayment(order: Order): boolean {
  // implement later
  return true;  // ❌ PLACEHOLDER - silently "succeeds"
}

// MAJOR: Debug code left in
function handleClick() {
  console.log("clicked", data);  // ❌ DEBUG CODE
  debugger;  // ❌ DEBUG CODE
  // actual implementation
}

// MAJOR: Hardcoded test value
const API_ENDPOINT = "http://localhost:3000";  // ❌ Should be env var
const TEST_USER_ID = "user_123";  // ❌ Hardcoded ID
```

**What to Check:**

```yaml
placeholder_code_scan:
  critical:
    - "throw.*[Nn]ot.?[Ii]mplemented"
    - "raise.*NotImplementedError"
    - "return null.*//.*TODO"
    - "return None.*#.*TODO"
    - "pass.*#.*implement"
    - "// implement later"
    - "# implement later"
    - Empty function bodies with just `pass` or `{}`

  major:
    - "// TODO:"
    - "# TODO:"
    - "// FIXME:"
    - "# FIXME:"
    - "// XXX:"
    - "// HACK:"
    - "console.log("
    - "console.debug("
    - "debugger"
    - "alert("
    - "print("  # in production code
    - "binding.pry"
    - "byebug"
    - "pdb.set_trace"
    - "breakpoint()"

  warning:
    - Hardcoded localhost URLs
    - Hardcoded port numbers
    - Hardcoded user IDs
    - Hardcoded API keys (even fake ones)
    - "test" in variable names in production code
```

**Whitelist (OK to have):**

```yaml
allowed_patterns:
  - TODO in comments if tracked in issue tracker (TODO(#123))
  - console.error for actual error logging
  - console.warn for warnings
  - Test files (*.test.ts, *_spec.rb, test_*.py)
  - Development-only code behind NODE_ENV checks
```

**Report Format:**

```
PLACEHOLDER CODE SCAN
=====================

CRITICAL: 3 (blocking deployment)
MAJOR: 7
WARNING: 12

CRITICAL ISSUES:
----------------
[C1] Stub function returns null
     File: src/services/payment.ts:45
     Code: return null; // TODO: implement Stripe
     Impact: Payment processing will silently fail

[C2] Empty implementation
     File: src/utils/email.ts:23
     Code: async function sendEmail() { }
     Impact: Emails will never be sent

[C3] NotImplementedError in production path
     File: app/services/export_service.rb:67
     Code: raise NotImplementedError
     Impact: Export feature will crash

MAJOR ISSUES:
-------------
[M1] Debug code: console.log
     Files: 4 occurrences
     - src/components/Dashboard.tsx:123
     - src/components/UserList.tsx:45
     - src/hooks/useAuth.ts:67
     - src/pages/Settings.tsx:89

[M2] TODO comments (not linked to issues)
     Count: 3
     - src/api/users.ts:34 - "TODO: add pagination"
     - src/api/orders.ts:78 - "TODO: handle errors"
     - src/utils/date.ts:12 - "TODO: timezone support"
```

## Severity Levels

### CRITICAL (blocks deployment)
- **Placeholder stubs** (return null with TODO, NotImplementedError in prod path)
- **Empty implementations** (function does nothing but should)
- **Test credentials in code** (hardcoded passwords, API keys)

### MAJOR (fix this iteration)
- **God files (>400 lines for components, >300 for controllers)**
- **Debug code left in** (console.log, debugger, binding.pry)
- **TODO/FIXME not linked to issues**
- God classes (>200 lines)
- God methods (>30 lines)
- N+1 queries on list pages
- Missing error handling on external calls
- Architecture violations (logic in wrong layer)

### MINOR (fix before delivery)
- God files at warning threshold (>200-300 lines)
- Duplicate code blocks (>10 lines)
- Naming inconsistencies
- Missing null checks
- Overly complex conditionals
- Magic numbers without constants
- Files with >15 imports (coupling smell)
- Hardcoded localhost URLs (should be env vars)

### SUGGESTION (optional)
- Minor style improvements
- Alternative approaches
- Performance micro-optimizations
- Additional test coverage ideas

## Report Format

```
CODE REVIEW RESULTS
===================

MAJOR: 3
MINOR: 4
SUGGESTIONS: 4

MAJOR ISSUES:
-------------
[M1] God file: utils.ts (487 lines)
     File: src/utils.ts
     Contains: date utils, string utils, API helpers, validators
     Recommendation: Split into src/utils/date.ts, string.ts, api.ts, validators.ts

[M2] God method: SessionService#process_login (47 lines)
     File: app/services/session_service.rb
     Lines: 23-70
     Recommendation: Extract validation, token generation,
                     and audit logging into separate methods

[M3] N+1 query in dashboard
     File: app/controllers/dashboard_controller.rb
     Line: 15
     Code: @sessions.each { |s| s.heartbeats.count }
     Fix: Add .includes(:heartbeats) to query

MINOR ISSUES:
-------------
[m1] Duplicate code: error handling pattern
     Files: app/controllers/api/*.rb
     Appears: 5 times
     Fix: Extract to concern or base controller

[m2] Inconsistent naming: mix of 'get_' and 'fetch_' prefixes
     Files: app/services/*.rb
     Fix: Standardize on one convention

[m3] Missing error handling on external API call
     File: app/services/weather_service.rb
     Line: 34
     Fix: Add rescue for HTTP::Error

SUGGESTIONS:
------------
[s1] Consider using memoization for repeated calculations
[s2] Could extract magic number 30 to constant SESSION_TIMEOUT_MINUTES
[s3] Consider adding index on sessions.project_name for faster lookups
[s4] Additional test coverage for edge case: empty heartbeats
```

## Auto-Fix Patterns

| Issue | Auto-Fix Available |
|-------|-------------------|
| N+1 queries | Yes - add includes/preload |
| Magic numbers | Yes - extract to constant |
| Unused imports | Yes - remove line |
| Dead code | Yes - remove block |
| Simple duplicates | Partial - can suggest extraction |
| God files | No - requires design decision on split |
| God methods | No - requires design decision |
| God classes | No - requires design decision |
| Architecture issues | No - requires understanding context |

## God File Detection Script

Add to CI/CD or run manually:

```bash
#!/bin/bash
# detect-god-files.sh

echo "=== GOD FILE DETECTION ==="
echo ""

# Thresholds
COMPONENT_WARN=200
COMPONENT_ERROR=400
CONTROLLER_WARN=150
CONTROLLER_ERROR=300

# Find large files
echo "Files over 300 lines:"
find src app -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.rb" -o -name "*.py" \) \
  -exec wc -l {} \; | awk '$1 > 300 {print $1 " lines: " $2}' | sort -rn

echo ""
echo "Files with >15 imports (coupling smell):"
for f in $(find src -name "*.ts" -o -name "*.tsx" 2>/dev/null); do
  count=$(grep -c "^import" "$f" 2>/dev/null || echo 0)
  if [ "$count" -gt 15 ]; then
    echo "$count imports: $f"
  fi
done

echo ""
echo "Files with >5 exports (too many concerns):"
for f in $(find src -name "*.ts" -o -name "*.tsx" 2>/dev/null); do
  count=$(grep -c "^export" "$f" 2>/dev/null || echo 0)
  if [ "$count" -gt 5 ]; then
    echo "$count exports: $f"
  fi
done
```

## Placeholder Code Detection Script

Add to CI/CD or run before deployment:

```bash
#!/bin/bash
# detect-placeholder-code.sh

set -e

echo "=== PLACEHOLDER CODE DETECTION ==="
echo ""

CRITICAL=0
MAJOR=0
WARNING=0

# Exclude patterns
EXCLUDE="node_modules\|vendor\|\.git\|dist\|build\|coverage\|__pycache__"

echo "CRITICAL: Stub implementations"
echo "------------------------------"
# NotImplementedError / throw not implemented
stubs=$(grep -rn "NotImplementedError\|throw.*[Nn]ot.*[Ii]mplemented\|raise.*NotImplemented" \
  --include="*.ts" --include="*.tsx" --include="*.js" --include="*.py" --include="*.rb" \
  . 2>/dev/null | grep -v "$EXCLUDE" || true)
if [ -n "$stubs" ]; then
  echo "$stubs"
  CRITICAL=$((CRITICAL + $(echo "$stubs" | wc -l)))
fi

# return null/None with TODO
null_stubs=$(grep -rn "return null.*TODO\|return None.*TODO\|return null.*FIXME" \
  --include="*.ts" --include="*.tsx" --include="*.js" --include="*.py" \
  . 2>/dev/null | grep -v "$EXCLUDE" || true)
if [ -n "$null_stubs" ]; then
  echo "$null_stubs"
  CRITICAL=$((CRITICAL + $(echo "$null_stubs" | wc -l)))
fi

echo ""
echo "CRITICAL: Test credentials in code"
echo "-----------------------------------"
creds=$(grep -rn "password.*=.*['\"]test\|api_key.*=.*['\"]xxx\|secret.*=.*['\"]test\|token.*=.*['\"]fake" \
  --include="*.ts" --include="*.tsx" --include="*.js" --include="*.py" --include="*.rb" \
  . 2>/dev/null | grep -v "$EXCLUDE" | grep -v "\.test\.\|_spec\.\|test_" || true)
if [ -n "$creds" ]; then
  echo "$creds"
  CRITICAL=$((CRITICAL + $(echo "$creds" | wc -l)))
fi

echo ""
echo "MAJOR: Debug code"
echo "-----------------"
# JavaScript/TypeScript
debug_js=$(grep -rn "console\.log(\|console\.debug(\|debugger\|alert(" \
  --include="*.ts" --include="*.tsx" --include="*.js" \
  . 2>/dev/null | grep -v "$EXCLUDE" | grep -v "\.test\.\|\.spec\." || true)
if [ -n "$debug_js" ]; then
  echo "$debug_js"
  MAJOR=$((MAJOR + $(echo "$debug_js" | wc -l)))
fi

# Python
debug_py=$(grep -rn "print(\|pdb\|breakpoint()\|import ipdb" \
  --include="*.py" \
  . 2>/dev/null | grep -v "$EXCLUDE" | grep -v "test_\|_test\.py" || true)
if [ -n "$debug_py" ]; then
  echo "$debug_py"
  MAJOR=$((MAJOR + $(echo "$debug_py" | wc -l)))
fi

# Ruby
debug_rb=$(grep -rn "puts \|binding\.pry\|byebug\|debugger" \
  --include="*.rb" \
  . 2>/dev/null | grep -v "$EXCLUDE" | grep -v "_spec\.rb" || true)
if [ -n "$debug_rb" ]; then
  echo "$debug_rb"
  MAJOR=$((MAJOR + $(echo "$debug_rb" | wc -l)))
fi

echo ""
echo "MAJOR: TODO/FIXME comments"
echo "--------------------------"
todos=$(grep -rn "TODO:\|FIXME:\|XXX:\|HACK:" \
  --include="*.ts" --include="*.tsx" --include="*.js" --include="*.py" --include="*.rb" \
  . 2>/dev/null | grep -v "$EXCLUDE" | grep -v "TODO(#" || true)
if [ -n "$todos" ]; then
  echo "$todos"
  MAJOR=$((MAJOR + $(echo "$todos" | wc -l)))
fi

echo ""
echo "WARNING: Hardcoded URLs"
echo "-----------------------"
urls=$(grep -rn "localhost:\|127\.0\.0\.1:\|http://\|:3000\|:8080\|:5000" \
  --include="*.ts" --include="*.tsx" --include="*.js" --include="*.py" --include="*.rb" \
  . 2>/dev/null | grep -v "$EXCLUDE" | grep -v "\.test\.\|_spec\.\|test_\|\.config\.\|\.env" || true)
if [ -n "$urls" ]; then
  echo "$urls" | head -20
  WARNING=$((WARNING + $(echo "$urls" | wc -l)))
fi

echo ""
echo "=== SUMMARY ==="
echo "CRITICAL: $CRITICAL"
echo "MAJOR: $MAJOR"
echo "WARNING: $WARNING"

if [ $CRITICAL -gt 0 ]; then
  echo ""
  echo "❌ DEPLOYMENT BLOCKED: $CRITICAL critical placeholder issues found"
  exit 1
fi

if [ $MAJOR -gt 10 ]; then
  echo ""
  echo "⚠️ WARNING: $MAJOR major issues found (threshold: 10)"
  exit 1
fi

echo ""
echo "✅ Placeholder code check passed"
```
