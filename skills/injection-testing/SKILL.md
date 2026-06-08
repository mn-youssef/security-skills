---
name: injection-testing
description: Use when dynamically testing whether untrusted input reaches an interpreter — SQL/NoSQL injection (incl. blind/time-based), OS command injection, server-side template injection (SSTI), XXE, LDAP/XPath, and CRLF/header injection. Confirms on a running app what security-code-audit finds in source.
---

# Injection Testing (FIND/EXPLOIT — input reaches an interpreter)

## Overview
Injection happens when input crosses into a language an interpreter executes — SQL, a shell, a
template engine, an XML/LDAP/XPath parser. This skill *proves* it dynamically; the code-level
patterns live in `security-code-audit/references/database.md` and `api-backend.md`.

**Core principle:** Find where input changes the *structure* of a command, not just its data.
A working injection is a tiny structural payload that changes the interpreter's behavior.

## Applies when / Skip when
- **Applies when:** untrusted input can reach an interpreter — SQL/NoSQL DB, OS shell, template
  engine, or XML/LDAP/XPath parser.
- **Skip when:** a given sink type is absent → that injection class is N/A.
- **If N/A:** name the absent interpreters (e.g. "no SQL → SQLi N/A") and test only the sinks
  that exist. Partial applicability is normal here.

## ⚠️ Authorization
Your own/authorized app, non-prod for stateful payloads. Never `--dump` real data, never
`--os-shell`/RCE on shared systems, never `; DROP`/destructive payloads. Confirm, don't damage.

## Method
1. **Map sinks** — every input that could reach a query/command/template/parser (use the audit findings).
2. **Probe** — send a structure-breaking character and watch for a behavior change (error,
   different result, timing).
3. **Confirm** — escalate from "something changed" to a controlled, benign proof.
4. **Bound impact** — note what an attacker could reach; do *not* exfiltrate real data.

## Quick reference — probe → confirm
| Type | Probe | Confirm (benign) |
|------|-------|------------------|
| SQLi | `'` `"` `')` → error/change | boolean `' OR '1'='1` ; time `' OR SLEEP(5)-- ` |
| Blind SQLi | no output? | time-based / boolean differential |
| NoSQLi | `'`, `{"$ne":null}`, `[$gt]=` | auth bypass via operator injection |
| Command | `;id` `|id` `` `id` `` `$(id)` | out-of-band DNS, or time `;sleep 5` |
| SSTI | `${7*7}` `{{7*7}}` `<%=7*7%>` | render `49` → engine-specific escalation |
| XXE | XML with external entity | OOB fetch / file read (benign canary) |
| LDAP | `*)(uid=*` | filter alteration |
| XPath | `' or '1'='1` | node selection change |
| CRLF | `%0d%0a` in reflected header | header/response splitting |

See `references/injection-recipes.md` for per-type detection and safe-confirmation payloads.

## Tooling
`sqlmap` for SQLi confirmation (scoped: `--batch --level/--risk` low, no `--dump`/`--os-shell`),
Burp/ZAP for fuzzing, OOB via an interaction server (Burp Collaborator/`interactsh`) for blind/XXE.
Drive them via `active-pentest`.

## Output
Per finding: the sink, the structural payload, the *evidence* of interpretation (error/time/OOB),
and the bounded impact (e.g. "DB read possible"). High–Critical.

## Hand-off
Often a foothold/escalation primitive → `vulnerability-chaining`; fix via `security-hardening`
(parameterization, safe APIs, allowlists, disabling external entities, output of structure-free queries).

## Common mistakes
- Only trying SQLi — SSTI, XXE, command, NoSQL hide in templates, file parsers, search, and JSON.
- Missing **blind** injection — no visible output ≠ not vulnerable; use time/boolean/OOB.
- Going destructive to "prove it" — a `SLEEP` or `49` proves it; `DROP` is vandalism.
- Forgetting second-order — stored input that injects when later used in a query.
