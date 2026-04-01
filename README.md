# Vibe Coding Security Scanner

A security audit skill for AI-assisted ("vibe-coded") web application codebases.

It works through 30+ vulnerability categories — from unauthenticated API routes and SQL injection to missing security headers and weak cryptography — and produces a severity-ranked report with working proof-of-concept exploit commands and step-by-step remediation guidance.

---

## What it checks

### 🔴 Critical
- API key & secret exposure (hardcoded secrets, leaked service role keys)
- SQL injection via string concatenation
- Authentication bypass (no server-side auth checks, JWT misuse)
- Broken access control / IDOR
- Command injection
- Sensitive data exposure (plaintext passwords, leaked API responses)
- RLS / database rules misconfiguration (Supabase, Firebase)
- Session hijacking
- Git secrets leaked in commit history

### 🟠 High
- Cross-Site Scripting (XSS) via `dangerouslySetInnerHTML`
- Weak or missing authentication controls
- Missing security headers (CSP, HSTS, X-Frame-Options, etc.)
- CORS misconfiguration
- CSRF vulnerabilities
- Brute force / credential stuffing (missing rate limiting)
- Insecure Direct Object Reference (IDOR)
- API security issues (over-exposure, GraphQL introspection)
- File upload security

### 🟡 Medium
- Insecure error handling (stack traces leaked to clients)
- Insecure cookies (missing HttpOnly/Secure/SameSite)
- Missing rate limiting
- Information disclosure
- Source map exposure in production
- Clickjacking
- Dependency vulnerabilities (`npm audit`)

### 🔵 Best Practices
- Environment & configuration hygiene
- Logging & monitoring
- Input validation (Zod, Joi, server-side)
- HTTPS & transport security
- Least privilege principles

### 🟤 Advanced
- JWT security (algorithm confusion, localStorage storage, token rotation)
- OAuth security (state param, PKCE, redirect URI validation)
- WebSocket security
- Subresource Integrity (SRI)
- SSRF (Server-Side Request Forgery)
- Open redirect
- Email security (SPF, DKIM, DMARC)
- DNS security
- AI prompt security (CVE-2025-48757 pattern — tables without RLS)

### Platform-specific checks
Supabase · Firebase · Vercel · Netlify · Next.js · Lovable · Bolt.new · Replit · v0.dev · Cursor · GitHub Copilot · MongoDB · PostgreSQL · Railway · Render · Fly.io · Bubble · Webflow · Framer · and more.

---

## Report output

For every finding the report includes:

- **Severity** (Critical / High / Medium / Low)
- **File path and line number**
- **Vulnerable code snippet**
- **Exploit scenario** — plain-English explanation of how an attacker abuses this
- **Proof of Concept** — working `curl` command or script using your app's actual URL, with the expected response shown
- **Fix** — specific remediation with code examples

---

## Installation

### Prerequisites
- An AI coding assistant that supports skills/custom prompts (e.g. any tool that reads from `~/.claude/skills/`)

### Install the skill

```bash
# Create the skills directory if it doesn't exist
mkdir -p ~/.claude/skills/vibecodingscanner

# Download the skill
curl -fsSL https://raw.githubusercontent.com/funky-monkey/vibecoding-security-scanner/main/SKILL.md \
  -o ~/.claude/skills/vibecodingscanner/SKILL.md
```

Or clone the repo and symlink:

```bash
git clone https://github.com/funky-monkey/vibecoding-security-scanner.git
mkdir -p ~/.claude/skills/vibecodingscanner
ln -s "$(pwd)/vibecoding-security-scanner/SKILL.md" ~/.claude/skills/vibecodingscanner/SKILL.md
```

Verify it's installed:

```bash
ls ~/.claude/skills/vibecodingscanner/SKILL.md
```

---

## Usage

### Basic scan — your own codebase

Open your AI assistant in the project root and type:

```
/vibecodingscanner
```

The skill will ask for the codebase path if it's not clear from context, then work through Phase 1 → Phase 2 → Phase 3 automatically.

### Scan a specific directory

```
scan /Users/you/projects/my-nextjs-app with vibecodingscanner
```

### Scan with a known base URL (for PoC exploit links)

Providing your deployment URL means every exploit example in the report uses your real endpoints as clickable links:

```
scan /Users/you/projects/my-app using vibecodingscanner, base URL is https://my-app.vercel.app
```

### Save the report as HTML and Markdown

```
scan /Users/you/projects/my-app with vibecodingscanner and save the report as SECURITY-REPORT.md and SECURITY-REPORT.html in the project root
```

### Example prompt (full)

```
Use vibecodingscanner to audit /Users/you/projects/my-nextjs-app.
The app is deployed at https://my-app.vercel.app.
Save the report as both SECURITY-REPORT.md and SECURITY-REPORT.html in the project root.
```

---

## Example report output

```
# Security Audit Report — my-nextjs-app

Date: 2026-04-01
Auditor: vibecodingscanner
Base URL: https://my-app.vercel.app

## Summary
| Severity  | Count |
|-----------|-------|
| 🔴 Critical | 3   |
| 🟠 High     | 5   |
| 🟡 Medium   | 4   |
| ✅ Passing  | 12  |

---

### CRIT-1 — All Admin API Routes Are Unauthenticated
- **Location:** src/app/api/admin/users/route.ts:4
- **Evidence:** `export async function GET() { const admin = getAdminClient() ...`
- **Exploit PoC:**
  ```bash
  curl https://my-app.vercel.app/api/admin/users
  # Returns: [{"id":"...","email":"admin@example.com","rol":"beheerder"}]
  ```
- **Fix:** Add a requireAdminAuth() guard at the top of every admin route.
```

---

## Three-phase methodology

The scanner follows a strict three-phase process — it never modifies code before completing the full analysis:

**Phase 1 — Systematic Analysis**
Reads every relevant file. For each item in the checklist, records the file path, line number, vulnerable snippet, and exploitability assessment.

**Phase 2 — Risk Planning**
Documents every failing check with: evidence, exploit scenario, working PoC, fix steps, and regression risk.

**Phase 3 — Remediation** *(optional, run separately)*
Applies only the fixes identified — no cosmetic changes, no scope creep.

---

## Sources

This checklist is compiled from:

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [vibeappscanner.com](https://vibeappscanner.com) — vulnerability library, checklists, guides, best practices
- [astoj/vibe-security](https://github.com/astoj/vibe-security)
- [Replit Vibe Code Security Checklist](https://docs.replit.com/tutorials/vibe-code-security-checklist)
- [namanyayg security audit prompt](https://gist.github.com/namanyayg/ed12fa79f535d0294f4873be73e7c69b)
- CVE-2025-48757 (Supabase tables without RLS — affected 170+ Lovable apps)

---

## Contributing

Pull requests welcome. If you find a vulnerability pattern that's missing, open an issue or add it to `SKILL.md` under the appropriate severity section.

---

## License

MIT
