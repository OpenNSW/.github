# OpenNSW
**_National Single Window for Trade Facilitation_**

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://www.apache.org/licenses/LICENSE-2.0)

**OpenNSW** is a centralized platform designed to streamline international trade by providing a single entry point for traders to interact with various Government Agencies. By decoupling process orchestration from domain-specific data, OpenNSW ensures a scalable and flexible ecosystem for managing consignments, certifications, and regulatory approvals.

**MVP Focus:** The initial release targets agricultural and food product exports (such as plant quarantine and coconut products), focusing on high-revenue HS codes with shared processes. The system handles consignment-level workflows such as Country of Origin certificates and Export Licenses, while injecting pre-consignment requirements (Business Registration, Environmental Protection License, TIN) via one-time verification.

<p align="center">
  • <a href="#repositories">Repositories</a>
  • <a href="#why-opennsw">Why OpenNSW?</a> •
  <a href="#key-features">Key Features</a> •
  <a href="#getting-started">Getting Started</a> •
  <a href="#deployment">Deployment</a> •
  <a href="#system-architecture">System Architecture</a> •
  <a href="#contributing">Contributing</a> •
  <a href="#license">License</a> •
</p>

---

## Repositories

The OpenNSW platform is organized across the following core repositories:

* **[core](https://github.com/OpenNSW/core):** A reusable Go SDK and workflow orchestration engine. It contains core packages for running long-lived Temporal workflows, micro interactive tasks (human-in-the-loop steps), payments, notifications, file storage, and authentication/authorization middleware.
* **[nsw-agency](https://github.com/OpenNSW/nsw-agency):** A pluggable, multi-tenant portal system that enables government or private agencies to review and approve trader-submitted data. A single codebase runs all agencies (e.g. NPQS, FCAU, CDA, SLPA), resolving branding and identity via environment variables at runtime.
* **[nsw-srilanka](https://github.com/OpenNSW/nsw-srilanka):** The deployer-specific application repository for the Sri Lanka instance of NSW. It wires together the `core` SDK with Sri Lanka–specific workflows (NPQS phytosanitary, FCAU health certificates, CDA, etc.), payment integrations (GovPay/LankaPay), and the Trader Portal frontend.
* **[nsw-gitops](https://github.com/OpenNSW/nsw-gitops):** The single source of truth for the continuous delivery of the NSW platform, utilizing ArgoCD and Helm umbrella charts for deploying infrastructure and applications to OpenShift.

---

## Why OpenNSW?

OpenNSW eliminates the complexity of manual, fragmented trade processes. It acts as the "orchestrator" for trade, allowing traders to submit documentation once and track the entire lifecycle of their consignment across multiple agencies.

Key architectural benefits include:

* **State vs. Data Decoupling:** The Core platform manages the workflow/process state, while independent Agency Portals handle specific domain-specific data.
* **Isolated Agency Modules:** Each agency maintains its own database and portal logic, ensuring data sovereignty and system stability.
* **Interoperability:** Seamlessly integrates with the National Data Exchange (NDX) for common data and ASYCUDA for customs finalization.
* **One-Time Verification:** Injects pre-consignment requirements (Business Registration, TIN, Environmental Protection License) directly into the workflow to reduce repetitive submissions.

---

## Key Features

OpenNSW offers powerful capabilities that streamline international trade processes:

| Feature | Status |
|---------|--------|
| **Core Platform Engine** – JSON-DSL-driven process orchestration managing process states without awareness of agency-specific data schemas | Implemented |
| **Trader Portal** – Single entry point for Traders to initiate consignments and track global status | Implemented |
| **Pluggable Agency Portals** – Independent units with agency-specific logic, databases, and officer portals (e.g., NPQS, FCAU, CDA, SLPA) | Implemented |
| **One-Time Verification** – Pre-consignment document injection (Business Registration, TIN, Environmental Protection License) | Implemented |
| **Automated Notifications** – Email and SMS alerts via background notification workers / Go task plugins | Implemented |
| **ASYCUDA Interface** – Automated handoff to Customs system upon completion of all agency approvals | Planned |
| **NDX Integration** – Fetching common data (BR number, VAT number) from external government providers | Under Evaluation |
| **Identity Provider Integration** – Centralized account management for Traders, Agency officers, and NSW Admins | Implemented |
| **Observability Stack** – Built-in OpenTelemetry for metrics, tracing, and logging | Planned |

---

## Project Structure

The OpenNSW ecosystem layout is structured as follows across its repositories:

```
OpenNSW/
├── core/                  # Reusable Go SDK (workflow interpreter, task manager, payments)
├── nsw-agency/            # Pluggable Agency review portals (SQLite/Postgres)
│   ├── backend/           # Go application server & database migrations
│   └── frontend/          # React/Vite portal for Agency officers
├── nsw-srilanka/          # Sri Lanka deployment instance
│   ├── configs/           # Agency workflows (CDA, FCAU, NPQS JSONForms)
│   ├── portals/           # Trader Portal frontend (React/Vite)
│   ├── cmd/server/        # NSW Backend API Server
│   └── idp/               # Identity Provider config
└── nsw-gitops/            # ArgoCD & Helm umbrella charts for continuous delivery
```

---

## System Architecture

The NSW system is built on a distributed microservices architecture to maintain high availability and modularity.

### Core Components

* **Identity Provider (IDP):** Manages all accounts for Traders, Agency officers, and NSW Admins. Provides centralized authentication and authorization.
* **Trader Portal:** Single entry point for Traders to initiate consignments and track global status across all agencies.
* **Core Workflow Engine:** Orchestrates process states (e.g., "Waiting for Approval") using a BPMN 2.0 interpreter, agnostic to agency-specific data schemas.
* **Agency Portals:** Independent, pluggable units containing agency-specific review logic, SQLite/Postgres databases, and officer portals.
* **National Data Exchange (NDX):** Bridge for fetching common trade data (BR number, VAT number) from external government systems.
* **ASYCUDA Integration:** Automated handoff to Customs system upon completion of all agency approvals.

### Architecture Principles

* **State vs. Data Decoupling:** The Core platform manages Process State, while independent Agency Portals manage Domain Data (e.g., certificate details, specifications). This separation ensures the core workflow remains agnostic to agency-specific schemas.
* **Isolated Agency Databases:** Each agency maintains its own database (SQLite/Postgres) and portal logic, ensuring data sovereignty and system stability.
* **Dual Portal Views:** Traders interact with the unified Trader Portal (in `nsw-srilanka`) to submit data, while Agency Officers use dedicated Agency Portals (in `nsw-agency`) to review and approve submissions.
* **Source of Truth:** Agency-specific data resides in the agency's own database, keeping it isolated from the core NSW database.
* **Callback-Based Workflow:** Agency systems send success/fail callbacks to the Core backend via M2M APIs to advance the workflow state machine.

### The Consignment Journey

1. **Initialization:** Trader selects an HS Code through the Trader Portal; the Core Workflow Engine triggers the relevant BPMN workflow.
2. **Submission:** Trader submits agency-specific forms via the Trader Portal, which are injected into the respective Agency's backend.
3. **Notification:** The Agency system alerts the relevant officer via email or SMS.
4. **Review:** An Agency Officer reviews the submission within their isolated Agency Portal.
5. **Decision:** The Officer approves or denies the request within their portal.
6. **Callback:** The Agency backend sends a success/fail callback to the NSW Core backend to advance the workflow state.
7. **Finalization:** Once all agency approvals are complete, the Core backend initiates the Customs (ASYCUDA) interface.

---

## Getting Started

To run the full local development environment, you will run `nsw-srilanka` and `nsw-agency` side-by-side.

**Prerequisites:** Go 1.26+, Node.js (with `pnpm`), Docker, Temporal CLI

### Setup & Run Commands

```bash
# Terminal 1 – Sri Lanka Core Platform (IDP, Temporal, backend, trader-app)
cd nsw-srilanka
cp .env.example .env
cp idp/.env.example idp/.env
cp configs/services.docker.example.json configs/services.docker.json
cp configs/payment_methods.example.json configs/payment_methods.json
cp configs/notification.example.json configs/notification.json

# Start core platform
make dev

# Terminal 2 – Agency Portals (NPQS, FCAU, CDA, SLPA)
cd ../nsw-agency
# Copy environmental configurations
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env

# Start all agency portals
./start-dev.sh all
# Wipe databases and start fresh:
./start-dev.sh all --clean-run
```

### Local Services URL Map

| Service / Portal | URL | Port / Config |
|------------------|-----|---------------|
| **Trader Portal** | http://localhost:5173 | Frontend Vite Server |
| **NSW Core API Backend** | http://localhost:8080 | Go HTTP Server |
| **Identity Provider (IDP)** | https://localhost:8090 | Thunder IDP |
| **NPQS Agency Portal** | http://localhost:5174 | Backend Port: `8081` |
| **FCAU Agency Portal** | http://localhost:5175 | Backend Port: `8082` |
| **CDA Agency Portal** | http://localhost:5176 | Backend Port: `8083` |
| **SLPA Agency Portal** | http://localhost:5177 | Backend Port: `8084` |

---

## Deployment

Deployment of the OpenNSW ecosystem is fully containerized and automated:

* **Continuous Delivery (GitOps):** Managed via [nsw-gitops](https://github.com/OpenNSW/nsw-gitops), using ArgoCD to deploy Helm umbrella charts (`infra-umbrella` for backing services and `apps-umbrella` for the application tier) on OpenShift.
* **CI/CD Workflows:** Individual repositories contain GitHub Actions workflows to validate builds, run tests, and publish Docker images to GHCR (e.g. `nsw-agency` release workflows).

---

## Contributing

Thank you for wanting to contribute to the National Single Window project. Please see the individual repository `CONTRIBUTING.md` files for more details.

## License 

Distributed under the Apache 2.0 License. See [License](https://www.apache.org/licenses/LICENSE-2.0) for more information.
