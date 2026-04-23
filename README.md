# Capital Investment — Project Requirements

This document captures all requirements for the Capital Investment project. Each requirement is tagged with its source so the reason for it is always clear.

**Tag key**

| Tag | Meaning |
|---|---|
| `[SEC]` | Security — a specific vulnerability class or hardening standard |
| `[STD]` | Industry standard — expected practice in professional environments |
| `[BP]` | Best practice — not required but strongly advisable |
| `[FUNC]` | Functional — the feature must exist for the product to work |
| `[REL]` | Reliability — prevents downtime or data loss |
| `[OBS]` | Observability — enables monitoring, debugging, and auditing |

---

## Table of contents

1. [Project-wide](#1-project-wide)
2. [Database (PostgreSQL + Flyway)](#2-database-postgresql--flyway)
3. [Data ingestion](#3-data-ingestion)
4. [AI enrichment](#4-ai-enrichment)
5. [API (FastAPI)](#5-api-fastapi)
6. [Frontend](#6-frontend)
7. [Container security (Podman)](#7-container-security-podman)
8. [CI/CD (GitHub Actions)](#8-cicd-github-actions)
9. [DNS & Cloudflare](#9-dns--cloudflare)
10. [Server & hosting](#10-server--hosting)
11. [Nginx routing](#11-nginx-routing)
12. [Web interface](#12-web-interface)
13. [Network & ISP (home server)](#13-network--isp-home-server)
14. [Dynamic DNS](#14-dynamic-dns)
15. [Hardware & uptime](#15-hardware--uptime)

---

## 1. Project-wide

**PRJ-001** `[STD]` `[BP]`
All source code lives in a single git repository with branch protection on `main`. Direct pushes to main are blocked; all changes go through pull requests.

**PRJ-002** `[SEC]`
Secrets are never stored in the repository. All credentials live in a `secrets/.env` file that is gitignored.
> OWASP A02 — Cryptographic Failures. A leaked API key or DB password is the most common breach vector. A secret committed once stays in git history permanently even after deletion.

**PRJ-003** `[BP]`
A `secrets/.env.example` file documents every required variable with a description. It is committed to the repo with no real values.
> Ensures the project is reproducible without exposing credentials.

**PRJ-004** `[STD]` `[BP]`
The project supports two runtime profiles: `dev` and `prod`. Profile controls AI backend, TLS, and log verbosity.
> Prevents prod credentials and config from leaking into dev and vice versa.

**PRJ-005** `[SEC]`
All services run as non-root users inside containers.
> CIS Docker Benchmark 4.1. If a container is compromised, a non-root process limits the blast radius significantly.

**PRJ-006** `[SEC]`
All inter-service communication uses internal Podman network names, not exposed host ports, except the Nginx public port.
> Reduces attack surface. Postgres, FastAPI, and Grafana are never exposed directly to the host network.

**PRJ-007** `[OBS]` `[STD]`
Every component writes structured logs (JSON preferred) with a consistent set of fields: `timestamp`, `level`, `component`, `message`.
> Industry standard for log aggregation. Enables Grafana Loki or similar tooling without reformatting.

**PRJ-008** `[STD]` `[REL]`
A `/health` endpoint exists for every service. Returns `200` + status JSON when healthy, `503` when dependencies are unavailable.
> Required for Podman health checks and CI readiness probes.

---

## 2. Database (PostgreSQL + Flyway)

**DB-001** `[STD]`
All schema changes are managed by Flyway versioned migration files. No manual DDL changes are applied directly to the database.
> Industry standard: schema-as-code. Enables reproducible environments and safe rollouts across dev, CI, and prod.

**DB-002** `[STD]`
Migration files are named with the Flyway convention: `V{n}__{description}.sql` and are never modified after being applied.
> Modifying applied migrations breaks Flyway's checksum validation and causes failures in other environments.

**DB-003** `[SEC]`
The application connects to the database using a dedicated low-privilege user. This user has `SELECT`, `INSERT`, `UPDATE`, `DELETE` only — no `DROP`, `CREATE`, or superuser rights. Flyway uses a separate higher-privilege user only at migration time.
> Principle of Least Privilege (PoLP). Limits damage from SQL injection or a compromised application container.

**DB-004** `[BP]` `[REL]`
All upserts use `ON CONFLICT DO UPDATE` (upsert pattern). Ingestion jobs are idempotent — re-running produces the same result with no duplicates.
> Best practice for data pipelines. Prevents duplicates without requiring coordination between runs.

**DB-005** `[BP]`
Bills and their AI enrichment are stored in separate tables (`bills` and `bill_ai_enrichment`). Source data is never mutated by the enrichment process.
> Data engineering best practice: raw source data is immutable. Derived/computed data is clearly separated.

**DB-006** `[BP]`
Economic indicator data is stored in a generic `econ_snapshots` table keyed by `series_id` and `date`. Adding a new economic series requires no schema change.
> Open/closed principle applied to schema design. Extensible by configuration, not migration.

**DB-007** `[OBS]`
All ingestion and enrichment runs are logged to an `ingestion_runs` table with status, record counts, errors, and timestamps.
> Audit trail and operational observability. Enables Grafana dashboards for pipeline health over time.

**DB-008** `[BP]`
Indexes exist on all columns used in `WHERE`, `JOIN`, and `ORDER BY` clauses in application queries. At minimum: `introduced_date`, `status`, `series_id`, `snapshot_date`, `keyword`.
> Performance best practice. Unindexed queries on a growing bills table will degrade over time.

---

## 3. Data ingestion

**ING-001** `[BP]` `[REL]`
All incoming data is validated against a Pydantic schema before any database write. Invalid records are logged and skipped, not crashed on.
> Defensive programming / data quality. One bad API response should not kill the entire ingestion run.

**ING-002** `[STD]`
Ingestion jobs are scheduled via systemd timers, not cron. The timer unit file and service unit file are both version-controlled in the repo.
> Systemd timers are the RHEL/enterprise Linux standard. They provide journald integration, dependency management, and restart policies out of the box.

**ING-003** `[BP]`
The Congress.gov ingestion job fetches only bills updated since the last successful run. It does not re-fetch the full dataset on every run.
> API stewardship / rate limit best practice. Respects the API provider's resources and stays within free-tier limits.

**ING-004** `[REL]`
All outbound HTTP calls have explicit timeouts set. No request blocks indefinitely.
> A hung API call should not stall a systemd service or block a container indefinitely.

**ING-005** `[SEC]`
External API keys (Congress.gov, FRED) are read from environment variables at runtime. They are never hardcoded in source files.
> OWASP A02. Secrets in source code are a critical vulnerability and will be exposed in git history permanently even after removal.

**ING-006** `[REL]`
Bill ingestion and economic ingestion run as separate jobs on separate schedules. Failure of one does not block the other.
> Fault isolation. Two independent systemd services with independent timers and independent failure states.

---

## 4. AI enrichment

**ENR-001** `[BP]`
The AI backend is selected by the `AI_BACKEND` environment variable: `ollama` for dev (local model), `claude` for prod (Anthropic API). Both backends produce identical output schemas.
> Dev/prod parity best practice. No Claude API costs or external network calls during local development.

**ENR-002** `[BP]` `[REL]`
The enrichment job only processes bills that have no existing enrichment record. Re-enriching requires an explicit `--force` flag.
> Idempotency and cost control. Prevents duplicate API calls and redundant LLM spend on reruns.

**ENR-003** `[BP]`
The prompt template version is stored alongside every enrichment result. If the prompt changes, old results can be identified and selectively re-enriched.
> ML ops best practice. Prompt changes are breaking changes to output quality — version them like code.

**ENR-004** `[BP]` `[REL]`
The AI is instructed via system prompt to respond only with structured JSON. The enrichment job validates the JSON schema before writing to the database. Malformed responses are logged and retried once.
> Defensive programming. LLMs can deviate from instructions; the application must not crash or write garbage on unexpected output.

**ENR-005** `[SEC]`
The Anthropic API key is read from the environment and is never logged, printed to stdout, or included in error messages or stack traces.
> OWASP A02. Secrets leaked in logs are as dangerous as secrets committed to code. Many log aggregation tools forward to third parties.

**ENR-006** `[OBS]`
Each enrichment record stores the model name used to generate it (e.g. `claude-sonnet-4-20250514` or `ollama/llama3.2`).
> Audit trail and reproducibility. Knowing which model produced a result is required for debugging output quality regressions.

---

## 5. API (FastAPI)

**API-001** `[SEC]` `[FUNC]`
All endpoints are read-only (`GET` only). No public write endpoints exist.
> Security by design. This is a public data product — no external actor should be able to modify data.

**API-002** `[STD]`
OpenAPI documentation is auto-generated by FastAPI and served at `/api/docs`. All endpoints, parameters, and response schemas are fully documented via type annotations.
> Industry standard for REST APIs (OpenAPI 3.0). FastAPI generates this automatically — no extra work required.

**API-003** `[BP]` `[REL]`
All list endpoints are paginated. No endpoint returns an unbounded result set.
> An unbounded query against a large bills table will exhaust memory and timeout under any reasonable load.

**API-004** `[SEC]`
The database connection string is never logged. All query parameters are passed via parameterized queries only — no string interpolation into SQL under any circumstance.
> OWASP A03 — SQL Injection. Parameterized queries are the only compliant approach. String interpolation into SQL is never acceptable.

**API-005** `[SEC]`
CORS is configured to allow read access from any origin in dev. In prod it is restricted to the project's own domain only.
> OWASP A01 — Access Control. Open CORS in prod allows any third-party website to make credentialed requests to the API on behalf of a visiting user.

**API-006** `[SEC]`
HTTP response headers include: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`. These are set at the Nginx layer so they apply universally.
> OWASP Secure Headers Project baseline. Set once in Nginx; applies to all services behind it.

---

## 6. Frontend

**FE-001** `[FUNC]`
Phase 1 delivers bill detail pages. URL pattern: `/bills/{id}`. Displays bill number, title, status, sponsor, AI summary, theme label, keyword tags, and an economic snapshot table for the week the bill was introduced.
> Simplest complete user story. Validates the full pipeline end-to-end before building more complex views.

**FE-002** `[FUNC]`
Phase 2 adds the shared timeline view: bills rendered as events on a vertical time axis with economic indicator lines overlaid on the same axis.
> Core product differentiator. Puts legislative activity and economic context in the same visual frame.

**FE-003** `[FUNC]`
Phase 3 adds the keyword and theme dashboard: aggregate views of which themes are most active, filterable by date range.
> Adds analytical value beyond individual bill lookup.

**FE-004** `[BP]`
The frontend is a static site (HTML/CSS/JS). It is served by Nginx, not FastAPI. It calls the API via `fetch`.
> Separation of concerns. Static assets served by Nginx are faster and do not consume FastAPI worker threads.

**FE-005** `[SEC]`
No user data is collected. No cookies, no analytics scripts, no tracking pixels. The site is read-only public data.
> Privacy by design (GDPR Article 25). No consent banner or cookie notice required if nothing is collected.

---

## 7. Container security (Podman)

**CTR-001** `[SEC]` `[STD]`
Podman is used instead of Docker. All containers run rootless — no daemon, no root socket.
> CIS Benchmark / enterprise standard. Rootless Podman is the default on RHEL/Fedora. No root socket means a container escape does not grant root on the host.

**CTR-002** `[SEC]` `[BP]`
All container images are pinned to a specific version tag. `:latest` is never used in Podman Compose files or Dockerfiles.
> Supply chain security. `:latest` silently changes between pulls and can introduce breaking changes or unreviewed vulnerabilities.

**CTR-003** `[SEC]`
Container images are scanned with Trivy for known CVEs before being used in prod. High and Critical severity findings block deployment.
> DevSecOps standard. Trivy is the CNCF-adopted open source container scanner.

**CTR-004** `[SEC]`
Static assets in the Nginx container are mounted read-only. No container has a writable mount to the host filesystem except the Postgres data volume.
> CIS Benchmark 5.12. Read-only mounts prevent a compromised container from modifying host files.

**CTR-005** `[BP]`
The Postgres data volume is a named Podman volume, not a bind mount to a host path. Backups are scripted and run on a schedule.
> Named volumes are managed by Podman and are portable between hosts. Bind mounts introduce host path dependencies that break on migration.

**CTR-006** `[SEC]` `[STD]`
Nginx terminates TLS in prod using Cloudflare Origin Certificates. HTTP traffic on port 80 is redirected to HTTPS. See DNS-004 for Full (Strict) TLS requirement.
> OWASP A02 / industry baseline. All public web services must use TLS. Cloudflare Origin Certificates avoid Let's Encrypt ACME challenges behind a proxy.

---

## 8. CI/CD (GitHub Actions)

**CI-001** `[STD]`
Every pull request triggers the CI pipeline. Merging to `main` is blocked if CI fails.
> DevSecOps standard. CI is a quality gate, not a reporting tool. It has no value if failures can be ignored.

**CI-002** `[SEC]`
Bandit runs on all Python files in CI. Medium severity and above findings fail the build.
> SAST — Static Application Security Testing. Bandit catches common Python vulnerabilities: hardcoded passwords, shell injection, insecure deserialization, use of `eval`.

**CI-003** `[SEC]`
Trivy scans the built container image in CI. High and Critical CVEs fail the build and block the image push to the registry.
> DevSecOps shift-left principle. Catch vulnerabilities in CI before they reach prod, not after a breach.

**CI-004** `[BP]` `[SEC]`
Python dependencies are pinned to exact versions in `requirements.txt`. A separate `requirements-dev.txt` covers development and test tools.
> Reproducible builds. Unpinned dependencies silently change between CI runs and can introduce vulnerabilities or breakage.

**CI-005** `[BP]`
Unit tests cover ingestion transform logic and enrichment JSON parsing. Integration tests run against a real Postgres container spun up in the CI job.
> Testing pyramid. Transform and parse logic are the highest-risk areas — a bug here corrupts stored data silently.

**CI-006** `[SEC]`
GitHub Actions secrets store the prod Anthropic API key, FRED key, and any registry credentials. They are referenced in workflow YAML via `${{ secrets.NAME }}` — never pasted inline.
> GitHub-recommended secret management. Secrets pasted into YAML are exposed in build logs and in git history.

**CI-007** `[SEC]`
A weekly scheduled GitHub Actions workflow runs dependency vulnerability scanning independently of pull requests.
> New CVEs are published daily. A project with no new commits can still become vulnerable overnight. Weekly automated scans catch this without requiring manual checks.

---

## 9. DNS & Cloudflare

**DNS-001** `[BP]`
The domain is purchased through Cloudflare Registrar or transferred to Cloudflare. Cloudflare acts as the authoritative DNS provider.
> Cloudflare Registrar charges at-cost with no markup. Keeping registrar and DNS in one place eliminates nameserver delegation complexity.

**DNS-002** `[SEC]`
The A record for the root domain and `www` subdomain point to the server IP with the Cloudflare proxy enabled (orange cloud). The server's real IP is never publicly exposed in any DNS record.
> Hides origin IP, preventing attackers from bypassing Cloudflare's WAF and DDoS protection by connecting directly to the server.

**DNS-003** `[STD]`
An `api` subdomain (`api.yourdomain.com`) is created as a separate proxied DNS record routing to the same server. The API and frontend share a server but are logically separated by subdomain.
> Standard REST API convention. Subdomain separation allows independent routing and makes future service separation straightforward.

**DNS-004** `[SEC]`
Cloudflare SSL/TLS mode is set to **Full (Strict)**. This requires a valid certificate on the origin Nginx server, not just at the Cloudflare edge.
> Security critical. The default "Flexible" mode leaves traffic between Cloudflare and the origin server completely unencrypted. Full (Strict) encrypts both hops and validates the origin certificate.

**DNS-005** `[SEC]`
Cloudflare's "Always Use HTTPS" setting is enabled. HTTP requests are redirected to HTTPS at the Cloudflare edge.
> Defense in depth. Even if Nginx's own redirect is misconfigured, Cloudflare enforces HTTPS before traffic reaches the origin.

**DNS-006** `[SEC]`
Cloudflare WAF is enabled with the default managed ruleset active. No custom rules are required initially.
> The free-tier WAF blocks common attack patterns (SQLi, XSS, bad bots) at zero additional cost.

**DNS-007** `[SEC]`
The Grafana subdomain (`grafana.yourdomain.com`) is restricted to authorized users via Cloudflare Access or IP allowlisting. Grafana is never publicly accessible.
> Principle of least exposure. Grafana has its own auth but exposing admin tooling publicly is an unnecessary attack surface.

**DNS-008** `[SEC]`
The Cloudflare API token used for DNS management is scoped to the minimum permissions needed: Zone / DNS / Edit for this specific zone only. It is stored in `secrets/.env` and GitHub Actions secrets.
> Principle of Least Privilege. A full-account API token, if stolen, compromises all Cloudflare zones and account settings.

---

## 10. Server & hosting

**HST-001** `[SEC]`
The server exposes only ports 80 and 443 to the public internet. All other ports (5432 Postgres, 3000 Grafana, 8000 FastAPI) are firewalled at the host level using `ufw` or `firewalld`.
> Defense in depth. Podman's internal network isolation is the first layer; a host firewall is the second. Both are required.

**HST-002** `[SEC]`
SSH access to the server is key-based only. Password authentication is disabled in `/etc/ssh/sshd_config` (`PasswordAuthentication no`).
> CIS Benchmark / SSH hardening standard. Password-based SSH is trivially brute-forced. Key authentication eliminates this entire class of attack.

**HST-003** `[SEC]`
The server runs automatic security updates for OS packages. `unattended-upgrades` (Debian/Ubuntu) or `dnf-automatic` (RHEL/Fedora) is configured and enabled.
> CIS Benchmark. OS-level CVEs are patched automatically without requiring manual intervention on every disclosure.

**HST-004** `[SEC]`
The origin server's IP address is not published or referenced in any public repository, DNS record, or documentation. If the IP changes, only Cloudflare's proxied A record is updated.
> The Cloudflare proxy is only effective if the origin IP remains unknown. A leaked IP allows attackers to bypass Cloudflare entirely.

**HST-005** `[REL]`
The Postgres data volume is backed up on a daily schedule to a location external to the server (object storage or a separate machine). Backups are tested by performing a restore to a local dev environment at least once.
> Recovery Point Objective (RPO) standard. A backup that has never been restored is not a backup — it is an untested assumption.

**HST-006** `[BP]`
Cloudflare Origin Certificates (issued free from the Cloudflare dashboard, valid for 15 years) are used on the Nginx origin server to satisfy Full (Strict) TLS mode.
> Practical tradeoff. Let's Encrypt certificates require port 80 to be reachable for ACME challenges, which is complicated behind a Cloudflare proxy. Origin certificates avoid this entirely and are trusted only by Cloudflare's infrastructure.

---

## 11. Nginx routing

**NGX-001** `[STD]`
Nginx uses `server_name` blocks to route by subdomain. `yourdomain.com` and `www` serve static frontend files. `api.yourdomain.com` proxies to FastAPI. `grafana.yourdomain.com` proxies to Grafana.
> Standard virtual hosting pattern. One Nginx instance handles all routing via subdomain without requiring multiple servers or ports.

**NGX-002** `[SEC]`
Nginx sets the full security header suite on all responses: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, `Permissions-Policy`, and a `Content-Security-Policy` appropriate to each subdomain.
> OWASP Secure Headers Project baseline. Set once in Nginx and they apply to every response from every service behind it.

**NGX-003** `[SEC]`
Nginx is configured to accept inbound connections only from Cloudflare's published IP ranges. Direct connections to the server IP on ports 80 and 443 are rejected with a 444 (no response).
> Without this, an attacker who discovers the origin IP can connect directly and bypass Cloudflare's WAF, DDoS protection, and access controls entirely. Cloudflare publishes its IP ranges at `https://www.cloudflare.com/ips/`.

**NGX-004** `[STD]`
Nginx passes the `CF-Connecting-IP` header to FastAPI as the real client IP. FastAPI logs use this value, not the Cloudflare proxy IP.
> Without this, all access logs show Cloudflare's IP ranges instead of actual visitor IPs, making abuse detection and debugging impossible.

**NGX-005** `[STD]` `[BP]`
Static frontend assets with content-hashed filenames are served with `Cache-Control: public, max-age=31536000, immutable`. The main `index.html` is served with `Cache-Control: no-cache`.
> Web performance standard. Immutable assets are cached indefinitely by Cloudflare's CDN and browsers. The HTML always revalidates so users receive updated asset references after each deploy.

---

## 12. Web interface

**WEB-001** `[BP]`
The web interface is built with plain HTML, CSS, and vanilla JavaScript for phase 1. No frontend framework is introduced until complexity justifies it.
> Simplicity first. A static site has no build pipeline, no Node.js dependency tree, and deploys as a file copy. Add a framework when it earns its place.

**WEB-002** `[FUNC]`
Phase 1 — Bill detail page. URL: `/bills/{id}`. Displays: bill number, title, status, chamber, sponsor name and party, date introduced, AI summary, theme label, keyword tags, and an economic snapshot table showing DOW, mortgage rate, dollar index, and median income for the week of introduction.

**WEB-003** `[FUNC]`
Phase 1 — Bill list page. URL: `/bills`. Paginated table of bills with columns for date, number, title, status, and theme. Filterable by status, theme, and chamber via URL query parameters.

**WEB-004** `[BP]`
All API calls from the frontend use the `api.yourdomain.com` subdomain. The API base URL is a single configurable constant at the top of the JS — not scattered across files.
> Changing the API URL in dev vs prod requires changing one value in one place.

**WEB-005** `[REL]`
The frontend handles all API response states explicitly: loading, empty result, and error. A blank page or unhandled promise rejection is never acceptable output.
> A blank page on API failure gives users no signal and makes debugging significantly harder.

**WEB-006** `[STD]`
The site is responsive and usable at a minimum viewport width of 375px. CSS Grid and media queries are sufficient — no mobile framework required.
> Web standard. Google's mobile-first indexing affects search ranking. A civic data tool should be accessible on any device.

**WEB-007** `[STD]`
Page titles, meta descriptions, and Open Graph tags are set on every page. The bill detail page uses the bill title and AI summary in its `og:description` tag.
> SEO and shareability. A civic data tool's reach depends on people sharing specific bills. Unfurled link previews in messaging apps drive that behavior.

**WEB-008** `[STD]`
A `robots.txt` is served at the domain root. Initially permissive (allow all crawlers). A `sitemap.xml` is generated or exposed via the API to help search engines discover individual bill pages.

---

## 13. Network & ISP (home server)

**NET-001** `[FUNC]`
The ISP is confirmed to allow inbound connections on ports 80 and 443 before any other work begins. Test by running a temporary server on port 443 and attempting to reach it from a device on mobile data (not home WiFi).
> Some residential ISPs block these ports silently at their infrastructure level or prohibit server hosting in their terms of service. This is a hard prerequisite — if blocked, no other requirement in this document is achievable without alternative approaches (see Cloudflare Tunnel as a workaround).

**NET-002** `[REL]`
The router is configured to forward ports 80 and 443 to the home server's local IP. The server is assigned a static local IP via DHCP reservation on the router — not a manually assigned static IP on the interface.
> Without a DHCP reservation, the server's local IP can change on reboot and silently break port forwarding rules.

**NET-003** `[SEC]`
Only ports 80, 443, and the custom SSH port are forwarded through the router. Postgres (5432), Grafana (3000), and FastAPI (8000) are never forwarded.
> The router firewall is the outermost security layer. Internal services must not be reachable from outside the LAN regardless of host firewall configuration.

**NET-004** `[SEC]`
The SSH port is changed from the default 22 to a non-standard port (e.g. 2222 or above 10000). Only this custom port is forwarded through the router.
> Not a substitute for key-based auth (HST-002), but significantly reduces automated brute-force scan noise. Residential IPs attract more scanning than cloud IPs because they are perceived as less monitored.

**NET-005** `[SEC]`
Fail2ban is installed and configured to ban IPs after repeated failed SSH attempts. The ban log is included in the Grafana dashboard.
> Standard SSH hardening for any publicly reachable server. Residential connections attract more automated scanning than cloud VMs.

---

## 14. Dynamic DNS

**DDNS-001** `[REL]`
A service running on the server monitors the public IP address and updates the Cloudflare DNS A record automatically whenever it changes.
> Residential ISPs assign dynamic IP addresses that can change on router reboot or at the ISP's discretion. Without DDNS the site goes down silently on every IP change.

**DDNS-002** `[BP]` `[STD]`
The DDNS updater uses the Cloudflare API with a scoped token (Zone / DNS / Edit for this zone only). It runs as a systemd timer every 5 minutes, consistent with the project's scheduling pattern.
> Cloudflare has no native DDNS client but its API makes this approximately 20 lines of Python. The systemd timer pattern is already established in this project for ingestion jobs.

**DDNS-003** `[OBS]`
The DDNS updater logs every IP check result and every update made. An alert (email or webhook) fires when the IP actually changes.
> An IP change indicates a router reboot or ISP action. Logging it helps diagnose any brief associated downtime.

**DDNS-004** `[REL]`
The Cloudflare DNS TTL for the A record is set to 1 minute (the minimum Cloudflare allows for proxied records).
> With a 5-minute DDNS check interval and 1-minute TTL, the maximum theoretical downtime window from an IP change is approximately 6 minutes. Higher TTLs extend this proportionally.

---

## 15. Hardware & uptime

**HW-001** `[REL]`
The server is configured to boot automatically after a power loss. The BIOS/UEFI "Restore on AC Power Loss" setting is set to "Power On".
> Home servers lose power during storms and outages. Without this setting the server stays off after power returns and requires manual intervention to resume service.

**HW-002** `[REL]`
All Podman services are configured with `restart: unless-stopped` in Podman Compose. The Podman socket service is enabled to start on boot via systemd (`systemctl enable podman`).
> After auto power-on (HW-001), containers must also restart automatically. Without this the server boots but serves nothing until manually started.

**HW-003** `[OBS]` `[REL]`
Uptime monitoring is configured via an external free-tier service (UptimeRobot or equivalent). It pings the `/health` endpoint every 5 minutes and sends an alert on failure.
> A server cannot reliably monitor its own uptime — if the server is down, any monitor running on it is also down. An external monitor is the only reliable signal for outages.

**HW-004** `[REL]`
The Postgres data volume resides on a separate drive or partition from the OS drive. An OS drive failure does not result in data loss.
> A single-drive setup means OS corruption, failed OS update, or drive failure simultaneously destroys all stored data.

**HW-005** `[FUNC]`
The project explicitly accepts home server uptime limitations. SLA expectations are best-effort, not 99.9%. This is a learning and portfolio project, not a production service with users depending on it.
> Honest scoping. A home server on a residential connection will have more downtime than a cloud VM. Documenting this prevents over-engineering of availability features to solve a problem that does not need solving at this stage.

---

## Requirement counts by section

| Section | Count |
|---|---|
| 1. Project-wide | 8 |
| 2. Database | 8 |
| 3. Data ingestion | 6 |
| 4. AI enrichment | 6 |
| 5. API | 6 |
| 6. Frontend | 5 |
| 7. Container security | 6 |
| 8. CI/CD | 7 |
| 9. DNS & Cloudflare | 8 |
| 10. Server & hosting | 6 |
| 11. Nginx routing | 5 |
| 12. Web interface | 8 |
| 13. Network & ISP | 5 |
| 14. Dynamic DNS | 4 |
| 15. Hardware & uptime | 5 |
| **Total** | **98** |

## Requirement counts by tag

| Tag | Count |
|---|---|
| `[SEC]` | 38 |
| `[BP]` | 28 |
| `[STD]` | 22 |
| `[FUNC]` | 12 |
| `[REL]` | 20 |
| `[OBS]` | 6 |

> Note: requirements with multiple tags are counted once per tag, so totals exceed 98.