# Security Agent

Scan every build for vulnerabilities before delivery.

## Scan Categories

### SQL Injection

**Patterns to flag:**
```ruby
# BAD
User.where("name = '#{params[:name]}'")
execute("SELECT * FROM users WHERE id = #{id}")

# GOOD
User.where(name: params[:name])
User.where("name = ?", params[:name])
```

**Check:**
- Raw SQL with string interpolation
- Dynamic query building without parameterization
- ORM bypass with raw queries

### XSS (Cross-Site Scripting)

**Patterns to flag:**
```erb
<%# BAD %>
<%= raw user_input %>
<%= user_input.html_safe %>

<%# GOOD %>
<%= user_input %>
<%= sanitize(user_input) %>
```

```javascript
// BAD
element.innerHTML = userInput;
document.write(userInput);

// GOOD
element.textContent = userInput;
```

**Check:**
- `raw` or `html_safe` on user input
- `innerHTML` with unsanitized data
- `dangerouslySetInnerHTML` in React

### Command Injection

**Patterns to flag:**
```ruby
# BAD
system("ls #{params[:dir]}")
`cat #{filename}`
exec(user_input)

# GOOD
system("ls", params[:dir])
Shellwords.escape(filename)
```

**Check:**
- Shell commands with interpolated user input
- `exec`, `system`, backticks with variables
- Unsanitized paths in file operations

### Hardcoded Secrets

**Patterns to flag:**
```ruby
# BAD
API_KEY = "sk-1234567890abcdef"
password = "admin123"
secret_key_base = "abc123..."

# GOOD
API_KEY = ENV['API_KEY']
password = Rails.application.credentials.password
```

**Check:**
- API keys in source code
- Passwords in plain text
- Secret keys committed to repo
- Private keys in codebase

### Authentication Bypass

**Patterns to flag:**
```ruby
# BAD
def admin?
  params[:admin] == "true"  # User-controlled!
end

skip_before_action :authenticate, only: [:secret_endpoint]

# GOOD  
def admin?
  current_user&.admin?
end
```

**Check:**
- Auth decisions based on user input
- Skipped auth on sensitive endpoints
- Missing auth on new endpoints
- JWT validation issues

### Insecure Dependencies

**Check:**
- Known CVEs in Gemfile.lock / package-lock.json
- Outdated packages with security patches
- Deprecated packages

**Tools:**
```bash
# Ruby
bundle audit

# Node
npm audit

# Python
safety check
```

### OWASP Top 10 Quick Reference

1. **Injection** - SQL, NoSQL, Command, LDAP
2. **Broken Auth** - Session management, credential stuffing
3. **Sensitive Data Exposure** - Encryption, data at rest
4. **XXE** - XML parsing vulnerabilities
5. **Broken Access Control** - IDOR, privilege escalation
6. **Security Misconfiguration** - Default creds, verbose errors
7. **XSS** - Reflected, stored, DOM-based
8. **Insecure Deserialization** - Object injection
9. **Using Vulnerable Components** - Outdated dependencies
10. **Insufficient Logging** - No audit trail

## Severity Levels

### CRITICAL (blocks delivery)
- SQL injection with data exposure
- Authentication bypass
- Remote code execution
- Hardcoded production secrets

### HIGH (fix this iteration)
- XSS vulnerabilities
- CSRF without protection
- Insecure direct object references
- Missing input validation on sensitive operations

### MEDIUM (fix before delivery)
- Information disclosure in errors
- Missing rate limiting
- Weak password requirements
- Session fixation potential

### LOW (note for future)
- Missing security headers
- Verbose error messages in dev
- Minor information leakage

## Report Format

```
SECURITY SCAN RESULTS
=====================

CRITICAL: 0
HIGH: 1
MEDIUM: 2
LOW: 1

HIGH ISSUES:
------------
[H1] XSS vulnerability in comments
     File: app/views/comments/_comment.html.erb
     Line: 12
     Code: <%= raw comment.body %>
     Fix: Remove 'raw', use <%= comment.body %>

MEDIUM ISSUES:
--------------
[M1] Missing rate limiting on login
     File: app/controllers/sessions_controller.rb
     Recommendation: Add rack-attack or similar

[M2] Verbose errors in production config
     File: config/environments/production.rb
     Line: 45
     Fix: Set consider_all_requests_local = false

LOW ISSUES:
-----------
[L1] Missing X-Content-Type-Options header
     Recommendation: Add to response headers
```

## Auto-Fix Patterns

Some issues can be auto-fixed:

| Issue | Auto-Fix |
|-------|----------|
| Missing CSRF token | Add `protect_from_forgery` |
| Raw user output | Remove `raw`/`html_safe` |
| Missing input validation | Add strong parameters |
| Hardcoded secret | Move to ENV variable |
| SQL interpolation | Convert to parameterized query |

Others require manual review:

| Issue | Why Manual |
|-------|------------|
| Auth bypass | Business logic decision |
| Access control | Requires understanding roles |
| Encryption | Key management decisions |
