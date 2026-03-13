# ✈️ Contrail

**A Microsoft Flight Simulator 2024 companion app — and a production-grade DevSecOps portfolio platform.**

Contrail is intentionally small in feature scope. The engineering investment is in the platform around it: secure CI/CD pipelines, GitOps delivery, Kubernetes orchestration, policy-as-code enforcement, and full-stack observability.

---

## Features

| Feature | Description |
|---------|-------------|
| 🗺️ **Flight Planner** | Search real-world routes between airports, preview on an interactive map, export directly as `.pln` for MSFS 2024 |
| 📡 **Live Tracker** | Connects to MSFS 2024 via SimConnect SDK — streams real-time telemetry (position, altitude, airspeed) to a live map |
| 📒 **Flight Logbook** | Auto-logs flights from SimConnect takeoff/landing detection. Manual entry supported |
| 📊 **Stats Dashboard** | Total hours, aircraft flown, airports visited, route heatmap |

Airport and route data sourced from [OurAirports](https://ourairports.com/data/) and [OpenFlights](https://openflights.org/data.html) — open datasets, no runtime API calls.

---

## Architecture

```
React + Vite  ──REST/WS──►  Go Backend  ──SimConnect──►  MSFS 2024
                                │
                    ┌───────────┴────────────┐
              contrail_world          contrail_app
              (airports, routes)      (users, logbook, plans)
```

See [`/docs/diagrams/`](./docs/diagrams/) for full system, infrastructure, database, and CI/CD pipeline diagrams.

---

## Tech Stack

### Application
| Layer | Technology |
|-------|-----------|
| Frontend | React 18, Vite, Leaflet.js |
| Backend | Go 1.23, Chi router |
| Database | PostgreSQL 16 (two databases) |
| Real-time | WebSockets (gorilla/websocket) |
| SimConnect | Go bindings for SimConnect SDK |

### Platform & DevSecOps
| Concern | Tooling |
|---------|---------|
| Containerization | Docker, Docker Compose |
| Orchestration | Kubernetes (K3s local, AKS cloud) |
| IaC | Terraform |
| GitOps | Argo CD |
| CI/CD | GitHub Actions |
| Secret Scanning | Gitleaks |
| SAST | CodeQL |
| Container Scanning | Trivy |
| Dependency Audit | Snyk |
| SBOM | Syft (SPDX format) |
| Image Signing | Cosign (keyless, Sigstore) |
| IaC Scanning | Checkov |
| Policy as Code | OPA + Conftest |
| Observability | Prometheus, Grafana, Loki |

---

## Repository Structure

```
contrail/
├── .github/
│   └── workflows/
│       ├── ci.yaml              # Main CI pipeline
│       ├── release.yaml         # Tag-triggered release + signing
│       └── dependency-review.yaml
├── backend/
│   ├── cmd/contrail/            # main.go entry point
│   ├── internal/
│   │   ├── api/                 # HTTP handlers
│   │   ├── simconnect/          # SimConnect bridge goroutine
│   │   ├── websocket/           # WebSocket hub
│   │   ├── db/                  # Postgres pool, queries
│   │   ├── seeder/              # OurAirports/OpenFlights CSV import
│   │   └── pln/                 # .pln XML generator
│   ├── Dockerfile
│   └── go.mod
├── frontend/
│   ├── src/
│   │   ├── components/          # React components
│   │   ├── pages/               # Route-level pages
│   │   ├── hooks/               # useSimConnect, useFlightPlan, etc.
│   │   └── api/                 # Typed API client
│   ├── Dockerfile
│   └── package.json
├── k8s/
│   ├── base/                    # Kustomize base manifests
│   │   ├── api-deployment.yaml
│   │   ├── frontend-deployment.yaml
│   │   ├── postgres-statefulset.yaml
│   │   └── ingress.yaml
│   └── overlays/
│       ├── local/               # K3s overrides
│       └── production/          # AKS overrides
├── terraform/
│   ├── local/                   # K3s / namespace bootstrap
│   └── azure/                   # AKS cluster, networking
├── policy/
│   └── conftest/                # OPA Rego policies
│       ├── no_root.rego
│       ├── required_limits.rego
│       ├── no_latest_tag.rego
│       └── required_labels.rego
├── monitoring/
│   ├── prometheus/
│   │   └── contrail-rules.yaml  # Custom alerting rules
│   └── grafana/
│       └── dashboards/          # Provisioned dashboard JSON
├── docs/
│   ├── diagrams/                # Mermaid source files
│   ├── API.md                   # Full API reference
│   └── ADR/                     # Architecture Decision Records
│       └── 001-two-databases.md
├── docker-compose.yaml
├── docker-compose.dev.yaml
└── README.md
```

---

## Getting Started

### Prerequisites
- Docker & Docker Compose
- Go 1.23+
- Node.js 20+
- MSFS 2024 (for SimConnect features)

### Quick Start (Docker Compose)

```bash
git clone https://github.com/GraysonMor31/contrail.git
cd contrail

# Start all services (API, frontend, two Postgres instances)
docker compose up -d

# Seed world data (runs once — imports OurAirports + OpenFlights CSVs)
docker compose exec api ./contrail seed

# Open in browser
open http://localhost:5173
```

### Local K8s (K3s)

```bash
# Apply manifests via Kustomize
kubectl apply -k k8s/overlays/local/

# Or let Argo CD manage it after bootstrapping:
kubectl apply -f k8s/argocd/app.yaml
```

---

## Security Pipeline

Every pull request runs:

1. **Gitleaks** — secret detection, fails build on any hit
2. **CodeQL** — static analysis (Go + JavaScript)
3. **Trivy** — container image vulnerability scan (CRITICAL = hard fail)
4. **Snyk** — dependency audit (Go modules + npm)
5. **Syft** — SBOM generation in SPDX format, attached to releases
6. **Checkov** — Terraform and K8s manifest IaC scanning
7. **Conftest/OPA** — custom policy gates (no root containers, required resource limits, no `:latest` image tags)
8. **Cosign** — keyless image signing via OIDC (Sigstore transparency log)

---

## Observability

- **Prometheus** scrapes `/metrics` from the Go backend (custom flight event counters, SimConnect connection state, DB pool metrics)
- **Grafana** dashboards provisioned from `monitoring/grafana/dashboards/`
- **Loki** aggregates structured JSON logs from all containers

---

## API

Full reference: [`docs/API.md`](./docs/API.md)

Base URL: `http://localhost:8080/api/v1`  
Auth: Bearer token (`Authorization: Bearer <token>`)

---

## Data Sources

| Dataset | Source | License |
|---------|--------|---------|
| Airport/Runway/Region/Country/Frequency data | [OurAirports](https://ourairports.com/data/) | Public Domain |
| Route/Airline data | [OpenFlights](https://openflights.org/data.html) | Open Database License |
| Navaid/Airway/Waypoint data | [X-Plane Developer](https://developer.x-plane.com/article/navdata-in-x-plane-11/) | GPL v.3 |
| Weather | [OpenWeatherMap](https://openweathermap.org/api) | Rate Limit API |
| METAR/TAF/SIGMETS/AIRMETS/PIREPS | [AviationWeather]( https://aviationweather.gov/api/data) | Rate Limit API |

Data is seeded into a local Postgres instance at startup. No runtime API calls to external services.

---

## License

Free Software Foundation General Public License (GPL) v.3 — see [LICENSE](./LICENSE)

---

## Author

**Grayson** — [@GraysonMor31](https://github.com/GraysonMor31)  
Built as a DevSecOps portfolio project.
