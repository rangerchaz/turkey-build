# Runtime Verification Phase

**When:** After build complete, before declaring "done"
**Goal:** Prove the code actually runs and does what was specified
**Rule:** No green checkmark until runtime verification passes

## Philosophy

Static analysis lies. "Compiles clean" means nothing if the app crashes on startup. "Tests pass" means nothing if there are no tests. "98/100 QA score" means nothing if the dashboard returns blank.

The only truth: **run it, hit it, verify the response.**

## Phase 1: Detect Stack

Don't ask. Detect.

```
scan project root:
  
  Gemfile + config.ru           → ruby/rails
  Gemfile + no config.ru        → ruby/script
  package.json + next.config.*  → node/nextjs
  package.json + nuxt.config.*  → node/nuxt
  package.json + vite.config.*  → node/vite
  package.json + extension      → node/vscode-extension
  package.json                  → node/generic
  pyproject.toml                → python/modern
  requirements.txt              → python/legacy
  manage.py                     → python/django
  go.mod                        → go
  Cargo.toml                    → rust
  *.csproj                      → dotnet
  pom.xml                       → java/maven
  build.gradle                  → java/gradle
  mix.exs                       → elixir
  pubspec.yaml                  → dart/flutter
  
  multiple matches              → monorepo, test each
```

## Phase 2: Database Schema Verification

**CRITICAL: Run this BEFORE starting the server.**

Database schema mismatches are the #1 cause of runtime failures that pass health checks but break auth/user flows.

```
procedure verify_database_schema():

  # 1. Detect migration tool from project files
  migration_tool = detect_migration_tool()
  #   Look for: alembic.ini, prisma/, migrations/, db/migrate/, etc.

  # 2. Run pending migrations
  run(migration_tool.upgrade_command)
  if failed:
    diagnose_migration_failure()
    # Common issues:
    #   - Migration references wrong parent (check revision chain)
    #   - Missing default for NOT NULL on existing table
    #   - Syntax error in migration file
    fix()
    retry(max=3)

  # 3. Verify schema matches models
  for each model/table modified in this build:
    run_simple_query(table)  # SELECT * FROM {table} LIMIT 1
    if "column does not exist":
      migration_did_not_run()
    if "type mismatch":
      migration_has_wrong_type()

  # 4. Test auth queries specifically (most common failure)
  if project_has_auth:
    test_user_query()  # Query user by email
    if failed:
      auth_will_break_at_runtime()
      fix_before_proceeding()

migration_file_validation:
  before_commit:
    - Parent reference points to actual previous migration ID
    - Migration runs forward successfully
    - Migration runs backward successfully (reversibility)
    - New columns on existing tables have defaults
```

### Common Migration Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Wrong parent reference | "Revision not found" | Check actual ID in previous migration file |
| No default for NOT NULL | "Cannot add NOT NULL column" | Add server_default or make nullable |
| Missing migration file | "Target revision not found" | Ensure file exists and is tracked in git |

## Phase 2b: Determine Start Command

Each stack has conventions. Use them.

```yaml
ruby/rails:
  install: "bundle install"
  setup: "rails db:create db:migrate db:seed"
  start: "rails server -b 0.0.0.0 -p 3000"
  ready_signal: "Listening on"
  ready_timeout: 60s
  port: 3000

node/nextjs:
  install: "npm install"
  start: "npm run dev"
  ready_signal: "Ready on"
  ready_timeout: 30s
  port: 3000

node/vite:
  install: "npm install"
  start: "npm run dev"
  ready_signal: "Local:"
  ready_timeout: 15s
  port: 5173

node/generic:
  install: "npm install"
  start: "npm start"
  ready_signal: "listening"
  ready_timeout: 30s
  port: 3000

node/vscode-extension:
  install: "npm install"
  compile: "npm run compile"
  verify: "no TypeScript errors + vsix builds OR extension.js exists"
  # no server to start - different verification

python/django:
  install: "pip install -r requirements.txt"
  setup: "python manage.py migrate"
  start: "python manage.py runserver 0.0.0.0:8000"
  ready_signal: "Starting development server"
  ready_timeout: 30s
  port: 8000

python/flask:
  install: "pip install -r requirements.txt"
  start: "flask run --host=0.0.0.0"
  ready_signal: "Running on"
  ready_timeout: 15s
  port: 5000

python/fastapi:
  install: "pip install -r requirements.txt"
  start: "uvicorn main:app --host 0.0.0.0 --port 8000"
  ready_signal: "Uvicorn running"
  ready_timeout: 15s
  port: 8000

go:
  install: "go mod download"
  start: "go run ."
  ready_signal: "listening"
  ready_timeout: 30s
  port: 8080

rust:
  install: "cargo build"
  start: "cargo run"
  ready_signal: "listening"
  ready_timeout: 60s
  port: 8080

dotnet:
  install: "dotnet restore"
  start: "dotnet run"
  ready_signal: "Now listening on"
  ready_timeout: 45s
  port: 5000

elixir:
  install: "mix deps.get"
  setup: "mix ecto.setup"
  start: "mix phx.server"
  ready_signal: "Running"
  ready_timeout: 30s
  port: 4000
```

## Phase 3: Discover Endpoints

Don't guess. Parse.

```yaml
ruby/rails:
  command: "rails routes -g ''"
  parse: extract verb + path from output

python/django:
  file: "**/urls.py"
  parse: regex for path() and url() calls

python/flask:
  files: "**/*.py"
  parse: regex for @app.route decorators

python/fastapi:
  files: "**/*.py"  
  parse: regex for @app.get/post/put/delete decorators

node/express:
  files: "**/*.js, **/*.ts"
  parse: regex for app.get/post/put/delete and router.get/post/put/delete

node/nextjs:
  dirs: "app/**/page.*, pages/**/*"
  parse: file paths become routes

go:
  files: "**/*.go"
  parse: regex for http.HandleFunc, mux.Handle, gin routes

elixir:
  file: "**/router.ex"
  parse: regex for get/post/put/delete macros

generic:
  files: "openapi.yaml, openapi.json, swagger.yaml, swagger.json"
  parse: OpenAPI spec paths
  
fallback:
  check: "/", "/health", "/api", "/api/health"
```

## Phase 4: Classify & Test Endpoints

Different endpoints need different verification.

```yaml
endpoint_types:

  html_page:
    indicators:
      - "GET method"
      - "no /api prefix"
      - "controller renders template"
    verify:
      - status: 200
      - content-type: contains "text/html"
      - body: non-empty, contains "<html" or "<!DOCTYPE"
    
  json_api_read:
    indicators:
      - "GET method"
      - "/api prefix OR respond_to :json"
    verify:
      - status: 200 or 204
      - content-type: contains "application/json"
      - body: valid JSON (if 200)
  
  json_api_write:
    indicators:
      - "POST/PUT/PATCH method"
    verify:
      - send: minimal valid payload based on param parsing
      - status: 200, 201, or 422 (validation error is OK - means it ran)
      - content-type: contains "application/json"
    
  json_api_delete:
    indicators:
      - "DELETE method"
    verify:
      - status: 200, 204, or 404 (not found is OK - means it ran)
      
  static_file:
    indicators:
      - "/assets", "/static", "/public"
      - file extension in path
    verify:
      - status: 200 or 304
      - body: non-empty (if 200)

  websocket:
    indicators:
      - "ws://", "wss://", "cable", "socket"
    verify:
      - connection: opens without immediate close
      - optional: send ping, receive pong
      
  auth_required:
    indicators:
      - "authenticate", "authorize", "current_user"
    verify:
      - status: 401 or 403 (proves auth check runs)
      - with_token: 200 (if we can generate test token)
```

## Phase 5: Run Verification

```
procedure verify_project():
  
  stack = detect_stack()
  config = get_stack_config(stack)
  
  # Install dependencies
  run(config.install)
  if failed:
    diagnose_install_failure()
    fix()
    retry(max=3)
  
  # Run setup (migrations, etc)
  if config.setup:
    run(config.setup)
    if failed:
      diagnose_setup_failure()
      fix()
      retry(max=3)
  
  # Start server
  process = start_background(config.start)
  wait_for(config.ready_signal, timeout=config.ready_timeout)
  
  if timeout:
    output = process.stderr + process.stdout
    diagnose_startup_failure(output)
    fix()
    retry(max=3)
  
  # Discover endpoints
  endpoints = discover_endpoints(stack)
  
  if empty(endpoints):
    endpoints = ["/", "/health", "/api"]  # fallback
  
  # Test each endpoint
  results = []
  for endpoint in endpoints:
    type = classify_endpoint(endpoint)
    result = verify_endpoint(endpoint, type)
    results.append(result)
  
  # Report
  passed = results.filter(r => r.passed)
  failed = results.filter(r => !r.passed)
  
  if failed.any():
    for f in failed:
      diagnose_failure(f)
      fix()
    
    # Re-run full verification after fixes
    process.kill()
    verify_project()  # recursive retry
  
  process.kill()
  return all_passed
```

## Phase 6: Diagnose & Fix

This is the hard part. Pattern match errors to fixes.

```yaml
diagnosis_patterns:

  # Startup failures
  - pattern: "cannot load such file"
    cause: "missing gem"
    fix: "bundle install"
    
  - pattern: "Module not found"
    cause: "missing npm package"
    fix: "npm install"
    
  - pattern: "No module named"
    cause: "missing python package"
    fix: "pip install {module}"
    
  - pattern: "database .* does not exist"
    cause: "database not created"
    fix: "rails db:create" | "createdb" | "python manage.py migrate"
    
  - pattern: "Migrations are pending"
    cause: "migrations not run"
    fix: "rails db:migrate" | "python manage.py migrate"
    
  - pattern: "Address already in use"
    cause: "port conflict"
    fix: "kill process on port OR use different port"
    
  - pattern: "EACCES: permission denied"
    cause: "file permissions"
    fix: "chmod +x" | "sudo chown"

  # Response failures  
  - pattern: "204 No Content on HTML route"
    cause: "controller not rendering"
    search: "ApplicationController"
    check: "inherits from ActionController::API?"
    fix: "change to ActionController::Base"
    
  - pattern: "500 Internal Server Error"
    cause: "unhandled exception"
    search: "server logs for stack trace"
    fix: "address the exception"
    
  - pattern: "404 Not Found"
    cause: "route not defined OR controller missing"
    search: "routes file"
    fix: "add route OR create controller"
    
  - pattern: "template not found"
    cause: "view file missing"
    search: "app/views or templates/"
    fix: "create the template file"
    
  - pattern: "JSON parse error on response"
    cause: "endpoint returning non-JSON"
    search: "controller action"
    fix: "add render json: OR respond_to"
    
  - pattern: "CORS error"
    cause: "cross-origin blocked"
    fix: "add CORS middleware/headers"
    
  - pattern: "connection refused"
    cause: "server not running OR wrong port"
    fix: "verify server started on expected port"
    
  - pattern: "SSL certificate error"
    cause: "https required but not configured"
    fix: "use http for local dev OR configure SSL"

  # Extension-specific
  - pattern: "Cannot find module 'vscode'"
    cause: "running outside extension host"
    fix: "this is expected - verify compile only"
    
  - pattern: "Activating extension .* failed"
    cause: "extension activation error"
    search: "extension.ts activate function"
    fix: "address activation error"
```

## Phase 6b: Auth Flow Verification

**Health checks lie.** `/api/health` returning 200 does not mean auth works.

If the project has authentication, you MUST test the actual auth flow:

```
procedure verify_auth_flow():

  # Skip if project has no auth
  if not has_auth_endpoints():
    return SKIP

  # 1. Test registration (if exists)
  if has_register_endpoint:
    response = POST /register (or /auth/register, /api/auth/register)
      body: { email: "test-{uuid}@example.com", password: "TestPass123!" }

    verify:
      - status: 201 or 200 (not 500)
      - response contains user data or success message
      - no "column does not exist" in logs

  # 2. Test login
  response = POST /login (or /auth/login, /api/auth/login)
    body: { email: "test@example.com", password: "wrongpassword" }

  verify:
    - status: 401 or 400 (not 500)
    - response contains error message
    - no database errors in logs

  # 3. Test authenticated endpoint (if exists)
  if has_me_endpoint:
    response = GET /me (or /auth/me, /api/users/me)
      headers: { Authorization: "Bearer {invalid_token}" }

    verify:
      - status: 401 (not 500)
      - proper error response

auth_verification_required:
  when:
    - User model was modified
    - Auth endpoints were added/changed
    - Any migration touched users table

  failure_means:
    - Do NOT proceed to E2E testing
    - Database schema is likely wrong
    - Fix migration and re-verify
```

### Why This Matters

| What You Test | What It Proves |
|---------------|----------------|
| GET /health → 200 | Server starts |
| POST /login → 401 | Auth code runs without crashing |
| POST /login → 500 | **Database schema is broken** |

A 500 on login means the ORM query failed - usually a missing column.

## Phase 7: Verify Against Spec

Not just "does it run" but "does it do what was asked."

```
procedure verify_spec(original_request, endpoints):
  
  # Extract requirements from original request
  requirements = parse_requirements(original_request)
  
  # Example: "passive tracking - sends heartbeat every 2 min"
  # → need: POST /heartbeats or similar endpoint
  # → need: endpoint accepts project, branch, timestamp
  
  for req in requirements:
    matched = find_matching_endpoint(req, endpoints)
    
    if not matched:
      report: "SPEC FAIL: '{req}' has no endpoint"
      action: "create missing endpoint"
      
    else:
      # Verify endpoint does what spec says
      result = test_endpoint_behavior(matched, req)
      
      if not result.matches_spec:
        report: "SPEC FAIL: '{matched}' doesn't {req}"
        action: "fix endpoint to match spec"
  
  # Check for specified features
  specified_features = [
    "passive tracking",          # → heartbeat endpoint exists
    "git branch detection",      # → branch field accepted
    "30 min gap = new session",  # → session creation logic
    "dashboard",                 # → / returns HTML with session data
    "analytics",                 # → /analytics returns charts
  ]
  
  for feature in specified_features:
    verify_feature_implemented(feature)
```

## Phase 8: Extension-Specific Verification

VS Code extensions can't be "started" like a server. Different process.

```
procedure verify_vscode_extension():
  
  # 1. Compile check
  run("npm run compile")
  if typescript_errors:
    fix_typescript_errors()
    retry()
  
  # 2. Package check
  if has_vsce:
    run("vsce package --no-dependencies")
    if failed:
      diagnose_packaging_error()
      fix()
      retry()
  
  # 3. Manifest check
  package_json = read("package.json")
  
  verify:
    - activationEvents defined
    - main or browser entry point exists
    - contributes section matches features
  
  # 4. Entry point check
  main_file = package_json.main
  verify:
    - file exists
    - exports activate function
    - activate doesn't throw on import
  
  # 5. Dependencies check
  for dep in package_json.dependencies:
    if dep == "vscode":
      warn: "should use @types/vscode + engines.vscode instead"
    verify: dep is importable
  
  # 6. Simulated activation (if possible)
  # Some CI systems support headless VS Code
  if headless_vscode_available:
    run("code --extensionDevelopmentPath=. --disable-gpu --headless")
    verify: no activation errors in output
```

## Integration with Build Skill

Add to SKILL.md after QA phase:

```markdown
## Phase 7: Runtime Verification

After QA scoring, before declaring complete:

1. **Start the app** - Use stack-appropriate command
2. **Wait for ready** - Watch for listening signal
3. **Hit every endpoint** - GET pages, POST to APIs
4. **Verify responses** - Right status, right content-type, non-empty
5. **Check against spec** - Does it do what was asked?
6. **Fix failures** - Diagnose, patch, restart, re-verify
7. **Loop until clean** - No green checkmark until runtime passes

### Failure Recovery

When verification fails:
1. Capture error output
2. Match against known patterns
3. Apply fix
4. Restart verification from step 1
5. Max 5 fix attempts before asking for help

### What "Done" Means

The build is NOT done until:
- [ ] Server starts without error
- [ ] All HTML routes return 200 with HTML body
- [ ] All API routes return expected status codes
- [ ] POST endpoints accept valid payloads
- [ ] Extension compiles and packages (if applicable)
- [ ] Original spec requirements are verifiable

"Compiles" ≠ "Works"
"Tests pass" ≠ "Works"  
"Looks good" ≠ "Works"

Only "ran it, hit it, verified response" = Works.
```

## Example: What Should Have Caught the Rails Bug

```
verify_project() called after build...

stack: ruby/rails
start: rails server -b 0.0.0.0 -p 3000
ready: "Listening on http://0.0.0.0:3000"

endpoints discovered via 'rails routes':
  GET  /           → dashboard#index
  GET  /analytics  → analytics#index
  POST /api/heartbeats → api/heartbeats#create
  GET  /api/current_session → api/heartbeats#current_session

testing GET / ...
  status: 204
  content-type: none
  body: empty
  FAIL: HTML route returned no content

diagnosing...
  searching: ApplicationController
  found: "class ApplicationController < ActionController::API"
  pattern match: "API controller can't render HTML"
  
fix: change ActionController::API to ActionController::Base
applying fix...
restarting server...

testing GET / ...
  status: 200
  content-type: text/html
  body: contains "<html"
  PASS

testing GET /analytics ...
  status: 200
  content-type: text/html
  body: contains "<html"
  PASS

testing POST /api/heartbeats ...
  payload: {"project_name":"test","branch":"main","file_type":".rb"}
  status: 201
  content-type: application/json
  body: valid JSON with session_id
  PASS

all endpoints pass.
runtime verification complete.
```

This would have caught the bug before declaring "98/100 - done!"
