---
name: security-review
description: >-
  Systematic, OWASP-informed security code review that identifies exploitable
  vulnerabilities with confidence-based reporting. Use when asked to security
  review, find vulnerabilities, check for security issues, audit security, or
  OWASP review code. Traces data flow, checks framework mitigations, and
  reports only high-confidence findings to minimize false positives.
---

<!-- See NOTICE.md for attribution (getsentry/skills Apache 2.0, OWASP CC BY-SA 4.0) -->

# Security Review

Identify exploitable security vulnerabilities in code. Report only
**HIGH CONFIDENCE** findings -- clear vulnerable patterns with
attacker-controlled input confirmed through data-flow tracing.

## Scope: Research vs. Reporting

**CRITICAL DISTINCTION:**

- **Report on**: Only the specific file, diff, or code provided by the user
- **Research**: The ENTIRE codebase to build confidence before reporting

Before flagging any issue, you MUST research the codebase to understand:

- Where does this input actually come from? (Trace data flow)
- Is there validation/sanitization elsewhere?
- How is this configured? (Check settings, config files, middleware)
- What framework protections exist?

**Do NOT report issues based solely on pattern matching.** Investigate first,
then report only what you are confident is exploitable.

---

## Confidence Levels

| Level      | Criteria                                                 | Action                           |
| ---------- | -------------------------------------------------------- | -------------------------------- |
| **HIGH**   | Vulnerable pattern + attacker-controlled input confirmed | **Report** with severity         |
| **MEDIUM** | Vulnerable pattern, input source unclear                 | **Note** as "Needs verification" |
| **LOW**    | Theoretical, best practice, defense-in-depth             | **Do not report**                |

---

## Do Not Flag

### General Exclusions

- Test files (unless explicitly reviewing test security)
- Dead code, commented code, documentation strings
- Patterns using **constants** or **server-controlled configuration**
- Code paths that require prior authentication to reach (note the auth
  requirement instead)

### Server-Controlled Values (NOT Attacker-Controlled)

These are configured by operators, not controlled by attackers:

| Source                | Example                                      | Why Safe                         |
| --------------------- | -------------------------------------------- | -------------------------------- |
| Framework settings    | `settings.API_URL`, `settings.ALLOWED_HOSTS` | Set via config/env at deployment |
| Environment variables | `os.environ.get('DATABASE_URL')`             | Deployment configuration         |
| Config files          | `config.yaml`, `app.config['KEY']`           | Server-side files                |
| Framework constants   | `django.conf.settings.*`, `process.env.*`    | Not user-modifiable              |
| Hardcoded values      | `BASE_URL = "https://api.internal"`          | Compile-time constants           |

**SSRF -- NOT a vulnerability:**

```python
# SAFE: URL comes from server-controlled settings
response = requests.get(f"{settings.API_URL}{path}")
```

**SSRF -- IS a vulnerability:**

```python
# VULNERABLE: URL comes from request (attacker-controlled)
response = requests.get(request.GET.get('url'))
```

### Framework-Mitigated Patterns

Check the framework context before flagging. Common false positives:

| Pattern                             | Why Usually Safe               |
| ----------------------------------- | ------------------------------ |
| Django `{{ variable }}`             | Auto-escaped by default        |
| React `{variable}`                  | Auto-escaped by default        |
| Vue `{{ variable }}`                | Auto-escaped by default        |
| Angular `{{ variable }}`            | Auto-sanitized by default      |
| `User.objects.filter(id=input)`     | ORM parameterizes queries      |
| `cursor.execute("...%s", (input,))` | Parameterized query            |
| `innerHTML = "<b>Loading...</b>"`   | Constant string, no user input |

**Only flag these when bypass mechanisms are used:**

- Django: `{{ var|safe }}`, `{% autoescape off %}`, `mark_safe(user_input)`
- React: `dangerouslySetInnerHTML={{__html: userInput}}`
- Vue: `v-html="userInput"`
- Angular: `bypassSecurityTrustHtml(userInput)`
- ORM: `.raw()`, `.extra()`, `RawSQL()` with string interpolation

---

## Review Process

### 1. Detect Context

What type of code is being reviewed?

| Code Type               | Focus Areas                                    |
| ----------------------- | ---------------------------------------------- |
| API endpoints, routes   | Authorization, authentication, injection       |
| Frontend, templates     | XSS, CSRF                                      |
| File handling, uploads  | Path traversal, file type validation, XXE      |
| Crypto, secrets, tokens | Algorithm strength, key management, randomness |
| Data serialization      | Deserialization attacks                        |
| External requests       | SSRF                                           |
| Business workflows      | Race conditions, logic bypass                  |
| GraphQL, REST design    | Mass assignment, query depth, rate limiting    |
| Config, headers, CORS   | Misconfiguration, permissive origins           |
| CI/CD, dependencies     | Supply chain, secret exposure in logs          |
| Error handling          | Fail-open patterns, information disclosure     |
| Audit, logging          | Log injection, missing audit trails            |

### 2. Identify Language/Framework

Based on file extension or imports, apply language-specific knowledge:

| Indicators                             | Key Patterns                                                                    |
| -------------------------------------- | ------------------------------------------------------------------------------- |
| `.py`, Django, Flask, FastAPI          | Auto-escape templates, ORM parameterization, `mark_safe`, `pickle`, `yaml.load` |
| `.js`/`.ts`, Express, React, Vue, Next | JSX auto-escape, `dangerouslySetInnerHTML`, `eval`, `child_process`             |
| `.go`, `go.mod`                        | `html/template` auto-escape, `text/template` unsafe, `os/exec`                  |
| `.rs`, `Cargo.toml`                    | `unsafe` blocks, FFI boundaries, raw pointer dereference                        |
| `.java`, Spring, `@Controller`         | Spring Security defaults, `@RequestBody` deserialization, JDBC parameterization |
| `.php`, Laravel, Symfony               | Blade auto-escape, `{!! !!}` raw output, PDO parameterization                   |

### 3. Check Infrastructure (if applicable)

| File Type                        | Key Patterns                                                            |
| -------------------------------- | ----------------------------------------------------------------------- |
| `Dockerfile`, `.dockerignore`    | Root user, secrets in layers, unverified base images                    |
| K8s manifests, Helm charts       | RBAC scope, secret mounting, privileged containers                      |
| `.tf`, Terraform                 | Hardcoded credentials, overly permissive IAM, public exposure           |
| GitHub Actions, `.gitlab-ci.yml` | Secret leakage in logs, untrusted inputs in `run:`, pull_request_target |
| AWS/GCP/Azure configs            | Overly permissive policies, public buckets/storage, missing encryption  |

### 4. Research Before Flagging

**For each potential issue, research the codebase to build confidence:**

- Where does this value actually come from? Trace the data flow
- Is it configured at deployment (settings, env vars) or from user input?
- Is there validation, sanitization, or allowlisting elsewhere?
- What framework protections apply?

Only report issues where you have HIGH confidence after understanding the
broader context.

### 5. Verify Exploitability

For each potential finding, confirm:

**Is the input attacker-controlled?**

| Attacker-Controlled (Investigate)              | Server-Controlled (Usually Safe)                 |
| ---------------------------------------------- | ------------------------------------------------ |
| `request.GET`, `request.POST`, `request.args`  | `settings.X`, `app.config['X']`                  |
| `request.json`, `request.data`, `request.body` | `os.environ.get('X')`, `process.env.X`           |
| `request.headers` (most headers)               | Hardcoded constants                              |
| `request.cookies` (unsigned)                   | Internal service URLs from config                |
| URL path segments: `/users/<id>/`              | Database content from admin/system               |
| File uploads (content and names)               | Signed session data                              |
| Database content from other users              | Framework settings                               |
| WebSocket messages                             | Validated/signed JWT claims (after verification) |

**Does the framework mitigate this?**

- Check language/framework for auto-escaping, parameterization
- Check for middleware/decorators that sanitize

**Is there validation upstream?**

- Input validation before this code
- Sanitization libraries (DOMPurify, bleach, html_escape, etc.)

### 6. Report HIGH Confidence Only

Skip theoretical issues. Report only what you have confirmed is exploitable
after research.

---

## Severity Classification

| Severity     | Impact                                          | Examples                                                                                  |
| ------------ | ----------------------------------------------- | ----------------------------------------------------------------------------------------- |
| **Critical** | Direct exploit, severe impact, no auth required | RCE, SQL injection to data exfiltration, auth bypass, hardcoded secrets in source         |
| **High**     | Exploitable with conditions, significant impact | Stored XSS, SSRF to cloud metadata, IDOR to sensitive data, privilege escalation          |
| **Medium**   | Specific conditions required, moderate impact   | Reflected XSS, CSRF on state-changing actions, path traversal with restrictions           |
| **Low**      | Defense-in-depth, minimal direct impact         | Missing security headers, verbose error messages, weak algorithms in non-critical context |

---

## Quick Patterns Reference

### Always Flag (Critical)

```
eval(user_input)             # Any language
exec(user_input)             # Any language
pickle.loads(user_data)      # Python
yaml.load(user_data)         # Python (not safe_load)
unserialize($user_data)      # PHP
deserialize(user_data)       # Java ObjectInputStream
shell=True + user_input      # Python subprocess
child_process.exec(user)     # Node.js
os.system(f"cmd {user}")     # Python
Runtime.exec(userInput)      # Java
```

### Always Flag (High)

```
innerHTML = userInput                  # DOM XSS
dangerouslySetInnerHTML={user}         # React XSS
v-html="userInput"                     # Vue XSS
{!! $userInput !!}                     # Blade XSS
f"SELECT * FROM x WHERE {user}"        # SQL injection
`SELECT * FROM x WHERE ${user}`        # SQL injection
os.system(f"cmd {user_input}")         # Command injection
subprocess.call(cmd, shell=True)       # Command injection (if cmd has user input)
```

### Always Flag (Secrets)

```
password = "hardcoded"
api_key = "sk-..."
AWS_SECRET_ACCESS_KEY = "AKIA..."
private_key = "-----BEGIN"
token = "ghp_..."
```

### Check Context First (MUST Investigate Before Flagging)

```
# SSRF -- ONLY if URL is from user input, NOT from settings/config
requests.get(request.GET['url'])       # FLAG: User-controlled URL
requests.get(settings.API_URL)         # SAFE: Server-controlled config
requests.get(f"{settings.BASE}/{x}")   # CHECK: Is 'x' user input?

# Path traversal -- ONLY if path is from user input
open(request.GET['file'])              # FLAG: User-controlled path
open(settings.LOG_PATH)                # SAFE: Server-controlled config
open(f"{BASE_DIR}/{filename}")         # CHECK: Is 'filename' validated?

# Open redirect -- ONLY if URL is from user input
redirect(request.GET['next'])          # FLAG: User-controlled redirect
redirect(settings.LOGIN_URL)           # SAFE: Server-controlled config

# Weak crypto -- ONLY if used for security purposes
hashlib.md5(file_content)              # SAFE: File checksums, caching
hashlib.md5(password)                  # FLAG: Password hashing
random.random()                        # SAFE: Non-security (UI, sampling)
random.random() for token              # FLAG: Use secrets module
```

---

## Output Format

````markdown
## Security Review: [File/Component Name]

### Summary

- **Findings**: X (Y Critical, Z High, ...)
- **Risk Level**: Critical/High/Medium/Low
- **Confidence**: High/Mixed

### Findings

#### [VULN-001] [Vulnerability Type] (Severity)

- **Location**: `file.py:123`
- **Confidence**: High
- **Issue**: [What the vulnerability is]
- **Impact**: [What an attacker could do]
- **Evidence**:
  ```python
  [Vulnerable code snippet]
  ```
````

- **Fix**: [How to remediate]

### Needs Verification

#### [VERIFY-001] [Potential Issue]

- **Location**: `file.py:456`
- **Question**: [What needs to be verified]

```

If no vulnerabilities found, state:
"No high-confidence vulnerabilities identified in the reviewed scope."
```
