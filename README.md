# Security Skills

A portable set of **defensive security skills** for testing and hardening an application
**you own or are authorized to test**. They apply attacker thinking (red team) to find
vulnerabilities, then guide you to fix them (blue team).

Lifecycle: **RECON → PLAN → FIND → EXPLOIT → FIX → REPORT**

```
security-testing         🧭  orchestrator / entry point (authorization gate + routing)

  RECON   recon-and-osint          🔭  discover the real attack surface (subdomains, secrets, endpoints)
  PLAN    threat-modeling          🗺️  map surface, rank risks
  FIND/   security-code-audit      🔍  static (white-box) review: read source for vuln patterns
  EXPLOIT active-pentest           💥  execution harness: run tests safely on YOUR app, capture PoCs
          ── deep-dive specialists ──
          access-control-testing   🚪  IDOR/BOLA, privesc, multi-tenant (OWASP #1)
          authentication-testing   🔑  login/session/OAuth/JWT/MFA → account takeover
          business-logic-testing   🧠  workflow/pricing/race-conditions/abuse
          api-security-testing     🔌  REST/GraphQL/gRPC/WS (OWASP API Top 10)
          client-side-exploitation 🖥️  DOM XSS, CSP/CORS, prototype pollution, smuggling, cache poisoning
          injection-testing        💉  SQLi/NoSQLi, command, SSTI, XXE, LDAP/XPath, CRLF
          file-upload-and-ssrf     📎  upload bypasses, SSRF, insecure deserialization
          secrets-management-audit 🗝️  exposed keys/tokens in code, git history, logs, config
          vulnerability-chaining   🔗  combine findings into end-to-end attack paths (the lead)
  FIX     security-hardening       🛡️  secure-coding fixes + re-test
  REPORT  pentest-reporting        📝  CVSS scoring, findings, retest & closure
```

## Scope coverage
Web frontend · Backend API (REST/GraphQL/gRPC/WebSocket) · Database · Auth & sessions · Business logic

## Install

This repo is packaged as a **skills plugin**: manifests live in `.claude-plugin/` and `.plugin/`,
the skills themselves in `skills/`. Push it to GitHub, then install with your tool of choice.

```
security-skills/
├── .claude-plugin/
│   ├── marketplace.json   # lists the "security" plugin (source ".")
│   └── plugin.json        # metadata + "skills": "./skills/"
├── .plugin/plugin.json    # generic manifest (read by the `npx skills` CLI)
├── README.md
└── skills/                # the 16 skills, each a <name>/SKILL.md (+ references/)
```

### As a plugin / via CLI (recommended)
```bash
# generic skills CLI (arg is your repo, per the skills CLI docs)
npx skills add mn-youssef/security-skills
```
In **Claude Code** (plugin marketplace):
```
/plugin marketplace add mn-youssef/security-skills
/plugin install security@security-skills
```
The manifest points at `./skills/`, so all 16 install together.

### Manual (no plugin tooling)
Copy any dir from `skills/` into your agent's skills path — e.g. `~/.claude/skills/` (Claude Code
personal) or `.claude/skills/` (project); for opencode/spec-kit use your configured skills path and
point at the `SKILL.md`. Frontmatter `name`/`description` are the standard fields.

> Manifests are configured for GitHub user **mn-youssef**. Adjust author/license in the
> manifests if you wish.

## Usage

Start with **`security-testing`** — it confirms authorization and routes you to the right
phase. Or invoke a phase skill directly if you know what you need.

```
security-testing            → guided full lifecycle
recon-and-osint             → "what's actually exposed?"
threat-modeling             → "what should I worry about and test first?"
security-code-audit         → "find vulnerabilities in this code" (static review)
active-pentest              → "run the tests safely on my staging app" (authorized only)
access-control-testing      → "can a user reach another user's / admin's data?"
authentication-testing      → "can someone take over an account?"
business-logic-testing      → "can someone abuse the workflow / pricing?"
api-security-testing        → "is my REST/GraphQL API authz solid?"
client-side-exploitation    → "any advanced browser/HTTP-layer bugs?"
injection-testing           → "does any input reach an interpreter?"
file-upload-and-ssrf        → "can uploads or URL-fetches be abused?"
secrets-management-audit    → "are any keys/tokens leaked anywhere?"
vulnerability-chaining      → "combine findings into a critical attack path"
security-hardening          → "fix this finding the right way"
pentest-reporting           → "score, write up, and track findings to closure"
```

## ⚠️ Authorization

Only use these against systems you **own** or have **written authorization** to test.
`active-pentest` is scoped to your own lab/staging and avoids destructive/DoS techniques —
it validates vulnerabilities, it does not weaponize them. Unauthorized testing is illegal
in most jurisdictions.
