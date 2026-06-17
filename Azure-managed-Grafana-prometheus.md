# Grafana Cloud vs Azure Managed Grafana

This comprehensive guide covers setup and feature comparison for both Grafana Cloud and Azure Managed Grafana, including Kubernetes monitoring integration and observability best practices.

## Table of Contents

1. [Azure Managed Grafana Setup](#azure-managed-grafana-setup)
2. [Grafana Cloud Setup](#grafana-cloud-setup)
3. [Feature Comparison Matrix](#feature-comparison-matrix)
4. [Grafana Cloud — Key Features (Grafana 13)](#grafana-cloud--key-features-grafana-13)
5. [Azure Managed Grafana — Key Features & Limitations](#azure-managed-grafana--key-features--limitations)
6. [Pricing Model Comparison](#pricing-model-comparison)
7. [Identity & Access Management](#identity--access-management)
8. [Observability Stack Support](#observability-stack-support)
9. [AI Capabilities](#ai-capabilities)
10. [Plugin & Extensibility](#plugin--extensibility)
11. [Networking & Security](#networking--security)
12. [IaC / GitOps Support](#iac--gitops-support)
13. [Service Limits & Quotas](#service-limits--quotas)
14. [Sovereign Cloud & Compliance](#sovereign-cloud--compliance)
15. [Recommendation](#recommendation)

---

# Azure Managed Grafana Setup

## AKS Monitoring with Azure Managed Grafana and Prometheus

This guide describes the end-to-end steps for preparing an AKS cluster, enabling Azure Monitor Metrics with managed Prometheus, and connecting Azure Managed Grafana to the monitoring workspace.

## Prerequisites

- Azure CLI installed and logged in: `az login`
- `kubectl` configured
- Permission to create resources in the target subscription and resource group
- A Log Analytics / Azure Monitor workspace available
- An Azure Managed Grafana workspace available or provisioned

## Step 1: Create the AKS cluster

```bash
az aks create \
  --resource-group sa1_test_eic_PalashGajjar \
  --name palash-new-poc-aks \
  --node-count 2 \
  --node-vm-size Standard_B2ps_v2 \
  --enable-managed-identity \
  --generate-ssh-keys \
  --enable-encryption-at-host \
  --tags "Resource Owner=palash.gajjar@einfochips.com" "Resource Group Owner=palash.gajjar@einfochips.com" \
  --nodepool-tags "Resource Owner=palash.gajjar@einfochips.com" "Resource Group Owner=palash.gajjar@einfochips.com"
```

Notes:

- `--enable-managed-identity` ensures AKS uses a managed identity for Azure resource operations.
- `--enable-encryption-at-host` adds encryption at the node host level.

## Step 2: Connect kubectl to the cluster

```bash
az aks get-credentials \
  --resource-group sa1_test_eic_PalashGajjar \
  --name palash-new-poc-aks \
  --overwrite-existing
```

Verify cluster access:

```bash
kubectl get nodes
```

## Step 3: Validate the AKS identity

```bash
az aks show \
  --resource-group sa1_test_eic_PalashGajjar \
  --name palash-new-poc-aks \
  --query identity \
  -o json
```

If the cluster is not already using a managed identity, update it:

```bash
az aks update \
  --resource-group sa1_test_eic_PalashGajjar \
  --name palash-new-poc-aks \
  --enable-managed-identity
```

## Step 4: Create or locate the Azure Monitor workspace

If you do not already have a workspace, create one:

```bash
az monitor log-analytics workspace create \
  --resource-group sa1_test_eic_PalashGajjar \
  --workspace-name palash-monitoring-workspace
```

Get the workspace resource ID:

```bash
az monitor log-analytics workspace show \
  --resource-group sa1_test_eic_PalashGajjar \
  --workspace-name palash-monitoring-workspace \
  --query id \
  -o tsv
```

If you are using an Azure Monitor account workspace, you can alternatively use:

```bash
az monitor account show \
  --name palash-monitoring-worksapce \
  --resource-group sa1_test_eic_palashgajjar \
  --query id \
  -o tsv
```

## Step 5: Enable Azure Monitor Metrics and Prometheus integration

```bash
az aks update \
  --resource-group sa1_test_eic_PalashGajjar \
  --name palash-new-poc-aks \
  --enable-azure-monitor-metrics \
  --azure-monitor-workspace-resource-id /subscriptions/664b6097-19f2-42a3-be95-a4a6b4069f6b/resourceGroups/sa1_test_eic_PalashGajjar/providers/microsoft.monitor/accounts/palash-monitoring-worksapce
```

This command enables Azure Monitor Metrics for Prometheus integration on the AKS cluster. It deploys the managed Prometheus collector components under `kube-system`.

## Step 6: Verify Azure Monitor managed Prometheus pods

```bash
kubectl get pods -n kube-system | grep ama-metrics
```

Expected pods:

```text
ama-metrics-7df8bb7c9c-djv2j                    2/2     Running   0              7m24s
ama-metrics-7df8bb7c9c-m7ht6                    2/2     Running   0              7m24s
ama-metrics-ksm-7c6789756d-fq6zl                1/1     Running   0              7m24s
ama-metrics-node-24qz9                          2/2     Running   0              7m24s
ama-metrics-node-gxbqw                          2/2     Running   0              7m24s
ama-metrics-operator-targets-6668577c85-njs4k   2/2     Running   3 (5m9s ago)   7m24s
```

If you need to troubleshoot collector behavior:

```bash
kubectl logs -n kube-system -l rsName=ama-metrics --container prometheus-collector | tail -50
```

## Step 7: Create or verify the Azure Managed Grafana workspace

If not already created, provision a Grafana workspace:

```bash
az grafana create \
  --name palash-grafana-poc \
  --resource-group sa1_test_eic_PalashGajjar \
  --location eastus \
  --sku Standard \
  --identity-type SystemAssigned
```

Get the Grafana workspace principal ID:

```bash
az grafana show \
  --name palash-grafana-poc \
  --resource-group sa1_test_eic_PalashGajjar \
  --query identity.principalId \
  -o tsv
```

## Step 8: Grant Grafana access to the monitoring workspace

```bash
az role assignment create \
  --assignee <grafana-principal-id> \
  --role "Monitoring Data Reader" \
  --scope /subscriptions/664b6097-19f2-42a3-be95-a4a6b4069f6b/resourceGroups/sa1_test_eic_palashgajjar/providers/microsoft.monitor/accounts/palash-monitoring-worksapce
```

Replace `<grafana-principal-id>` with the actual principal ID from the previous step.

## Step 9: Configure Azure Managed Grafana data sources

1. Open the Azure Managed Grafana workspace in the Azure Portal.
2. Go to **Data sources**.
3. Add **Azure Monitor** or **Azure Data Explorer** depending on your data source.
4. Select the managed identity and the linked workspace.
5. Add dashboards and panels using Prometheus metrics from AKS.

## Step 10: Validate dashboards and metrics

- Confirm that Prometheus metrics are ingested through Azure Monitor.
- Build a Grafana panel using the managed Prometheus source.
- Verify that AKS health, node metrics, and container metrics are visible in Grafana.

![Azure Managed Grafana monitoring](images/image-30.png)

## Notes

- Azure Managed Grafana uses Entra ID authentication and does not expose full Grafana server admin controls.
- Use Standard tier if you need Private Link, managed private endpoints, or deterministic outbound IPs.
- Ensure all resource IDs and resource group names match your environment.

# Grafana Cloud Setup

## Kubernetes Monitoring setup

To set up Kubernetes Monitoring in Grafana Cloud, use the built-in setup guide:

1. Go to **Connections > Collector Setup** and search for **Kubernetes**.
2. Select **Kubernetes Monitoring** — this walks you through activating the feature.

![Kubernetes connection setup](images/image32.png)

3. You'll get a generated Helm chart command to deploy Grafana Alloy on your cluster, which ships metrics, logs, and events automatically.

![Grafana Alloy deployment command](images/image33.png)

## Grafana Cloud Kubernetes Monitoring pod roles

These pods are part of the Grafana Alloy stack deployed on your EKS cluster for Grafana Cloud Kubernetes monitoring.

- `beyla-k8s-cache-...` — Kubernetes resource cache for Alloy. It watches API objects and keeps cluster metadata available for other Alloy components.
- `grafana-k8s-monitoring-alloy-operator-...` — The Alloy operator that manages deployment lifecycle, reconciles configuration, and ensures Alloy components are running.
- `grafana-k8s-monitoring-alloy-singleton-...` — A single control pod for cluster-level coordination and central Alloy status/config handling.
- `grafana-k8s-monitoring-alloy-receiver-...` — Ingests telemetry from Alloy collectors and forwards metrics/logs/events to Grafana Cloud.
- `grafana-k8s-monitoring-alloy-metrics-0` — The main metrics collector that scrapes Prometheus-style metrics and ships them to Grafana Cloud.
- `grafana-k8s-monitoring-alloy-logs-...` — Collects logs from the cluster and forwards them into Grafana Cloud logging.
- `grafana-k8s-monitoring-alloy-profiles-...` — Collects profiling data from the cluster for continuous profiling support.
- `grafana-k8s-monitoring-kube-state-metrics-...` — Exports Kubernetes control-plane state metrics such as pod/deployment counts, resource requests, and object health.
- `grafana-k8s-monitoring-node-exporter-...` — Collects host/node metrics like CPU, memory, disk, and network usage.
- `grafana-k8s-monitoring-kepler-...` — Collects granular node and container resource usage metrics, often used for efficiency and utilization analysis.
- `grafana-k8s-monitoring-opencost-...` — Collects cost and usage data for cloud cost monitoring and chargeback insights.
- `grafana-k8s-monitoring-k8s-manifest-tail-...` — Watches manifest/config changes to keep Alloy deployment config in sync.

This setup matches the Grafana Cloud Kubernetes monitoring flow: deploy Grafana Alloy, collect cluster telemetry, and stream it into Grafana Cloud for dashboards and alerts.

## Grafana Native Kubernetes Monitoring Experience

The Grafana Cloud Kubernetes monitoring experience is built as a unified drill-down workflow from cluster to workload.

- Clusters → Namespaces → Workloads → Nodes → All Jobs — all connected in one navigation flow
- Within each workload you get tabs for **Overview, CPU, Memory, Network, Storage, Energy, Logs, Events, Changes**
- **Per-pod filtering** is available inline on the dashboard
- **Scheduling & Alignment panels** show CPU request percentage and usage vs request alignment
- **Right-sizing intelligence** is built in with p95 usage vs requests comparisons
- **AI Insights toggle** is available per workload
- **Profiles tab** exposes Pyroscope / continuous profiling data
- **Cost** is surfaced in the sidebar for cluster and workload views

![Grafana native Kubernetes monitoring navigation](images/image36.png)

## Azure Managed Grafana Kubernetes Monitoring Experience

Azure Managed Grafana with Azure Managed Prometheus exposes Kubernetes telemetry through separate dashboards rather than one cohesive drill-down path.


### Complete Pre-built Azure Managed Grafana K8s Dashboards

### Group 1 — Compute Resources (`kubernetes-mixin` tag)
From Image 1, filtered by `kubernetes-mixin`:

| Dashboard | Location |
|---|---|
| Kubernetes / Compute Resources / Cluster | Azure Managed Prometheus |
| Kubernetes / Compute Resources / Cluster (Windows) | Azure Managed Prometheus |
| Kubernetes / Compute Resources / Namespace (Pods) | Azure Managed Prometheus |
| Kubernetes / Compute Resources / Namespace (Workloads) | Azure Managed Prometheus |
| Kubernetes / Compute Resources / Namespace (Windows) | Azure Managed Prometheus |
| Kubernetes / Compute Resources / Node (Pods) | Azure Managed Prometheus |
| Kubernetes / Compute Resources / Pod | Azure Managed Prometheus |
| Kubernetes / Compute Resources / Pod (Windows) | Azure Managed Prometheus |
| Kubernetes / Compute Resources / Workload | Azure Managed Prometheus |
| Kubernetes / Kubelet | Azure Managed Prometheus |
| Kubernetes / USE Method / Cluster (Windows) | Azure Managed Prometheus |
| Kubernetes / USE Method / Node (Windows) | Azure Managed Prometheus |

![Azure Managed Grafana compute dashboards](images/image34.png)
### Group 2 — Networking (`k8s:network-observability` tag)
From Image 2, filtered by `k8s:network-observability`:

| Dashboard | Location |
|---|---|
| Azure / Insights / Containers / Networking (v1) | Azure Monitor |
| Azure / Insights / Containers / Networking (v2) | Azure Monitor |
| Kubernetes / Networking / Clusters | Azure Managed Prometheus |
| Kubernetes / Networking / DNS (Cluster) | Azure Managed Prometheus |
| Kubernetes / Networking / DNS (Workload) | Azure Managed Prometheus |
| Kubernetes / Networking / Drops (Workload) | Azure Managed Prometheus |
| Kubernetes / Networking / L7 (Namespace) | Azure Managed Prometheus |
| Kubernetes / Networking / L7 (Workload) | Azure Managed Prometheus |
| Kubernetes / Networking / Pod Flows (Namespace) | Azure Managed Prometheus |
| Kubernetes / Networking / Pod Flows (Workload) | Azure Managed Prometheus |

![Azure Managed Grafana networking dashboards](images/image35.png)

## Compared to Grafana Native K8s

| Category | Grafana Native K8s | Azure Managed Grafana |
|---|---|---|
| **Compute — Cluster level** | ✅ Unified drill-down | ✅ Dedicated dashboard |
| **Compute — Namespace level** | ✅ Tab within flow | ✅ Pods + Workloads separate |
| **Compute — Node level** | ✅ Tab within flow | ✅ Dedicated dashboard |
| **Compute — Pod level** | ✅ Tab within flow | ✅ Dedicated dashboard |
| **Compute — Workload level** | ✅ Tab within flow | ✅ Dedicated dashboard |
| **Windows node pools** | ❌ Not shown | ✅ Cluster, Pod, USE Method |
| **Kubelet metrics** | ❌ Not separate | ✅ Dedicated dashboard |
| **USE Method dashboards** | ❌ No | ✅ Windows specific |
| **Networking — DNS** | ❌ No | ✅ Cluster + Workload level |
| **Networking — Drops** | ❌ No | ✅ Workload level |
| **Networking — L7** | ❌ No | ✅ Namespace + Workload |
| **Networking — Pod Flows** | ❌ No | ✅ Namespace + Workload |
| **Azure Monitor Networking** | ❌ No | ✅ 2 dashboards (v1, v2) |
| **Right-sizing / Alignment** | ✅ Built-in panels | ❌ Not pre-built |
| **AI Insights** | ✅ Yes | ❌ No |
| **Logs tab** | ✅ Integrated | ❌ Separate |
| **Events tab** | ✅ Per workload | ❌ Not pre-built |
| **Profiling** | ✅ Pyroscope tab | ❌ No |
| **Navigation style** | Unified drill-down | Flat independent dashboards |

## Key Takeaway

**Azure Managed Grafana wins on breadth** — especially networking observability (DNS, L7, Pod Flows, Drops) and Windows workload support which Grafana native doesn't pre-build.

**Grafana Native wins on depth** — unified UX, right-sizing intelligence, logs/events integrated per workload, AI insights, and profiling — all things Azure Managed Grafana doesn't have out of the box.

---

## Overview

| | Grafana Cloud | Azure Managed Grafana |
|---|---|---|
| **Managed by** | Grafana Labs | Microsoft |
| **Underlying version** | Grafana 13 (latest, always up-to-date) | Grafana Enterprise (lags behind, Microsoft-controlled cadence) |
| **Deployment model** | SaaS (multi-cloud, hosted by Grafana Labs) | PaaS within Azure subscription |
| **Primary use case** | Full-stack observability, LGTM stack, AI-driven ops | Azure-centric monitoring, Azure Monitor integration |
| **Best for** | Teams on multi-cloud / hybrid stacks, needing latest Grafana features | Teams deeply embedded in Azure with Entra ID, Azure Monitor as primary signal source |

---

## Quick Verdict

| Scenario | Winner |
|---|---|
| You live in Azure and use Azure Monitor + ADX | **Azure Managed Grafana** |
| You need latest Grafana features (Grafana 13, AI Assistant) | **Grafana Cloud** |
| You need full RBAC control | **Grafana Cloud** |
| You need k6, Loki, Tempo, Mimir, Pyroscope | **Grafana Cloud** |
| You need Azure Entra ID SSO without extra config | **Azure Managed Grafana** |
| You need Grafana Plugin Catalog access | **Grafana Cloud** |
| You need SLA with zone redundancy | Both (Azure Managed Grafana Standard; Grafana Cloud Pro+) |
| Budget is usage-based and teams are small | **Grafana Cloud** (free tier available) |
| Predictable per-user billing preferred | **Azure Managed Grafana** |

---

## Feature Comparison Matrix

| Feature | Grafana Cloud | Azure Managed Grafana |
|---|:---:|:---:|
| Always on latest Grafana version | ✅ | ❌ (Microsoft-controlled cadence) |
| Grafana 13 / Grafana Assistant (AI) | ✅ GA | ❌ Not available |
| AI Observability for LLMs/agents | ✅ Public Preview | ❌ |
| Full Grafana RBAC | ✅ | ❌ Disabled |
| Grafana Server Admin role | ✅ | ❌ Not available to customers |
| Plugin Catalog (install/upgrade/remove) | ✅ | ❌ |
| Grafana Enterprise plugins | ✅ (paid add-on) | ✅ (optional, Standard only) |
| Git Sync (GitOps) | ✅ GA | ❌ |
| Grafana Advisor (health checks) | ✅ | ❌ |
| Loki (log aggregation) | ✅ Kafka-backed, v3.x | ❌ (not bundled) |
| Tempo (distributed tracing) | ✅ | ❌ (not bundled) |
| Mimir (scalable Prometheus) | ✅ | ❌ (not bundled) |
| Pyroscope (continuous profiling) | ✅ v2.0 | ❌ |
| k6 (performance testing) | ✅ v2.0 with AI | ❌ |
| Synthetic Monitoring | ✅ | ❌ |
| Azure Monitor native integration | ✅ (plugin) | ✅ Built-in, first-class |
| Azure Data Explorer (ADX) | ✅ (plugin) | ✅ Built-in (throttling caveats) |
| Azure Entra ID (AAD) SSO | ✅ (configurable) | ✅ Native, mandatory |
| Third-party identity providers | ✅ | ❌ (Entra ID only) |
| Managed Identity for data sources | ✅ | ✅ (one per workspace) |
| Private Link / Private Endpoints | ✅ (Enterprise) | ✅ Standard tier |
| Zone redundancy | ✅ | ✅ Standard tier |
| Deterministic outbound IPs | ✅ | ✅ Standard tier |
| Unified Alerting | ✅ | ✅ (Standard; manual enable for pre-Dec-2022 workspaces) |
| SMTP / Email reporting | ✅ | ✅ Standard only |
| PDF / Image rendering | ✅ | ✅ Standard only |
| Dashboard import from Azure Portal | ❌ | ✅ |
| Grafana Marketplace (plugins) | ✅ Pilot | ❌ |
| IRM (Incident Response & Mgmt) | ✅ | ❌ |
| Adaptive Telemetry (cost control) | ✅ | ❌ |
| Free tier | ✅ (forever free tier) | ❌ (Essential in preview, being deprecated) |
| SLA | ✅ Pro+ | ✅ Standard |

---

## Grafana Cloud — Key Features (Grafana 13)

### Dashboards & Visualization

- **Suggested Dashboards** — automatically surfaces pre-built dashboards tailored to connected data sources.
- **Dynamic Dashboards** — GA; more responsive and scalable for growing teams with section-level variables for rows and tabs.
- **Graphviz Panel** — new visualization option for graph/network diagrams.
- **Git Sync (GA)** — manage dashboards and config as code with native GitOps workflows.
- **Plugin lifecycle management** — Cloud automatically keeps plugins up-to-date and compatible.

### AI & Machine Learning

- **Grafana Assistant (GA)** — purpose-built LLM agent embedded in Grafana Cloud:
  - Build and debug PromQL, LogQL, TraceQL, SQL, and k6 queries.
  - Auto-build and customize dashboards from natural language.
  - Root cause analysis and incident investigation.
  - Accessible via Slack, Microsoft Teams, API, and the new `gcx` CLI.
  - Available to OSS/Enterprise users via one-click plugin (requires Cloud account).
- **Assistant Skills (GA)** — documents that give agents instructions, runbooks, workflow context, auto-approval of tool calls, and auto-remediation pipelines.
- **Assistant Automations** — scheduled activity summaries.
- **AI Observability (Public Preview)** — monitor LLM agents and agentic pipelines in production:
  - Observe agent inputs, outputs, and execution flows in real time.
  - Continuous output evaluation with alerting for policy violations, anomalies, and low-quality responses.
  - Anthropic integration for model usage and cost monitoring.
- **Grafana Cloud CLI (gcx)** — new agentic CLI for automated and agent-driven workflows.
- **o11y-bench (OSS)** — open benchmark for evaluating AI agents on observability tasks.
- **Adaptive Telemetry** — filters unused metrics/logs/traces, reducing cost 35–50%.

### OSS Stack (LGTM+)

| Component | Version / Status | Highlights |
|---|---|---|
| **Loki** | 3.x (redesigned) | Kafka-backed ingestion; redesigned query engine; up to 20x less data scanned, 10x faster aggregated queries; needle-in-haystack full-text indexing (via Logline acquisition) |
| **Tempo** | Latest | Distributed tracing, TraceQL |
| **Mimir** | Latest | Scalable Prometheus-compatible metrics |
| **Pyroscope** | 2.0 | Ground-up rearchitecture; stateless querying; metrics from profiles; heatmap queries; significantly lower cost |
| **k6** | 2.0 (upcoming) | AI-assisted test authoring (`agent`, `mcp`, `docs`, `explore` subcommands); Playwright-inspired Assertions API; formalized extension ecosystem; k6 Operator 1.0 for Kubernetes distributed testing |

### Alerting & IRM

- Unified alerting (Grafana Alertmanager).
- Full RBAC on notification policies.
- Incident Response & Management (IRM) module.
- SRE agent for automated root cause analysis.

### Pricing Tiers (Grafana Cloud)

| Tier | Details |
|---|---|
| **Free** | Actually useful forever-free tier; 13 months metric retention; 30 days logs/traces/profiles/k6 |
| **Pro** | Pay-as-you-go; $8/active user/month (base); $55/user/month with Enterprise plugins |
| **Advanced / Enterprise** | Minimum $25,000/year; BYOC (Bring Your Own Cloud) option |

---

## Azure Managed Grafana — Key Features & Limitations

### Strengths

- **First-class Azure integration** — Azure Monitor, Azure Data Explorer, Azure Managed Prometheus, and Azure Log Analytics are all natively supported with minimal configuration.
- **Microsoft Entra ID (AAD) SSO** — mandatory; centralized identity management; supports Conditional Access policies.
- **Dashboard import from Azure Portal** — one-click import of existing Azure Monitor charts.
- **Managed Identity support** — system-assigned or user-assigned (one per workspace) for secure data source authentication.
- **Zone redundancy** — available on Standard tier with SLA.
- **Private Link + Managed Private Endpoints** — Standard tier; restrict all traffic to private network.
- **Deterministic outbound IPs** — Standard tier; for firewall allowlisting.
- **Grafana Enterprise plugins** — optional add-on on Standard tier (e.g., AppDynamics, Datadog, Dynatrace, Splunk, ServiceNow, Oracle).
- **Encryption** — data at rest (Microsoft-managed keys) and in transit (TLS 1.2).
- **Compliance** — 50+ compliance certifications via Azure infrastructure.
- **Billing via Azure** — fits into existing EA/MACC spending commitments.

### Known Limitations

| Limitation | Detail |
|---|---|
| **Grafana version lag** | Runs Grafana Enterprise but on Microsoft's update cadence — NOT always the latest. Grafana 13 features (AI Assistant, Git Sync, Advisor, etc.) are not yet available. |
| **No Grafana RBAC** | Full Grafana Role-Based Access Control is disabled. Only Admin / Editor / Viewer org-level roles available. |
| **No Server Admin role** | Customers cannot access Server Admin. Admin API, User API, and Admin Organizations API are all blocked. |
| **No Plugin Catalog** | Cannot install, uninstall, or upgrade plugins from the Grafana Plugin Catalog. Plugins are managed by Microsoft. |
| **Entra ID only** | All users must have Entra ID accounts. Third-party OAuth/SAML providers not natively supported. |
| **One managed identity per workspace** | Cannot use both system-assigned and user-assigned managed identities simultaneously. |
| **No bundled LGTM stack** | Loki, Tempo, Mimir, Pyroscope, and k6 are NOT included. You must bring your own or use Azure equivalents. |
| **Azure Data Explorer throttling** | High-cardinality or multi-panel ADX queries may result in 50x errors or slow responses. |
| **Current User auth limitation** | Automated background tasks (alerts, reporting) fail if no user is interactively logged in when using Current User auth. |
| **Essential tier deprecating** | Essential (preview) tier is being retired; Microsoft recommends migrating to Standard or Azure Monitor dashboards. |
| **No Grafana Marketplace** | Third-party plugin marketplace not available. |
| **No AI/ML features** | Grafana Assistant, AI Observability, Adaptive Telemetry — none available. |
| **Enterprise only via Microsoft** | CSP (Cloud Solution Provider) subscriptions cannot purchase Grafana Enterprise through Azure. |

### Service Tiers

| Feature | Essential (Deprecated) | Standard X1 | Standard X2 |
|---|:---:|:---:|:---:|
| Dashboards | 20 max | Unlimited | Unlimited |
| Data sources | 5 max | Unlimited | Unlimited |
| Alert rules | ❌ | 500/org | 1,000/org |
| API keys | 2 max | 100 | 100 |
| Zone redundancy | ❌ | ✅ | ✅ |
| Private endpoints | ❌ | ✅ | ✅ |
| SMTP/Reporting | ❌ | ✅ | ✅ |
| Grafana Enterprise (optional) | ❌ | ✅ | ✅ |
| Max instances / subscription / region | 1 | 50 | 50 |
| SLA | ❌ | ✅ | ✅ |

---

## Pricing Model Comparison

| Aspect | Grafana Cloud | Azure Managed Grafana |
|---|---|---|
| **Model** | Usage-based (metrics series, log GB, trace spans, etc.) + per active user | Per-active-user + hourly compute (Standard: ~$0.043/hr/unit in Central US; Zone-redundant: ~$0.051/hr/unit) |
| **Free tier** | Yes — forever free, meaningful limits | No (Essential deprecated; 30-day Azure trial credits only) |
| **Cost predictability** | Variable — scales with ingestion volume | More predictable — compute + user count |
| **Azure billing integration** | ❌ (separate Grafana Labs invoice) | ✅ (rolls into Azure subscription / EA) |
| **Enterprise add-on** | $55/user/month (Enterprise plugins) | Optional, licensing through Microsoft |

---

## Identity & Access Management

| Capability | Grafana Cloud | Azure Managed Grafana |
|---|---|---|
| SSO via Entra ID / AAD | ✅ Configurable | ✅ Native, mandatory |
| SAML / OAuth (third-party) | ✅ | ❌ |
| Grafana RBAC | ✅ Full | ❌ Disabled |
| Grafana Server Admin | ✅ | ❌ |
| Team sync with Entra ID | ✅ (via config) | ✅ (Preview in sovereign clouds) |
| Managed Identity for data sources | ✅ | ✅ (one per workspace) |
| Conditional Access (Azure) | ❌ | ✅ |

---

## Observability Stack Support

| Signal Type | Grafana Cloud (native) | Azure Managed Grafana (native) |
|---|---|---|
| Metrics (Prometheus) | ✅ Mimir | ✅ Azure Managed Prometheus / Azure Monitor |
| Logs | ✅ Loki 3.x (Kafka-backed) | ❌ (bring your own; Azure Monitor Logs via plugin) |
| Traces | ✅ Tempo | ❌ (bring your own; Azure Monitor / App Insights via plugin) |
| Profiles | ✅ Pyroscope 2.0 | ❌ |
| Synthetic tests | ✅ | ❌ |
| Performance/load testing | ✅ k6 2.0 | ❌ |
| Azure Monitor | ✅ (plugin) | ✅ First-class native |
| Azure Data Explorer | ✅ (plugin) | ✅ First-class (throttle caveats) |
| Azure Log Analytics | ✅ (plugin) | ✅ |

---

## AI Capabilities

| Feature | Grafana Cloud | Azure Managed Grafana |
|---|---|---|
| Grafana Assistant (AI chat for dashboards/queries) | ✅ GA | ❌ |
| AI Observability (monitor LLMs/agents) | ✅ Public Preview | ❌ |
| AI-assisted k6 test authoring | ✅ (k6 2.0) | ❌ |
| SRE agent for root cause analysis | ✅ | ❌ |
| Adaptive Telemetry (AI cost optimization) | ✅ | ❌ |
| o11y-bench (AI agent benchmarking) | ✅ OSS | ❌ |
| Assistant via Slack / Teams | ✅ | ❌ |
| gcx CLI for agentic workflows | ✅ | ❌ |

---

## Plugin & Extensibility

| Aspect | Grafana Cloud | Azure Managed Grafana |
|---|---|---|
| Plugin Catalog (self-serve install) | ✅ | ❌ |
| Plugin auto-updates | ✅ Managed by Grafana Labs | ❌ Managed by Microsoft |
| Grafana Enterprise plugins | ✅ (paid) | ✅ (paid, Standard only) |
| Grafana Marketplace (ISV plugins) | ✅ Pilot phase | ❌ |
| Custom / private plugins | ✅ | Limited |

---

## Networking & Security

| Feature | Grafana Cloud | Azure Managed Grafana |
|---|---|---|
| Private Link | ✅ (Enterprise) | ✅ Standard |
| Managed Private Endpoints | ✅ | ✅ Standard |
| Deterministic outbound IPs | ✅ | ✅ Standard |
| Zone redundancy | ✅ | ✅ Standard |
| TLS in transit | ✅ TLS 1.2+ | ✅ TLS 1.2 |
| Encryption at rest | ✅ | ✅ (Microsoft-managed keys) |
| Customer-managed keys (CMK) | ✅ Enterprise | ❌ |
| Data residency control | ✅ (region selection) | ✅ (stored in workspace region) |
| Compliance certifications | SOC2, ISO 27001, HIPAA, etc. | 50+ Azure certifications |

---

## IaC / GitOps Support

| Capability | Grafana Cloud | Azure Managed Grafana |
|---|---|---|
| Git Sync (GA) | ✅ Native in Grafana 13 | ❌ |
| Terraform provider | ✅ (grafana/grafana) | ✅ (azurerm_dashboard_grafana) |
| Grafana Advisor (health checks) | ✅ | ❌ |
| API-driven dashboard management | ✅ Full API | ⚠️ Partial (Server Admin APIs blocked) |
| Azure Resource Manager (ARM/Bicep) | ❌ | ✅ |

---

## Service Limits & Quotas

| Limit | Grafana Cloud (Pro) | Azure Managed Grafana (Standard X1) | Azure Managed Grafana (Standard X2) |
|---|---|---|---|
| Dashboards | Unlimited | Unlimited | Unlimited |
| Alert rules | Unlimited | 500/instance | 1,000/instance |
| Data sources | Unlimited | Unlimited | Unlimited |
| Metric retention | 13 months | Depends on Azure Monitor config | Depends on Azure Monitor config |
| Log retention | 30 days | Depends on Log Analytics workspace | Depends on Log Analytics workspace |
| Max instances / region | N/A (SaaS) | 50/subscription | 50/subscription |
| API keys | Unlimited | 100 | 100 |
| Requests per IP/s | N/A | 90 req/s | 90 req/s |
| Requests per HTTP host/s | N/A | 45 req/s | 45 req/s |
| Data query timeout | N/A | 200 seconds | 200 seconds |
| Image/PDF render timeout | N/A | 220 seconds | 220 seconds |
| Data source query size | N/A | 80 MB | 80 MB |

---

## Sovereign Cloud & Compliance

| Region | Grafana Cloud | Azure Managed Grafana |
|---|---|---|
| Azure Government | ✅ (separate configuration) | ✅ (with limitations: no Enterprise plugins, no Essential plan) |
| Azure China (21Vianet) | ❌ | ✅ Preview (same limitations as Gov) |
| Standard commercial regions | ✅ Global | ✅ All major Azure regions |

---

## Recommendation

### Choose Grafana Cloud if:

- You need access to the **latest Grafana features** (Grafana 13, AI Assistant, Git Sync, Advisor, Marketplace).
- Your stack is **multi-cloud or hybrid** (AWS + Azure + GCP, on-prem).
- You need the **full LGTM+ stack** (Loki, Tempo, Mimir, Pyroscope, k6) under one roof.
- You need **full RBAC** and self-serve **plugin management**.
- You are building or monitoring **AI/agentic applications** (AI Observability, o11y-bench).
- Your team is small and the **free tier** is valuable for getting started.
- You want **GitOps/IaC** workflows for dashboard management (Git Sync).

### Choose Azure Managed Grafana if:

- Your primary observability signals live in **Azure Monitor, Azure Data Explorer, or Azure Log Analytics**.
- Your organization is **Entra ID-first** and needs Conditional Access enforcement.
- You need dashboards to **roll into Azure billing** (EA/MACC commitments).
- You want a **simple, managed Grafana** with minimal ops overhead and strong Azure SLA guarantees.
- Your compliance posture requires **Azure-native certifications** and data residency guarantees.
- You are already using **Azure Managed Prometheus** and want a co-located UI.

### Hybrid Approach (Recommended for Azure-heavy teams)

Use **Azure Managed Grafana** for your Azure-native dashboards and Azure Monitor signals (leveraging its first-class ADX/Monitor integration and Entra ID SSO), while using **Grafana Cloud** (or connecting your Azure Managed Grafana to Grafana Cloud) to access Grafana Assistant for AI-driven analysis, query building, and incident response. Grafana Cloud's Assistant plugin can connect to a self-managed or Azure-hosted Grafana instance via one-click setup, giving you the best of both worlds.

---

## References

- [Grafana 13 Release & GrafanaCON 2026 Announcements](https://grafana.com/blog/grafanacon-2026-announcements/)
- [Azure Managed Grafana Overview — Microsoft Learn](https://learn.microsoft.com/en-us/azure/managed-grafana/overview)
- [Azure Managed Grafana Service Limits — Microsoft Learn](https://learn.microsoft.com/en-us/azure/managed-grafana/known-limitations)
- [Azure Managed Grafana FAQ — Microsoft Learn](https://learn.microsoft.com/en-us/azure/managed-grafana/faq)
- [Grafana Cloud Pricing](https://grafana.com/pricing/)
- [Azure Managed Grafana Pricing](https://azure.microsoft.com/en-us/pricing/details/managed-grafana/)
- [Grafana Assistant Documentation](https://grafana.com/docs/grafana-cloud/machine-learning/assistant/get-started/)
- [AI Observability in Grafana Cloud](https://grafana.com/products/cloud/ai-observability/)