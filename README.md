# Vibe Coding Security Scanner

A security audit skill for AI-assisted ("vibe-coded") web application codebases. It works through 30+ vulnerability categories and produces a severity-ranked report with working proof-of-concept exploit commands and step-by-step remediation guidance.

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
- AI prompt security (CVE-2025-48757 pattern, tables without RLS)

### Platform-specific checks
Supabase, Firebase, Vercel, Netlify, Next.js, Lovable, Bolt.new, Replit, v0.dev, Cursor, GitHub Copilot, MongoDB, PostgreSQL, Railway, Render, Fly.io, Bubble, Webflow, Framer, and more.

---

## Report output

For every finding the report includes a severity rating, file path and line number, the vulnerable code snippet, a plain-English exploit scenario, a working `curl` proof-of-concept using your app's actual URL, and a specific fix with code examples.

---

## Installation

This is a skill file for **Claude Code**. Install Claude Code first, then drop the skill into the right directory and it becomes available as a slash command.

### Step 1 - Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

Or download the desktop app from [claude.ai/code](https://claude.ai/code).

### Step 2 - Install the skill

**Option A: one-liner**

```bash
mkdir -p ~/.claude/skills/vibecodingscanner && \
  curl -fsSL https://raw.githubusercontent.com/funky-monkey/vibecoding-security-scanner/main/SKILL.md \
  -o ~/.claude/skills/vibecodingscanner/SKILL.md
```

**Option B: clone and symlink** (easier to update later)

```bash
git clone https://github.com/funky-monkey/vibecoding-security-scanner.git ~/vibecoding-security-scanner
mkdir -p ~/.claude/skills/vibecodingscanner
ln -s ~/vibecoding-security-scanner/SKILL.md ~/.claude/skills/vibecodingscanner/SKILL.md
```

### Step 3 - Verify

```bash
ls ~/.claude/skills/vibecodingscanner/SKILL.md
```

---

## Skills included

This repo contains two skills that work together:

- `vibecodingscanner` - full codebase audit, produces a severity-ranked report
- `vibecodingexploits` - generates working exploit chains for individual findings

Install both:

```bash
mkdir -p ~/.claude/skills/vibecodingscanner && \
  curl -fsSL https://raw.githubusercontent.com/funky-monkey/vibecoding-security-scanner/main/SKILL.md \
  -o ~/.claude/skills/vibecodingscanner/SKILL.md

mkdir -p ~/.claude/skills/vibecodingexploits && \
  curl -fsSL https://raw.githubusercontent.com/funky-monkey/vibecoding-security-scanner/main/vibecodingexploits/SKILL.md \
  -o ~/.claude/skills/vibecodingexploits/SKILL.md
```

---

## Usage

### Step 1 - Run the scanner

Open Claude Code inside your project and run:

```
/vibecodingscanner
```

The skill works through Phase 1 (analysis), Phase 2 (risk planning), and Phase 3 (remediation) automatically. It never modifies code before completing the full analysis.

The report is saved to `_security/SECURITY-REPORT.md` and `_security/SECURITY-REPORT.html` in your project. The `_security/` folder is automatically added to `.gitignore`.

**Scan a specific path:**

```
scan /path/to/my-app with vibecodingscanner
```

**Include your deployment URL for clickable exploit links:**

```
scan /path/to/my-app using vibecodingscanner, base URL is https://my-app.vercel.app
```

### Step 2 - Generate exploit chains

At the bottom of every report there is an Exploit Chains section with ready-to-copy lines. Paste any line into Claude Code:

```
/vibecodingexploits HIGH-4 xss-stored https://my-app.vercel.app/api/cms content
/vibecodingexploits HIGH-5 csrf https://my-app.vercel.app/api/admin/posts
/vibecodingexploits HIGH-6 open-redirect https://my-app.vercel.app/auth/callback to_path
```

The exploit skill produces:

- HTML bait pages for XSS, CSRF, clickjacking, WebSocket hijacking
- Shell scripts for JWT forgery and command injection
- curl commands for open endpoints, SSRF, and session replay
- Receiver setup instructions (webhook.site, netcat, python http.server)

All output files are saved to `_security/` alongside the report.

---

## Sources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [vibeappscanner.com](https://vibeappscanner.com)
- [astoj/vibe-security](https://github.com/astoj/vibe-security)
- [Replit Vibe Code Security Checklist](https://docs.replit.com/tutorials/vibe-code-security-checklist)
- [namanyayg security audit prompt](https://gist.github.com/namanyayg/ed12fa79f535d0294f4873be73e7c69b)
- CVE-2025-48757

---

## Contributing

Pull requests welcome. If you find a missing vulnerability pattern, open an issue or add it to `SKILL.md` under the appropriate severity section.

---

## License

MIT
