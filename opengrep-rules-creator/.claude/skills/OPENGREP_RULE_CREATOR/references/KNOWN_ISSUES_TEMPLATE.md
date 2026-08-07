# Known Issues

<!--
  WHAT THIS FILE IS
  ─────────────────
  This file is the vulnerability register for the Opengrep Rules Creator
  pipeline. It documents previously identified security issues in your
  codebase so that Stage 2 (/opengrep-rule-creator) can generate Opengrep
  rules that target the exact patterns your team has already encountered.

  Rules derived from your own vulnerability history are the most valuable:
  they target patterns that have actually appeared in your code, which
  maximises detection accuracy and minimises false positives.

  WHEN TO UPDATE THIS FILE
  ─────────────────────────
  - After a penetration test: add each finding with its root cause pattern.
  - After a bug bounty report is triaged: add confirmed vulnerabilities.
  - After a CVE is disclosed that affects a library you use: add the package
    and the specific method / API that introduces the risk.
  - After an internal security review surfaces new patterns.
  - When a vulnerability is patched: update Status to "Patched" and keep the
    entry — patched issues are still useful for variant detection.

  HOW STAGE 2 USES THIS FILE
  ───────────────────────────
  Stage 2 reads this file before the interview and code recon hot spots.
  Each entry becomes a high-priority rule candidate. The more specific the
  "Root cause pattern" field, the more precise and low-false-positive the
  generated rule will be.

  FIELD GUIDE
  ───────────
  ID                  A sequential identifier. Format: KI-NNN.
  CVE / Reference     CVE ID, Snyk advisory ID, internal ticket, or bug
                      bounty report ID. Use "Internal" if no external ref.
  Vulnerability class Short label. See the taxonomy below.
  Affected component  File path and function / method name where the issue
                      was found. Be as specific as possible.
  Affected language   The language of the affected file.
  Status              Open | Patched | Won't Fix | Under Investigation
  Severity            Critical | High | Medium | Low
  Description         One to three sentences. What can an attacker do?
                      What is the impact?
  Root cause pattern  The code pattern that causes the vulnerability. This
                      is the most important field for rule generation. Describe
                      the pattern in terms that map to code: which function,
                      which argument, which data flow.
  Safe alternative    What the fix looks like. Concrete function name, library,
                      or pattern — not abstract advice.
  Recurrence risk     High | Medium | Low. How likely is the same pattern
                      to appear elsewhere in the codebase?

  VULNERABILITY CLASS TAXONOMY (use these labels for consistency)
  ───────────────────────────────────────────────────────────────
  Injection
    SQLi              SQL injection
    CMDi              Command / OS injection
    LDAPi             LDAP injection
    XPathi            XPath injection
    TMPLi             Template injection (server-side)
    HTMLi             HTML injection

  Client-side
    XSS-R             Reflected XSS
    XSS-S             Stored XSS
    XSS-D             DOM-based XSS
    OpenRedirect      Unvalidated redirect

  Authentication & Session
    AuthBypass        Authentication bypass
    SessionFixation   Session fixation
    CSRF              Cross-site request forgery
    JWT-Weak          JWT weakness (weak secret, alg:none, missing validation)
    MissingAuth       Missing authentication check on sensitive endpoint

  Access Control
    IDOR              Insecure direct object reference / BOLA
    PrivEsc           Privilege escalation
    MassAssign        Mass assignment / parameter binding
    PathTraversal     Path traversal / directory traversal

  Crypto
    WeakCrypto        Weak algorithm (MD5/SHA1 for passwords, ECB mode, DES)
    WeakPRNG          Insecure random number generation
    HardcodedKey      Hardcoded secret, credential, or key
    InsecureTLS       TLS misconfiguration (verify=False, self-signed accepted)

  Deserialization
    UnsafeDeser       Unsafe deserialization (pickle, yaml.load, unserialize)

  Server-side
    SSRF              Server-side request forgery
    XXE               XML external entity injection
    LFI               Local file inclusion
    RFI               Remote file inclusion

  Application Logic
    RaceCondition     Race condition in auth / payment / state-change flow
    ReDoS             Regular expression denial of service
    BizLogic          Business logic flaw (price manipulation, order bypass, etc.)
    IDOR-BizLogic     Object-level auth bypass with business impact

  Supply Chain
    DepVuln           Known vulnerable dependency (CVE in used library)
    ProtoPollu        Prototype pollution (JS)

  Infrastructure / Config
    SecretLeak        Secret / credential leaked in logs, response, or headers
    MissingHeader     Missing security header (CSP, HSTS, X-Frame-Options, etc.)
    CORS-Misc         CORS misconfiguration (wildcard, credentials + wildcard)
    LogInjection      Log injection / log forging
-->

---

## Format reference

Each entry follows this structure. Copy the entry template below to add new
issues. Do not leave placeholder text in `<angle brackets>` — replace every
field or delete the row.

```
### KI-NNN: <Short descriptive title>

| Field              | Value                                      |
|--------------------|--------------------------------------------|
| ID                 | KI-NNN                                     |
| CVE / Reference    | CVE-XXXX-XXXX or internal ticket ID        |
| Vulnerability class| <class from taxonomy above>                |
| Affected component | `path/to/file.ext` — `functionName()`      |
| Affected language  | Python / JS / TS / Java / Kotlin / Go / PHP / Swift / Dart |
| Status             | Open / Patched / Won't Fix / Under Investigation |
| Severity           | Critical / High / Medium / Low             |
| Description        | What can an attacker do? What is the impact? (1–3 sentences) |
| Root cause pattern | The specific code pattern: which function, which argument, which data flow |
| Safe alternative   | The correct pattern, library, or function to use instead |
| Recurrence risk    | High / Medium / Low                        |
```

---

## Worked examples

The entries below are complete examples. They illustrate the level of detail
that produces the most effective Opengrep rules. Remove or replace them with
your actual issues before committing this file.

---

### KI-001: SQL injection via unsanitized user ID in search endpoint

| Field | Value |
|---|---|
| ID | KI-001 |
| CVE / Reference | Internal — pentest Q3 2024, finding PT-042 |
| Vulnerability class | SQLi |
| Affected component | `src/api/users.js` — `searchUsers()` |
| Affected language | JS |
| Status | Patched |
| Severity | High |
| Description | The `q` query parameter was concatenated directly into a raw SQL string passed to `db.query()`. An attacker could enumerate all users, extract password hashes, or read arbitrary tables. |
| Root cause pattern | `db.query("SELECT * FROM users WHERE name LIKE '%" + req.query.q + "%'")` — user input concatenated into a SQL string argument to `db.query()` or `db.execute()` without parameterization. |
| Safe alternative | Use parameterized queries: `db.query("SELECT * FROM users WHERE name LIKE ?", ["%" + sanitized + "%"])`. The project uses `mysql2` — use `connection.execute()` with the `?` placeholder syntax. |
| Recurrence risk | High |

---

### KI-002: Prototype pollution via lodash _.merge with user-supplied body

| Field | Value |
|---|---|
| ID | KI-002 |
| CVE / Reference | SNYK-JS-LODASH-15869619 / CVE-2020-8203 |
| Vulnerability class | ProtoPollu |
| Affected component | `src/config/merge-defaults.js` — `applyUserPreferences()` |
| Affected language | JS |
| Status | Open |
| Severity | High |
| Description | `_.merge()` is called with `req.body` as the source object. An attacker can send `{"__proto__": {"isAdmin": true}}` to pollute the prototype chain and elevate privileges or alter application behaviour globally. |
| Root cause pattern | `_.merge(target, req.body)` — first or subsequent arguments to `_.merge()` originate from `req.body`, `req.query`, or any user-controlled HTTP input. |
| Safe alternative | Replace `_.merge()` with `structuredClone(target)` + `Object.assign()`, or switch to `lodash/fp`'s `merge` which does not mutate. If `_.merge` is required, sanitize keys: strip entries where key is `__proto__`, `constructor`, or `prototype` before merging. |
| Recurrence risk | High |

---

### KI-003: Insecure deserialization of YAML configuration uploaded by users

| Field | Value |
|---|---|
| ID | KI-003 |
| CVE / Reference | CVE-2017-18342 (PyYAML) |
| Vulnerability class | UnsafeDeser |
| Affected component | `app/pipeline/loader.py` — `load_pipeline_config()` |
| Affected language | Python |
| Status | Patched |
| Severity | Critical |
| Description | User-uploaded pipeline configuration files were loaded with `yaml.load()` without specifying a Loader. The default Loader allows arbitrary Python object instantiation, enabling remote code execution via a crafted YAML file. |
| Root cause pattern | `yaml.load(user_file)` or `yaml.load(user_file, Loader=yaml.Loader)` — any call to `yaml.load()` where the first argument is user-controlled input (file upload, API field, environment variable) and no safe Loader is specified. |
| Safe alternative | Use `yaml.safe_load()` for all untrusted input. For trusted configuration files that require full YAML features, use `yaml.load(f, Loader=yaml.FullLoader)` and document the trust assumption explicitly. |
| Recurrence risk | Medium |

---

### KI-004: Missing authentication on internal admin API routes

| Field | Value |
|---|---|
| ID | KI-004 |
| CVE / Reference | Internal — bug bounty report BB-2024-017 |
| Vulnerability class | MissingAuth |
| Affected component | `src/routes/admin.ts` — multiple route handlers |
| Affected language | TS |
| Status | Patched |
| Severity | Critical |
| Description | Several `/admin/*` endpoints were accessible without authentication. An unauthenticated attacker could list all users, reset passwords, and modify subscription plans. |
| Root cause pattern | Express route handler functions registered under `/admin/*` without `requireAuth` or `passport.authenticate()` middleware applied either to the router or to the individual handler. Pattern: `router.get('/admin/...', handlerFn)` with no auth middleware in the chain. |
| Safe alternative | Apply `requireAuth` middleware at the router level: `const adminRouter = express.Router(); adminRouter.use(requireAuth);`. Do not rely on per-route application — a missing decorator on one new route silently creates an auth bypass. |
| Recurrence risk | Medium |

---

### KI-005: Hardcoded JWT secret in application configuration

| Field | Value |
|---|---|
| ID | KI-005 |
| CVE / Reference | Internal — code review 2024-01 |
| Vulnerability class | HardcodedKey |
| Affected component | `config/auth.go` — `NewJWTService()` |
| Affected language | Go |
| Status | Patched |
| Severity | High |
| Description | The JWT signing secret was hardcoded as a string literal. Any developer with read access to the repository could forge valid JWT tokens and impersonate any user, including administrators. |
| Root cause pattern | `jwt.NewWithClaims(jwt.SigningMethodHS256, claims).SignedString([]byte("my-secret-key"))` — a string literal or `const` value passed as the signing key argument to `SignedString()` or equivalent JWT signing functions. |
| Safe alternative | Load the signing key from an environment variable or secrets manager at startup: `secret := os.Getenv("JWT_SECRET")`. Validate that the variable is non-empty and meets minimum length (≥32 bytes for HS256). |
| Recurrence risk | Low |

---

### KI-006: Path traversal in file download endpoint

| Field | Value |
|---|---|
| ID | KI-006 |
| CVE / Reference | Internal — pentest Q4 2024, finding PT-091 |
| Vulnerability class | PathTraversal |
| Affected component | `src/controllers/files.py` — `download_report()` |
| Affected language | Python |
| Status | Open |
| Severity | High |
| Description | The `filename` parameter from the request is joined directly to the base report directory path using `os.path.join()`. An attacker can supply `../../../etc/passwd` to read arbitrary files from the server filesystem. |
| Root cause pattern | `os.path.join(BASE_DIR, request.args.get('filename'))` followed by `open()` — user input passed to `os.path.join()` and then to any file-reading function (`open`, `send_file`, `FileResponse`) without path normalisation or containment check. |
| Safe alternative | Normalise and validate the resolved path: `resolved = os.path.realpath(os.path.join(BASE_DIR, filename))`. Then assert `resolved.startswith(os.path.realpath(BASE_DIR))` before opening. Reject the request if the assertion fails. |
| Recurrence risk | High |

---

<!-- ═══════════════════════════════════════════════════════════════════════ -->
<!-- ADD YOUR PROJECT'S KNOWN ISSUES BELOW THIS LINE                        -->
<!-- Use the entry template from the Format reference section at the top.    -->
<!-- Remove the worked examples above once your own entries are in place.    -->
<!-- ═══════════════════════════════════════════════════════════════════════ -->

