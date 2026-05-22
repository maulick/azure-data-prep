# JD Crash Course — The Gaps Beyond Session 1

> Read tonight to fill the gaps between what you built and what the JD asks. Prioritized — SQL DB first (it's mandatory), then everything else.

## Your strengths to lead with

Before the gaps, remind yourself of your edges:

- **DevOps background** — Azure DevOps Pipelines, YAML, IaC (Bicep / Terraform / ARM)
- **ADF hands-on** — what you built this session, the JSON in your repo
- **CI/CD patterns** — branching strategies, environment promotion, secrets via Key Vault

When you're asked about a gap area, your move is: **"I haven't done X hands-on, but I understand the deployment and CI/CD patterns because I've done similar shape of work in Y."** Don't apologize. Reframe to your strength.

---

## Reading order for tonight

1. **SQL DB crash course** (deepest read — 30 min)
2. **Databricks** (15 min)
3. **Key Vault + PIM + IAM** (15 min)
4. **Networking + App Service** (15 min)
5. **Front Door** (10 min)
6. **Power BI** (10 min)
7. **The "how to handle a gap" framing section** (5 min — most important)
8. Re-read `notes.md` cheat sheet section

Total: ~90 minutes focused, then sleep.

---

# 1. Azure SQL Database (MANDATORY — go deep)

## What it is

**Azure SQL Database** is Microsoft's managed SQL Server in the cloud. PaaS — you don't manage the OS, patching, or backups; Azure does.

**Three deployment models:**
- **Single Database** — most common. One DB per server. Predictable compute and storage.
- **Elastic Pool** — multiple DBs sharing a compute pool. Cost-effective when DB load is uneven (one spikes while others idle).
- **Managed Instance** — closest to traditional SQL Server. Supports SQL Agent, cross-DB queries, CLR. Use when migrating on-prem SQL workloads with minimal changes.

## Service tiers (memorize)

**DTU-based** (older, simpler):
- **Basic** — tiny, dev/test (5 DTU)
- **Standard** (S0–S12) — most common general workloads
- **Premium** (P1–P15) — high-IOPS / high-throughput

**vCore-based** (newer, more flexible — this is what new projects use):
- **General Purpose** — balanced, most workloads
- **Business Critical** — local SSD, very low latency, readable replicas
- **Hyperscale** — for very large DBs (up to 100 TB), with fast restores and rapid scale

**Serverless** (within vCore General Purpose): auto-pause when idle, billed per second. Great for dev/test or sporadic workloads.

## Authentication

- **SQL authentication**: traditional username/password. Easy but less secure.
- **Azure AD authentication**: identity is your Entra user, group, service principal, or managed identity. Preferred for new work — no passwords to rotate, full audit trail, MFA-capable.

Production default: **Azure AD authentication with managed identity** for app-to-DB connections, **AAD admin group** (not individual users) for human admin access.

## Firewall rules (the gotcha that catches everyone)

Two layers:
1. **Server-level firewall** — IP ranges that can reach the SQL Server. Set in the Azure portal under "Networking" on the server (not the database).
2. **VNet rules** — if you want traffic from a specific subnet only.
3. **Database-level firewall** (rare) — overrides server-level for a specific DB.

**Allow Azure services**: a checkbox that lets all Azure IPs through. Convenient for dev, dangerous for prod (any Azure customer's VM could try to connect). For prod, use **Private Endpoint** instead.

## CI/CD for SQL DB — the two main patterns

This is the JD's mandatory item. Know both.

### Pattern A — DACPAC (state-based)

A **DACPAC** is a "Data-tier Application Package" — a compiled snapshot of your database schema. The build process compares your schema project (`.sqlproj`) against the target DB and generates the diff to apply.

**Workflow:**
1. Developer edits schema in a SQL Database Project in VS Code or Visual Studio (tables, views, stored procs, etc.)
2. Project is committed to Git like normal code
3. CI pipeline runs `dotnet build` on the `.sqlproj` → produces `.dacpac` artifact
4. CD pipeline deploys the `.dacpac` to target environment using the Azure DevOps task `SqlAzureDacpacDeployment@1`

**The task takes:**
- Server name + DB name
- Auth method (SQL auth credentials, AAD, or service connection)
- Path to the DACPAC
- Publish profile (XML file with environment-specific settings — connection strings, disable destructive changes per env, etc.)

**Pros:**
- Pure declarative — your schema project IS the desired state
- IDE support, IntelliSense, build-time validation of schema
- Microsoft-native, well-supported

**Cons:**
- Refactoring is hard (renames look like drop + create unless you use the Refactor menu)
- Data migrations need separate scripts
- Drift detection only on deploy

### Pattern B — Migration-based (Flyway, DbUp, EF Migrations, Liquibase)

A series of **forward-only migration scripts** (`V001__create_orders.sql`, `V002__add_index.sql`, ...). The tool tracks which migrations have been applied to which database in a metadata table.

**Workflow:**
1. Developer writes a new numbered migration script
2. Commits to Git
3. CI builds (sometimes nothing to build — just package the scripts)
4. CD runs the migration tool against target DB → it applies any new migrations in order

**Pros:**
- Imperative — each change is its own auditable script
- Data migrations live alongside schema migrations
- Simpler mental model for developers

**Cons:**
- No declarative "what should this DB look like?" — only "what changes have I made?"
- Drift between environments is harder to spot
- You can't "edit" old migrations once applied

### When to pick which

- **DACPAC** for greenfield projects with one team, clean schema, mostly schema changes
- **Migration-based** for multi-team work, complex data migrations, existing DBs that were created outside the framework
- Real shops sometimes use **both** — DACPAC for schema, separate scripts for data ops

## Interview answers for SQL DB

### Q: "How would you do CI/CD for an Azure SQL Database?"

> Two main patterns. State-based with DACPAC: developers maintain a SQL Database Project that compiles to a `.dacpac` artifact. CI builds it, CD deploys via the Azure DevOps SqlAzureDacpacDeployment task — it diffs the DACPAC against the target DB and applies the changes. The deploy can be gated with publish profiles per environment. Migration-based with Flyway or DbUp: forward-only numbered SQL scripts in Git, the tool tracks what's been applied in a metadata table. I'd default to DACPAC for greenfield work where schema is the primary concern, and migrations for projects with complex data ops or multi-team contention. Both deploy through Azure DevOps pipelines with environment approvals between stages.

### Q: "How do you handle secrets for SQL connection strings in a pipeline?"

> Connection strings go in Key Vault, referenced by an Azure DevOps variable group linked to the vault. The pipeline service principal needs Key Vault Secrets User on the vault. For the SQL DB connection itself, in production I'd prefer Azure AD authentication with the app's managed identity rather than SQL auth — no password to manage at all. The DACPAC deployment task supports AAD service principal auth via the service connection.

### Q: "How would you handle a schema change that drops a column with live data?"

> Carefully and in two deploys. First deploy: add a new column or table that consumers can migrate to, and update the application code to write to both old and new. Wait for the migration to complete (could be days). Second deploy: drop the old column. DACPAC needs the `DropObjectsNotInSource` setting enabled deliberately, and you'd typically set `BlockOnPossibleDataLoss=false` only for the second deploy. Migration-based tools force you to write the drop as an explicit migration script.

### Q: "What's the difference between DTU and vCore?"

> DTU is a bundled, opaque unit of compute + storage + IO — simpler but less flexible. vCore exposes compute and storage as separate axes, lets you pick General Purpose vs Business Critical service tiers, and supports advanced features like Hyperscale and serverless. For new work I'd default to vCore — DTU is mostly legacy.

### Q: "How do you secure a SQL DB?"

> Layers. Network: VNet integration via Private Endpoint, no public endpoint at all. Authentication: AAD admin group, managed identities for app connections, no SQL auth in production. Authorization: granular database-level roles, Always Encrypted for the most sensitive columns. Audit: enable SQL auditing to a Storage Account or Log Analytics, threat detection via Microsoft Defender for SQL. Backups: PITR for 35 days, geo-redundant backup for DR. TDE is on by default.

---

# 2. Azure Databricks (infra deployment)

## What Databricks is

A managed Apache Spark platform with notebooks, clusters, jobs, and Unity Catalog (governance). The "infra" piece in the JD means the workspace setup, not the notebook development.

## Workspace creation patterns

A Databricks workspace is created via:
- **Portal/CLI** (one-off)
- **Bicep / ARM / Terraform** (IaC — what the JD wants)

In Bicep:
```bicep
resource workspace 'Microsoft.Databricks/workspaces@2023-02-01' = {
  name: 'dbx-mycompany-prod'
  location: location
  sku: { name: 'standard' }  // or 'premium' for Unity Catalog, RBAC features
  properties: {
    managedResourceGroupId: managedRgId
    parameters: {
      enableNoPublicIp: { value: true }  // SCC — secure cluster connectivity
      customVirtualNetworkId: { value: vnetId }  // VNet injection
      customPublicSubnetName: { value: 'snet-dbx-public' }
      customPrivateSubnetName: { value: 'snet-dbx-private' }
    }
  }
}
```

**Key concepts to know:**
- **Standard vs Premium SKU** — Premium is required for RBAC on notebooks, Unity Catalog, table ACLs
- **VNet injection** — deploys the workspace into your VNet (two subnets: public for Databricks plane, private for clusters)
- **Managed resource group** — Databricks creates its own RG for cluster VMs and disks; you can't directly manage these resources
- **Secure cluster connectivity (SCC)** — no public IP on cluster nodes; outbound through NAT

## CI/CD for Databricks

Two layers, both deploy via Azure DevOps:

1. **Infrastructure** (workspace, networking, Key Vault-backed secret scopes) → Bicep/Terraform
2. **Code** (notebooks, jobs, libraries) → Databricks Asset Bundles (modern), or Databricks CLI workflows

**Databricks Asset Bundles** (DAB) is the current recommended pattern:
- YAML files describe your jobs, notebooks, clusters, etc.
- `databricks bundle deploy` deploys them to a target workspace
- Runs from Azure DevOps Pipelines with a Databricks PAT (stored in Key Vault) or AAD token

## Interview answers

### Q: "How would you deploy Databricks workspaces across environments?"

> Workspace itself goes through Bicep with parameters per environment — VNet IDs, SKU, public IP toggle. CI on commit, CD with manual approvals dev → test → prod. For workspace contents — notebooks, jobs, cluster definitions — I'd use Databricks Asset Bundles, which is the YAML-based deploy pattern Databricks recommends now. PAT or service principal auth stored in Key Vault, referenced from the pipeline.

### Q: "What's a managed resource group in Databricks?"

> When you create a Databricks workspace, Databricks provisions a separate Azure resource group it owns — that's where cluster VMs, disks, NICs live. You name the managed RG when you create the workspace but you can't directly manage its contents — only Databricks can. Comes up in interviews because people get confused why "their" resource group has all this stuff in it.

---

# 3. Azure AD / IAM / PIM / Key Vault

## Azure AD = now called "Entra ID"

Same product, rebranded. If you say "Azure AD" you sound dated; if you say "Entra ID" you sound current. Use both interchangeably.

## RBAC vs Access Policies (Key Vault specifically)

**RBAC mode** (current best practice):
- Permissions managed via Azure RBAC roles
- Roles: `Key Vault Administrator`, `Key Vault Secrets Officer`, `Key Vault Secrets User`, etc.
- Centralized in Azure RBAC, supports PIM, custom roles
- Recommended for all new vaults

**Access Policies** (legacy):
- Vault-local permission model
- Limited to predefined combinations
- Microsoft recommends migrating to RBAC

Enable RBAC mode at vault creation: `--enable-rbac-authorization true` in CLI, or `enableRbacAuthorization: true` in Bicep.

## Key Vault concepts

- **Secrets**: arbitrary strings (passwords, connection strings, API keys)
- **Keys**: cryptographic keys for encryption operations (HSM-backed or software)
- **Certificates**: X.509 certs, auto-renewable from CAs

**Soft delete**: mandatory for new vaults. Deleted items recoverable for N days (7-90).
**Purge protection**: optional. Prevents anyone (even admins) from hard-deleting before retention expires. Required in production; not always available in sandboxes.

## PIM (Privileged Identity Management)

Just-in-time, time-bound role activation. Lives within Entra ID P2 license tier.

**Concept:**
- Instead of giving someone permanent Owner on the prod RG, you make them **eligible** for Owner.
- When they need it, they **activate** the role for a defined time window (e.g., 4 hours).
- Activation can require MFA, business justification, and/or approval workflow.

**Why this matters:** standing privileged access is the #1 source of breaches. PIM turns "always admin" into "admin only when needed, with audit trail."

## Managed Identity

Two types:
- **System-assigned**: lifecycle tied to the parent resource. Auto-cleaned when the resource is deleted.
- **User-assigned**: standalone identity, can be attached to multiple resources, lives independently.

Use **system-assigned** when one identity-per-resource is fine (most cases). Use **user-assigned** when you want multiple resources to share an identity, or when you want the identity to outlive the resource.

## Interview answers

### Q: "How does PIM work and why use it?"

> PIM converts standing permissions into just-in-time activations. Instead of having permanent Owner on a subscription, I'd be configured as eligible for Owner. When I need to do work that requires it, I activate the role for a time window — say 4 hours — and it can require MFA, business justification, or approval from a peer. After the window, the role drops automatically. The benefit: standing privileged access is the biggest attack surface in cloud, and PIM shrinks that surface dramatically while keeping an audit trail of every activation.

### Q: "What's the difference between system-assigned and user-assigned managed identity?"

> System-assigned identity lifecycle is tied to the parent resource — created with it, deleted with it, one per resource. User-assigned identity is a standalone resource you create separately and attach to one or more parent resources. Default to system-assigned for simplicity; use user-assigned when multiple resources need to share an identity, or when you want the identity to persist across resource recreations.

### Q: "Why RBAC mode on Key Vault instead of access policies?"

> Access policies are a vault-local permission model with a limited set of predefined combinations. RBAC mode integrates with Azure RBAC, which means I can use PIM for just-in-time elevation, define custom roles, and centralize all permission management. Microsoft recommends RBAC for all new vaults. The only reason to use access policies anymore is migration of existing vaults that haven't been switched over.

---

# 4. Azure Networking + App Service + App Insights + Service Endpoint

## Service Endpoint vs Private Endpoint (the high-frequency interview question)

| | Service Endpoint | Private Endpoint |
|---|---|---|
| **What it does** | Extends your VNet identity to a PaaS service | Gives the PaaS service a private IP in your VNet |
| **Public endpoint on the PaaS service?** | Still exists, just filtered by source subnet | Replaced by private IP |
| **DNS configuration** | None needed | Required (private DNS zones) |
| **Cost** | Free | ~$0.01/hour per endpoint |
| **Granularity** | Service-level (Storage, SQL, etc.) | Resource-level (this specific storage account) |
| **Use case** | Quick wins for VNet-to-PaaS hardening | Production where no public endpoint should exist |

**Default for new architectures: Private Endpoint.** Service Endpoint was the older model; Private Endpoint is the current best practice.

## App Service with VNet integration

App Service is PaaS for web apps. By default, it runs on Microsoft's shared infrastructure and talks to your back-end resources (DBs, storage) over the public internet.

**Regional VNet integration** lets the App Service make outbound calls through a delegated subnet in your VNet. Combined with Service Endpoint or Private Endpoint on the target resource, traffic stays private.

```
App Service → snet-app (delegated, VNet integration) → Private Endpoint → SQL DB
                                                       (in snet-data)
```

## App Insights

Application telemetry — requests, dependencies, exceptions, custom events, traces. Wired to your app via a connection string (`APPLICATIONINSIGHTS_CONNECTION_STRING` env var).

**Modern best practice**: App Insights resource is backed by a Log Analytics workspace (workspace-based App Insights). Older classic mode is being deprecated.

**Sampling**: high-traffic apps drop a percentage of telemetry to control cost. Adaptive sampling is the default.

## Interview answers

### Q: "Service Endpoint vs Private Endpoint?"

> Both keep traffic off the public internet, but they work differently. Service Endpoint extends your VNet's identity to a PaaS service — traffic stays on Microsoft's backbone, but the service still has a public IP and just trusts your VNet's source IP. Private Endpoint goes further — it gives the PaaS service an actual private IP inside your VNet. Traffic never touches a public endpoint. Private Endpoint is more secure but requires DNS work — you need private DNS zones so the FQDN resolves to the private IP. For new architectures I default to Private Endpoint.

### Q: "How does App Service connect privately to a SQL DB?"

> Three pieces. Regional VNet integration on the App Service into a delegated subnet — that gives the App Service an outbound path through my VNet. Private Endpoint on the SQL Server in a separate subnet — gives SQL a private IP. Private DNS zone linked to my VNet so the SQL FQDN resolves to that private IP. Then on the SQL Server I disable public network access entirely. Result: App Service can talk to SQL, nothing on the internet can.

---

# 5. Azure Front Door

## What it is

Azure's global edge load balancer + CDN + WAF (Web Application Firewall). Lives outside any region — your traffic enters via the nearest Microsoft point of presence, then routes to your backend (App Service, Storage, anything HTTPS).

## SKUs

- **Standard** — basic edge + WAF
- **Premium** — adds private link to origins, bot protection, advanced WAF rules

## Key concepts

- **Origins** — your backend services (App Service, Storage Static Sites, custom HTTPS endpoint)
- **Routes** — rules mapping incoming requests to origins (path patterns, etc.)
- **WAF policies** — OWASP rule sets, custom rules (e.g., block specific countries, rate limiting)
- **`X-Azure-FDID` header** — Front Door sends this header with your Front Door's unique ID; backends should validate it to refuse direct traffic that bypasses Front Door

## Production pattern

1. App Service exists at `myapp.azurewebsites.net` (public)
2. Front Door at `myapp.azurefd.net` routes to App Service
3. App Service IP restrictions: only allow traffic from Azure Front Door's service tag AND require the correct `X-Azure-FDID` header
4. WAF policy on Front Door in Prevention mode with OWASP rules
5. Custom domain `www.myapp.com` points to Front Door

## Interview answer

### Q: "When would you use Front Door vs Application Gateway?"

> Front Door is global — it operates at the edge across Microsoft's worldwide POPs, terminates TLS close to your users, and routes to backends anywhere. Application Gateway is regional — it sits in one Azure region. Use Front Door when you have a global user base or multi-region backends. Use Application Gateway when you have a single-region app that needs L7 features like cookie-based session affinity or path-based routing within one region. Many architectures use both — Front Door at the edge for global routing and WAF, Application Gateway inside each region for finer L7 control.

---

# 6. Power BI Integration with Azure

## The basics

Power BI is Microsoft's BI/visualization platform. The "integration with Azure" part of the JD means programmatic/automated workflows around Power BI.

## Key concepts

- **Workspaces** — containers for reports, dashboards, datasets, dataflows
- **Datasets** — the data models (often pointing at Azure SQL, Synapse, Databricks, etc. as sources)
- **Reports** (`.pbix` files) — visualizations on top of datasets
- **Premium vs Pro vs Free** — Premium gives you dedicated capacity, paginated reports, larger datasets, API access without per-user licensing

## Service principal access

For automation, you don't use a human account — you use a **service principal** added to a Power BI workspace as a Member or Admin. Critical setting: in the Power BI admin portal, **"Service principals can use Power BI APIs"** must be enabled.

## REST API patterns

Common operations:
- **Import a `.pbix`** to a workspace
- **Trigger a dataset refresh**
- **Update connection strings** in a dataset (e.g., point dev report at prod data on deploy)
- **Manage workspace permissions** programmatically

PowerShell module: `MicrosoftPowerBIMgmt`. Has cmdlets for everything above.

## CI/CD for Power BI

Two patterns coexist:

**Pattern A — Power BI Deployment Pipelines** (built into Power BI Premium):
- Workspaces in Dev → Test → Prod tiers
- Click "Deploy" in the UI to promote
- Limited automation; UI-centric

**Pattern B — Azure DevOps deployment via REST API**:
- `.pbix` file committed to Git
- Azure DevOps pipeline calls the Power BI REST API to import to target workspace + trigger refresh
- Full control, fits with the rest of your IaC

## Interview answers

### Q: "How would you integrate Power BI with Azure data sources and CI/CD?"

> Power BI reports connect to Azure SQL, Synapse, or Databricks as data sources. For CI/CD, I'd commit `.pbix` files to Git and use an Azure DevOps pipeline to deploy them via the Power BI REST API — import the file to the target workspace, then trigger a dataset refresh. Authentication via a service principal that's a member of the workspace, with the "Service principals can use Power BI APIs" tenant setting enabled. For parameters that differ between environments — connection strings, refresh schedules — I'd template them in the pipeline.

### Q: "What's the difference between Power BI Pro and Premium?"

> Pro is per-user licensing — every consumer of a report needs a Pro license. Premium is capacity-based — you buy dedicated compute and storage, and viewers can be free users. Premium also unlocks larger datasets, paginated reports, advanced AI features, and more aggressive API access. For a public dashboard or a large internal audience, Premium is more cost-effective. For a small team of report developers, Pro is fine.

---

# 7. How to handle a gap gracefully in the interview

The single most important section. Memorize this framing.

## The honest reframe template

When asked about something you haven't done:

> "I haven't done X hands-on, but I understand the pattern because I've done **[similar shape of work]** — [specific example from your DevOps background]. The deployment / CI/CD piece is what I'm strongest at; the data-specific configuration I'd ramp up on quickly because the underlying mechanics are the same."

Example for Power BI:

> "I haven't built a Power BI integration end to end, but the deployment pattern is familiar — service principal in a target workspace, REST API calls from an Azure DevOps pipeline, secrets in Key Vault, parameter overrides per environment. Same shape as how I'd deploy any other Azure resource that has a REST API. I'd ramp up on the Power BI-specific objects (workspaces, datasets, refresh schedules) but the pipeline scaffolding I could write today."

## Don't say

- "I've never used that." (closes the door)
- "I've watched videos about it." (shows you know you're bluffing)
- "Yeah we used it" (when you didn't — verifiable lies are interview death)

## Do say

- "I haven't deployed that specific service, but..."
- "I've done the pattern with X, would expect Y to be similar..."
- "Walk me through what your team uses for that — I want to understand your stack."

## The cardinal rule

**You are interviewing as an Azure DevOps engineer who works with data services, not as a senior data engineer.** Your CI/CD muscles are the job. The data services are what those pipelines deploy. Lead with the CI/CD strength every time.

---

# Final pep talk

You went from "what's HNS?" to "I built a Data Flow with derived columns and aggregates and version-controlled the JSON" in one session. That's real movement. The other JD items are gaps, but they're gaps you can close with the right framing tomorrow.

The interview won't be 7 deep technical drills on every JD bullet. It'll be:
- Conversation about your background
- 1-2 deep dives on things you've actually built (lean into ADF, lean into your DevOps work)
- Some "what would you do for X?" hypotheticals (the crash course above covers these)
- Behavioral questions

You have the ADF story for the deep dive. You have the crash course for the hypotheticals. You have the DevOps background for everything CI/CD-shaped.

Sleep. Walk in tomorrow with the screenshots ready, the GitHub repo open, and the headline answer at the tip of your tongue. The rest is conversation.

You've got this.
