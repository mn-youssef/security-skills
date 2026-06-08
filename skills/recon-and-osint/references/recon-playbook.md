# Recon & OSINT Playbook

> Authorized / owned assets only. Passive steps are safe anywhere; active steps need scope.
> Replace `example.com` with your domain.

## 1. Passive footprint (no traffic to target)
- **Certificate transparency:** `crt.sh/?q=%25.example.com` — reveals subdomains via issued certs.
- **Search dorking:** `site:example.com`, `inurl:`, `filetype:`, `intitle:"index of"`,
  exposed `.env`, `.git`, backups, admin panels.
- **Wayback / archives:** `waybackurls example.com`, `gau example.com` — old endpoints & params.
- **DNS:** `dig`, `dnsrecon`, passive DNS (SecurityTrails). MX/TXT/SPF reveal vendors.

## 2. Subdomain & asset discovery
```bash
subfinder -d example.com -all -silent | tee subs.txt
amass enum -passive -d example.com
# resolve & probe which are live
cat subs.txt | dnsx -silent | httpx -silent -title -tech-detect -status-code
```
- **Cloud buckets:** name-permutation tools; check `*.s3.amazonaws.com`, GCS, Azure blobs for public ACLs.
- **Related orgs/domains:** ASN lookups, reverse-WHOIS, favicon-hash pivoting (`shodan`/`fofa`).

## 3. Service & tech fingerprinting
```bash
whatweb https://example.com
httpx -l subs.txt -tech-detect -title -server -status-code
nmap -sV -Pn -top-ports 1000 TARGET     # active — scoped hosts only
```
Note versions for known-CVE lookups; note frameworks/CMS for targeted checks.

## 4. Content & endpoint discovery
```bash
# crawl + collect URLs/params
katana -u https://example.com -jc -silent | tee urls.txt
# pull endpoints/secrets from JS
cat urls.txt | grep -Ei '\.js$' | while read u; do curl -s "$u"; done | \
  grep -Eo '("|'"'"')/[a-zA-Z0-9_/-]+("|'"'"')'   # path-like strings
# directory/endpoint brute (scoped)
ffuf -u https://example.com/FUZZ -w wordlist.txt -mc 200,301,401,403 -rate 50
```
Look for: API docs (Swagger/OpenAPI `/swagger`, `/openapi.json`), GraphQL (`/graphql`),
admin/debug routes, `.git/`, `.env`, backups (`.bak`, `~`, `.old`).

## 5. Secret & exposure hunting (defensive)
```bash
gitleaks detect --source . --redact            # your repos
trufflehog github --org=YOUR_ORG               # your org's public repos
```
- GitHub dorking on **your org**: `org:YOUR_ORG password`, `api_key`, `BEGIN PRIVATE KEY`.
- Check JS bundles & source maps for inlined secrets/endpoints.
- Know what's already leaked: search breach corpora for your org's emails so you can force resets.

## 6. Consolidate
Produce the **target inventory** (asset · type · tech · notable exposure). Flag anything that
should be internal but is public (staging, debug, admin, buckets). That list drives
`threat-modeling`.

## Tool index
`subfinder` `amass` `dnsx` `httpx` `katana` `gau` `waybackurls` `whatweb` `ffuf` `nmap`
`gitleaks` `trufflehog` `shodan` — plus crt.sh and search-engine dorks.
