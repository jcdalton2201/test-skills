---
name: secure-code-review-rust
description: |
  Security review for Rust code using AST analysis, data flow tracking, cargo tools (clippy/audit), and pattern matching. Creates Jira/Linear issues for findings. Integrates with rust-tdd workflow. Provides auto-fix suggestions with code diffs.
---

# secure-code-review-rust: Comprehensive Security Analysis

Security scanning for Rust with AST-based analysis, type system awareness, macro expansion, cargo tool integration, and automatic issue creation.

## Overview

1. **AST Analysis** - Parse with `syn` for accurate data flow tracking
2. **Cargo Tools** - Run `cargo clippy` and `cargo audit`
3. **Pattern Matching** - Regex for obvious issues (secrets, SQL, etc.)
4. **Type System** - Recognize validated wrapper types (reduce false positives)
5. **Macro Expansion** - Understand compile-time checked code (sqlx!, etc.)
6. **Issue Creation** - Create Jira/Linear issues for findings
7. **Auto-Fix** - Generate remediation code diffs

---

## Part 1: Invocation

### User Triggers

```text
A: "Review codebase for security issues"      → Full scan
B: "Review src/payment.rs for security"       → Single file
C: After rust-tdd [ready-for-test]            → Automatic scan
D: "Security review for branch feature/pay"   → Branch diff
```

### Configuration (.secure-code-review.toml)

```toml
[scan]
include = ["src/**/*.rs"]
exclude = ["tests/**", "benches/**"]

[tools]
run_clippy = true
run_cargo_audit = true
expand_macros = true

[tracker]
type = "linear"  # or "jira"
project = "PROJ"
create_issues = true
labels = ["security", "needs-info"]

[severity]
fail_on = ["CRITICAL", "HIGH"]
```

---

## Part 2: AST-Based Analysis

### Data Flow Tracking with `syn`

**Taint Sources** (mark as untrusted):
```rust
axum::extract::{Path, Query, Json}
actix_web::web::{Path, Query, Json}
std::env::args()
std::io::stdin()
reqwest::Response::text()
```

**Dangerous Sinks** (check if tainted data reaches):
```rust
sqlx::query() with format!
Command::new().arg()
std::fs::{read, write}
reqwest::get()
unsafe { }
```

**Algorithm**:
```
1. Parse function with syn::parse_file()
2. Mark parameters from extractors as "tainted"
3. Walk AST, track variable assignments
4. If tainted var reaches dangerous sink → FINDING
```

**Example Detection** (data flow through function calls):
```rust
// NOW DETECTED with AST analysis:
fn build_query(id: &str) -> String {
    format!("WHERE id = {}", id)  // Returns tainted
}

async fn handler(Path(id): Path<String>) -> Result<User> {
    let clause = build_query(&id);  // Tainted!
    let sql = format!("SELECT * FROM users {}", clause);
    sqlx::query_as(&sql)  // ← FINDING: SQL injection
        .fetch_one(&pool).await
}
```

---

## Part 3: Cargo Tools Integration

### Run Clippy

```bash
cargo clippy --all-targets --message-format=json
```

**Security-relevant lints**:
- `clippy::unwrap_used` / `expect_used`
- `clippy::panic` / `todo` / `unimplemented`
- `clippy::indexing_slicing`
- `clippy::arithmetic_side_effects`
- `clippy::cast_possible_truncation`

### Run Cargo Audit

```bash
cargo audit --json
```

**Parse vulnerabilities**:
- Advisory ID, severity, description
- Affected crate and version
- Patched version recommendation
- Generate Cargo.toml fix

---

## Part 4: Type System Awareness

### Recognize Validated Types

**Pattern: Newtype with validation constructor**
```rust
struct ValidatedEmail(String);  // Private field

impl ValidatedEmail {
    pub fn new(email: &str) -> Result<Self, Error> {
        if !email.contains('@') { return Err(...) }
        Ok(Self(email.to_string()))
    }
}

// Function takes ValidatedEmail → input pre-validated → LOWER SEVERITY
fn create_user(email: ValidatedEmail) -> Result<User> { }
```

**Detection**:
```
1. Find newtype definition (tuple struct, 1 field)
2. Check if constructor returns Result (validates)
3. Check if inner field is private (can't bypass)
4. If all true → ValidationStatus::TypeValidated → lower severity
```

---

## Part 5: Macro Expansion

### Safe Macros (Compile-Time Checked)

```rust
// SAFE - compile-time checked by sqlx
sqlx::query!("SELECT * FROM users WHERE id = $1", id)

// NOT SAFE - dynamic query
sqlx::query(&format!("SELECT * FROM users WHERE id = {}", id))
```

**Detection**: Recognize `query!`, `query_as!`, `query_scalar!` macros as safe.

**For complex macros**: Run `cargo expand` before analysis.

---

## Part 6: Vulnerability Detection

### 6.1 Hardcoded Secrets

**Detect**:
```rust
const API_KEY: &str = "sk_live_abc123";  // ← CRITICAL
```

**Auto-Fix**:
```diff
- const API_KEY: &str = "sk_live_abc123";
+ fn api_key() -> Result<String, VarError> {
+     std::env::var("API_KEY")
+ }
```

### 6.2 SQL Injection

**Detect** (with data flow):
```rust
let sql = format!("SELECT * FROM users WHERE id = '{}'", user_id);
sqlx::query(&sql)  // ← CRITICAL
```

**Auto-Fix**:
```diff
- let sql = format!("SELECT * FROM users WHERE id = '{}'", user_id);
- sqlx::query(&sql).fetch_one(&pool).await
+ sqlx::query("SELECT * FROM users WHERE id = $1")
+     .bind(user_id)
+     .fetch_one(&pool).await
```

### 6.3 Command Injection

**Detect**:
```rust
Command::new("sh").arg("-c").arg(format!("grep {}", input))  // ← CRITICAL
```

**Auto-Fix**:
```diff
- Command::new("sh").arg("-c").arg(format!("grep {}", input))
+ Command::new("grep").arg("--").arg(input).arg("file.txt")
```

### 6.4 Path Traversal

**Detect**:
```rust
let path = base_dir.join(user_filename);  // ← HIGH if user_filename tainted
```

**Auto-Fix**:
```diff
- let path = base_dir.join(user_filename);
+ fn safe_join(base: &Path, name: &str) -> Result<PathBuf, AppError> {
+     if name.contains("..") || name.starts_with('/') {
+         return Err(AppError::InvalidPath);
+     }
+     Ok(base.join(name))
+ }
+ let path = safe_join(&base_dir, &user_filename)?;
```

### 6.5 Unsafe Code

**Detect**:
```rust
unsafe { std::mem::transmute(...) }  // ← HIGH if undocumented
```

**Auto-Fix** (add safety docs):
```diff
+ /// # Safety
+ /// Caller must ensure ptr is valid and aligned.
  unsafe fn read_raw(ptr: *const u8) -> u8 { *ptr }
```

### 6.6 Weak Cryptography

**Detect**:
```rust
let hash = md5::compute(password);  // ← CRITICAL for passwords
let token: u64 = rand::random();    // ← HIGH for secrets
```

**Auto-Fix**:
```diff
- let hash = md5::compute(password);
+ use argon2::{Argon2, PasswordHasher};
+ use password_hash::{SaltString, rand_core::OsRng};
+ let salt = SaltString::generate(&mut OsRng);
+ let hash = Argon2::default().hash_password(password.as_bytes(), &salt)?;
```

### 6.7 JWT Issues

**Detect**:
```rust
jsonwebtoken::dangerous::insecure_decode(...)  // ← CRITICAL
validation.validate_exp = false;               // ← HIGH
```

### 6.8 SSRF / Open Redirect

**Detect**:
```rust
reqwest::get(user_url).await  // ← HIGH if user_url not validated
Redirect::to(user_redirect)   // ← MEDIUM
```

### 6.9 Async Mistakes

**Detect**:
```rust
std::fs::read_to_string(...)  // ← MEDIUM in async fn (blocking)
service.get().await.unwrap()  // ← HIGH panic in handler
```

**Auto-Fix**:
```diff
- std::fs::read_to_string("file.txt")
+ tokio::fs::read_to_string("file.txt").await

- service.get().await.unwrap()
+ service.get().await.map_err(AppError::from)?
```

---

## Part 7: Issue Creation (Jira/Linear)

### Detect Tracker

```rust
1. Check for Linear MCP tools → use Linear API
2. Check JIRA_URL env var → use Jira API
3. Check .secure-code-review.toml → use configured tracker
4. Fallback → generate report only
```

### Create Linear Issue

```rust
mcp__linear__save_issue(
    title: "Security: SQL Injection in payment_service.rs",
    description: format_issue_body(finding),
    labelIds: ["security", "critical"],
    teamId: config.team_id,
    priority: 1,  // Urgent for CRITICAL
)
```

### Create Jira Issue

```rust
POST {jira_url}/rest/api/2/issue
{
  "fields": {
    "project": { "key": "SEC" },
    "summary": "Security: SQL Injection in payment_service.rs",
    "description": format_jira_body(finding),
    "issuetype": { "name": "Bug" },
    "priority": { "name": "Blocker" },
    "labels": ["security", "critical"]
  }
}
```

### Issue Body Template

```markdown
## 🔴 CRITICAL Security Finding

**File**: src/services/payment_service.rs:42
**CWE**: CWE-89 | **OWASP**: A03:2021 - Injection

### Description
User input flows to SQL query without parameterization.

### Vulnerable Code
```rust
let sql = format!("SELECT * FROM users WHERE id = '{}'", user_id);
sqlx::query(&sql).fetch_one(&pool).await
```

### Auto-Fix
```diff
- let sql = format!("SELECT * FROM users WHERE id = '{}'", user_id);
- sqlx::query(&sql).fetch_one(&pool).await
+ sqlx::query("SELECT * FROM users WHERE id = $1")
+     .bind(user_id)
+     .fetch_one(&pool).await
```

### Verification
After fixing: `cargo test && cargo clippy`
```

---

## Part 8: Integration with rust-tdd

### Automatic Security Check Workflow

When rust-tdd completes and adds `[ready-for-test]`:

```
1. SCAN: Run secure-code-review-rust on implemented files

2. IF CRITICAL/HIGH findings:
   - Create tracker issue with [security][critical] labels
      - Update parent issue: Remove [ready-for-test], add [needs-security-fix]
      - Add comment: "⚠️ Security issues found - see SECURITY-FINDINGS.md"
      - Status stays: In Progress (not Done)

   3. IF MEDIUM/LOW only:
      - Keep [ready-for-test] label
      - Add comment: "ℹ️ Security scan passed. N minor findings noted."
      - Continue to code review

4. OUTPUT: SECURITY-FINDINGS.md for human review
```

### Label Changes

```
Before security scan:
  [must][s2][epic-PREFIX][ready-for-test]

If CRITICAL/HIGH found:
  [must][s2][epic-PREFIX][needs-security-fix]
  + New issue: [security][critical] or [security][high]

After security fix:
  [must][s2][epic-PREFIX][ready-for-test]
  (re-scan passes)

After merge:
  [shipped]
```

### Comment Templates

**Security Issues Found**:
```
⚠️ **Security Scan Failed**

Found: 2 CRITICAL, 1 HIGH issues

| Severity | Issue | File | Auto-Fix |
|----------|-------|------|----------|
| CRITICAL | SQL Injection | payment.rs:42 | ✅ Available |
| CRITICAL | Hardcoded Secret | config.rs:10 | ✅ Available |
| HIGH | Path Traversal | upload.rs:88 | ✅ Available |

See SECURITY-FINDINGS.md for details and auto-fix diffs.

Created issues:
- SEC-101: SQL Injection in payment_service
- SEC-102: Hardcoded API key in config
- SEC-103: Path traversal in file upload
```

**Security Scan Passed**:
```
✅ **Security Scan Passed**

No CRITICAL or HIGH findings.

Minor findings (2 MEDIUM, 1 LOW):
- MEDIUM: Consider stronger RNG for session tokens
- MEDIUM: Unbounded deserialization in import endpoint
- LOW: Missing CORS headers on public endpoint

See SECURITY-FINDINGS.md for details.

Ready for code review.
```

---

## Part 9: Report Generation

### SECURITY-FINDINGS.md

```markdown
# Security Findings Report

**Scan Date**: 2026-05-12T14:30:00Z
**Files Scanned**: 12
**Duration**: 2.34s

## Summary

| Severity | Count | Action |
|----------|-------|--------|
| CRITICAL | 2 | Fix immediately |
| HIGH | 1 | Fix before merge |
| MEDIUM | 2 | Track for remediation |
| LOW | 1 | Document |

## 🔴 CRITICAL Issues

### 1. SQL Injection
**File**: src/services/payment_service.rs:42
**CWE**: CWE-89 | **OWASP**: A03:2021

**Vulnerable Code**:
```rust
let sql = format!("SELECT * FROM users WHERE id = '{}'", user_id);
```

**Auto-Fix**:
```diff
- let sql = format!("SELECT * FROM users WHERE id = '{}'", user_id);
- sqlx::query(&sql).fetch_one(&pool).await
+ sqlx::query("SELECT * FROM users WHERE id = $1")
+     .bind(user_id)
+     .fetch_one(&pool).await
```

**Tracker Issue**: SEC-101

---

### 2. Hardcoded AWS Key
**File**: src/config/aws.rs:10
**CWE**: CWE-798

**Vulnerable Code**:
```rust
const AWS_KEY: &str = "AKIAIOSFODNN7EXAMPLE";
```

**Auto-Fix**:
```diff
- const AWS_KEY: &str = "AKIAIOSFODNN7EXAMPLE";
+ fn aws_key() -> Result<String, VarError> {
+     std::env::var("AWS_KEY")
+ }
```

**Tracker Issue**: SEC-102

---

## 🟠 HIGH Issues

### 3. Path Traversal
**File**: src/handlers/upload.rs:88

...

## Exemptions

| File | Finding | Reason | Expires |
|------|---------|--------|---------|
| tests/fixtures.rs:* | hardcoded-secret | Test data | Never |
| src/ffi/crypto.rs:42 | unsafe-block | ADR-0005 approved | 2026-08 |
```

### security-findings.json

```json
{
  "summary": {
    "critical": 2,
    "high": 1,
    "medium": 2,
    "low": 1,
    "timestamp": "2026-05-12T14:30:00Z",
    "duration_ms": 2340
  },
  "findings": [
    {
      "id": "sql-injection-001",
      "severity": "CRITICAL",
      "confidence": "HIGH",
      "file": "src/services/payment_service.rs",
      "line": 42,
      "type": "sql-injection",
      "title": "SQL Injection",
      "cwe": "CWE-89",
      "owasp": "A03:2021",
      "description": "User input flows to SQL query",
      "vulnerable_code": "let sql = format!(\"SELECT...",
      "auto_fix": {
        "description": "Use parameterized query",
        "diff": "- let sql = format!...\n+ sqlx::query..."
      },
      "tracker_issue": "SEC-101"
    }
  ],
  "tools_run": {
    "ast_analysis": true,
    "cargo_clippy": true,
    "cargo_audit": true,
    "macro_expansion": false
  }
}
```

---

## Part 10: False Positive Handling

### Automatic Detection

```rust
// Test file → lower severity
if file.contains("/tests/") || file.contains("/benches/") {
    finding.severity = "INFO";
    finding.notes.push("Test file - verify not real credential");
}

// Documented unsafe → lower severity
if has_safety_documentation(unsafe_block) {
    finding.severity = lower_severity(finding.severity);
}

// Validated type → lower severity
if is_validated_type(param_type) {
    finding.confidence = "LOW";
}

// Compile-time checked macro → skip
if is_compile_time_safe(macro_call) {
    return None;  // Not a finding
}
```

### Exemption File (.secure-code-review-ignore)

```text
# Test credentials
tests/**:*:hardcoded-secret - Test data only

# Approved unsafe FFI (ADR-0005)
src/ffi/crypto.rs:42:unsafe-block - FFI boundary, approved

# False positive - validated at type level
src/api/handlers.rs:*:sql-injection - Uses ValidatedQuery type
```

---

## Part 11: Checklist

### Per-Scan Checklist

```
[ ] AST analysis completed
[ ] Cargo clippy run (if enabled)
[ ] Cargo audit run (if enabled)
[ ] Data flow tracking applied
[ ] Type validation recognized
[ ] Macro safety recognized
[ ] False positives filtered
[ ] Auto-fixes generated
[ ] Report written (MD + JSON)
[ ] Tracker issues created (if CRITICAL/HIGH)
[ ] Parent issue updated (if rust-tdd integration)
```

### Finding Quality

```
[ ] Severity is accurate (not over/under-stated)
[ ] CWE/OWASP references correct
[ ] Vulnerable code snippet included
[ ] Auto-fix is valid Rust (compiles)
[ ] Auto-fix doesn't change behavior (except security)
[ ] Remediation is actionable
```

---

## Summary

This skill provides comprehensive Rust security analysis:

1. **AST Analysis** - Accurate data flow, not just regex
2. **Cargo Integration** - Clippy + audit for compiler-level checks
3. **Type Awareness** - Reduces false positives for validated types
4. **Macro Understanding** - Recognizes compile-time safe patterns
5. **Auto-Fix** - Actionable code diffs for each finding
6. **Tracker Integration** - Creates Jira/Linear issues automatically
7. **rust-tdd Integration** - Automatic security gate before code review

**Severity Guide**:
- **CRITICAL**: Fix immediately (SQL injection, hardcoded secrets, RCE)
- **HIGH**: Fix before merge (path traversal, weak crypto, unsafe without docs)
- **MEDIUM**: Track and fix soon (weak patterns, missing headers)
- **LOW**: Document and improve opportunistically
