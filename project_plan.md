# LexWatch — Build Roadmap

This document defines the order in which to build the project and the rationale for each phase. The ordering follows one principle: **never build on an unvalidated foundation**. Each phase proves something before the next phase depends on it.

---

## Table of contents

1. [Phase 1 — Git repo + secrets baseline](#phase-1--git-repo--secrets-baseline)
2. [Phase 2 — Home server + network hardening](#phase-2--home-server--network-hardening)
3. [Phase 3 — Domain + Cloudflare DNS + DDNS](#phase-3--domain--cloudflare-dns--ddns)
4. [Phase 4 — Database + Flyway migrations](#phase-4--database--flyway-migrations)
5. [Phase 5 — Data ingestion + CI/CD pipeline](#phase-5--data-ingestion--cicd-pipeline)
6. [Phase 6 — AI enrichment + FastAPI](#phase-6--ai-enrichment--fastapi)
7. [Phase 7 — Frontend + Nginx routing + security headers](#phase-7--frontend--nginx-routing--security-headers)
8. [Phase 8 — Observability + backups + timeline views](#phase-8--observability--backups--timeline-views)

---

## Phase 1 — Git repo + secrets baseline

**Requirements:** PRJ-001 · PRJ-002 · PRJ-003 · PRJ-004

### What to build

- Initialize the git repository and push to GitHub
- Configure branch protection on `main` — block direct pushes, require pull requests
- Add `.gitignore` covering `secrets/.env`, `__pycache__`, `.venv`, and log files
- Create `secrets/.env.example` documenting every required environment variable with a description and placeholder value
- Define the two runtime profiles (`dev` and `prod`) and document how they differ

### Why first

Once a secret touches git history it is there permanently — even after deletion, it lives in the reflog and any clones made before the removal. This is the one mistake that cannot be undone without a destructive force-push and invalidating every contributor's local clone.

Setting up the repository structure and secrets discipline before writing a single line of application code costs about 30 minutes and eliminates an entire class of irreversible mistakes.

### Unlocks

Everything else. No phase can start without a repository and a safe place to store credentials.

---

## Phase 2 — Home server + network hardening

**Requirements:** NET-001 · NET-002 · NET-003 · NET-004 · NET-005 · HST-001 · HST-002 · HST-003 · HW-001 · HW-002

### What to build

- Confirm the ISP allows inbound traffic on ports 80 and 443 — test by running a temporary listener and hitting it from a mobile data connection (not home WiFi)
- Assign the server a static local IP via DHCP reservation on the router
- Forward ports 80, 443, and the custom SSH port through the router — nothing else
- Change the SSH port from 22 to a non-standard port
- Disable password authentication in `/etc/ssh/sshd_config` (`PasswordAuthentication no`) and confirm key-based auth works before closing the session
- Configure `ufw` or `firewalld` to block all ports except 80, 443, and the SSH port
- Enable automatic OS security updates (`unattended-upgrades` on Debian/Ubuntu, `dnf-automatic` on RHEL/Fedora)
- Install and configure Fail2ban for SSH brute-force protection
- Set BIOS/UEFI "Restore on AC Power Loss" to Power On
- Enable the Podman socket service to start on boot (`systemctl enable podman`)

### Why second

Developers instinctively want to write code first and deal with deployment later. On a home server that is backwards. If the ISP blocks port 443 (NET-001), nothing built after this point will be publicly reachable. Find that out on day two, not day thirty.

These are physical and network-layer concerns. They do not change once set, and they need to be correct before anything is put on the internet.

### Unlocks

Domain purchase and DNS configuration — it is pointless to buy a domain and configure Cloudflare if the server cannot receive inbound traffic.

---

## Phase 3 — Domain + Cloudflare DNS + DDNS

**Requirements:** DNS-001 · DNS-002 · DNS-003 · DNS-004 · DNS-005 · DNS-006 · DNS-007 · DNS-008 · DDNS-001 · DDNS-002 · DDNS-003 · DDNS-004 · HST-004 · HST-006

### What to build

- Purchase the domain through Cloudflare Registrar (or transfer an existing domain)
- Create proxied A records for the root domain, `www`, `api`, and `grafana` subdomains pointing to the server IP
- Set Cloudflare SSL/TLS mode to **Full (Strict)**
- Enable "Always Use HTTPS" in the Cloudflare dashboard
- Enable the Cloudflare WAF with the default managed ruleset
- Restrict the `grafana` subdomain via Cloudflare Access or IP allowlisting
- Issue a Cloudflare Origin Certificate (free, 15-year validity) from the Cloudflare dashboard and install it in Nginx
- Create a scoped Cloudflare API token (Zone / DNS / Edit for this zone only) and store it in `secrets/.env`
- Write the DDNS updater script — approximately 20 lines of Python using the Cloudflare API — and deploy it as a systemd timer running every 5 minutes
- Set the Cloudflare DNS TTL for the A record to 1 minute

### Why third

Do this before any application exists so the routing layer can be tested independently with a simple Nginx "hello world" response. Verifying the full `browser → Cloudflare → Nginx → response` path early means any application problems encountered later are definitely application problems, not DNS or TLS problems. Mixing those failure modes wastes significant debugging time.

The DDNS updater belongs here because a residential IP can change at any time. Deploying it at the same time as DNS setup means the domain stays pointed at the server from day one, regardless of IP changes.

### Unlocks

All public-facing services. The domain now resolves through Cloudflare to the server with TLS working end-to-end.

---

## Phase 4 — Database + Flyway migrations

**Requirements:** DB-001 · DB-002 · DB-003 · DB-004 · DB-005 · DB-006 · DB-007 · DB-008 · CTR-001 · CTR-002 · CTR-005 · PRJ-005 · PRJ-006

### What to build

- Write the Podman Compose file defining the `db` (Postgres) and `flyway` containers
- Pin both images to specific version tags — no `:latest`
- Configure both containers to run as non-root users
- Create a named Podman volume for Postgres data
- Configure the Podman internal network so Postgres is not exposed to the host
- Write `V1__init_schema.sql` implementing the full schema: `bills`, `bill_ai_enrichment`, `bill_keywords`, `econ_snapshots`, `ingestion_runs`
- Create the low-privilege application database user with `SELECT`, `INSERT`, `UPDATE`, `DELETE` only
- Create a separate Flyway migration user with DDL rights, used only at startup
- Add all indexes defined in DB-008
- Verify: running `flyway migrate` against a fresh database produces correct tables with no errors

### Why fourth

Everything else in the stack reads from or writes to this database. It must exist and be correct before any other service is built.

Schema mistakes caught here cost minutes to fix. Schema mistakes caught after the ingestion code is written and has been populating tables cost hours — potentially requiring data migration scripts or a full database wipe in development.

Doing the Podman and container security work here (CTR-001, CTR-002, CTR-005, PRJ-005, PRJ-006) also establishes the container patterns that every subsequent service will follow.

### Unlocks

All ingestion, enrichment, and API work.

---

## Phase 5 — Data ingestion + CI/CD pipeline

**Requirements:** ING-001 · ING-002 · ING-003 · ING-004 · ING-005 · ING-006 · CI-001 · CI-002 · CI-003 · CI-004 · CI-005 · CI-006 · CI-007 · PRJ-007 · PRJ-008

### What to build

- Write the Congress.gov ingestion script with Pydantic validation and upsert logic
- Write the FRED and Yahoo Finance economic ingestion script
- Deploy both as systemd service + timer pairs, version-controlled in the repo
- Set explicit timeouts on all outbound HTTP calls
- Write the GitHub Actions CI workflow with:
  - Bandit SAST on all Python files (medium severity and above fails the build)
  - pytest unit tests covering transform and parse logic
  - Integration tests running against a Postgres container spun up in CI
  - Trivy container image scanning (High/Critical CVEs fail the build)
  - Pinned `requirements.txt` and `requirements-dev.txt`
  - Weekly scheduled vulnerability scan workflow
- Store all CI secrets in GitHub Actions secrets — not in workflow YAML
- Configure structured JSON logging across both ingestion jobs
- Add the `/health` liveness check to each service

### Why fifth

This is the first real Python code in the project, and CI-002 (Bandit SAST) and CI-004 (pinned dependencies) need to be enforced from the very first commit of application code. Running security scanning on code that already exists in production has already failed the shift-left principle.

The Congress.gov and FRED ingestion jobs are built together because they have identical patterns — fetch, validate, upsert, log the run — and can share utility code. By the end of this phase, real congressional bill data and real economic data are sitting in the Postgres tables, and CI is blocking any commit that fails security checks or tests.

### Unlocks

AI enrichment and the API — there is now real data to enrich and serve.

---

## Phase 6 — AI enrichment + FastAPI

**Requirements:** ENR-001 · ENR-002 · ENR-003 · ENR-004 · ENR-005 · ENR-006 · API-001 · API-002 · API-003 · API-004 · API-005 · API-006 · CTR-003 · CTR-004

### What to build

- Stand up Ollama in a Podman container (dev profile only) and pull the configured local model
- Write the enrichment job with the `AI_BACKEND` switcher (`ollama` vs `claude`)
- Write the system prompt instructing the AI to return structured JSON only
- Implement JSON schema validation on the AI response before any database write
- Store `model_used` and `prompt_version` with every enrichment record
- Verify enrichment produces clean summaries and keyword tags against a sample of real bills
- Build the FastAPI application with all read-only endpoints:
  - `GET /api/bills` — paginated, filterable list
  - `GET /api/bills/{id}` — full bill detail with enrichment and economic snapshot
  - `GET /api/econ/{series_id}` — time-series data for one indicator
  - `GET /health` — liveness check
- Configure CORS to restrict to the project domain in prod
- Add Trivy scanning for the FastAPI container image in CI (CTR-003)
- Mount static assets read-only in all applicable containers (CTR-004)

### Why sixth

These two components go together because the API's bill detail endpoint is only meaningful once enrichment data exists. Testing it against un-enriched bills returns nulls and gives no useful signal about whether the endpoint is working correctly.

By the end of this phase the full backend pipeline runs end-to-end: ingest → enrich → serve. The API is live at `api.yourdomain.com` with auto-generated OpenAPI documentation.

### Unlocks

The frontend and the public-facing site.

---

## Phase 7 — Frontend + Nginx routing + security headers

**Requirements:** WEB-001 · WEB-002 · WEB-003 · WEB-004 · WEB-005 · WEB-006 · WEB-007 · WEB-008 · NGX-001 · NGX-002 · NGX-003 · NGX-004 · NGX-005 · FE-001 · FE-004 · FE-005 · CTR-006

### What to build

- Write the static frontend — bill list page (`/bills`) and bill detail page (`/bills/{id}`) in plain HTML, CSS, and vanilla JavaScript
- Handle all three response states explicitly in JS: loading, empty, and error
- Set the API base URL as a single configurable constant
- Implement responsive layout working at 375px minimum viewport width
- Add page titles, meta descriptions, and Open Graph tags to every page
- Add `robots.txt` and `sitemap.xml`
- Configure Nginx `server_name` blocks for all subdomains
- Set the full security header suite on all Nginx responses: `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`, `Content-Security-Policy`
- Configure Nginx to accept connections only from Cloudflare's published IP ranges — reject direct connections to the server IP with a 444 response
- Configure `CF-Connecting-IP` passthrough to FastAPI for real client IP logging
- Set `Cache-Control: immutable` on hashed static assets, `no-cache` on `index.html`
- Install the Cloudflare Origin Certificate in the Nginx container (CTR-006)
- Switch the enrichment job to the Claude API backend (`AI_BACKEND=claude`) in prod

### Why seventh

The security headers (NGX-002) and Cloudflare IP restriction (NGX-003) apply to every response Nginx sends. Getting them right before the site is publicly indexed is important — a site indexed by Google without correct security headers, or with origin IP bypass possible, is harder to remediate after the fact.

NGX-003 in particular closes the origin IP bypass risk that exists from the moment the domain goes live in Phase 3. Combining it with the frontend build here means the site goes from internal-only to fully public in one controlled step.

### Unlocks

The project going live. This is the first moment a public user can visit the site and see real data.

---

## Phase 8 — Observability + backups + timeline views

**Requirements:** HW-003 · HW-004 · HW-005 · HST-005 · DDNS-003 · FE-002 · FE-003

### What to build

- Configure UptimeRobot (or equivalent external monitor) to ping `/health` every 5 minutes with email alerting on failure
- Set up Grafana connected to Postgres — dashboards for pipeline run history, ingestion record counts, enrichment queue depth, and economic indicator charts
- Write and schedule the Postgres backup script — daily export to external storage
- Perform a test restore of the backup to a local dev environment and document the procedure
- Add the IP change alert to the DDNS updater (email or webhook on actual IP change)
- Build the Phase 2 frontend view: shared timeline of bills and economic indicators on a common date axis
- Build the Phase 3 frontend view: keyword and theme dashboard with date range filtering

### Why last

Grafana dashboards and the timeline views require accumulated data to be meaningful. Dashboards with two days of pipeline runs tell you nothing useful. After six to eight weeks of daily ingestion they show trends, gaps, and anomalies that are actually actionable.

Similarly, the Phase 2 timeline view — bills overlaid on economic indicator lines — only tells a visual story once there are weeks of bills and weeks of economic data to correlate. Building it in Phase 7 would produce a nearly empty chart.

Observability before these views also means pipeline issues appear in Grafana before users notice anything wrong in the frontend.

### Unlocks

Long-term maintainability. The project is now self-monitoring, backed up, and expanding in feature scope with real data to support it.

---

## Summary

| Phase | Focus | Key unlock |
|---|---|---|
| 1 | Git + secrets | Safe foundation for everything |
| 2 | Server + network | Confirmed the server can receive traffic |
| 3 | Domain + Cloudflare + DDNS | Public domain with TLS working end-to-end |
| 4 | Database + Flyway | Correct schema before any code writes to it |
| 5 | Ingestion + CI/CD | Real data in the database, security gates in CI |
| 6 | AI enrichment + API | Full backend pipeline running, API live |
| 7 | Frontend + Nginx | Site publicly accessible with security hardening |
| 8 | Observability + backups + views | Self-monitoring, recoverable, feature-complete |

## Key sequencing principles

**Network before application.** Confirming the ISP allows inbound traffic (NET-001) is a hard prerequisite. If it does not, no amount of application work matters. Find this out in Phase 2, not Phase 7.

**CI before features.** GitHub Actions is wired up in Phase 5 at the same time as the first application code — not retrofitted after. Security scanning on code that already exists in production has already failed the shift-left principle.

**Schema before code.** The Flyway migration is written and verified in Phase 4 before any ingestion or API code exists. Schema mistakes caught against an empty database cost minutes. Schema mistakes caught after weeks of ingestion data has been written can be costly to remediate.

**Data before dashboards.** Grafana and the advanced frontend views are Phase 8 because they require accumulated data to be visually meaningful. Building them earlier produces empty or misleading displays.