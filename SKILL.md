---
name: vibecodingscanner
description: Full security audit skill for vibe-coded / AI-generated web apps. Scans a codebase across 30+ vulnerability categories and produces a severity-ranked report with remediation steps. Trigger whenever the user asks for a security scan, audit, or review of a web app codebase.
---

# Vibe Coding Security Scanner

You are performing a **professional security audit** of an AI-assisted / vibe-coded web application codebase. Follow the three-phase methodology below **exactly and in order**. Do not skip phases, do not make code changes before completing the analysis phase.

---

## HOW TO USE THIS SKILL

### Before you start — always ask these two questions first

Before doing any analysis, ask the user:

1. **Codebase path** — "What is the path to the codebase you want me to scan?" (skip if already provided in the invocation)
2. **Live / staging URL** — "What is the live or staging URL of this app?" (e.g. `https://my-app.vercel.app`) — used for clickable links and working curl PoCs in the report; answer "unknown" to skip

Wait for both answers before proceeding to Phase 1.

### Then proceed in order:
1. Use the confirmed codebase path as the root for all file reads
2. Work through **Phase 1 → Phase 2 → Phase 3**
3. Output the full **Security Report** at the end, using the provided URL as `BASE_URL` throughout
4. Always save the report as both `SECURITY-REPORT.md` and `SECURITY-REPORT.html` inside a `_security/` folder in the project root (create the folder if it does not exist)
5. Always append `_security/` to the project's `.gitignore` (create `.gitignore` if it does not exist) — the folder contains exploit PoCs and must never be committed

---

## PHASE 1 — SYSTEMATIC ANALYSIS

Read the codebase. Search for every item in the checklist below. For each finding:
- Record the **file path and line number**
- Note the **vulnerability type and severity**
- Collect the **exact vulnerable code snippet**
- Assess **exploitability** (can it be triggered from outside?)

Do NOT modify any code in this phase.

---

## MASTER SECURITY CHECKLIST

Work through every category. Mark each item as ✅ PASS, ❌ FAIL, or ⚠️ REVIEW.

---

### 🔴 CRITICAL

#### C1 — API Key & Secret Exposure
- [ ] Scan ALL files (including `.js`, `.ts`, `.jsx`, `.tsx`, `.env*`, config files) for hardcoded secrets
- [ ] Flag any key matching patterns: `sk-`, `AKIA`, `AIza`, `sk_live_`, `pk_live_` appearing in non-`.env` files
- [ ] Check for `process.env.*` values being passed to frontend bundles via `NEXT_PUBLIC_`, `VITE_`, `REACT_APP_` prefixes for secrets that should be server-only
- [ ] Verify `.env` is in `.gitignore` and no `.env` file is committed
- [ ] Scan git history for accidentally committed secrets (`gitleaks`, `trufflehog` patterns)
- [ ] Check for `service_role` keys, admin SDK keys, or database passwords in client-side code
- [ ] Safe to expose: Supabase anon key, Firebase client apiKey, Stripe publishable key (`pk_`)
- [ ] Never expose: OpenAI `sk-`, Stripe `sk_`, AWS `AKIA`, any key with `secret`/`private` in name

#### C2 — SQL Injection
- [ ] Search for string concatenation in SQL queries: `` `SELECT * FROM ... ${userInput}` ``
- [ ] Check all raw query calls: `db.query()`, `client.query()`, `connection.execute()`
- [ ] Verify ORM raw escape hatches are parameterized: `prisma.$queryRaw`, `sequelize.query`
- [ ] Confirm all user-supplied values go through parameterized queries or ORM methods
- [ ] Validate numeric inputs with `parseInt`/`parseFloat` + `isNaN` check before use in queries

#### C3 — Authentication Bypass
- [ ] Verify every protected API route has server-side auth check (not just frontend guards)
- [ ] Check that JWT tokens are verified server-side (signature + expiry), not just decoded
- [ ] Look for endpoints that trust a `userId` or `role` value coming from the request body/query
- [ ] Check for sequential/predictable IDs (`/api/users/1`, `/api/orders/123`) without ownership validation
- [ ] Verify session expiry is enforced and sessions are invalidated on logout
- [ ] Check for missing auth middleware on sensitive routes

#### C4 — Broken Access Control
- [ ] Verify every data-fetching endpoint checks ownership: `WHERE id = ? AND user_id = ?`
- [ ] Check for IDOR: can user A access user B's data by changing an ID in the URL/request?
- [ ] Verify admin-only endpoints check role server-side, not just in UI
- [ ] Confirm RLS (Row Level Security) is enabled on all Supabase/PostgreSQL tables
- [ ] Check Firestore/Firebase security rules are not in test mode and restrict by `request.auth.uid`
- [ ] Look for client-side-only role checks (hiding UI ≠ enforcing access)
- [ ] Implement deny-by-default: grants must be explicit

#### C5 — Command Injection
- [ ] Search for `exec()`, `execSync()`, `spawn()`, `eval()` with user-controlled input
- [ ] Flag any use of `child_process` where input is concatenated into the command string
- [ ] Check template literal shell commands: `` exec(`git clone ${repoUrl}`) ``

#### C6 — Sensitive Data Exposure
- [ ] Check API responses: are passwords, hashes, internal tokens, or excessive user data returned?
- [ ] Verify passwords are hashed with bcrypt/argon2 (never MD5/SHA1, never plaintext)
- [ ] Check database fields: sensitive columns should be excluded from wildcard `SELECT *`
- [ ] Verify PII is not logged to console or sent to third-party logging services unmasked

#### C7 — RLS / Database Rules Misconfiguration (Supabase/Firebase)
- [ ] For every Supabase table: confirm `ALTER TABLE x ENABLE ROW LEVEL SECURITY;` is present
- [ ] Verify policies exist for SELECT, INSERT, UPDATE, DELETE separately
- [ ] Policies must use `auth.uid()` checks, not just boolean `true`
- [ ] For Firebase: no rule set to `allow read, write: if true;` in production
- [ ] Test by querying as anonymous user — should return zero rows

#### C8 — Session Hijacking
- [ ] Verify session tokens are sufficiently random (not guessable/sequential)
- [ ] Check cookie flags: `HttpOnly`, `Secure`, `SameSite=Strict` or `Lax`
- [ ] Confirm HTTPS is enforced before setting session tokens
- [ ] Check token rotation on privilege escalation (login, role change)

#### C9 — Git Secrets Leak
- [ ] Run: `git log --all --full-history -- "*.env"` to check if env files were ever committed
- [ ] Check for `.env`, `credentials.json`, `secrets.yaml`, `*.pem`, `*.key` in git history
- [ ] Verify pre-commit hooks or CI secret scanning (GitGuardian, GitHub secret scanning) is set up

---

### 🟠 HIGH

#### H1 — Cross-Site Scripting (XSS)
- [ ] Search for `dangerouslySetInnerHTML` — each instance needs DOMPurify sanitization
- [ ] Search for `innerHTML =`, `outerHTML =`, `document.write(` with user data
- [ ] Verify URL parameters are encoded before rendering
- [ ] Check for user-generated content rendered in the DOM without escaping
- [ ] Confirm Content-Security-Policy header is set (see H4)
- [ ] Stored XSS: data saved to DB then rendered without encoding is highest risk

#### H2 — Weak / Missing Authentication Controls
- [ ] Passwords: minimum length enforced (≥ 8 chars), complexity requirements exist
- [ ] Password reset uses time-limited tokens (≤ 1 hour), not guessable values
- [ ] Account lockout or exponential backoff after repeated failed logins
- [ ] MFA available for privileged accounts
- [ ] OAuth flows use `state` parameter to prevent CSRF
- [ ] Refresh tokens rotated and revoked on logout

#### H3 — Missing / Misconfigured Security Headers
Check response headers for every page and API endpoint:
- [ ] `Content-Security-Policy: default-src 'self'; script-src 'self'` (no unsafe-inline unless nonce)
- [ ] `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- [ ] `X-Frame-Options: DENY` (or `SAMEORIGIN`)
- [ ] `X-Content-Type-Options: nosniff`
- [ ] `Referrer-Policy: strict-origin-when-cross-origin`
- [ ] `Permissions-Policy` restricting camera/mic/geolocation as needed
- [ ] No `Server` or `X-Powered-By` headers leaking stack info

Platform config locations:
- Next.js: `next.config.js` → `headers()`
- Vercel: `vercel.json` → `headers`
- Netlify: `_headers` file
- nginx: `add_header` directives
- Express: `helmet` middleware

#### H4 — CORS Misconfiguration
- [ ] No `Access-Control-Allow-Origin: *` combined with `Access-Control-Allow-Credentials: true`
- [ ] Whitelist only known trusted origins (not `*` for credentialed requests)
- [ ] Verify `Access-Control-Allow-Methods` is restricted to needed methods only
- [ ] Dynamic origin validation must use an allowlist, not reflect arbitrary `Origin` headers
- [ ] Check preflight OPTIONS handling

#### H5 — CSRF (Cross-Site Request Forgery)
- [ ] All state-changing requests (POST/PUT/DELETE) use CSRF tokens or `SameSite=Strict` cookies
- [ ] Forms include anti-CSRF tokens generated server-side and validated on submission
- [ ] Cookie-based auth uses `SameSite=Strict` or `SameSite=Lax`
- [ ] Check JSON APIs: Content-Type validation prevents simple-form CSRF

#### H6 — Brute Force / Credential Stuffing
- [ ] Rate limiting on `/login`, `/register`, `/forgot-password`, `/api/auth/*`
- [ ] Rate limit is per-IP and per-account (both axes)
- [ ] Typical threshold: 5-10 attempts per 15-minute window
- [ ] CAPTCHA or proof-of-work on auth endpoints after threshold
- [ ] Check for credential stuffing protection (breach password database check)

#### H7 — IDOR (Insecure Direct Object Reference)
- [ ] Every resource fetch query includes ownership check: `WHERE id = $1 AND owner_id = $2`
- [ ] UUIDs alone are insufficient — ownership must be verified
- [ ] Test: can authenticated user A access user B's resources by changing the ID?
- [ ] Bulk endpoints (lists) must be filtered by the authenticated user's scope

#### H8 — NoSQL Injection (MongoDB/Document DBs)
- [ ] Check Mongoose/MongoDB queries using `req.body` directly as query filter
- [ ] Sanitize objects: user input should not be spread directly into query objects
- [ ] Use `mongoose-sanitize` or validate that input is a scalar string/number, not an object
- [ ] Example risk: `User.find({ email: req.body.email })` where `email` is `{ $gt: "" }`

#### H9 — API Security
- [ ] All sensitive endpoints require authentication (not just UI-level guards)
- [ ] GraphQL: disable introspection in production, implement query depth/complexity limits
- [ ] REST: validate and restrict HTTP methods per endpoint
- [ ] API versioning doesn't leave old insecure endpoints alive
- [ ] Verify response payloads don't over-expose fields (select only needed columns)

#### H10 — File Upload Security
- [ ] Validate file MIME type server-side (not just extension or Content-Type header)
- [ ] Enforce file size limits
- [ ] Generate server-side filenames — never use user-provided filenames
- [ ] Store uploads outside the web root or in object storage (S3, Supabase Storage)
- [ ] Scan uploads for malware before serving
- [ ] Prevent path traversal: `filename.replace(/\.\./g, '')`

---

### 🟡 MEDIUM

#### M1 — Insecure Error Handling
- [ ] Production errors return generic messages (`"Something went wrong"`) — never stack traces
- [ ] Internal errors are logged server-side only
- [ ] No database errors, file paths, or framework version info leaked to clients
- [ ] `try/catch` blocks in API routes return consistent, non-descriptive error shapes

#### M2 — Insecure Cookies
- [ ] All auth cookies set with `HttpOnly` flag (no JS access)
- [ ] All cookies set with `Secure` flag (HTTPS only)
- [ ] All cookies set with `SameSite=Strict` or `SameSite=Lax`
- [ ] Sensitive cookies not accessible to subdomains unless needed

#### M3 — Missing Rate Limiting
- [ ] Rate limiting on ALL public API endpoints, not just auth
- [ ] Different limits for read vs. write operations
- [ ] 429 responses include `Retry-After` header
- [ ] Implement with `express-rate-limit`, `rate-limiter-flexible`, or edge middleware

#### M4 — Information Disclosure
- [ ] Remove `X-Powered-By`, `Server` response headers
- [ ] Disable directory listing on file servers
- [ ] Verify build artifacts don't expose source maps in production (`//# sourceMappingURL`)
- [ ] Check `robots.txt` doesn't reveal sensitive admin/API paths
- [ ] No version numbers in error messages or headers

#### M5 — Source Map Exposure
- [ ] Confirm production builds have source maps disabled or restricted
- [ ] Next.js: `productionBrowserSourceMaps: false` in `next.config.js`
- [ ] Vite: `build.sourcemap: false` in `vite.config.js`
- [ ] If source maps needed for error tracking: serve only to authenticated monitoring tools

#### M6 — Clickjacking
- [ ] `X-Frame-Options: DENY` set (covered in H3 but verify specifically)
- [ ] CSP `frame-ancestors 'none'` as modern alternative

#### M7 — Mixed Content
- [ ] No HTTP resources (scripts, images, fonts) loaded from HTTPS pages
- [ ] Check for hardcoded `http://` URLs in frontend code
- [ ] CSP `upgrade-insecure-requests` directive set

#### M8 — Dependency Vulnerabilities
- [ ] Run `npm audit` (or `yarn audit` / `pnpm audit`) — zero high/critical findings
- [ ] Check for packages known to be malicious or abandoned
- [ ] Verify `package-lock.json` / `yarn.lock` is committed and up to date
- [ ] Set up Dependabot or Snyk for ongoing monitoring
- [ ] Flag any package installed from a non-registry source (git URLs, local paths)

---

### 🔵 BEST PRACTICES / HARDENING

#### BP1 — Environment & Configuration
- [ ] All secrets use environment variables, never hardcoded
- [ ] `.env.example` exists with placeholder values (not real secrets)
- [ ] Different secrets for dev/staging/production environments
- [ ] Infrastructure-as-code scanned with Checkov or similar

#### BP2 — Logging & Monitoring
- [ ] Authentication events (login, logout, failed attempts) are logged
- [ ] Admin actions are logged with actor identity
- [ ] Logs are structured (JSON), timestamped, and stored securely
- [ ] Alerting on repeated failures or anomalous patterns (Sentry, Datadog, etc.)

#### BP3 — Input Validation (Defense in Depth)
- [ ] All user input is validated at the API boundary (type, length, format)
- [ ] Validation library used (Zod, Joi, Yup, class-validator)
- [ ] Input validation is server-side, not just client-side
- [ ] Numeric IDs validated as integers, emails validated as RFC-compliant

#### BP4 — HTTPS & Transport Security
- [ ] HTTPS enforced for all traffic (redirect HTTP → HTTPS)
- [ ] TLS certificate valid and auto-renewing
- [ ] HSTS header set (see H3)
- [ ] No mixed content (see M7)

#### BP5 — Least Privilege
- [ ] Database users have only the permissions they need (no `SUPERUSER` for app)
- [ ] API keys scoped to minimum required permissions
- [ ] Cloud IAM roles follow least privilege
- [ ] Service accounts don't share credentials across environments

#### BP6 — Backup & Recovery
- [ ] Automated database backups configured
- [ ] Backup restoration tested periodically
- [ ] Disaster recovery procedure documented

#### BP7 — Security Testing Integration
- [ ] SAST tool integrated in CI (SonarQube, Semgrep, ESLint security plugin)
- [ ] Dependency scanning in CI (Snyk, `npm audit --audit-level=high`)
- [ ] Secrets scanning in CI/pre-commit (GitGuardian, gitleaks)
- [ ] Use `npm ci` (not `npm install`) in CI/CD pipelines for reproducible, locked builds
- [ ] Security-critical packages (auth, crypto) pinned to exact versions — no `^` or `~`
- [ ] `npx depcheck` run periodically to identify and remove unused dependencies
- [ ] `package-lock.json` or `yarn.lock` committed to repository

#### BP8 — Output Encoding
- [ ] All user-generated content encoded for the correct context before rendering (HTML, JS, URL, CSS)
- [ ] Text content uses `textContent` not `innerHTML`
- [ ] URL parameters encoded with `encodeURIComponent` before use in links or redirects
- [ ] JSON responses set `Content-Type: application/json` to prevent MIME sniffing as HTML

#### BP9 — Cryptographic Standards
- [ ] No MD5 or SHA-1 used for any security purpose (passwords, tokens, signatures)
- [ ] Passwords hashed with bcrypt (cost ≥ 12) or argon2id
- [ ] Random tokens generated with `crypto.randomBytes()` or equivalent CSPRNG — not `Math.random()`
- [ ] TLS 1.2+ enforced; TLS 1.0 and 1.1 disabled
- [ ] Sensitive data encrypted at rest for any database containing PII

---

### 🟤 ADVANCED / OFTEN MISSED

#### ADV1 — JWT Security
- [ ] Signing algorithm explicitly specified and allowlisted during verification (prevents "none" algorithm attack)
- [ ] Use RS256 (asymmetric) for public APIs, or HS256 with a secret ≥ 32 bytes
- [ ] Access token expiry set to 15–30 minutes maximum
- [ ] Refresh tokens stored in `httpOnly; Secure; SameSite=Lax` cookies only — never localStorage
- [ ] Access tokens stored in memory only (JS variable) — never localStorage or sessionStorage
- [ ] All claims validated on every request: `iss`, `aud`, `exp`, `iat`
- [ ] Refresh tokens validated against a database (allows revocation)
- [ ] Token rotation on refresh (old refresh token invalidated)

#### ADV2 — OAuth Security
- [ ] Never implement OAuth from scratch — use Auth.js, Passport.js, Supabase Auth, Clerk, Auth0
- [ ] `state` parameter used and validated in OAuth callback (prevents CSRF)
- [ ] PKCE (Proof Key for Code Exchange) implemented for public clients (SPAs, mobile)
- [ ] Redirect URIs use exact matching — no wildcards, no prefix matching
- [ ] OAuth tokens stored in `httpOnly` cookies or server-side sessions — never localStorage
- [ ] `SameSite=Lax` (not `Strict`) on OAuth session cookies — Strict blocks OAuth redirect callbacks

#### ADV3 — WebSocket Security
- [ ] `wss://` used in production — never unencrypted `ws://`
- [ ] Authentication verified during the HTTP upgrade handshake (check JWT/session before upgrading)
- [ ] `Origin` header validated server-side to prevent cross-origin WebSocket hijacking
- [ ] All incoming WebSocket messages treated as untrusted user input and validated
- [ ] Rate limiting on WebSocket connections and messages per user (e.g., 10 msg/sec max)
- [ ] Disconnection cleanup: remove user from all rooms/channels on disconnect
- [ ] Ping/pong heartbeat implemented to detect and close stale connections

#### ADV4 — Subresource Integrity (SRI)
- [ ] All externally-loaded scripts (`<script src="https://cdn...">`) include `integrity` attribute with SHA-384 hash
- [ ] All externally-loaded stylesheets include `integrity` attribute
- [ ] `crossorigin="anonymous"` attribute present alongside `integrity`
- [ ] SRI hashes regenerated whenever the external resource version changes
- [ ] Use `https://vibeappscanner.com/tools/sri-generator` or `openssl dgst -sha384` to generate hashes

#### ADV5 — SSRF (Server-Side Request Forgery)
- [ ] Any server-side code that fetches a URL validates that URL against an allowlist
- [ ] Block requests to internal/private IP ranges: `127.0.0.1`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.169.254` (cloud metadata)
- [ ] Webhooks and URL-parameter-driven fetches are the highest-risk areas — audit all of them
- [ ] DNS rebinding protection: resolve URL once and validate, then use resolved IP

#### ADV6 — Open Redirect
- [ ] No redirect that uses a user-supplied URL without validation
- [ ] Redirect destinations validated against an allowlist of known-safe URLs or domains
- [ ] `?redirect=`, `?next=`, `?url=`, `?return=` parameters in auth flows validated before use
- [ ] Relative-only redirects preferred: strip protocol and host, keep only path

#### ADV7 — Email Security (Domain Protection)
- [ ] SPF record configured for sending domain (prevents spoofing)
- [ ] DMARC policy set (`p=quarantine` minimum, `p=reject` preferred)
- [ ] DKIM signing configured for transactional email provider
- [ ] Check with `vibeappscanner.com/tools/email-security` or `MXToolbox`
- [ ] Transactional email API keys server-side only (never exposed in frontend)

#### ADV8 — DNS Security
- [ ] DNSSEC enabled on domain (prevents DNS hijacking)
- [ ] CAA (Certification Authority Authorization) records set to limit which CAs can issue certificates
- [ ] Check with `vibeappscanner.com/tools/dns-security`
- [ ] No dangling DNS records pointing to deprovisioned infrastructure

#### ADV9 — Responsible Disclosure
- [ ] `/.well-known/security.txt` file present with contact email and disclosure policy
- [ ] Check validity with `vibeappscanner.com/tools/security-txt`
- [ ] Security contact address monitored and responsive

#### ADV10 — AI Prompt Security (Vibe Coding Specific)
- [ ] Real API keys, database passwords, or credentials never pasted into AI chat context
- [ ] `.env` files never attached or copy-pasted into AI tools (Cursor, Claude Code, Copilot, etc.)
- [ ] AI-generated code reviewed for CVE-2025-48757 pattern: Supabase tables created without RLS (affects 170+ Lovable apps)
- [ ] AI-suggested secret placeholder values (e.g., `"your-api-key-here"`) never committed — treat as a real secret leak if they reach git
- [ ] Post-session: review what sensitive context was shared with AI during the session

---

### 🟣 PLATFORM-SPECIFIC CHECKS

Detect which platforms/tools the project uses (check `package.json`, config files, imports) and run the relevant section(s).

#### PS-SUPABASE — Supabase Projects
- [ ] `ALTER TABLE x ENABLE ROW LEVEL SECURITY` present for ALL tables
- [ ] Separate RLS policies for SELECT, INSERT, UPDATE, DELETE on each table
- [ ] Policies use `(select auth.uid())` pattern (not just `auth.uid()` — performance matters at scale)
- [ ] `service_role` key never appears in any client-side file
- [ ] Anon key is acceptable in frontend; service_role key is never acceptable
- [ ] Email confirmation enabled in Auth settings
- [ ] Password policy configured with minimum requirements
- [ ] Auth rate limiting enabled
- [ ] RPC/Edge Functions include `auth.uid()` checks at the top
- [ ] `SECURITY DEFINER` functions understood and justified
- [ ] Rotate keys immediately if service_role was ever exposed

#### PS-FIREBASE — Firebase Projects
- [ ] No rule: `allow read, write: if true;` in any Firestore/Realtime DB ruleset
- [ ] All rules check `request.auth != null` before granting access
- [ ] Rules validate data structure and field types, not just auth
- [ ] Rules tested with Firebase Emulator before deployment
- [ ] Firebase Admin SDK / service account credentials are server-only (never in frontend)
- [ ] Client-side Firebase config (apiKey, projectId) is acceptable to expose
- [ ] API key restricted in Google Cloud Console to only needed services
- [ ] Email verification enabled for user accounts
- [ ] Storage rules restrict file types and sizes
- [ ] App Check configured to prevent unauthorized API usage

#### PS-MONGODB — MongoDB / Atlas Projects
- [ ] Authentication enabled (MongoDB never run without auth)
- [ ] MongoDB NOT exposed to public internet (`bindIp` = localhost or private network only)
- [ ] Atlas IP Access List configured — not `0.0.0.0/0`
- [ ] Application uses a dedicated low-privilege user (not `admin`)
- [ ] TLS/SSL enforced for all connections
- [ ] NoSQL injection prevention: validate that user-supplied values are not objects/operators
- [ ] Block `$where`, `$gt`, `$regex`, `$ne` operators from raw user input used in queries
- [ ] Mongoose strict schema validation enabled
- [ ] Audit logging configured for sensitive collections
- [ ] Encryption at rest enabled (Atlas default; verify for self-hosted)
- [ ] VPC peering used for cloud deployments (no public endpoint)

#### PS-VERCEL — Vercel Deployments
- [ ] `NEXT_PUBLIC_` prefix only on variables that should be client-visible
- [ ] Production secrets NOT shared with Preview or Development environments
- [ ] Preview deployments password-protected (Vercel Authentication enabled)
- [ ] Preview environments connected to staging DB, NOT production DB
- [ ] Security headers configured in `vercel.json` or `next.config.js` `headers()`
- [ ] CORS configured in API routes — not `Access-Control-Allow-Origin: *`
- [ ] Rate limiting on API routes (Vercel native or custom middleware)
- [ ] Function timeout set to prevent timeout-based DoS
- [ ] Team member access to production secrets audited
- [ ] Source maps disabled for production builds

#### PS-NETLIFY — Netlify Deployments
- [ ] `_headers` file configured in publish directory with security headers
- [ ] Build-time variables vs. runtime function variables understood and separated
- [ ] No sensitive secrets printed in build logs or bundled into frontend
- [ ] Function endpoints authenticated before accepting sensitive operations
- [ ] Scheduled functions run with minimum required permissions
- [ ] Deploy preview URLs not exposing production data
- [ ] Netlify Identity configured for access control where needed

#### PS-POSTGRES — PostgreSQL (direct / Neon / PlanetScale / Turso / Upstash)
- [ ] RLS enabled on all multi-tenant tables: `ALTER TABLE x ENABLE ROW LEVEL SECURITY`
- [ ] App connects as a low-privilege role — never `postgres` / superuser
- [ ] `scram-sha-256` authentication in `pg_hba.conf` (not `trust` or `md5`)
- [ ] All connections require SSL (`?sslmode=require` in connection string)
- [ ] All queries use parameterized statements — never string-concatenated SQL
- [ ] ORM raw-query escape hatches (`prisma.$queryRaw`, `sequelize.query`) also parameterized
- [ ] `pgAudit` or equivalent logging enabled for sensitive tables
- [ ] Connection strings stored in env vars, never in source code
- [ ] **PlanetScale:** production branch protection enabled; no app-level access control bypasses (no RLS — enforce in app)
- [ ] **Neon:** separate branches per environment; branch credentials not shared with production
- [ ] **Turso:** auth tokens scoped to minimum permissions; no RLS — access control must be in app layer
- [ ] **Upstash Redis:** REST URL and token in env vars; webhook endpoints verify Upstash signatures; no sensitive PII stored unencrypted in Redis
- [ ] **Upstash Kafka/QStash:** queue credentials server-side only; incoming message payloads validated before processing

#### PS-RAILWAY — Railway Deployments
- [ ] All secrets stored in Railway's variable management, not in repository
- [ ] Databases on private network (not public internet)
- [ ] Service-to-service communication uses internal DNS
- [ ] Only services needing public access are exposed
- [ ] SSL enforced for all database connections
- [ ] Container images scanned for vulnerabilities before deployment
- [ ] Separate variable values for staging vs production

#### PS-RENDER — Render Deployments
- [ ] Secrets stored as Render env vars — not in repo or build logs
- [ ] Databases on private service (not public-facing)
- [ ] Build logs reviewed to ensure secrets are not printed during build
- [ ] Internal services communicate via Render's private network
- [ ] Auto-deploy to production gated (consider manual deploy requirement)

#### PS-FLYIO — Fly.io Deployments
- [ ] Secrets set via `fly secrets set` — never committed to git
- [ ] Databases and internal services on private networking
- [ ] Dockerfile runs as non-root user
- [ ] Container image scanned for vulnerabilities
- [ ] TLS termination enabled at edge
- [ ] WireGuard used for secure remote access to private services

#### PS-BUBBLE — Bubble Apps
- [ ] Privacy rules configured for ALL data types (not just some)
- [ ] Privacy rules tested as: logged-out user, regular user, and admin
- [ ] No data type set to "everyone can see" for sensitive data
- [ ] "This Thing's Creator" pattern used to restrict user data to its owner
- [ ] API workflows secured with authentication — no unauthenticated data-modifying workflows
- [ ] All installed plugins audited for data access scope; unused plugins removed
- [ ] Network requests inspected in browser DevTools to verify no data over-exposure
- [ ] Page-level access rules configured (login required where appropriate)

#### PS-WEBFLOW — Webflow Sites
- [ ] No secrets or API keys stored in CMS collections (CMS data is accessible via API)
- [ ] Only trusted third-party scripts embedded (embedded scripts execute arbitrary code)
- [ ] Sensitive CMS collections restricted with Memberships, not URL obscurity
- [ ] Forms use reCAPTCHA / spam protection
- [ ] Form webhook destinations are legitimate and not publicly writable
- [ ] Staging site password-protected before launch

#### PS-FRAMER — Framer Sites
- [ ] No secrets stored in CMS (accessible via API)
- [ ] Code overrides and custom components reviewed for XSS
- [ ] Only trusted scripts embedded in custom code blocks
- [ ] Form webhook destinations verified as legitimate
- [ ] Framer Auth used for protected pages — not URL obscurity
- [ ] Staging versions password-protected

#### PS-RETOOL — Retool Internal Tools
- [ ] SSO configured for user authentication (not local passwords)
- [ ] 2FA required for all users
- [ ] All database/API queries use parameterized inputs — no raw concatenation
- [ ] Database credentials encrypted in Retool credential store
- [ ] App-level permissions configured per group
- [ ] Custom JavaScript in apps audited for vulnerabilities
- [ ] Audit logging enabled and reviewed
- [ ] Read replicas used for dashboards to protect production DB

#### PS-LOVABLE — Lovable Apps
- [ ] Supabase RLS enabled on all tables (Lovable uses Supabase by default)
- [ ] RLS policies cover SELECT, INSERT, UPDATE, DELETE
- [ ] No OpenAI, Stripe, or service keys hardcoded in generated source
- [ ] Auth uses server-side validation — not client-side only
- [ ] Security headers (CSP, HSTS, X-Frame-Options) configured

#### PS-BOLT — Bolt.new Apps
- [ ] All AI-generated code reviewed for security before deployment
- [ ] Database (Supabase/Firebase) rules/RLS configured — Bolt often skips this
- [ ] Session tokens stored in HttpOnly cookies, not localStorage
- [ ] Deployment config (Vercel/Netlify) reviewed for exposed vars

#### PS-REPLIT — Replit Projects
- [ ] Paid tier used for any project with secrets or sensitive logic (free Repls are public)
- [ ] ALL credentials in Replit Secrets, never in source files
- [ ] Database connection strings (contain credentials) in Secrets
- [ ] Custom domain configured with HTTPS for production
- [ ] Authentication implemented server-side — not relying on obscurity
- [ ] Deployed Repl logs monitored for errors and anomalies

#### PS-V0 — v0.dev Components
- [ ] All v0-generated components reviewed for `dangerouslySetInnerHTML` usage
- [ ] Props and user data sanitized before rendering
- [ ] API keys never passed to v0-generated client-side components
- [ ] Server-side data fetching used for sensitive operations
- [ ] CSRF protection added when integrating generated forms
- [ ] Authentication tokens stored in secure cookies, not localStorage

#### PS-WINDSURF — Windsurf / Cascade Projects
- [ ] Cascade agent NOT granted access to production systems or live credentials
- [ ] All Cascade multi-step flows reviewed before execution
- [ ] MCP servers audited — only trusted, necessary servers installed
- [ ] `.gitignore` covers all secret file types (Windsurf respects it)
- [ ] Git checkpoint made before major Cascade operations (easy rollback)
- [ ] AI-generated auth flows manually reviewed — Cascade auth often has gaps

#### PS-BASE44 — Base44 Apps
- [ ] No API keys (OpenAI, Stripe, DB) in generated frontend code — use server-side env vars
- [ ] Auth routes reviewed: rate limiting and account lockout present
- [ ] API routes check authorization (not just authentication)
- [ ] All API endpoints validate input with Zod or similar
- [ ] Debug/test routes removed before production deployment
- [ ] CORS restricted to actual domain (not `*`)
- [ ] Source maps disabled in production build config

#### PS-ANTIGRAVITY — Antigravity Apps
- [ ] Auto-generated API integrations reviewed for exposed credentials in component config
- [ ] Admin-only UI components enforce access at data layer, not just by hiding UI
- [ ] Database (Supabase/Firebase) rules/RLS enabled and tested
- [ ] Visual form builders' client-side validation supplemented with server-side validation
- [ ] Preview deployment URLs not publicly indexed
- [ ] Third-party visual integration credentials moved to env vars

#### PS-AI-ASSISTANTS — AI Code Assistants (Copilot, Cody, Tabnine, etc.)
- [ ] **Copilot/Tabnine:** Never accept a suggestion that contains a literal secret value — rotate immediately if one was accepted and committed
- [ ] **Copilot:** If Copilot suggests a secret value, assume it may be from training data — never use it
- [ ] **Cody:** Repository indexing scope reviewed — sensitive repos excluded from Sourcegraph context
- [ ] **Tabnine:** Local-only model mode enabled for sensitive/proprietary code
- [ ] **All:** AI-generated password hashing code verified to use bcrypt/argon2 (not MD5/SHA1)
- [ ] **All:** AI-generated JWT code verified to check signature and expiry server-side
- [ ] **All:** File exclusions configured (`.env`, `credentials.json`, `*.pem`) in IDE plugin settings

#### PS-NEXTJS — Next.js Applications
- [ ] `NEXT_PUBLIC_` env vars contain NO secrets
- [ ] API routes in `pages/api/` or `app/api/` all check authentication
- [ ] Server Actions validate authentication before executing
- [ ] `next.config.js` security headers configured
- [ ] `productionBrowserSourceMaps: false` in `next.config.js`
- [ ] `dangerouslySetInnerHTML` usages audited and all use DOMPurify

---

### 🤖 AI-ASSISTED CODE SPECIFIC CHECKS

Apply these checks to ALL codebases built with AI assistants (Claude Code, Cursor, Copilot, Lovable, Bolt, v0, Windsurf, etc.):

#### AI1 — AI Code Trust & Review
- [ ] All AI-generated authentication code reviewed by a human — AI auth is frequently insecure
- [ ] AI-generated SQL/database queries checked for parameterization
- [ ] No placeholder secrets accepted from AI suggestions (e.g., `"your-api-key-here"` committed)
- [ ] AI-generated error handling doesn't leak stack traces or internal details
- [ ] Input validation not assumed because AI generated a form — validate server-side
- [ ] AI-generated session handling reviewed (token generation, expiry, rotation)
- [ ] Password hashing: AI often suggests MD5/SHA1 — must be bcrypt or argon2

#### AI2 — Vibe-Coding Specific Patterns
- [ ] Client-side-only access control (hiding UI elements ≠ enforcing security)
- [ ] AI-generated code doesn't trust `userId`/`role` values from request body (not verified by server)
- [ ] AI-generated API routes have auth middleware, not just UI-level guards
- [ ] AI-scaffolded database schemas have RLS/rules configured (AI often skips this)
- [ ] Build configuration generated by AI: verify source maps, debug flags are off in prod
- [ ] AI-generated `.gitignore`: verify it covers all secret file types
- [ ] AI-generated CORS config: verify it's not `origin: '*'` in production

#### AI3 — .cursorignore / Context Hygiene (Cursor/Claude Code users)
- [ ] `.cursorignore` or equivalent excludes `.env`, `credentials.json`, `*.pem`, `secrets/`
- [ ] Terminal/shell history not shared with AI context (no secrets in recent commands)
- [ ] MCP servers reviewed — only trusted, necessary servers installed
- [ ] AI not run in production environments

---

## PHASE 2 — RISK PLANNING

For each ❌ FAIL finding from Phase 1, document:

```
FINDING: [vulnerability name]
FILE: [path:line]
SEVERITY: [CRITICAL / HIGH / MEDIUM / LOW]
EVIDENCE: [exact code snippet]
EXPLOIT SCENARIO: [how an attacker would exploit this]
EXPLOIT POC: [working curl command or code snippet that demonstrates the attack. For open/unauthenticated endpoints, always start with a GET listing request that fetches all records first — this proves the endpoint is open and shows real data before escalating to write/delete examples. Only add write or delete examples after the read PoC.]
FIX: [specific remediation steps]
REGRESSION RISK: [could fixing this break existing functionality?]
```

---

## PHASE 3 — REMEDIATION (only after Phase 1 & 2 complete)

- Fix only the issues identified — no cosmetic or unrelated changes
- Make minimal targeted changes
- Document what changed and why
- After each fix, verify no new vulnerabilities were introduced
- Re-run affected checklist items to confirm resolution

The report must include a **Fix Todo List** (see OUTPUT section). This list is designed to be fed directly to an AI coding assistant as a follow-up prompt. Each item must be self-contained: file path, line number, what to change, and why — so the AI can act on it without needing the full report context.

---

## OUTPUT: SECURITY REPORT

**Always save the report as both `_security/SECURITY-REPORT.md` and `_security/SECURITY-REPORT.html` (create the `_security/` folder if it does not exist).**

After saving, append this line to the project's `.gitignore` (create the file if it does not exist):
```
_security/
```
The `_security/` folder contains working exploit commands and attack payloads — it must never be committed to version control.

Produce the report in this exact format:

---

# Security Audit Report
**Project:** [name]
**Date:** [today]
**Auditor:** vibecodingscanner
**Codebase:** [root path]

## Executive Summary
[2–3 sentence summary of overall security posture]

**Risk Score:** [CRITICAL / HIGH / MEDIUM / LOW]
**Total Findings:** [n]  |  Critical: [n]  |  High: [n]  |  Medium: [n]  |  Best Practice: [n]

---

## Critical Findings (fix immediately)
[Table or list — one entry per finding]

| # | Vulnerability | File | Line | Evidence |
|---|--------------|------|------|----------|
| 1 | ... | ... | ... | `code snippet` |

### [Finding Title]
- **Severity:** CRITICAL
- **Location:** `file:line`
- **Description:** What it is and why it matters
- **Exploit Scenario:** How an attacker abuses this
- **Proof of Concept:** Working curl command or minimal script that demonstrates the attack. Use the actual base URL of the app if known. Commands must be copy-pasteable and correct — not pseudocode. Include the expected response so the reader knows what a successful exploit looks like. For open or unauthenticated endpoints, always lead with a GET listing request (e.g. `curl GET /api/admin/users`) that fetches all records — this confirms the endpoint is open and shows real data exposure before showing any write or delete examples.
- **Fix:** Step-by-step remediation with code examples

---

## High Severity Findings

[same structure as Critical]

---

## Medium Severity Findings

[same structure]

---

## Passed Checks
[Bullet list of what is correctly implemented — give credit where due]

---

## Recommended Tools to Add
Based on findings, recommend from:
- **Secrets scanning:** GitGuardian, gitleaks, trufflehog
- **Dependency scanning:** Snyk, Dependabot
- **SAST:** SonarQube, Semgrep, ESLint security plugin
- **Runtime monitoring:** Sentry, Datadog
- **Headers validation:** securityheaders.com
- **Auth:** Clerk, Auth0, NextAuth.js, Supabase Auth
- **Rate limiting:** express-rate-limit, rate-limiter-flexible
- **Input validation:** Zod, Joi
- **HTTP security headers:** helmet (Express), next.config.js headers()

---

## Remediation Priority Order
1. [Most critical fix]
2. [Second most critical]
3. [...]

---

## Exploit Chains

> Run any line below in Claude Code to generate a full attack chain for that finding — JS payload, HTML bait page, curl commands, and receiver setup where applicable.

```
/vibecodingexploits [FINDING-ID] [exploit-type] [url] [param?]
```

For each finding that has a meaningful exploit chain (XSS, CSRF, open redirect, JWT, SSRF, command injection, clickjacking, WebSocket hijacking, session replay), generate one ready-to-copy line. Format:

```
/vibecodingexploits CRIT-1  open-endpoint    https://[base]/api/admin/users
/vibecodingexploits HIGH-4  xss-stored       https://[base]/blog/[slug]      content
/vibecodingexploits HIGH-5  csrf             https://[base]/api/admin/posts
/vibecodingexploits HIGH-6  open-redirect    https://[base]/auth/callback    to_path
/vibecodingexploits ADV1    jwt-none         https://[base]/api/admin/me
/vibecodingexploits ADV3    websocket-hijack wss://[base]/ws
/vibecodingexploits ADV5    ssrf             https://[base]/api/fetch         url
/vibecodingexploits C5      cmd-injection    https://[base]/api/run           cmd
/vibecodingexploits M6      clickjacking     https://[base]/admin/settings
/vibecodingexploits C8      session-replay   https://[base]/api/me
```

Only include lines for findings actually present in this scan. Skip exploit types not applicable to the codebase.

---

## Fix Todo List

> This section is designed to be copy-pasted directly into an AI coding assistant as a self-contained fixing prompt. Each item includes everything the AI needs to act: the file, the line, what is wrong, and exactly what to change.

Paste the following into your AI assistant to fix all findings:

```
Fix the following security vulnerabilities in this codebase. Work through them in order. For each item: make only the change described, verify it does not break existing functionality, and move to the next.

[ ] CRIT-1 | src/[file]:[line] | [one-line description of what to change and why]
[ ] CRIT-2 | src/[file]:[line] | [one-line description]
[ ] HIGH-1 | src/[file]:[line] | [one-line description]
[ ] HIGH-2 | src/[file]:[line] | [one-line description]
[ ] MED-1  | src/[file]:[line] | [one-line description]
...

After all items are complete, confirm each fix with a short summary of the change made.
```

Rules for populating the todo list:
- One line per finding — no multi-line descriptions
- Include the exact file path and line number
- State the change imperatively: "Replace X with Y", "Add auth guard at top of handler", "Change condition from A to B"
- Order by severity: Critical first, then High, then Medium
- If a fix requires creating a new shared helper (e.g. `requireAdminAuth`), list that as the first item so subsequent items can reference it

---

*Report generated by the Vibe Coding Security Scanner skill.*
*Sources: OWASP Top 10 · vibeappscanner.com (checklists, guides, best-practices, glossary, sample-report, tools, safety-guides, vibe-coding-security) · astoj/vibe-security · Replit Vibe Code Security Checklist · namanyayg security audit prompt · CVE-2025-48757*

---

## APPENDIX: Platform Checklist Coverage Reference

This skill incorporates checks from vibeappscanner.com's full library of 55 platform-specific checklists. When the scanned codebase uses one of these platforms, apply the corresponding checks from the PS-* sections above AND note the platform in the report header.

| Category | Platforms Covered |
|----------|-------------------|
| AI builders | Lovable, Bolt.new, Replit, v0.dev, Windsurf, Base44, Antigravity |
| AI assistants | GitHub Copilot, Claude Code, Cursor, Cody, Tabnine, Amazon Q, Gemini Code, Cline, Augment |
| Databases | Supabase, Firebase, MongoDB, PostgreSQL, PlanetScale, Neon, Turso, Upstash |
| Hosting | Vercel, Netlify, Railway, Render, Fly.io |
| No-code/Low-code | Bubble, Webflow, Framer, Retool |

**Listed in vibeappscanner.com index but pages not yet published (404 at time of writing — check back for updates):**
Amazon Q, Gemini Code, Cline, Appwrite, Convex, Xano, FlutterFlow, Glide, Trae, Devin, OpenAI Codex, Augment Code, Emergent, Wix Harmony, Hostinger Horizons, Firebase Studio, Softr, ToolJet, DronaHQ, Jotform Apps, UI Bakery, Orchids, VibeSDK, SuperNinja, Tempo Labs

**Key stat:** ~80% of AI-built applications contain at least one exploitable vulnerability at launch (vibeappscanner.com research, 2026).
