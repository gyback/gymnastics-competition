---
status: accepted
date: 2026-11-12
decision-makers: Gustav Gybäck
---

# Cloud hosting platform and Infrastructure as Code — Azure Container Apps in Sweden Central, Bicep with Azure Verified Modules in the private repo and a portable container deployment contract

<!-- Related issue: #8 -->

## Context and Problem Statement

The [TeamGym](https://www.europeangymnastics.com/discipline/teamgym/intro) competition service needs a hosted home. Traffic is low most of the year and peaks on competition days, when clubs, coaches and organisers check **Schedules** across **Floor**, **Tumble** and **Trampet** from their phones in sports halls. The project is run by a solo maintainer on a small budget.

Earlier ADRs have already fixed most of what is deployed:

- [ADR-0002](0002-repository-strategy.md): this public repository publishes versioned, immutable OCI images to GHCR. A **private infrastructure repository** deploys pinned versions by digest. The public repository holds no cloud credentials or environment identifiers.
- [ADR-0004](0004-backend-runtime-and-architectural-style.md) and [ADR-0005](0005-frontend-framework-and-api-contract.md): **one** `.NET 10` container (`TeamGym.Arena.Api`) serves the API, the BFF and the compiled React SPA on the same origin. `ServiceDefaults` provides OpenTelemetry and health checks.
- [ADR-0007](0007-primary-data-store-and-data-access.md): PostgreSQL 17 on **Azure Database for PostgreSQL — Flexible Server** in `swedencentral`, Entra managed identity authentication, and schema migrations that run as a separate step **before** a new API revision rolls out.
- [ADR-0008](0008-identity-authentication-and-authorisation.md): Microsoft Entra External ID, Data Protection keys in Blob Storage protected by Key Vault, and an e-mail delivery provider that was left for this ADR.

We need to decide:

1. **The compute platform and region** for the API container and the migration step.
2. **The IaC tool** used in the private infrastructure repository.
3. **Environments**, and whether per-PR preview environments are wanted.
4. **How deployments run** from GitHub Actions without long-lived secrets.
5. **Secrets, networking, observability and e-mail delivery.**
6. **The public deployment contract**: what the published images promise, so that anyone can deploy them without the private repository.
7. **The answers to the open questions**: the monthly budget ceiling, and who owns the custom domain.

## Decision Drivers

- **Low running cost at small scale**, with scale-to-zero where it does not hurt users, and headroom for competition-day peaks.
- **Managed services over self-operated infrastructure** for a solo maintainer.
- **EU data residency (GDPR):** all data, logs and e-mail processing stay in the EU/EEA, preferably in Sweden.
- **A portable application:** 12-factor configuration, standard containers, and no cloud provider SDKs in domain code. Self-hosting clubs or federations must be able to run the images elsewhere.
- **No long-lived secrets:** GitHub Actions deploys with OIDC federated credentials, and the application reaches Azure services with managed identities.
- **An IaC tool that is industry standard** and well understood by both humans and AI agents ([ADR-0003](0003-engineering-workflow-and-quality-gates.md)).

## Considered Options

### 1. Compute Platform

- **Option C1:** **Azure Container Apps** (Consumption plan), with the migration step as a Container Apps **job**.
- **Option C2:** **Azure App Service** (Linux, Web App for Containers), optionally with **Static Web Apps** for the frontend.
- **Option C3:** **Azure Kubernetes Service (AKS)**.

### 2. Infrastructure as Code Tool (private repository)

- **Option T1:** **Bicep** with **Azure Verified Modules (AVM)** and deployment stacks.
- **Option T2:** **Terraform / OpenTofu** with the `azurerm` and `azuread` providers.
- **Option T3:** **Pulumi** in C#.

## Decision Outcome

Chosen options:

1. **Compute: Option C1.** Host `TeamGym.Arena.Api` on **Azure Container Apps** (Consumption plan) in **Sweden Central** (`swedencentral`), and run schema migrations as a **Container Apps job** before each rollout.
2. **IaC: Option T1.** The private infrastructure repository uses **Bicep with Azure Verified Modules**, deployed as **deployment stacks** from GitHub Actions with OIDC federated credentials.

The application only has to meet the **public deployment contract** (section 7). Nothing in this repository depends on Azure Container Apps or Bicep.

### 1. Region and Resource Topology

- **Region:** Every resource is created in **Sweden Central** (`swedencentral`), matching the database ([ADR-0007](0007-primary-data-store-and-data-access.md)). The Entra external tenants are located in **Europe** ([ADR-0008](0008-identity-authentication-and-authorisation.md)). Azure Communication Services uses the **Europe** data location.
- **Per environment**, in its own resource group (`rg-teamgym-<env>`):

| Resource | Purpose |
| --- | --- |
| Virtual network with two subnets | One subnet for the Container Apps environment, one delegated to PostgreSQL. |
| Container Apps environment (workload profiles, **Consumption profile only**) | Hosts the `api` container app and the `migrations` job. No dedicated profile, so there is no fixed environment charge. |
| Container app `api` | Runs the `api` image, with HTTPS ingress, a custom domain and a free managed certificate. |
| Container Apps job `migrations` | Manually triggered job that runs the `migrations` image to completion ([section 4](#4-deployment-pipeline)). |
| Azure Database for PostgreSQL — Flexible Server | `Burstable B1ms` with **private access (VNet integration)**, Entra authentication only, and password authentication disabled ([ADR-0007](0007-primary-data-store-and-data-access.md)). |
| User-assigned managed identities | `id-api` (runtime: DML-only database role, Blob, Key Vault, e-mail) and `id-migrator` (DDL database role for migrations only). |
| Key Vault (RBAC mode) | Data Protection key-wrapping key, OIDC client certificate, and any remaining third-party secret. Soft delete and purge protection are enabled. |
| Storage account | A private Blob container for ASP.NET Core Data Protection keys. Later, also music uploads ([ADR-0007](0007-primary-data-store-and-data-access.md)). |
| Log Analytics workspace + workspace-based Application Insights | Logs, traces and metrics ([section 5](#5-observability-baseline)). |
| Azure Communication Services + Email Communication Service | Transactional e-mail from the custom domain ([section 6](#6-e-mail-delivery)). |

- **Shared** resources (`rg-teamgym-shared`): the **Azure DNS** zone for the custom domain, and the **Cost Management budget**.
- **Networking:** Choosing PostgreSQL private access now matters because Flexible Server's network mode is fixed when the server is created and cannot be switched later. The database has **no public endpoint**. Storage and Key Vault keep public endpoints, restricted by firewall to the Container Apps subnet (service endpoints) and Azure Resource Manager. This avoids private endpoint charges while blocking direct internet access to data. Maintainer database access goes through a short-lived Container Apps job or `az containerapp exec`, never through an opened firewall.
- **No container registry of our own:** Container Apps pulls the public images directly from GHCR **by digest** ([ADR-0002](0002-repository-strategy.md)). Azure Container Registry is added only if GHCR availability or pull limits become a problem.
- **No CDN or WAF in the MVP:** Azure Front Door's base fee alone would exceed most of the budget. The app's own ASP.NET Core rate limiter protects `/bff/login` and anonymous public endpoints. Front Door is reconsidered when public competition pages see real traffic or abuse.

### 2. Compute Sizing and Scaling

| Setting | `staging` | `prod` |
| --- | --- | --- |
| CPU / memory per replica | 0.25 vCPU / 0.5 GiB | 0.25 vCPU / 0.5 GiB to start; raised through IaC if load testing or production metrics require it |
| Min / max replicas | **0** / 1 (scale-to-zero; a cold start of a few seconds is acceptable) | **1** / 5 (no cold start for the first sign-in of the day) |
| Scale rule | HTTP concurrency | HTTP concurrency (e.g. 50 concurrent requests per replica) |
| Revision mode | Single | Single |

- **Why `prod` keeps one replica:** Container Apps bills an idle replica at a reduced rate, so one small always-on replica costs only a few euros a month after the free monthly grant. In return, users never wait for a cold start, and it keeps a warm process ready for SignalR later.
- **Competition days:** Organisers know the dates in advance. Before a large event (for example a national championship), the maintainer raises `prod` min replicas through an IaC pull request, and lowers them afterwards. Automating this from the competition calendar is a later improvement.
- **Statelessness:** Any replica can serve any request. The session cookie is decrypted with shared Data Protection keys ([ADR-0008](0008-identity-authentication-and-authorisation.md)), so no session affinity is needed.
- **SignalR later:** When the `Runner` and `Scoring` modules add SignalR hubs ([ADR-0004](0004-backend-runtime-and-architectural-style.md)) and `prod` runs more than one replica, a backplane is required. The choice between Azure SignalR Service and session affinity is made in the Runner ADR.

### 3. Environments

- **`staging`** and **`prod`** only. Both are defined by the **same Bicep templates** with different parameter files, so staging is a faithful rehearsal of production.
- **Separate Entra external tenants per environment** (both free at our scale). Test accounts and staging app registrations never exist in the production tenant. Staging runs against a real external tenant, which closes the Keycloak-vs-Entra gap noted in [ADR-0008](0008-identity-authentication-and-authorisation.md).
- **Staging data is synthetic.** No production data, and no real gymnast data, is ever copied into staging.
- **Staging database cost control:** The staging Flexible Server is stopped on a schedule outside working sessions by a scheduled workflow in the private repository. It must be restarted at least every 7 days, because Azure starts a stopped server automatically after 7 days.
- **No per-PR preview environments.** Pull requests in this public repository cannot deploy anything anyway, because the repository holds no credentials ([ADR-0002](0002-repository-strategy.md)). `.NET Aspire` locally and Testcontainers in CI cover pre-merge verification. Revisit if there is more than one active contributor and reviewing UI changes without running them locally becomes a bottleneck.
- **Local development** remains `dotnet run --project src/AppHost`, with no cloud account ([ADR-0004](0004-backend-runtime-and-architectural-style.md)).

### 4. Deployment Pipeline

Deployments run only from the **private** infrastructure repository, as decided in [ADR-0002](0002-repository-strategy.md):

```text
public repo                              private infrastructure repo
───────────                              ───────────────────────────
release vX.Y.Z
 └─ GHCR: api@sha256:…, migrations@sha256:…, build provenance attestations
                                         Renovate PR: bump digests in staging parameters
                                          └─ PR checks: bicep lint, what-if, `gh attestation verify`
                                         merge → workflow (GitHub environment `staging`)
                                          1. azure/login with OIDC federated credential
                                          2. deploy Bicep deployment stack
                                          3. start `migrations` job with new digest, wait for success
                                          4. update `api` to new digest; new revision goes live after its readiness probe passes
                                          5. smoke test: GET /health = 200, GET /bff/user = 401
                                         PR: promote the same digests to prod parameters
                                          └─ same workflow, GitHub environment `prod` (required reviewer)
```

- **No long-lived secrets:** Each GitHub environment (`staging`, `prod`) logs in with its own **user-assigned managed identity** that has a **federated credential** bound to the subject `repo:<owner>/<infra-repo>:environment:<env>`. Each identity has rights only on its own resource groups. No client secrets or publish profiles exist.
- **Only verified images are deployed:** The PR check verifies the GHCR build provenance attestation of each digest before merging.
- **Migrations first:** The `migrations` job runs with `id-migrator` and must finish successfully before the `api` revision changes. If it fails, the deployment stops and the previous revision keeps serving traffic. Because migrations follow **expand-and-contract** ([ADR-0007](0007-primary-data-store-and-data-access.md)), the old revision keeps working against the new schema.
- **Rollback** means reverting the digest pull request. This rolls back the application only. The database is never rolled back automatically; if needed, a forward-fix migration is written instead.
- **Infrastructure changes** follow the same path: pull request with `what-if` output, merge, deploy to `staging`, then promote to `prod`.

### 5. Observability Baseline

- **Signals:** The application emits **OpenTelemetry** logs, traces and metrics through `ServiceDefaults` ([ADR-0004](0004-backend-runtime-and-architectural-style.md)), plus structured JSON logs to stdout.
- **Export:** `ServiceDefaults` follows the standard `.NET Aspire` pattern:
  - If `OTEL_EXPORTER_OTLP_ENDPOINT` is set, it exports through **OTLP**. This is used by the Aspire dashboard locally and by self-hosters (Grafana, Jaeger, SigNoz, …).
  - If `APPLICATIONINSIGHTS_CONNECTION_STRING` is set, it enables the **Azure Monitor OpenTelemetry distro** and sends all three signals to **Application Insights**.

  The Azure Monitor package is referenced **only in `ServiceDefaults`** and never by modules, so the application stays portable. We use it rather than the Container Apps managed OpenTelemetry agent because the managed agent does not forward metrics to Application Insights.
- **Cost guardrails:** Log Analytics uses a **daily ingestion cap**, **30-day retention**, adaptive trace **sampling** in production, and health probe requests excluded from tracing. Ingestion should stay within the free monthly allowance.
- **Alerts** (to the maintainer by e-mail): an availability test on `/health`, the `api` failing to start or restarting repeatedly, an HTTP 5xx rate threshold, PostgreSQL CPU or storage above 80 %, a failed `migrations` job, and the budget thresholds below.
- **Personal data in telemetry:** Logs and traces must not contain gymnast personal data, e-mail addresses or tokens. Only identifiers (`UserId`, `OrganisationId`, …) may be logged ([ADR-0007](0007-primary-data-store-and-data-access.md), [ADR-0008](0008-identity-authentication-and-authorisation.md)).

### 6. E-mail Delivery

This resolves the provider choice deferred in [ADR-0008](0008-identity-authentication-and-authorisation.md):

- **Production:** **Azure Communication Services Email** with the **Europe** data location, sending from a verified subdomain of the custom domain (for example `noreply@mail.<domain>`) with SPF, DKIM and DMARC records managed in Azure DNS. The `api` authenticates with its managed identity, so there is no API key.
- **Application side:** Modules depend on an `IEmailSender` port. Two adapters live in the infrastructure layer of the host, selected by `Email__Provider`:
  - `Smtp`: used for **Mailpit** in `.NET Aspire` locally and by self-hosters with any SMTP relay.
  - `AzureCommunicationServices`: used in Azure.
- **Volume and cost:** Invitations and notifications only. This costs cents per month at MVP scale.

### 7. Public Deployment Contract

This is the part of the decision that belongs to the **public** repository. Any operator, whether our private infrastructure repository, a self-hosting club, or a contributor running Docker Compose, can deploy the application by satisfying this contract and nothing more. The authoritative list of settings is kept in **`docs/deployment.md`**, which is updated in the same pull request as any code that adds a setting.

**Images** (published by the release workflow, [ADR-0002](0002-repository-strategy.md)):

| Image | Role |
| --- | --- |
| `ghcr.io/gyback/teamgym-arena/api:<version>` | The long-running web process: API, BFF and SPA ([ADR-0005](0005-frontend-framework-and-api-contract.md)). |
| `ghcr.io/gyback/teamgym-arena/migrations:<version>` | The `src/MigrationService` worker ([ADR-0007](0007-primary-data-store-and-data-access.md)). It applies all module migrations, then exits with code `0` on success and non-zero on failure. Run it to completion before starting a new `api` version. |

- Each image is tagged with the SemVer version and the commit SHA, built for `linux/amd64` and `linux/arm64`, and accompanied by a build provenance attestation. **Operators should deploy by digest.**
- The images are based on Microsoft's **chiselled** `.NET 10` runtime images. They run as the **non-root** `app` user and contain no shell.

**Runtime behaviour of `api`:**

| Contract | Detail |
| --- | --- |
| **Port** | Listens on plain HTTP on port **8080** (`ASPNETCORE_HTTP_PORTS`). TLS is terminated by the platform's ingress or reverse proxy. |
| **Reverse proxy** | Honours `X-Forwarded-Proto` / `X-Forwarded-For` when `ASPNETCORE_FORWARDEDHEADERS_ENABLED=true`, so that `Secure` cookies and OIDC redirect URIs use `https`. Only set this when the container is reachable through the proxy alone. |
| **Health endpoints** | `GET /alive`: liveness, the process is running. `GET /health`: readiness, including database connectivity. Both are anonymous (on the [ADR-0008](0008-identity-authentication-and-authorisation.md) allow-list), are mapped in **every** environment (not only Development, unlike the Aspire template default), and return only `Healthy` / `Unhealthy` with status `200` / `503`, with no details. |
| **Statelessness** | No local state. Compatible with a read-only root filesystem, with writes only to `/tmp`. Any number of replicas can run behind a load balancer once the shared Data Protection key ring is configured. |
| **Shutdown** | Handles `SIGTERM` and finishes in-flight requests within the platform's grace period (30 s on Container Apps). |
| **Logs** | Structured JSON to stdout. OpenTelemetry as described in section 5. |
| **Database migrations** | The `api` never migrates the schema ([ADR-0007](0007-primary-data-store-and-data-access.md)). It runs with a DML-only database role. |

**Configuration** comes only from **environment variables** (or files mounted by the platform), using ASP.NET Core's `__` separator. Indicative groups:

| Group | Example keys | Azure | Self-hosted alternative |
| --- | --- | --- | --- |
| Hosting | `ASPNETCORE_ENVIRONMENT`, `App__PublicBaseUrl` | `Production` | Same |
| Database | `ConnectionStrings__teamgym` | No password: Entra token via managed identity (`AZURE_CLIENT_ID`) | Password in the connection string |
| Identity | `Authentication__Oidc__Authority`, `__ClientId`, `__SubjectClaim`, client credential settings | Entra external tenant, certificate from Key Vault ([ADR-0008](0008-identity-authentication-and-authorisation.md)) | Any OIDC provider, for example Keycloak |
| Data Protection | `DataProtection__BlobUri`, `DataProtection__KeyVaultKeyUri` | Blob Storage + Key Vault key | A persistent volume (`DataProtection__KeyPath`) |
| E-mail | `Email__Provider`, `Email__From`, provider-specific keys | `AzureCommunicationServices` | `Smtp` |
| Telemetry | `OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES` (`deployment.environment=<env>`), `OTEL_EXPORTER_OTLP_ENDPOINT` or `APPLICATIONINSIGHTS_CONNECTION_STRING` | Application Insights | Any OTLP backend |

- **Every Azure-specific setting has a non-Azure alternative**, so the images run on any container host.
- **Secrets are never baked into images.** In Azure, almost no secrets exist, because managed identities replace connection passwords and API keys. Any remaining secret is a Container Apps secret that references Key Vault.
- **Application code reaches Azure only through standard abstractions**: `DefaultAzureCredential` in infrastructure registrations in the host, never in module domain or application code. The [ADR-0003](0003-engineering-workflow-and-quality-gates.md) architecture tests enforce this.

### 8. Answers to Open Questions

1. **Monthly budget ceiling?**
   - **€50 per month** for all environments together, excluding the domain registration.
   - An Azure **Cost Management budget** in IaC alerts the maintainer at **80 % actual** and **100 % forecast** spend.
   - Indicative monthly cost at list prices, at the time of writing:

   | Item | Estimate |
   | --- | --- |
   | Container Apps `prod` (one mostly idle 0.25 vCPU replica, after the free monthly grant) | €3–8 |
   | Container Apps `staging` (scale-to-zero) | ~€0 |
   | PostgreSQL `prod` (`B1ms`, 32 GiB, 7-day backups) | €16–20 |
   | PostgreSQL `staging` (`B1ms`, stopped outside test sessions) | €4–7 |
   | Log Analytics / Application Insights (with daily cap) | €0–5 |
   | Key Vault, Storage, DNS zones, Communication Services e-mail | €2–4 |
   | Entra External ID (below 50,000 monthly active users) | €0 |
   | **Total** | **≈ €25–45** |

   - If spending approaches the ceiling, the first levers are: stop the staging database more aggressively, lower the log cap, and allow `prod` to scale to zero outside competition periods.
2. **Custom domain, and who owns it?**
   - The maintainer **registers and owns** the domain personally, and pays the registration outside Azure. Ownership is transferred to a project organisation or association if one is formed.
   - The domain's **DNS is delegated to an Azure DNS zone** managed by the private IaC repository. This covers the `api` custom domain record and validation, the e-mail SPF/DKIM/DMARC records, and the IdP custom URL domain if one is added later.
   - **The domain name is not recorded in this public repository** ([ADR-0002](0002-repository-strategy.md)). The application learns it only through `App__PublicBaseUrl`.
   - `staging` uses a subdomain of the same domain.

### Consequences

- Good, because the whole hosted service costs an estimated €25–45 per month, within the €50 ceiling, with a budget alert as a guardrail.
- Good, because Container Apps, PostgreSQL Flexible Server, Key Vault and Communication Services are all managed services. There are no VMs, clusters or certificates to patch or renew by hand.
- Good, because every resource and all personal data stay in Sweden Central or the EU, which supports the GDPR commitments in [ADR-0007](0007-primary-data-store-and-data-access.md) and [ADR-0008](0008-identity-authentication-and-authorisation.md).
- Good, because GitHub Actions and the running application use federated credentials and managed identities, so there are no deploy keys, client secrets or database passwords to rotate or leak.
- Good, because the deployment contract keeps the images portable. Self-hosters can run them with Docker Compose, Kubernetes or another cloud with only environment variables.
- Good, because staging and production come from the same templates, and every deployment rehearses migrations-then-rollout in staging first.
- Bad, because `prod` runs a single small replica in a single region. A regional outage or a slow replica means downtime. This is accepted for the MVP; backups and point-in-time restore ([ADR-0007](0007-primary-data-store-and-data-access.md)) allow recovery into another EU region.
- Bad, because Bicep ties the IaC to Azure. This is accepted because the application itself stays portable, and AVM modules exist for Terraform too if a migration is ever needed.
- Bad, because PostgreSQL private access makes ad-hoc database access less convenient (through a job or `exec` instead of a direct connection). This is accepted for the stronger security posture.
- Bad, because pulling images from GHCR adds a dependency on GitHub availability at deploy and scale-out time. Container Apps keeps running existing replicas during a GHCR outage.

### Confirmation

- **Public repository:**
  - A container smoke test in CI starts the built `api` image with only environment variables against a PostgreSQL Testcontainer, after running the `migrations` image. It asserts that `/alive` and `/health` return `200`, that the image runs as non-root on port `8080`, and that `/api/*` without a session returns `401`.
  - The release workflow publishes both images with provenance attestations ([ADR-0002](0002-repository-strategy.md)).
  - Architecture tests assert that no module project references Azure SDK packages. Only the host and `ServiceDefaults` may.
  - Code review checks that `docs/deployment.md` is updated whenever a configuration key is added or changed.
- **Private repository:**
  - Every infrastructure PR runs `bicep lint` and `what-if` and verifies image attestations.
  - Deployment stacks with deny settings prevent manual drift in `prod`.
  - Each deployment ends with the smoke test in section 4.
  - IaC-level details (module structure, naming, parameter files, stack settings) are recorded as ADRs in the private repository.

## Pros and Cons of the Options

### Compute Platform

#### Option C1: Azure Container Apps (Consumption)

- Good, because it runs standard OCI images with per-second billing, a free monthly grant, scale-to-zero and HTTP-based autoscaling. This suits low baseline traffic with competition-day peaks.
- Good, because Container Apps **jobs** fit the "run migrations to completion before rollout" step ([ADR-0007](0007-primary-data-store-and-data-access.md)) without extra infrastructure.
- Good, because it includes managed ingress, free managed certificates for custom domains, revisions, managed identities, Key Vault secret references and VNet integration with no fixed charge on the Consumption profile.
- Good, because the `.NET Aspire` tooling targets it directly. The deployment model is still plain containers, so Aspire is not required in production.
- Neutral, because it abstracts Kubernetes. Advanced networking or custom sidecars are more limited than on AKS, but none are needed.
- Bad, because scale-to-zero means cold starts. We avoid this in `prod` by keeping one replica.

#### Option C2: Azure App Service (+ Static Web Apps)

- Good, because it is mature and well documented, with deployment slots and built-in authentication.
- Neutral, because Static Web Apps adds nothing, since the SPA is served by the API container ([ADR-0005](0005-frontend-framework-and-api-contract.md)).
- Bad, because there is no scale-to-zero. A Basic plan is paid around the clock per environment, and deployment slots and VNet integration for zero-downtime swaps need a more expensive Standard or Premium plan. Two environments would use most of the budget.
- Bad, because there is no native run-to-completion job for migrations. We would need a separate container instance or a WebJob workaround.

#### Option C3: Azure Kubernetes Service

- Good, because it is the most flexible and portable orchestration platform.
- Bad, because it needs always-on node VMs, cluster upgrades, ingress controllers and certificate management. That is far more cost and operational work than one container justifies for a solo maintainer.

### Infrastructure as Code Tool

#### Option T1: Bicep with Azure Verified Modules

- Good, because it is Azure's native IaC language, with day-one support for new resource types and API versions, and no state file to host, lock or protect.
- Good, because **Azure Verified Modules** provide Microsoft-maintained, secure-by-default modules for Container Apps, PostgreSQL, Key Vault and more.
- Good, because `what-if` previews changes in PRs, and **deployment stacks** add managed cleanup and deny settings against drift.
- Good, because the Microsoft Graph Bicep extension can declare the Entra app registrations and federated credentials next to the Azure resources.
- Bad, because it is Azure-only, which is accepted since the hosting target is Azure and portability is guaranteed at the container level.
- Bad, because some Entra External ID settings (tenant creation, user flows, branding) cannot be expressed in Bicep or Graph yet. These are recorded as scripted or documented steps in the private repository.

#### Option T2: Terraform / OpenTofu

- Good, because it is the most widely used multi-cloud IaC tool, with a large ecosystem and strong agent familiarity, and the `azuread` provider covers Entra objects well.
- Bad, because remote state must be hosted, locked and secured (a storage account with its own access), which is extra moving parts for a solo maintainer.
- Bad, because its multi-cloud strength brings no benefit to an Azure-only deployment, and new Azure features sometimes reach the `azurerm` provider after Bicep.

#### Option T3: Pulumi (C#)

- Good, because the infrastructure would be written in the same language as the application.
- Bad, because it needs a state backend (Pulumi Cloud or self-managed), has the smallest community of the three, and general-purpose code in IaC invites more complexity than declarative templates.

## More Information

- Related issue: [#8](https://github.com/gyback/teamgym-arena/issues/8)
- Parent epic: [#9](https://github.com/gyback/teamgym-arena/issues/9)
- Related ADRs:
  - [ADR-0002 Repository strategy — public app monorepo, private IaC repo and artefact hand-off](0002-repository-strategy.md)
  - [ADR-0003 Engineering workflow and quality gates for agent-assisted development](0003-engineering-workflow-and-quality-gates.md)
  - [ADR-0004 Backend runtime and architectural style — .NET 10 modular monolith with vertical slices](0004-backend-runtime-and-architectural-style.md)
  - [ADR-0005 Frontend framework and API contract — React 19 SPA with Vite, ASP.NET Core BFF and OpenAPI TypeScript client](0005-frontend-framework-and-api-contract.md)
  - [ADR-0007 Primary data store and data access](0007-primary-data-store-and-data-access.md)
  - [ADR-0008 Identity, authentication and authorisation](0008-identity-authentication-and-authorisation.md)
- Follow-up implementation issues:
  - [#37 Implement the container deployment contract (health endpoints, forwarded headers, Data Protection, managed identity, OpenTelemetry export, `docs/deployment.md`)](https://github.com/gyback/teamgym-arena/issues/37)
  - [#38 Implement `IEmailSender` with SMTP (Mailpit) and Azure Communication Services adapters](https://github.com/gyback/teamgym-arena/issues/38)
  - [#39 Add Docker Compose self-hosting example and `docs/self-hosting.md`](https://github.com/gyback/teamgym-arena/issues/39)
  - [#12 Add release workflow to publish versioned container images to GHCR](https://github.com/gyback/teamgym-arena/issues/12) (extended with the `migrations` image, multi-arch builds and attestations)
- Revisit this decision when:
  - monthly spend regularly exceeds the €50 ceiling,
  - the `Runner` / `Scoring` modules need multiple `prod` replicas with SignalR (backplane choice),
  - public pages need a CDN or WAF (Azure Front Door),
  - more than one contributor works on the codebase and per-PR preview environments become valuable, or
  - an availability target beyond single-region, single-replica hosting is required.
