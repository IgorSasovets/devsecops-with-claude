---
name: opengrep-code-recon
description: >-
  Stage 1 of the Opengrep Rules Creator pipeline. Scans a project directory to
  map tech stack, data flows, validators/sanitizers, data processing functions,
  business logic, config files, DB schemas, and third-party dependencies across
  JS/TS, Python, Java/Kotlin, Go, PHP, and mobile (Swift/Dart/Kotlin) projects.
  Produces CODE_RECON.md in opengrep-rules-creator/. Uses Grep/Glob first;
  reads files only when targeted. Output feeds /opengrep-rule-creator.
  Invocation depth auto-scales to project size.
  Use when asked to "recon the codebase", "map the source code", or as the
  first step of an Opengrep rules creation workflow.
context: fork
argument-hint: "<target-dir> [--exclude <glob>] [--lang <js|ts|python|java|kotlin|go|php|swift|dart>] [--depth <fast|balanced|deep>] [--no-deps]"
allowed-tools:
  - Read
  - Glob
  - Grep
  - Write
  - Bash(find:*)
  - Bash(ls:*)
  - Bash(wc:*)
  - Bash(head:*)
  - Bash(cat:*)
  - Bash(grep:*)
  - Bash(rg:*)
  - Bash(jq:*)
---

# /opengrep-code-recon

Stage 1 of the Opengrep Rules Creator pipeline. Maps a project's source code
to produce a structured `CODE_RECON.md` that feeds `/opengrep-rule-creator`
with everything needed to write precise, low-false-positive Opengrep rules.

**Token efficiency contract:** Glob and Grep first — always. Read individual
files only when targeted extraction is needed. Never read large files in full;
use `head -n 80` or targeted `grep` instead. Depth scales automatically with
project size (see Step 2).

**Tool fallback:** Prefer Glob/Grep native tools. When unavailable, fall back
to `find`, `rg --files`, `grep -rn`, `ls -R` via Bash. Only permitted Bash
commands listed above are allowed.

---

## Arguments

- `<target-dir>` (required) — project root to scan. Relative or absolute.
- `--exclude <glob>` — extra exclusion glob on top of config + built-in
  defaults. Repeatable.
- `--lang <id>` — restrict discovery to one language/ecosystem. When omitted,
  all supported languages are auto-detected.
- `--depth <fast|balanced|deep>` — override auto-scaling (see Step 2).
- `--no-deps` — skip dependency inventory (faster for very large repos).

---

## Step 0 — Pre-flight

1. Confirm `<target-dir>` exists and is readable.
2. State clearly: **this skill is read-only — no target files will be modified
   or executed.**
3. Create output directory `<target-dir>/opengrep-rules-creator/` if absent.
4. Check for existing `CODE_RECON.md`; if found, note its timestamp and ask
   operator whether to overwrite or append.

---

## Step 1 — Load configuration

Look for `<target-dir>/.opengrep-config.yml`. If present, parse:

- `exclude_paths` — merge with built-in exclusions below.
- `languages` — list override (same ids as `--lang`).
- `depth` — `fast | balanced | deep` (overridable by CLI).
- `output_dir` — override default `opengrep-rules-creator`.
- `known_issues_file` — path to `KNOWN_ISSUES.md` for downstream stage.
- `project_context` — free-text block copied verbatim into CODE_RECON.md.

If config is absent, use defaults. Note absence in CODE_RECON.md.
Merge CLI args on top of config values (CLI wins).

**Built-in exclusions (always applied):**
```
.git/**, **/node_modules/**, **/__pycache__/**, **/vendor/**, **/dist/**,
**/build/**, **/.gradle/**, **/.idea/**, **/.vscode/**, **/Pods/**,
**/.dart_tool/**, **/.pub-cache/**, **/target/**, **/.cargo/,
**/*.min.js, **/*.map, **/*.lock, **/coverage/**, **/.tox/**
```

---

## Step 2 — Auto-scale depth

Count total source files before reading anything:

```bash
find <target-dir> -type f \( -name "*.js" -o -name "*.ts" -o -name "*.py" \
  -o -name "*.java" -o -name "*.kt" -o -name "*.go" -o -name "*.php" \
  -o -name "*.swift" -o -name "*.dart" \) \
  | grep -v node_modules | grep -v vendor | grep -v dist | wc -l
```

| File count | Auto depth | Behaviour |
|---|---|---|
| < 200 | `deep` | Read key files in full; trace all entry points |
| 200–2000 | `balanced` | Targeted head+grep; sample large files |
| > 2000 | `fast` | Grep-only passes; no full file reads |

`--depth` CLI flag overrides auto-selection. Always report the chosen depth
in CODE_RECON.md.

**Cap:** At `fast` depth with > 5 000 files, sample the 100 most recently
modified + 200 random files from each language bucket and note the sampling
in CODE_RECON.md.

---

## Step 3 — Language and framework detection

Run all passes in parallel (Glob/Grep, no file reads).

### 3a — Language markers

| Language | Detection signals |
|---|---|
| JavaScript | `**/*.js`, `**/*.mjs`, `package.json` present |
| TypeScript | `**/*.ts`, `**/*.tsx`, `tsconfig.json` present |
| Python | `**/*.py`, `requirements.txt`, `pyproject.toml`, `setup.py` |
| Java | `**/*.java`, `pom.xml`, `build.gradle` |
| Kotlin | `**/*.kt`, `**/*.kts` |
| Go | `**/*.go`, `go.mod` |
| PHP | `**/*.php`, `composer.json` |
| Swift | `**/*.swift`, `Package.swift`, `*.xcodeproj` |
| Dart/Flutter | `**/*.dart`, `pubspec.yaml` |

### 3b — Framework/runtime detection (Grep, no read)

```bash
# JS/TS frameworks
grep -rl "from 'express'\|require('express')" <target-dir> --include="*.js" --include="*.ts" | head -5
grep -rl "from '@nestjs\|from 'fastify\|from 'koa'" <target-dir> --include="*.ts" | head -5
grep -rl "from 'react'\|from 'next'" <target-dir> --include="*.tsx" --include="*.jsx" | head -5

# Python frameworks
grep -rl "from django\|import django" <target-dir> --include="*.py" | head -5
grep -rl "from flask\|import flask" <target-dir> --include="*.py" | head -5
grep -rl "from fastapi\|import fastapi" <target-dir> --include="*.py" | head -5

# Java/Kotlin frameworks
grep -rl "springframework\|@SpringBootApplication" <target-dir> --include="*.java" --include="*.kt" | head -5
grep -rl "android.app.Activity\|androidx." <target-dir> --include="*.java" --include="*.kt" | head -5

# Go frameworks
grep -rl "\"github.com/gin-gonic\|\"github.com/gorilla\|\"net/http\"" <target-dir> --include="*.go" | head -5

# PHP frameworks
grep -rl "Illuminate\\\|Symfony\\\|Laravel" <target-dir> --include="*.php" | head -5
```

### 3c — Locale/charset detection

```bash
# Detect non-ASCII identifiers or comments (signals non-English codebase)
grep -rl $'[^\x00-\x7F]' <target-dir> --include="*.py" --include="*.js" \
  --include="*.ts" --include="*.java" --include="*.go" | head -10
# Check source file encoding declarations
grep -rn "coding:\|charset=" <target-dir> --include="*.py" --include="*.php" | head -10
```

Record detected locales and non-ASCII patterns in CODE_RECON.md. Downstream
rule creation will use this to avoid locale-specific false positives.

---

## Step 4 — Dependency inventory (skip if `--no-deps`)

Extract package lists without reading entire lock files.

### JS/TS
```bash
# package.json — extract dependencies block only (avoid reading full file)
cat <target-dir>/package.json | python3 -c "
import sys,json; d=json.load(sys.stdin)
deps={**d.get('dependencies',{}), **d.get('devDependencies',{})}
for k,v in sorted(deps.items()): print(f'{k}=={v}')" 2>/dev/null | head -100
```

### Python
```bash
cat <target-dir>/requirements.txt 2>/dev/null | grep -v "^#" | head -80
grep -A200 '\[project\]' <target-dir>/pyproject.toml 2>/dev/null | grep "dependencies" | head -30
```

### Java/Kotlin
```bash
grep -n "implementation\|compile\|api(" <target-dir>/build.gradle 2>/dev/null | head -60
grep -n "<artifactId>\|<groupId>" <target-dir>/pom.xml 2>/dev/null | head -80
```

### Go
```bash
cat <target-dir>/go.mod | grep "require" -A200 | head -80
```

### PHP
```bash
cat <target-dir>/composer.json | python3 -c "
import sys,json; d=json.load(sys.stdin)
reqs={**d.get('require',{}), **d.get('require-dev',{})}
for k,v in sorted(reqs.items()): print(f'{k}: {v}')" 2>/dev/null | head -60
```

### Mobile
```bash
# Swift (Package.swift or Podfile)
grep -n "\.package\|pod '" <target-dir>/Package.swift \
  <target-dir>/Podfile 2>/dev/null | head -40
# Dart
cat <target-dir>/pubspec.yaml | grep -A100 "dependencies:" | head -60
```

**Security-flagged packages** — after building the list, grep for known
high-risk package names:
```bash
# Packages known to introduce common vuln classes
echo "<dep-list>" | grep -iE \
  "lodash|moment|serialize-javascript|node-fetch|axios|request|got|\
  eval|vm2|child_process|shelljs|execa|pyyaml|pickle|marshal|\
  deserialize|jinja2|mako|mysql|psycopg2|pymongo|redis|jwt|jsonwebtoken|\
  bcrypt|crypto-js|md5|sha1|xml2js|cheerio|puppeteer|playwright"
```

Flag matched packages in a `## Security-Interesting Dependencies` section.

---

## Step 5 — Entry point mapping

Discover where external input enters the application. Grep-only for `balanced`
and `fast`; targeted head reads for `deep`.

### HTTP / API entry points
```bash
# Express/Node
grep -rn "app\.get\|app\.post\|app\.put\|app\.delete\|app\.patch\|router\." \
  <target-dir> --include="*.js" --include="*.ts" | grep -v node_modules | head -80

# FastAPI/Flask
grep -rn "@app\.route\|@router\.\|@app\.get\|@app\.post" \
  <target-dir> --include="*.py" | head -80

# Spring
grep -rn "@GetMapping\|@PostMapping\|@RequestMapping\|@RestController" \
  <target-dir> --include="*.java" --include="*.kt" | head -80

# Go net/http
grep -rn "http\.HandleFunc\|mux\.Handle\|router\.GET\|router\.POST" \
  <target-dir> --include="*.go" | head -60

# PHP
grep -rn "Route::\|$_GET\|$_POST\|$_REQUEST\|$_FILES" \
  <target-dir> --include="*.php" | head -60
```

### User-input sources (taint sources for rules)
```bash
# Common sources across ecosystems
grep -rn "req\.body\|req\.params\|req\.query\|request\.args\|request\.form\|\
request\.json\|request\.data\|getParam\|getQueryParam\|\$_GET\|\$_POST\|\
\$_REQUEST\|os\.Stdin\|bufio\.NewScanner\|flag\.\|os\.Args\|readLine\|\
scanner\.nextLine\|System\.in\|Console\.ReadLine" \
  <target-dir> --include="*.js" --include="*.ts" --include="*.py" \
  --include="*.java" --include="*.kt" --include="*.go" --include="*.php" \
  | grep -v node_modules | grep -v vendor | head -100
```

Record each unique entry point as: `file:line | method | framework`.

---

## Step 6 — Data flow mapping

Scale to depth. Build a source → transformation → sink map for each major
input source found in Step 5.

### 6a — Dangerous sinks (always scan, Grep-only)

```bash
# Injection sinks — SQL
grep -rn "query\|execute\|raw\|\.sql(" <target-dir> \
  --include="*.js" --include="*.ts" --include="*.py" \
  --include="*.java" --include="*.kt" --include="*.go" --include="*.php" \
  | grep -viE "(sanitize|escape|prepare|parameteriz|placeholder|\?)" \
  | grep -v node_modules | grep -v vendor | head -60

# Command injection sinks
grep -rn "exec\|spawn\|system\|popen\|subprocess\|Runtime\.exec\|os\.exec\|\
shell_exec\|passthru\|proc_open" \
  <target-dir> --include="*.py" --include="*.java" --include="*.kt" \
  --include="*.go" --include="*.php" | grep -v node_modules | head -60

# Template / code injection sinks
grep -rn "eval\|Function(\|render_template_string\|template\.Execute\|\
Twig::createTemplate\|eval(" \
  <target-dir> --include="*.js" --include="*.ts" --include="*.py" \
  --include="*.php" | grep -v node_modules | head -40

# File operation sinks
grep -rn "fs\.readFile\|fs\.writeFile\|open(\|fopen\|file_get_contents\|\
os\.Open\|ioutil\.ReadFile\|Files\.readAll" \
  <target-dir> --include="*.js" --include="*.ts" --include="*.py" \
  --include="*.java" --include="*.go" --include="*.php" | head -50

# Deserialization sinks
grep -rn "pickle\.load\|yaml\.load\|JSON\.parse\|unserialize\|\
ObjectInputStream\|Newtonsoft\.Json\|deserialize" \
  <target-dir> --include="*.py" --include="*.js" --include="*.php" \
  --include="*.java" --include="*.kt" | head -40

# HTTP redirect sinks (open redirect)
grep -rn "redirect\|location\.href\|res\.redirect\|header('Location" \
  <target-dir> --include="*.js" --include="*.ts" --include="*.py" \
  --include="*.php" | head -40

# XSS sinks
grep -rn "innerHTML\|outerHTML\|document\.write\|\.html(\|dangerouslySetInner\|\
v-html\|render.*html\|Markup(" \
  <target-dir> --include="*.js" --include="*.ts" --include="*.html" \
  --include="*.py" | grep -v node_modules | head -50
```

### 6b — Transformation chain (deep/balanced only)

For each entry-point file found in Step 5, extract the first 80 lines:
```bash
head -n 80 <entry-point-file>
```
Identify any intermediate function calls between input read and sink. Record
the chain as: `source → [fn1, fn2, ...] → sink`.

---

## Step 7 — Validator and sanitizer mapping

```bash
# Input validation libraries and functions
grep -rn "validator\.\|joi\.\|yup\.\|zod\.\|ajv\.\|express-validator\|\
marshmallow\|pydantic\|cerberus\|jsonschema\|Validate\|@Valid\|@NotNull\|\
binding\.ShouldBind\|govalidator" \
  <target-dir> --include="*.js" --include="*.ts" --include="*.py" \
  --include="*.java" --include="*.kt" --include="*.go" | head -60

# Sanitization / encoding functions
grep -rn "sanitize\|escape\|encode\|strip_tags\|htmlspecialchars\|\
DOMPurify\|xss\|bleach\|html\.EscapeString\|template\.HTMLEscaper\|\
encodeURIComponent\|encodeURI\|htmlentities" \
  <target-dir> --include="*.js" --include="*.ts" --include="*.py" \
  --include="*.php" --include="*.go" | grep -v node_modules | head -60

# Parameterized query / ORM usage (safe patterns — reduce false positives)
grep -rn "prepare\|parameteriz\|bindParam\|\?\|:param\|%s.*cursor\|\
\.filter(\|\.where(\|Model\.objects\|createQueryBuilder\|\.eq(\|\
gorm\.Where\|db\.Prepare" \
  <target-dir> --include="*.py" --include="*.js" --include="*.ts" \
  --include="*.java" --include="*.kt" --include="*.go" --include="*.php" \
  | head -60

# Auth/authz middleware (trust boundaries for rules)
grep -rn "isAuthenticated\|requireAuth\|@PreAuthorize\|jwt\.verify\|\
authenticate\|passport\.\|middleware.*auth\|authMiddleware\|checkPermission" \
  <target-dir> --include="*.js" --include="*.ts" --include="*.py" \
  --include="*.java" --include="*.kt" --include="*.go" | head -50
```

---

## Step 8 — Data processing function mapping

```bash
# Serialization / deserialization patterns
grep -rn "JSON\.stringify\|JSON\.parse\|pickle\.\|yaml\.\|toml\.\|\
serialize\|marshal\|protobuf\|MessagePack\|Gson\|ObjectMapper\|Codable" \
  <target-dir> --include="*.js" --include="*.ts" --include="*.py" \
  --include="*.java" --include="*.kt" --include="*.swift" | head -50

# Cryptographic operations (flag weak algos)
grep -rn "md5\|sha1\|sha256\|createCipher\|DES\|RC4\|ECB\|\
hashlib\.\|Cipher\.\|MessageDigest\|crypto\." \
  <target-dir> --include="*.js" --include="*.ts" --include="*.py" \
  --include="*.java" --include="*.kt" --include="*.go" | head -50

# Random number generation (flag insecure)
grep -rn "Math\.random\|random\.\|rand\.\|rand\.Intn\|rand\.Float\|\
Random()\|new Random\|random\.random\(\)" \
  <target-dir> --include="*.js" --include="*.ts" --include="*.py" \
  --include="*.java" --include="*.kt" --include="*.go" | head -40

# File upload / download handling
grep -rn "multer\|formidable\|busboy\|upload\|FileField\|InMemoryUploadedFile\|\
multipart\|ContentType.*octet\|@RequestPart" \
  <target-dir> --include="*.js" --include="*.ts" --include="*.py" \
  --include="*.java" | head -40
```

---

## Step 9 — Business logic mapping

Identify high-value business logic that warrants bespoke Opengrep rules.

```bash
# Payment / financial operations
grep -rn "charge\|payment\|stripe\|paypal\|braintree\|invoice\|billing\|\
amount\|price\|discount\|refund\|transfer\|withdraw\|balance" \
  <target-dir> --include="*.js" --include="*.ts" --include="*.py" \
  --include="*.java" --include="*.kt" --include="*.go" --include="*.php" \
  | grep -v node_modules | grep -v "\.test\.\|_test\.\|spec\." | head -40

# Access control / permission checks
grep -rn "role\|permission\|privilege\|canAccess\|isAdmin\|isOwner\|\
hasRole\|@RolesAllowed\|checkAccess\|authorize" \
  <target-dir> --include="*.js" --include="*.ts" --include="*.py" \
  --include="*.java" --include="*.kt" | head -40

# Rate limiting
grep -rn "rateLimit\|throttle\|RateLimiter\|limiter\.\|rate_limit\|\
429\|TooManyRequests" \
  <target-dir> --include="*.js" --include="*.ts" --include="*.py" \
  --include="*.java" --include="*.go" | head -30

# Token / session management
grep -rn "token\|session\|cookie\|JWT\|refreshToken\|accessToken\|\
SESSION_SECRET\|secret_key\|CSRF" \
  <target-dir> --include="*.js" --include="*.ts" --include="*.py" \
  --include="*.java" --include="*.kt" | grep -v node_modules | head -50
```

---

## Step 10 — Configuration file mapping

```bash
# Discover all config files (no reading — file list only)
find <target-dir> \( \
  -name "*.env" -o -name ".env*" -o -name "*.env.*" \
  -o -name "config.js" -o -name "config.ts" -o -name "config.py" \
  -o -name "settings.py" -o -name "application.yml" \
  -o -name "application.properties" -o -name "*.yaml" -o -name "*.yml" \
  -o -name "*.json" -o -name ".htaccess" -o -name "web.config" \
  -o -name "nginx.conf" -o -name "Dockerfile" -o -name "docker-compose*" \
\) -not -path "*/node_modules/*" -not -path "*/vendor/*" \
   -not -path "*/.git/*" | head -60

# Detect hardcoded secrets in config files (Grep only)
grep -rn "password\s*=\s*['\"][^'\"]\|secret\s*=\s*['\"][^'\"]\|\
api_key\s*=\s*['\"][^'\"]\|token\s*=\s*['\"][^'\"]\|\
AWS_SECRET\|PRIVATE_KEY\|DATABASE_URL.*://.*:.*@" \
  <target-dir> --include="*.env" --include="*.yml" --include="*.yaml" \
  --include="*.json" --include="*.py" --include="*.js" \
  | grep -v node_modules | grep -v ".example\|.sample\|.template" | head -40

# Security headers / CORS config
grep -rn "cors\|CORS\|Access-Control\|helmet\|Content-Security-Policy\|\
X-Frame-Options\|allowedOrigins\|origin.*\*" \
  <target-dir> --include="*.js" --include="*.ts" --include="*.py" \
  --include="*.java" --include="*.go" | grep -v node_modules | head -40

# TLS / SSL configuration
grep -rn "ssl\|tls\|https\|verify=False\|InsecureSkipVerify\|rejectUnauthorized\|\
CERT_NONE\|checkCert\|sslMode" \
  <target-dir> --include="*.py" --include="*.js" --include="*.ts" \
  --include="*.go" --include="*.java" | head -30
```

---

## Step 11 — Database structure mapping

```bash
# ORM model / schema definitions
grep -rn "db\.Model\|@Entity\|@Table\|models\.Model\|Schema(\|\
createTable\|sequelize\.define\|typeorm\|mongoose\.Schema\|\
gorm\.Model\|struct.*gorm\|@Column\|@PrimaryKey" \
  <target-dir> --include="*.py" --include="*.js" --include="*.ts" \
  --include="*.java" --include="*.kt" --include="*.go" | head -60

# Migration files
find <target-dir> -path "*/migrations/*" -name "*.py" \
  -o -path "*/migrations/*" -name "*.sql" \
  -o -path "*/db/migrate/*" -name "*.rb" \
  -o -name "*.migration.ts" -o -name "*_migration.go" \
  | head -30

# Raw SQL queries (flag for injection review)
grep -rn "SELECT\|INSERT\|UPDATE\|DELETE\|DROP\|CREATE TABLE\|ALTER" \
  <target-dir> --include="*.py" --include="*.js" --include="*.ts" \
  --include="*.java" --include="*.kt" --include="*.go" --include="*.php" \
  | grep -v node_modules | grep -v vendor | grep -v migrations | head -50

# Connection strings / DSNs
grep -rn "postgresql://\|mysql://\|mongodb://\|redis://\|sqlite://\|\
MongoClient\|createConnection\|getConnection\|DataSource\|DriverManager" \
  <target-dir> --include="*.py" --include="*.js" --include="*.ts" \
  --include="*.java" --include="*.kt" --include="*.go" | head -30
```

---

## Step 12 — Security hot-spot aggregation

Compile all findings from Steps 5–11 into a prioritised hot-spot list.
Format: `file:line | category | snippet (≤ 60 chars) | risk-signal`.

**Categories:**
- `SINK-SQL` — raw SQL composition near user input
- `SINK-CMD` — command execution
- `SINK-TMPL` — template/code injection
- `SINK-FILE` — path traversal candidate
- `SINK-DESER` — unsafe deserialization
- `SINK-REDIRECT` — open redirect candidate
- `SINK-XSS` — reflected/stored XSS candidate
- `SOURCE` — external input entry point
- `VALIDATOR` — sanitizer / validator (safe pattern — feeds rule allow-list)
- `CONFIG-SECRET` — potential hardcoded credential
- `CRYPTO-WEAK` — weak algorithm usage
- `AUTH-GATE` — auth check (feed as sanitizer/guard into taint rules)
- `BIZ-PAYMENT` — payment logic (high-value review target)

---

## Step 13 — Write CODE_RECON.md

Write `<target-dir>/opengrep-rules-creator/CODE_RECON.md`:

```markdown
# Source Code Recon Report
Generated: <ISO8601>
Target: <target-dir>
Depth: <fast|balanced|deep> (<auto|manual>)
Config: <found at path | not found — defaults used>

## Project Summary

| Field | Value |
|---|---|
| Project name | <dir name or config value> |
| Detected languages | <list> |
| Detected frameworks | <list> |
| Total source files | <N> |
| Files excluded | <N> |
| Locales detected | <en / non-ASCII files: N> |
| Scan timestamp | <ISO8601> |

## Project Context
<paste project_context block from config, or "Not configured.">

## Active Exclusions
<list of all active exclusion globs>

## Language & Framework Inventory

| Language | Files | Frameworks/Runtimes |
|---|---|---|

## Dependency Inventory
<omitted if --no-deps>

### All Dependencies (<N> total)
| Package | Version | Category |
|---|---|---|

### Security-Interesting Dependencies
| Package | Version | Risk Reason |
|---|---|---|

## Entry Points

| File | Line | Method / Route | Framework | Input Variables |
|---|---|---|---|---|

## Data Flow Map

### Taint Sources (external input)
<source file:line → description>

### Dangerous Sinks
| File | Line | Category | Sink Pattern | Apparent Safe? |
|---|---|---|---|---|

### Source → Sink Chains (deep/balanced only)
<For each chain: source:line → [fn1:line, fn2:line] → sink:line>

## Validators & Sanitizers

| File | Line | Function / Library | Type | Patterns Covered |
|---|---|---|---|---|

## Auth / Trust Boundaries
<middleware / decorators that gate access — used as sanitizers in taint rules>

| File | Line | Mechanism | Scope |
|---|---|---|---|

## Data Processing Functions

### Serialization / Deserialization
| File | Line | Library | Operation | Safe? |
|---|---|---|---|---|

### Cryptographic Operations
| File | Line | Algorithm | Operation | Concern |
|---|---|---|---|---|

### File Handling
| File | Line | Operation | Path Source |
|---|---|---|---|

## Business Logic Map

### High-Value Areas
| Domain | Files | Functions/Routes | Rule Opportunity |
|---|---|---|---|

## Configuration Files

### Config File Inventory
| File | Type | Sensitive? |
|---|---|---|

### Potential Hardcoded Secrets
<file:line references — values redacted>

### Security Header / CORS Config
<file:line references>

### TLS / SSL Flags
<file:line references>

## Database Structure

### ORM Models / Schemas
| File | Model Name | Key Fields |
|---|---|---|

### Raw SQL Queries
<file:line — categorised: parameterized vs concatenated>

### Connection Strings
<file:line references — credentials redacted>

## Security Hot Spots

| Priority | File:Line | Category | Signal | Rule Opportunity |
|---|---|---|---|---|

## Rule Opportunities — Recommended Starting Points

Based on the recon above, the following areas have the strongest signal for
Opengrep rules. Ordered by expected ROI (high signal + low false-positive risk):

| # | Area | Type | Files Affected | OWASP | Notes |
|---|---|---|---|---|---|

## Metadata for Downstream Stage

```yaml
# Consumed automatically by /opengrep-rule-creator
languages: [<list>]
frameworks: [<list>]
entry_point_files: [<list of up to 20 key files>]
taint_sources:
  - pattern: <source pattern>
    language: <lang>
taint_sanitizers:
  - pattern: <sanitizer pattern>
    language: <lang>
taint_sinks:
  - pattern: <sink pattern>
    category: <SINK-*>
    language: <lang>
known_issues_file: <path or null>
output_dir: opengrep-rules-creator
```

## Notes for /opengrep-rule-creator
- Recon depth: <fast|balanced|deep>
- Highest-priority rule opportunities: <top 3 one-liners>
- Non-obvious trust boundaries: <notes>
- Locales / encoding: <notes for pattern writing>
```

---

## Step 14 — Hand back

Tell the operator:

1. **Languages detected:** list with file counts.
2. **Frameworks:** list.
3. **Entry points found:** N total (top 3 files with most routes).
4. **Dangerous sinks:** N total by category (SINK-SQL: X, SINK-CMD: Y, …).
5. **Validators/sanitizers found:** N (important — reduces false positives in rules).
6. **Security-interesting dependencies:** list.
7. **Hot spots:** top 5 with file:line.
8. **Output written to:** `<path>/CODE_RECON.md`
9. **Suggested next step:**
   - `> /opengrep-rule-creator <target-dir>` — proceed to rule creation

---

## Constraints

- **Never read a file in full during discovery.** Glob/Grep for discovery;
  targeted `head` or `grep` for extraction.
- **Never execute any code from the target.** Read-only.
- **Never follow symlinks or `..` paths.** Stay within `<target-dir>`.
- **Redact credential values.** Record file:line references only — never the
  actual secret value — in CODE_RECON.md.
- **Cap per-category grep output at 100 lines.** Add `| head -N` to every
  grep pass.
- **At `fast` depth with > 5 000 files:** sample as described in Step 2 and
  note the sampling prominently in CODE_RECON.md.
