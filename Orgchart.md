# Capital Investment — Architecture Diagram

```mermaid
graph LR

    %% ── External sources ──────────────────────────────────────
    subgraph EXT["External services"]
        direction TB
        CONGRESS["Congress.gov API\nBills + votes"]
        FRED["FRED API\nEcon indicators"]
        YAHOO["Yahoo Finance\nDOW, S&P 500"]
        CLAUDE["Claude API\nAI enrichment"]
        UPTIME["UptimeRobot\nExternal monitor"]
    end

    %% ── Cloudflare ────────────────────────────────────────────
    subgraph CF["Cloudflare"]
        direction TB
        PROXY["Proxy + CDN\nHides origin IP"]
        TLS["TLS termination\nFull Strict mode"]
        WAF["WAF\nDDoS · SQLi · XSS"]
        DNS["DNS\nDDNS auto-update"]
        ACCESS["Access\nGrafana gating"]
        GHA["GitHub Actions\nCI/CD · Bandit · Trivy"]
    end

    %% ── Home server ───────────────────────────────────────────
    subgraph SERVER["Home server"]
        direction TB

        FW["ufw firewall\nPorts 80 · 443 · SSH only"]

        subgraph PODMAN["Podman — rootless containers"]
            direction TB

            NGINX["Nginx\nReverse proxy · TLS · CF-IP allowlist · sec headers"]

            subgraph SERVING["Serving layer"]
                direction LR
                STATIC["Static frontend\nBill list + detail"]
                API["FastAPI\nREST · OpenAPI docs"]
                GRAFANA["Grafana\nDashboards"]
            end

            PG["PostgreSQL\nbills · econ_snapshots · keywords · ingestion_runs"]

            subgraph JOBS["Background jobs"]
                direction LR
                FLYWAY["Flyway\nSchema migrations"]
                INGEST["Ingest jobs\nCongress + FRED"]
                ENRICH["AI enrichment job\nOllama dev · Claude API prod"]
            end

            TIMERS["Systemd timers\nIngest · enrich · DDNS updater"]
        end
    end

    %% ── Data flows — ingestion ────────────────────────────────
    CONGRESS -->|bills| INGEST
    FRED -->|indicators| INGEST
    YAHOO -->|prices| INGEST
    CLAUDE -->|summaries + keywords| ENRICH

    %% ── Data flows — internal ────────────────────────────────
    INGEST -->|upsert| PG
    ENRICH -->|write back| PG
    FLYWAY -->|migrate| PG
    TIMERS -.->|schedule| INGEST
    TIMERS -.->|schedule| ENRICH
    API -->|query| PG
    GRAFANA -->|query| PG

    %% ── Public traffic ───────────────────────────────────────
    CF -->|proxied HTTPS| NGINX
    NGINX --> STATIC
    NGINX --> API
    NGINX --> GRAFANA

    %% ── Monitoring + deploy ──────────────────────────────────
    UPTIME -.->|health check| NGINX
    GHA -.->|SSH deploy| SERVER

    %% ── Firewall sits in front of Podman ─────────────────────
    FW --> PODMAN

    %% ── Styles ───────────────────────────────────────────────
    classDef cloud    fill:#EEEDFE,stroke:#534AB7,color:#26215C
    classDef core     fill:#E1F5EE,stroke:#0F6E56,color:#04342C
    classDef ai       fill:#FAECE7,stroke:#993C1D,color:#4A1B0C
    classDef neutral  fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A
    classDef infra    fill:#E6F1FB,stroke:#185FA5,color:#042C53

    class PROXY,TLS,WAF,DNS,ACCESS,GHA cloud
    class NGINX,STATIC,API,GRAFANA,PG,FLYWAY,INGEST,TIMERS core
    class CLAUDE,ENRICH ai
    class CONGRESS,FRED,YAHOO,UPTIME neutral
    class FW infra
```

---

## Zone key

| Zone | Contents |
|---|---|
| External services | APIs and monitors outside your control |
| Cloudflare | Edge proxy, WAF, DNS, TLS termination, CI/CD |
| Home server | Everything you own and operate |
| Podman | All containerized services, rootless |

## Line key

| Line | Meaning |
|---|---|
| Solid arrow | Live data flow |
| Dashed arrow | Scheduled, automated, or deployment trigger |

## Color key

| Color | Meaning |
|---|---|
| Purple | Cloudflare infrastructure |
| Teal | Core services inside Podman |
| Coral | AI layer (Claude API + enrichment job) |
| Gray | External neutral services |
| Blue | Host-level infrastructure (firewall) |