# Build Vs Buy Monitoring Architecture (Unix, Databases, Kubernetes)

## Overview

This repository documents a target monitoring architecture to replace BMC Helix Operations Management (BHOM)/TrueSight for a large Unix and database estate (~60k Unix/Linux servers, ~8k database servers, >10k DB instances), with a roadmap to Kubernetes. The design assumes:

- Unix-only hosting for all monitoring components (no Windows).
- Heavy use of custom log and script-based monitoring today.
- A self-service portal for application teams to request log/process/filesystem monitoring.
- A strict 15-day OS patching cycle for infrastructure.
- A separate event-management/GMOM layer for enrichment and correlation.

Two open-source options are evaluated:

1. **Zabbix** as the primary monitoring platform.
2. **Prometheus stack** (Prometheus + Alertmanager + long-term storage + Grafana).

BMC Helix Ops Mgmt (BHOM) is used as the baseline for functional comparison.[cite:149][cite:154]

---

## Current State Summary

- ~60k Unix/Linux servers and ~8k database servers across UK, US, HK, MX.
- BHOM/TrueSight with PATROL agents and KMs for OS, DB, log, and script monitoring.[cite:149][cite:155][cite:159]
- Integration Services limited to ~750 agents each, leading to 60+ IS instances per region and overload issues at high monitor/attribute counts.[cite:90][cite:97]
- ~38k custom policies across ~40k servers, including ~12k in UK.
- At least 200 policy changes per month, mostly log/process/filesystem monitoring requested via a self-service portal.
- GMOM-like event layer used for enrichment, correlation, and downstream routing.

The replacement platform must:

- Preserve baseline OS and DB monitoring via tag-like assignments (e.g., LINUX tag).
- Support high-volume custom log/process/filesystem policies with per-host thresholds.
- Integrate cleanly with the existing event-management layer.
- Remain supportable by a production support team and an onboarding team.
- Extend cleanly to Kubernetes and containers without introducing a second monitoring product.

---

## Design Principles

1. **Unix-only hosting** – All servers (core, proxies, DB, UI) must run on Linux/Unix.
2. **Single logical monitoring platform** – Same platform covers Unix, DB, and Kubernetes.
3. **Strong policy abstraction** – Support a Helix/Patrol-like model: baselines by tag/template and per-host overrides via API.
4. **First-class log and script monitoring** – Support script results via logs and custom thresholds per host/filesystem.
5. **API-driven self-service** – Application teams interact via a portal, which uses APIs to create/update/remove monitoring definitions.
6. **High availability with 15-day patch cycles** – Monitoring remains available during rolling patching.
7. **Open source + commercial support** – Core stack is open source, with vendor support and training options available.
8. **Kubernetes-ready** – Native integrations for K8s metrics and state, with clear roadmap for containerization of the monitoring stack itself.

---

## Option 1 – Zabbix-Based Target Architecture

Zabbix is a mature, open-source monitoring platform with a central server, a distributed proxy layer, and agents on monitored hosts.[cite:39][cite:54][cite:103] It natively supports templates (similar to policies), log monitoring, custom scripts, and has a full HTTP API.

### 1.1 Core Components (Unix-Only)

**Central region (primary DC or main cloud region):**

- **Zabbix Server cluster**
  - 2 Linux nodes (active/active or active/passive behind a virtual IP).
  - Responsible for configuration, evaluation of triggers, and data storage orchestration.
- **Database cluster**
  - 3 Linux nodes running MariaDB or PostgreSQL dedicated to Zabbix.
  - Synchronous or semi-synchronous replication for HA.
- **Web/UI/API cluster**
  - 2 Linux nodes running the Zabbix web frontend and API, behind a load balancer.

**Regional data collection (UK, US, HK, MX):**

- **Zabbix Proxies**
  - UK: 6–8 proxies, sized for ~5–10k NVPS (new values per second) each, split by environment and domain (Unix, DB, app).
  - US: 6–8 proxies with similar sizing.
  - HK/MX: 3–4 proxies per region, scaled to host counts.
  - Proxies run on Linux, buffering data locally and actively pushing to the central server.[cite:54][cite:56]

**Agents:**

- Zabbix agent 2 deployed on all Unix/Linux servers and DB hosts.
- Agents use both passive and active checks; active checks allow configuration changes to propagate from server/proxy to agents without manual file edits.[cite:39][cite:103]

This architecture is sized with headroom beyond the current TrueSight integration service limits and is aligned with Zabbix performance guidance around NVPS and proxy distribution.[cite:76][cite:82][cite:85]

### 1.2 Policy and Custom Monitoring Model

Zabbix maps naturally to the existing Helix/Patrol policy model.

**Baselines (Tag = LINUX → Template):**

- `Template_Linux_Baseline` includes:
  - CPU, memory, uptime, load average, kernel metrics.
  - Filesystem discovery and utilization metrics via low-level discovery (LLD).
  - Default process checks (e.g., sshd, critical daemons).
  - ICMP ping (availability/latency) and agent health (`agent.ping`, nodata-based triggers).[cite:103][cite:115]
- Applied to all Linux hosts via host groups and auto-registration rules, mirroring the existing tag-based baseline pattern.[cite:103]

**Custom per-host/per-app policies:**

- Application and onboarding teams require per-host custom monitoring for:
  - Specific log files and patterns.
  - Application processes.
  - Specific filesystems and thresholds.
- These are modelled as:
  - **Items**: log/file/process items defined on the host or via custom templates.
  - **Triggers**: alert conditions on item values.
  - **Macros**: values for thresholds (e.g., `{$FS_WARN}`, `{$FS_CRIT}`, `{$PROC_MIN}`) overridable at host or template scope.[cite:103][cite:115]

**Script monitoring via logs:**

- Scripts continue to write status or metrics to log files.
- Zabbix agent uses native log item keys such as `log[]`, `logrt[]`, and associated `log.count[]` variants:[cite:106][cite:110][cite:114]
  - Example: `log[/var/log/appX/script_status.log,"SCRIPT_ERROR|CRITICAL"]`.
  - Triggers fire when matching patterns appear or counts exceed thresholds.
- This closely matches the existing use of PATROL scripting KM and log KM, but with a cleaner API-driven configuration model.[cite:155][cite:159]

### 1.3 API and Self-Service Portal

- Zabbix exposes a comprehensive HTTP API for configuration:
  - `item.create`, `item.update`, `item.delete` for log/process/filesystem items.[cite:107]
  - `trigger.create`, `trigger.update`, `trigger.delete` for alert conditions.[cite:111][cite:115]
  - Host, template, and macro APIs for baseline application and overrides.[cite:103]
- The self-service portal for application teams will:
  - Read current configuration via the API (for viewing existing log/process policies).
  - Allow app teams to request new or changed policies.
  - Translate requests into API calls that create/update items, triggers, and macros.
  - Integrate with change management for approval and (planned) versioning.

This approach allows the onboarding team to maintain standards while giving app teams the ability to drive changes via a controlled portal.

### 1.4 Patching and Operations Model (15-Day Cycle)

- **Central cluster:**
  - Patch web/API nodes one at a time, then DB nodes in sequence with failover, then server nodes.
  - Maintain N+1 capacity so monitoring remains available during patch windows.
- **Proxies:**
  - At least two proxies per domain/region to allow alternating patching.
  - During a proxy patch, hosts either buffer data locally (active checks) or temporarily send to another proxy in the same region.
- **Agents:**
  - Upgraded as part of OS patching; the lightweight nature of the agent minimizes risk.
  - Nodata/agent.ping triggers tuned to avoid noise during planned patch windows.

Zabbix’s architecture and proxy buffering make it well-suited to aggressive patching policies without major gaps in monitoring.[cite:54][cite:56]

### 1.5 Kubernetes and Containerization Roadmap

Zabbix can monitor Kubernetes and containers without introducing a second primary monitoring product.

- **In-cluster deployment:**
  - Use the Zabbix Helm chart to deploy a Zabbix proxy and agent DaemonSet into each cluster.[cite:152][cite:162]
  - The in-cluster proxy communicates back to the central Zabbix server over TLS.
- **Kubernetes templates:**
  - Official templates monitor Kubernetes nodes, API server, and other control-plane components via the Kubernetes API and kube-state-metrics.[cite:157][cite:162]
  - Linux node metrics can reuse `Template_Linux_Baseline`, maintaining consistency across bare-metal and container nodes.
- **Prometheus metrics integration:**
  - Zabbix can ingest Prometheus-format metrics via integrations, allowing reuse of existing exporters where needed, while keeping Zabbix as the single pane of glass.[cite:157][cite:162]

The Zabbix server, DB, and web components can initially run on Linux VMs and later be containerized onto internal Kubernetes clusters if required, without changing the overall architecture.

---

## Option 2 – Prometheus-Based Target Architecture

Prometheus is a CNCF-graduated metrics and alerting toolkit and the de facto standard for Kubernetes monitoring.[cite:121][cite:125][cite:129][cite:163] It excels at high-volume metrics ingestion and SRE-style observability but does not provide a native policy abstraction equivalent to BHOM/TrueSight or Zabbix.

### 2.1 Core Components (Unix-Only)

- **Regional Prometheus shards:**
  - 2–3 Prometheus servers per region, each handling ~2M active time series for comfortable headroom.[cite:77][cite:91][cite:99][cite:102]
  - Targets sharded by role (Unix vs DB vs app) and environment (prod vs non-prod).
  - Each shard deployed in an HA pair.
- **Long-term storage and global querying:**
  - Thanos, Cortex, Mimir, or VictoriaMetrics cluster for global queries and long-term retention.[cite:78][cite:96]
- **Alerting and visualization:**
  - Alertmanager cluster for deduplication, routing, and silencing.[cite:116][cite:104]
  - Grafana cluster for dashboards across Prometheus and long-term storage.[cite:83]

All components run on Linux and can be deployed as VMs or containers.

### 2.2 Custom Monitoring and Policy Model

Prometheus exposes primitives (metrics, labels, alerting rules) but does not define policies.

- **Baselines:**
  - node_exporter and DB exporters expose OS/DB metrics; labels encode host attributes (e.g., `os="linux"`, `env="prod"`).[cite:41][cite:113]
  - PromQL alerting rules apply to sets of series filtered by labels.
- **Custom script monitoring:**
  - Scripts expose metrics via custom exporters, or write to files interpreted by textfile collectors.[cite:113][cite:117]
  - Alerts are written as PromQL rules on those metrics.
- **Log monitoring:**
  - Prometheus does not tail logs; a separate system (e.g., Loki or ELK) is required.[cite:113][cite:117]
  - Alerts can be raised in that system or via metrics derived from logs.

To replicate the current BHOM-style policy model (38k+ policies, 200+ changes/month), a separate **policy management layer** must be built:

- A configuration service that stores policy definitions and translates them into:
  - Prometheus rule files or PrometheusRule CRDs (if using Prometheus Operator).
  - Log/alerting rules in Loki/ELK for log-based conditions.
- A CI/CD or orchestration layer that updates rule configurations and coordinates Prometheus reloads.
- Governance and versioning integrated with change management.

This is feasible but requires significant engineering effort and ongoing ownership.

### 2.3 Kubernetes and Containerization

Prometheus is natively aligned with Kubernetes:

- **Kubernetes monitoring:**
  - kube-prometheus-stack Helm chart installs Prometheus, Alertmanager, exporters, and Grafana with K8s service discovery out of the box.[cite:153][cite:158][cite:163]
  - kube-state-metrics exposes Kubernetes object state; node and cAdvisor metrics cover node and container resources.[cite:158][cite:163]
- **Containerizing the monitoring stack:**
  - All components (Prometheus, Alertmanager, Grafana, long-term store) can be run in containers on internal K8s clusters.

However, this creates an operational split:

- K8s and cloud-native stacks follow a GitOps/config-as-code model.
- Traditional Unix/DB workloads would still need a policy translation layer built for Prometheus.

### 2.4 Patching and Operations Model (15-Day Cycle)

- Prometheus shards and replicas are patched in a rolling fashion, with traffic failing over to remaining instances.
- Long-term store components (Thanos/Cortex) and Alertmanager/Grafana nodes are also patched sequentially.
- Exporters on hosts are patched alongside OS updates.

Prometheus can be operated under a strict patch cycle, but the larger number of components (shards, long-term store, Alertmanager, Grafana, exporters, and policy layer) increases operational complexity compared to Zabbix.

---

## Comparison Matrix – BHOM vs Prometheus vs Zabbix

| Dimension | BHOM (Helix Ops Mgmt) | Prometheus stack | Zabbix |
| --- | --- | --- | --- |
| Primary model | PATROL agents + KMs + monitor policies from BHOM console.[cite:149][cite:154][cite:155] | Metrics + alert rules; pull-based scraping with TSDB; configs as YAML/files.[cite:41][cite:113][cite:127] | Agents + templates + items + triggers; central server + proxies.[cite:39][cite:54][cite:103] |
| Policy abstraction | Strong: monitor policies with precedence, pushed to PATROL agents from UI.[cite:149][cite:160] | None natively; policies must be implemented as rule/config management on top.[cite:113][cite:108] | Strong: templates, items, triggers, macros; full HTTP API; maps directly to policies.[cite:103][cite:107][cite:115] |
| Log monitoring | PATROL Log KM, policies from TrueSight/BHOM consoles.[cite:155][cite:159] | Not built-in; logs handled by Loki/ELK or similar, Prometheus alerts only on derived metrics.[cite:113][cite:117] | Native log items (`log[]`, `logrt[]`, `log.count[]`) including rotation and regex support.[cite:106][cite:110][cite:114] |
| Script monitoring | PATROL scripting KM, often outputting to logs.[cite:155][cite:159] | Via custom exporters or textfile collectors; requires significant integration work.[cite:113][cite:117] | Scripts write to logs or return values via agent items; alerts via log or numeric items.[cite:106][cite:110][cite:114] |
| Custom policy volume | Tens of thousands of policies, but with Integration Service scaling limits.[cite:90][cite:97] | Scales in principle, but only if a custom policy/rules service is built and maintained.[cite:108][cite:104] | Designed for many objects; policies expressed as items/triggers/templates via API; suitable for 38k+ policies and high churn.[cite:103][cite:107][cite:115] |
| K8s / containers | Helix ITOM has containerized deployments and K8s integrations.[cite:149][cite:156][cite:161] | Native K8s monitoring with kube-prometheus-stack and service discovery.[cite:153][cite:158][cite:163] | Official K8s monitoring via Helm chart and templates; can ingest Prometheus metrics.[cite:152][cite:157][cite:162] |
| Scaling pattern | Integration Services per ~X agents; scaling via more IS and BHOM capacity.[cite:90][cite:97] | Shard Prometheus by targets/labels; use long-term store for big estates.[cite:77][cite:91][cite:96][cite:99] | Zabbix server + many proxies; proxies offload collection and buffer data; LTS releases with performance guidance.[cite:54][cite:56][cite:76][cite:82][cite:85] |
| Hosting | SaaS/containerized backends, agents on Linux/Windows.[cite:149][cite:154][cite:161] | Linux-only components; can run on VMs or internal K8s.[cite:163][cite:127] | All components run on Unix-like OS; no Windows required.[cite:162][cite:126] |
| Lifecycle / support | Commercial support from BMC; SaaS lifecycle controlled by vendor.[cite:149][cite:160] | CNCF-governed, large open community; multiple vendors offer commercial support.[cite:121][cite:125][cite:129][cite:163] | LTS releases with 5-year support windows; official commercial support and active community.[cite:118][cite:122][cite:126][cite:128] |
| Fit for current operating model | Current baseline, but costly and encountering scaling limits. | Strong for K8s and SRE, but requires a new config-as-code and policy layer. | Very close to Helix/Patrol workflows, supports heavy log/script monitoring, aligns with Unix-only and future K8s. |

---

## Recommendation

Given the requirements and operating model:

- **Adopt Zabbix as the unified monitoring platform** for Unix, databases, and Kubernetes:
  - Strong policy abstraction via templates and macros.
  - Native log and script monitoring, suitable for existing 700+ script-based hosts and thousands of log-based checks.[cite:106][cite:110][cite:114]
  - Proven proxy-based scaling for large environments and clear NVPS-based sizing guidance.[cite:76][cite:82][cite:85]
  - Unix-only hosting and an official Kubernetes integration path.[cite:162][cite:152][cite:157]
  - Full HTTP API to support the current self-service portal model.[cite:103][cite:107]
- **Use Prometheus tactically** where it adds clear value:
  - For advanced SLO/SRE metrics on Kubernetes workloads.
  - As a metrics source that Zabbix can ingest where Prometheus exporters are already standard.

This approach minimizes risk when migrating away from BHOM/TrueSight while providing a future-proof platform across bare metal, VMs, and containers.

---

## Next Steps

1. **UK Region Pilot (Unix + selected DBs)**
   - Deploy Zabbix server/DB/web in HA and 2–3 proxies for UK.
   - Onboard a subset of hosts (e.g., 1–2k servers) with baseline templates.
   - Migrate a slice of custom log/process policies via the portal and Zabbix API.
2. **Refine Capacity and Sizing**
   - Measure NVPS, queue lengths, DB load, and proxy lag.
   - Adjust proxy counts and hardware sizing before global rollout.
3. **Global Rollout**
   - Scale proxies and templates to remaining regions.
   - Migrate policies region by region while BHOM/TrueSight runs in parallel.
4. **Kubernetes Integration**
   - Introduce Zabbix Helm chart for initial clusters.
   - Align node metrics and cluster-state monitoring with existing templates.
5. **Prometheus Evaluation**
   - Optionally deploy a small Prometheus stack for a specific K8s domain.
   - Assess whether additional Prometheus usage is justified beyond what Zabbix provides.
