# 🔬 bk-sast-workflow

> **BakeFoundry GitHub Composite Action** — CodeQL Static Application Security Testing (SAST) for your CI/CD pipeline.

---

## 📌 Repository

🔗 **GitHub:** [BakeFoundry/bk-sast-workflow](https://github.com/BakeFoundry/bk-sast-workflow)

---

## 🤔 What is SAST and Why Do You Need It?

**SAST (Static Application Security Testing)** analyzes your source code **without executing it** to find security vulnerabilities *before* they reach production.

### 🚨 Why SAST is Required

| Without SAST | With SAST (CodeQL) |
|---|---|
| Security bugs silently enter production | Caught at PR time, before merge |
| Code reviews miss subtle vulnerabilities | Automated 250+ security query checks |
| Manual audits are slow and inconsistent | Runs in every CI pipeline automatically |
| Bugs cost 10–100x more to fix post-release | Fixed in development (cheapest stage) |
| Compliance failures (SOC2, PCI-DSS, ISO 27001) | Evidence of automated security controls |

### 🔍 What CodeQL Detects

CodeQL uses **semantic analysis** (not just text matching) — it understands how data flows through your program:

| Vulnerability Type | Example |
|---|---|
| **SQL Injection** | User input passed unsanitized into a DB query |
| **Cross-Site Scripting (XSS)** | Unescaped user data rendered in HTML |
| **Command Injection** | User input passed to `os.system()` or `exec()` |
| **Path Traversal** | `../../etc/passwd` style directory escape |
| **Insecure Deserialization** | Untrusted data deserialized unsafely |
| **Hardcoded Credentials** | Passwords/API keys in source code |
| **Weak Cryptography** | MD5/SHA1 used for password hashing |
| **Unvalidated Redirects** | User-controlled redirect URLs |
| **Race Conditions** | TOCTOU and thread-safety issues |

---

## 🏗️ How CodeQL Works

```
Your Source Code
      │
      ▼
┌─────────────────────────────────────────┐
│  STEP 1 — BUILD CODEQL DATABASE         │
│  CodeQL compiles/parses your code into  │
│  a relational database capturing:       │
│   • Data flow (where data travels)      │
│   • Control flow (execution paths)      │
│   • Call graph (who calls what)         │
└─────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────┐
│  STEP 2 — RUN SECURITY QUERIES          │
│  Pre-built queries written in the       │
│  CodeQL query language scan the DB:     │
│   • security-extended suite (250+ rules)│
│   • Taint tracking across functions     │
│   • Cross-file vulnerability detection  │
└─────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────┐
│  STEP 3 — GENERATE SARIF RESULTS        │
│  SARIF = Static Analysis Results        │
│  Interchange Format (standard JSON)     │
│   • File + line number of each finding  │
│   • Severity level                      │
│   • Remediation advice                  │
└─────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────┐
│  STEP 4 — UPLOAD & REPORT               │
│   • SARIF uploaded to GitHub Security   │
│     tab (Code Scanning Alerts)          │
│   • Summary posted as PR comment        │
│   • Quality gate fails build if high-   │
│     severity findings are found         │
└─────────────────────────────────────────┘
```

---

## ✅ Supported Languages

| Language | Auto-Detected By |
|---|---|
| JavaScript / TypeScript | `*.js`, `*.ts`, `*.jsx`, `*.tsx` |
| Python | `*.py` |
| Java / Kotlin | `*.java`, `*.kt` |
| Go | `*.go` |
| C# | `*.cs` |
| Ruby | `*.rb` |
| C / C++ | `*.c`, `*.cpp`, `*.h` |

---

## 🚀 Usage

### Basic Usage (Auto-detect Language)

```yaml
- name: Run SAST Scan
  uses: BakeFoundry/bk-sast-workflow@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Advanced Usage (Interpreted Languages)

```yaml
- name: Run SAST Scan
  uses: BakeFoundry/bk-sast-workflow@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    languages: "python,javascript"          # Optional: override auto-detection
    severity-threshold: "high"              # Fail on high or above (default)
```

### Advanced Usage (Compiled Languages — Manual Build)

For compiled languages (Java, C#, C/C++, Go) where CodeQL needs to trace an actual build:

```yaml
- name: Run SAST Scan
  uses: BakeFoundry/bk-sast-workflow@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    languages: "java"
    build-mode: "manual"                    # Disable autobuild; use your own build command
    build-command: "mvn clean package -DskipTests"
    severity-threshold: "high"
```

### Full Pipeline Integration

```yaml
name: Security & CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  secret-scan:
    name: Secret Scanning
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: BakeFoundry/bk-secret-scan-workflow@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}

  sast-scan:
    name: SAST Scan (CodeQL)
    runs-on: ubuntu-latest
    needs: secret-scan              # Run after secret scan
    permissions:
      security-events: write        # Required to upload SARIF results
      contents: read
      pull-requests: write          # Required to post PR comments
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0            # Full history needed for accurate analysis
      - uses: BakeFoundry/bk-sast-workflow@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          severity-threshold: "high"

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [secret-scan, sast-scan]  # Only build after security gates pass
    steps:
      - uses: actions/checkout@v4
      - name: Build Application
        run: make build
```

---

## ⚙️ Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `github-token` | ✅ Yes | — | GitHub token for SARIF upload and PR comments |
| `languages` | ✅ Yes | — | Comma-separated list of languages to scan (e.g. `python,javascript`) |
| `build-mode` | ❌ No | `none` | `none` for interpreted languages (Python, JS, Ruby) · `autobuild` for compiled (Java, C#, C++, Go) · `manual` for custom build |
| `build-command` | ❌ No | `""` | Build command to run when `build-mode` is `manual` (e.g. `mvn package`) |
| `severity-threshold` | ❌ No | `high` | Minimum severity to fail the build (`critical`, `high`, `medium`, `low`, `note`) |

> **`build-mode` quick guide:**
> | Language | `build-mode` to use |
> |---|---|
> | Python, JavaScript, TypeScript, Ruby | `none` *(default — no action needed)* |
> | Java, C#, C++, Go, Swift | `autobuild` or `manual` |

## 📤 Outputs

| Output | Description |
|---|---|
| `sarif-file` | Path to the generated SARIF results file |
| `findings-count` | Total number of findings discovered |

---

## 🔐 Required GitHub Permissions

Add these permissions to your workflow job:

```yaml
permissions:
  security-events: write   # Upload SARIF to code scanning
  contents: read           # Read repository contents
  pull-requests: write     # Post PR comments
```

---

## 📊 What Developers See

### PR Comment (on every pull request)

```
✅ CodeQL SAST Scan Results
No security vulnerabilities found! Your code looks clean. 🎉

| Severity | Count |
|----------|-------|
| 🔴 High  |  0    |
| 🟠 Medium|  0    |
| 🟡 Low   |  0    |
| Total    |  0    |

Languages Scanned: `python`
Query Suite: `security-extended`

📋 Full Details: View in GitHub Security Tab | View Workflow Run
```

### What Approvers See

- ✅ **GREEN** badge on PR if quality gate passes
- ❌ **RED** build failure if high-severity vulnerabilities are found
- 🔍 **GitHub Security tab** → Code Scanning Alerts with full details, file locations, and fix recommendations

---

## 🏷️ Severity Threshold Guide

| Threshold | Build Fails When |
|---|---|
| `critical` | Only on critical findings |
| `high` | High or critical findings *(recommended default)* |
| `medium` | Medium, high, or critical findings |
| `low` | Any finding at all |

---

## 📈 Pipeline Flow

```
Developer Commits Code
        │
        ▼
  ┌─────────────┐     FAIL ──► 🔔 Notify Developer
  │ Secret Scan │
  └──────┬──────┘
         │ PASS
         ▼
  ┌─────────────┐     FAIL ──► 🔔 Notify Developer
  │  SAST Scan  │◄── This Action
  │  (CodeQL)   │
  └──────┬──────┘
         │ PASS
         ▼
  ┌─────────────┐     FAIL ──► 🔔 Notify Developer
  │  SonarQube  │
  └──────┬──────┘
         │ PASS
         ▼
  ┌─────────────┐
  │    Build    │
  └──────┬──────┘
         │ PASS
         ▼
  ┌─────────────┐
  │  AMI Bake   │
  │ Image Scan  │
  └──────┬──────┘
         │ PASS
         ▼
      Deploy ✅
```

---

## 🔧 Internals

This composite action uses **CodeQL Action v4** internally:

| Internal Step | Action Used |
|---|---|
| Initialize CodeQL | `github/codeql-action/init@v4` |
| Auto-Build | `github/codeql-action/autobuild@v4` |
| Run Analysis | `github/codeql-action/analyze@v4` |
| Upload SARIF | `github/codeql-action/upload-sarif@v4` |

---

## 🔗 Related Actions

| Action | Purpose | Repository |
|---|---|---|
| `bk-secret-scan-workflow` | Detect hardcoded secrets using Trivy | [BakeFoundry/bk-secret-scan-workflow](https://github.com/BakeFoundry/bk-secret-scan-workflow) |
| `bk-sast-workflow` | Code vulnerability scanning using CodeQL | *This repository* |
