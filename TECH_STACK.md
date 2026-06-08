# BRFTools Tech Stack

**Domain:** brftools.uk  
**GitHub:** github.com/Namcoza  
**DNS / CDN:** Cloudflare (Free tier)  
**Built with:** Claude Code / Cowork  
**Last updated:** June 2026

---

## Core Philosophy

- Free tier first — no paid services unless absolutely justified
- One consistent stack across all project types
- Deployable via Claude Code or Cowork without manual DevOps
- Subdomains per project off brftools.uk (e.g. `tool.brftools.uk`)

---

## Hosting & Deployment

### Static sites, frontends, simple apps
**Cloudflare Pages** (free tier)

- Deploy directly from a GitHub repo or via `wrangler` CLI
- Automatic HTTPS, global CDN, custom subdomain via Cloudflare DNS
- Zero config for HTML/CSS/JS or React/Next.js static export
- Limit: 500 builds/month, 100GB bandwidth — more than enough

### APIs and backend logic
**Cloudflare Workers** (free tier)

- Serverless JS/TS functions running at the edge
- 100,000 requests/day free
- Use for: REST APIs, form handling, webhooks, auth callbacks, automation triggers
- Pairs naturally with Pages (same dashboard, same DNS)

### When you need a traditional server
Use **Railway** or **Render** (both have free tiers with sleep-on-idle)  
Only reach for these if Workers genuinely can't handle it (long-running processes, Python, heavy compute)

---

## Database (when needed)

**Cloudflare D1** — SQLite at the edge, free tier, integrates directly with Workers  
Use for: user records, app state, simple relational data

**Cloudflare KV** — Key-value store, free tier  
Use for: config, session tokens, caching, feature flags

**Rule of thumb:**
- No persistence needed → Workers alone
- Simple structured data → D1
- Fast lookups / session state → KV
- File/blob storage → Cloudflare R2 (free 10GB)

---

## Authentication (when needed)

**Cloudflare Access** (free for up to 50 users)  
Best for: internal tools, dashboards — protects a whole subdomain with a login page  
Zero code required, just configure in the Cloudflare dashboard

**Clerk** (free tier: 10,000 MAU)  
Best for: public-facing apps that need proper user accounts  
Drop-in React components, works with Pages + Workers

**No-auth default:** For personal/internal tools with no sensitive data, skip auth entirely and rely on the URL being unlisted.

---

## Frontend

**Default:** Vanilla HTML/CSS/JS — fast to build, no build step, deploys to Pages instantly

**When you need reactivity:** React (via Vite, static export to Pages)

**CSS approach:** Tailwind CSS (CDN version for small projects, installed for larger ones)

**Design system:** Keep it consistent across projects — pick one colour palette and font and reuse it. Suggested:
- Font: Inter (Google Fonts)
- Accent: `#F6821F` (Cloudflare orange — ties back to your infrastructure)
- Neutral: `#1a1a2e` dark, `#f5f5f5` light

---

## Project URL Structure

All projects live as subdomains of brftools.uk:

```
brftools.uk              → landing / index of tools
tool-name.brftools.uk    → individual projects
api.brftools.uk          → shared API worker (optional)
```

Set up in Cloudflare DNS: CNAME each subdomain to the relevant Pages or Worker deployment.

---

## File & Asset Storage

**Cloudflare R2** (free 10GB, no egress fees)  
Use for: user uploads, generated files, images, PDFs  
Access via Workers — never expose the bucket directly

---

## Source Control & Change Control

**GitHub account:** github.com/Namcoza

### Repo naming convention

Every project gets its own repo, named to match its subdomain:

```
brftools-[project-name]
```

Examples:
```
brftools-home          → brftools.uk
brftools-ev-survey     → ev-survey.brftools.uk
brftools-visitor-log   → visitor-log.brftools.uk
```

This makes the link between repo → subdomain → Cloudflare Pages deployment unambiguous.

### Branch strategy

```
main    → production (live on brftools.uk subdomain)
dev     → staging / work in progress
```

- **Never commit directly to `main`** — always work on `dev`, then merge
- Cloudflare Pages deploys `main` automatically to the live subdomain
- Optionally connect `dev` branch to a `-dev.brftools.uk` preview URL in Pages settings

### Commit message convention

Use this format consistently (Claude Code can follow this if you instruct it):

```
type: short description

Types:
  feat      → new feature or page
  fix       → bug fix
  style     → visual / CSS changes only
  refactor  → code change, no behaviour change
  content   → copy, text, or asset updates
  config    → config files, wrangler, DNS, env vars
  chore     → deps, tooling, cleanup
```

Examples:
```
feat: add postcode lookup to EV survey form
fix: correct mobile nav overflow on dashboard
config: add D1 database binding to wrangler.toml
content: update homepage hero text
```

### Per-project repo structure

Every repo should follow this layout:

```
/
├── README.md          ← what the project is, how to run it, stack used
├── CHANGELOG.md       ← human-readable log of meaningful changes
├── .gitignore
├── .gitattributes     ← line ending consistency across Windows + Mac
├── wrangler.toml      ← if project uses Workers (binding names only, no secrets)
├── src/               ← all source code
│   ├── index.html     ← or index.jsx for React projects
│   └── ...
└── public/            ← static assets (images, fonts, icons)
```

### CHANGELOG format

Maintain a `CHANGELOG.md` in every repo. Update it when merging to `main`:

```markdown
## [YYYY-MM-DD] - Short description of release

### Added
- New feature or page

### Changed
- What was modified and why

### Fixed
- What was broken
```

### Connecting GitHub → Cloudflare Pages (once per project)

1. Go to Cloudflare Dashboard → Pages → Create application
2. Connect to Git → select the `Namcoza/brftools-[name]` repo
3. Set production branch: `main`
4. Add build command if needed (leave blank for vanilla HTML)
5. After deploy, go to Custom Domains → add `[name].brftools.uk`
6. Cloudflare auto-creates the DNS record

From then on: **push to `main` = live in ~30 seconds**. No manual deploys needed.

---

## Secrets & Credentials

**Golden rule: a secret never touches a file that gets committed to GitHub. Ever.**

### How it works on Cloudflare

Cloudflare has two separate secret stores depending on what you're building:

**Workers → Cloudflare Worker Secrets**  
Encrypted at rest, injected as environment variables at runtime. Set via dashboard or wrangler CLI. Never visible after saving — not even to you.

**Pages → Cloudflare Pages Environment Variables**  
Same idea, set in the Pages dashboard under Settings → Environment Variables. Mark any sensitive value as "Encrypted".

Secrets are scoped per project and per environment (Production vs Preview), so your live API keys are completely separate from any dev/staging values.

### Setting a secret (Workers)

Via the Cloudflare dashboard:
```
Workers & Pages → your-worker → Settings → Variables → Add variable → Encrypt
```

Via wrangler CLI (Claude Code terminal):
```bash
wrangler secret put MY_API_KEY
# prompts you to type the value — it is never written to disk
```

### Setting a secret (Pages)

```
Pages → your-project → Settings → Environment Variables → Add variable
Toggle "Encrypt" for anything sensitive
Set separately for Production and Preview environments
```

### Accessing secrets in Worker code

Secrets are available as plain environment variables — no extra code needed:

```javascript
export default {
  async fetch(request, env) {
    const apiKey = env.MY_API_KEY  // injected by Cloudflare, never hardcoded
    // use apiKey to call external services
  }
}
```

### What goes in wrangler.toml (safe to commit)

`wrangler.toml` defines the *shape* of your config — names of bindings, D1 databases, KV namespaces — but **never values**:

```toml
name = "brftools-my-project"
main = "src/index.js"
compatibility_date = "2024-01-01"

# Safe — just the binding name, no value
[vars]
ENVIRONMENT = "production"

# Safe — just the DB name reference
[[d1_databases]]
binding = "DB"
database_name = "brftools-my-project-db"
database_id = "your-d1-id-here"
```

Actual secret values (API keys, tokens, passwords) are set via the dashboard or `wrangler secret put` — never in this file.

### .gitignore — every project must include these

```gitignore
# Secrets & environment
.env
.env.*
.dev.vars          ← wrangler's local secret file, never commit

# Cloudflare / wrangler
.wrangler/

# Dependencies
node_modules/

# OS
.DS_Store
Thumbs.db
```

### The `.dev.vars` file (local dev only — if ever needed)

If you ever do local dev with `wrangler dev`, secrets go in `.dev.vars` (wrangler's local equivalent of the dashboard secrets). This file is **always gitignored** and never committed:

```
# .dev.vars  ← gitignored, local only
MY_API_KEY=your-actual-key-here
SOME_OTHER_SECRET=value
```

### Secret management rules

| Rule | Why |
|------|-----|
| Never put secrets in `wrangler.toml` | It gets committed to GitHub |
| Never put secrets in source code | Same reason |
| Never log secrets with `console.log` | Ends up in Cloudflare logs |
| Never pass secrets in URLs | They appear in access logs |
| Use encrypted variables in Pages | Plain text vars are visible in the dashboard |
| Rotate keys if a repo is ever made public | Assume exposure, regenerate immediately |

### If you accidentally commit a secret

1. Immediately revoke/regenerate the key at the source (the API provider)
2. Remove it from the file and commit the fix
3. Run `git filter-repo` or contact GitHub support to purge from history — deleting the file is not enough, git history retains it

---

## Local Dev & Tooling

| Tool | Purpose |
|------|---------|
| `wrangler` CLI | Deploy Workers and Pages from terminal |
| Vite | Frontend dev server for React projects |
| GitHub | Source control — connect to Pages for auto-deploy |
| VS Code | Editor (if working outside Claude Code) |

**Claude Code / Cowork workflow:**  
Build and iterate in Claude Code → push to GitHub → Cloudflare Pages auto-deploys on commit.  
For Workers: `wrangler deploy` from Claude Code terminal.

---

## Multi-Device Deployment Strategy

**Devices:** Windows PC + Mac  
**Principle:** GitHub is the single source of truth — your local machine is always disposable.  
No project state, secrets, or progress should ever live only on one device.

---

### The core workflow (device-agnostic)

```
[Any device]
    │
    ├── Pull latest from GitHub        ← always start here
    ├── Build / make changes in Claude Code or Cowork
    ├── Push to dev branch on GitHub
    └── Merge dev → main when ready    ← Cloudflare deploys automatically
```

You never deploy from your machine directly. GitHub triggers Cloudflare. Your machine is just the editor.

---

### One-time setup per machine

Do this once on each device. After that, the device is fully ready for any project.

**1. Install Git**
- Windows: https://git-scm.com/download/win (use Git Bash as your terminal)
- Mac: run `git --version` in Terminal — it will prompt install if missing

**2. Install Node.js + wrangler**
```bash
# Install Node (use LTS version) from https://nodejs.org
# Then install wrangler globally
npm install -g wrangler
```

**3. Authenticate wrangler to your Cloudflare account**
```bash
wrangler login
# Opens browser → log in to Cloudflare → done
# Token is stored locally — never committed anywhere
```

**4. Authenticate Git to GitHub**
```bash
# Use SSH keys (recommended — no password prompts)
ssh-keygen -t ed25519 -C "your@email.com"
# Add the public key to GitHub: Settings → SSH Keys → New SSH Key
# Test with: ssh -T git@github.com
```

This setup is per-machine, not per-project. Do it once and every repo just works.

---

### Starting work on a new device

When you sit down at a machine you haven't used in a while:

```bash
# Navigate to where you keep projects
cd ~/projects          # Mac
cd C:\projects         # Windows

# Clone the repo (first time on this machine)
git clone git@github.com:Namcoza/brftools-[project-name].git

# OR if already cloned, just pull latest
cd brftools-[project-name]
git pull origin dev
```

Then open in Claude Code / Cowork and carry on. Everything is exactly where you left it.

---

### Daily workflow (on any machine)

```bash
# 1. Always pull before starting
git pull origin dev

# 2. Make changes in Claude Code / Cowork

# 3. Stage and commit
git add .
git commit -m "feat: description of what you did"

# 4. Push to dev
git push origin dev

# 5. When ready to go live — merge dev into main
git checkout main
git pull origin main
git merge dev
git push origin main
git checkout dev        ← go straight back to dev
```

Cloudflare Pages detects the push to `main` and deploys within ~30 seconds. No deploy command needed.

---

### Keeping secrets in sync across machines

Secrets **never** move between machines. They live only in Cloudflare.

When you set up wrangler on a new machine:
- `wrangler login` re-authenticates you to Cloudflare
- Your Worker Secrets are already there — Cloudflare injects them at runtime
- You never copy `.env` files between machines
- There is no `.env` file to lose or accidentally commit

If you need to check what secrets exist for a Worker:
```bash
wrangler secret list
# Shows names only — values are never retrievable, by design
```

---

### Cross-platform gotchas (Windows ↔ Mac)

| Issue | Fix |
|-------|-----|
| Line endings (CRLF vs LF) | Add `.gitattributes` to every repo (see below) |
| File path separators | Use forward slashes in code — Node handles both |
| SSH key location | Windows: `C:\Users\you\.ssh\` / Mac: `~/.ssh/` |
| Terminal | Windows: use Git Bash, not Command Prompt |

**Add this `.gitattributes` to every repo** — prevents line ending conflicts between machines:

```gitattributes
* text=auto eol=lf
*.bat text eol=crlf
```

Commit this once and both machines will always produce consistent line endings.

---

### If a machine is lost, stolen, or wiped

1. Revoke its SSH key from GitHub → Settings → SSH Keys → Delete
2. Revoke its wrangler token from Cloudflare → My Profile → API Tokens
3. Clone repos fresh on the replacement machine
4. All secrets remain safe in Cloudflare — nothing sensitive was on the machine

Recovery time: ~10 minutes to be fully operational again.

---

## Project Checklist (new project)

**GitHub setup**
- [ ] Create repo: `Namcoza/brftools-[project-name]`
- [ ] Create `main` and `dev` branches
- [ ] Add `README.md`, `CHANGELOG.md`, `.gitignore`

**Cloudflare setup**
- [ ] Connect repo to Cloudflare Pages (production branch: `main`)
- [ ] Add custom domain `[name].brftools.uk` in Pages settings
- [ ] Decide: needs DB? → add D1. Needs auth? → add Cloudflare Access or Clerk

**Before first commit**
- [ ] Add `wrangler.toml` if using Workers — binding names only, no secret values
- [ ] Add `.gitattributes` (line endings — copy from stack doc)
- [ ] Confirm `.gitignore` covers `.env`, `.dev.vars`, `node_modules`, `.wrangler`
- [ ] Add any API keys via Cloudflare dashboard (Workers Secrets or Pages Encrypted Vars)
- [ ] Verify no secrets appear anywhere in source files before pushing

**On each new machine (one-time)**
- [ ] Install Git + Node.js + wrangler
- [ ] `wrangler login` → authenticate to Cloudflare
- [ ] Generate SSH key and add to GitHub

**Ongoing**
- [ ] All work happens on `dev`
- [ ] Merge to `main` only when ready to go live
- [ ] Update `CHANGELOG.md` on every merge to `main`

---

## What to Avoid (on free tier)

| Avoid | Reason |
|-------|--------|
| Vercel (hobby) | 100GB bandwidth cap, no commercial use |
| Netlify (free) | 100GB limit, slower builds |
| Heroku | No free tier anymore |
| Firebase | Gets expensive fast, vendor lock-in |
| PlanetScale | Free tier removed |
| Supabase | Free tier pauses after 1 week inactivity |

---

## Quick Reference

```
Hosting:    Cloudflare Pages
Functions:  Cloudflare Workers  
Database:   Cloudflare D1 (SQL) / KV (key-value)
Storage:    Cloudflare R2
Auth:       Cloudflare Access (internal) / Clerk (public)
Secrets:    Cloudflare Worker Secrets / Pages Encrypted Vars
Frontend:   Vanilla JS or React + Tailwind
DNS/CDN:    Cloudflare (already set up)
Domain:     brftools.uk
GitHub:     github.com/Namcoza
Repos:      Namcoza/brftools-[project-name]
Branches:   main (live) / dev (staging)
```
