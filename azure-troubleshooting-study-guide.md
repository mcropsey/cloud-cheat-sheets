# Azure Troubleshooting Study Guide

Each topic: **Concept** (why it breaks), **Core** (CLI and portal paths), **Example** (a realistic incident, start to finish, ending in a verify), **Trap** (what wastes an hour). Portal labels are as of Aug 2026.

**The one rule:** work the five layers top-down and don't skip. **Platform → Identity → Network → Data plane → App.** Most "Azure is down" tickets are a subscription context, a permission, a network path, or a config change. Prove layers 1–4 before you read application code. And on every resource, run **Diagnose and solve problems** before you think — it's free and usually right.

| Layer | Smells like | First tool |
|---|---|---|
| 1 Platform / quota | stuck Creating/Failed, `AllocationFailed`, quota, lock, Policy deny | Service Health, Resource Health, Activity log, Usage + quotas |
| 2 Identity | 401/403, `AADSTS*`, `AuthorizationFailed`, Key Vault 403, MI not found | IAM + deny assignments, Entra sign-in logs |
| 3 Network | timeouts, RDP/SSH fail, PE 403, SNAT, DNS wrong IP | Network Watcher, effective NSG rules, next hop, private DNS |
| 4 Data plane | Storage 403/409, SQL login fail, disk attach fail | resource firewall, keys/SAS/Entra, metrics |
| 5 App / guest OS | 500s, crash loop, boot failure, probe fail | Boot diagnostics, serial console, Kudu, App Insights |

---

## 1. First 60 Seconds

**Concept:** Three health surfaces, three questions. Then: what changed?

| Surface | Question | Where |
|---|---|---|
| Azure Status | Is Azure down for everyone? | azure.status.microsoft (no login) |
| Service Health | Did an incident/maintenance hit *my* subscriptions and regions? | Portal → **Service Health** |
| Resource Health | Is *this* resource healthy — platform or me? | Resource → **Resource health** |

Resource Health statuses: **Available** (look at your config) · **Degraded** (up but slow — metrics, throttling) · **Unavailable** (read the cause: blue = user action, red = platform) · **Unknown** (no signal >10 min, normal on deallocated VMs — confirm power state).

```bash
az account show                                                   # which tenant/subscription am I in — ALWAYS first
az account set --subscription <id>
az monitor activity-log list -g RG --offset 24h --status Failed --max-events 50 -o table
az monitor activity-log list --correlation-id <guid>              # the whole ARM transaction
az deployment group list -g RG --query "[].{n:name,s:properties.provisioningState}" -o table
```
**Portal:** Resource → **Activity log** → Timespan + **Status = Failed** → open event → copy *Operation, Caller, Correlation ID, innermost error code/message*. Then Resource → **Diagnose and solve problems** → search the symptom ("can't connect", "503"). Portal **Copilot → Troubleshooting agent** can run detectors and draft the support request.

**Decision rule:** Service Health or Resource Health shows a platform event → stop changing config, open a support request **from the Resource Health blade** (no paid plan needed for platform unavailability). Health Available and Activity log clean → it's config, network, identity, or app.

**Example — "The web app started 503ing at 09:40":**
```bash
az monitor activity-log list -g web-rg --offset 2h --status Succeeded --query "[].{t:eventTimestamp,op:operationName.value,who:caller}" -o table
# → 09:38  Microsoft.Web/sites/config/write   svc-deploy@…    ← app settings changed
```
Verify: Activity log event → JSON → `properties` shows the setting that changed; App Service → Diagnose → **Application Changes** confirms it.

**Trap:** Activity log is control-plane only (ARM). Application stdout lives in diagnostic logs / App Insights / Log Analytics. Portal keeps 90 days; the wrong **directory + subscription filter** (top-right) is the #1 reason a resource "doesn't exist."

---

## 2. Identity, RBAC & Secrets — 401/403 Playbook

**Concept:** RBAC scope inherits Management group → Subscription → RG → Resource. **Control-plane** role (Contributor) ≠ **data-plane** role (`Storage Blob Data Contributor`). Deny assignments, resource locks, and Azure Policy can fail an operation IAM says is allowed. Role changes take **up to 10 minutes** to propagate.

```bash
az account show                                                          # wrong tenant is the #1 cause
az role assignment list --assignee <id> --all -o table
az role assignment list --scope <resource-id> --include-inherited -o table
az account get-access-token --resource https://storage.azure.com/        # token for the right audience?
az lock list -g RG -o table
```
**Portal:** Resource → **Access control (IAM)** → Role assignments + **Deny assignments** tab · Resource → **Locks** · Resource → **Identity** (managed identity on/off) then target resource → IAM → search the MI name · Microsoft Entra ID → **Sign-in logs** → failed sign-in → **Conditional Access** tab · Policy → Compliance.

| Check | Why it bites |
|---|---|
| Wrong tenant / subscription | Portal looks fine; script is elsewhere |
| RBAC vs data-plane | Contributor can't read blobs; Key Vault Access Policy vs RBAC is a second trap |
| Deny assignment / lock / Policy | `RequestDisallowedByPolicy` even as Owner |
| Managed identity | exists on the resource but has no role on the target; user-assigned MI not attached |
| Token audience | token for Graph used against ARM; guest clock skew breaks JWTs |
| Key / SAS / connection string | rotated key, expired SAS, string points at the wrong account |
| Conditional Access / MFA | works in portal, fails from automation or non-compliant device |

**Example — "Function app gets 403 reading a blob; its managed identity is enabled":**
```bash
az functionapp identity show -g RG -n fn --query principalId -o tsv          # → <guid>
az role assignment list --assignee <guid> --scope $(az storage account show -g RG -n sa --query id -o tsv) -o table
# → empty
az role assignment create --assignee <guid> --role "Storage Blob Data Reader" --scope <sa-id>
```
Verify: wait a few minutes, retry → 200. (Contributor on the RG would *not* have fixed this.)

**Trap:** Key Vault: choose **Access policies OR RBAC** in Access configuration; mixing them is a silent 403. Soft-deleted secrets return 404, not "deleted." "Allow trusted Microsoft services" doesn't cover every resource type.

---

## 3. Networking (Network Watcher)

**Concept:** NSGs apply at **subnet and NIC** — read *Effective security rules*, not the rule you remember writing. UDRs can send traffic to an NVA or to **None** (blackhole). Private endpoint + private DNS zone must be **linked to the VNet** or names resolve to the public IP. PaaS outbound dies on **SNAT port exhaustion**.

Decision tree — walk in order:
1. NIC / private endpoint / public IP exists? VM or app running?
2. DNS resolves to the IP you expect? Private DNS zone linked? Peering? Fallback 168.63.129.16?
3. Next hop: Internet, VNet, NVA, or **None**?
4. NSG on subnet **and** NIC + Azure Firewall/NVA + guest OS firewall — effective rules.
5. Destination listening? (`ss -lnt`, probe path, App Gateway backend health)
6. SNAT / outbound: intermittent timeouts from PaaS → Diagnose → SNAT Port Exhaustion.
7. Private endpoint + public access disabled without correct private DNS = timeout or 403.

```bash
az network nic list-effective-nsg -g RG -n NIC
az network watcher test-ip-flow -g RG --vm VM --direction Inbound --protocol TCP --local 10.0.0.4:22 --remote 203.0.113.5:50000
az network watcher show-next-hop -g RG --vm VM --source-ip 10.0.0.4 --dest-ip 8.8.8.8
az network watcher test-connectivity -g RG --source-resource VM --dest-address sql.example.com --dest-port 1433
az network vnet peering list -g RG --vnet-name V -o table          # both sides must be Connected
```
**Portal:** Network Watcher (one per **region** — pick the right one) → **IP flow verify** / **Next hop** / **Connection troubleshoot** / **NSG diagnostics** / **Packet capture** / **Flow logs** → Traffic Analytics · VM → Networking → NIC → **Effective security rules** · Private DNS zones → **Virtual network links**.

**Example — "App can't reach SQL after we added a private endpoint":**
```bash
az network private-endpoint show -g RG -n sql-pe --query 'customDnsConfigs[].ipAddresses'   # → 10.0.2.5
# from the app (Kudu console):  nameresolver sqlsrv.database.windows.net   → 40.x.x.x  ← public IP!
az network private-dns link vnet list -g RG -z privatelink.database.windows.net -o table    # → no link to app VNet
az network private-dns link vnet create -g RG -z privatelink.database.windows.net -n app-link -v app-vnet -e false
```
Verify: `nameresolver` now returns 10.0.2.5; `tcpping sqlsrv…:1433` succeeds.

**Trap:** Peering must show *Connected* on **both** VNets and CIDRs can't overlap. Connection troubleshoot from *another VM in the VNet* before you rebuild anything. Just-in-time VM access (Defender) silently closes 22/3389.

---

## 4. Virtual Machines

**Concept:** Resource Health tells you host vs guest. Boot diagnostics = screenshot + serial log. Serial console = live login when RDP/SSH is dead. Run command / Repair VM / Redeploy are the escalating fixes.

```bash
az vm get-instance-view -g RG -n VM --query instanceView.statuses -o table
az vm boot-diagnostics get-boot-log -g RG -n VM
az vm run-command invoke -g RG -n VM --command-id RunShellScript --scripts "ss -lnt; df -h"
az vm redeploy -g RG -n VM                       # new host; ephemeral public IP changes
az vm repair create -g RG -n VM                  # rescue VM with OS disk attached (az extension add -n vm-repair)
az vm list-skus --location eastus --size Standard_D4 -o table
```
**Portal:** VM → **Help** → Boot diagnostics · Serial console · Run command · Reset password · Redeploy · Repair VM · Reset NIC. VM → Networking (NSG on NIC + subnet). Subscription → **Usage + quotas**.

| Symptom | Check | Fix |
|---|---|---|
| Can't start / `AllocationFailed` | Activity log innermost error; quota; zone capacity | other size / zone / region; request quota |
| Won't boot | Boot diagnostics screenshot + serial log | serial console; snapshot OS disk → Repair VM → fix fstab/bcdedit → Swap OS disk |
| Can't RDP/SSH | power state, public IP, NSG NIC+subnet, JIT, guest firewall | Bastion; Connection troubleshoot; Reset NIC; Redeploy |
| Slow | CPU / memory / **disk IOPS vs disk SKU cap** / network | PerfInsights (Diagnose blade); bigger disk tier or VM size |
| `VMExtensionProvisioningError` | guest agent, outbound 443, CSE script | Extensions + applications → instance view |

**Example — VM stuck, "Unknown" Resource Health, no SSH:**
```bash
az vm get-instance-view -g RG -n VM --query "instanceView.statuses[].displayStatus"   # → "VM deallocated"
az vm start -g RG -n VM
```
Not broken — someone deallocated it (Activity log: `deallocate` by an auto-shutdown schedule). Verify: `PowerState/running`, Resource Health → Available.

**Trap:** Resource Health **Degraded** usually means host; in-guest counters mean workload. Redeploy changes the public IP unless it's static. Disk throttling shows as high latency with low CPU — check the disk SKU's IOPS limit, not the VM.

---

## 5. App Service & Functions

**Concept:** "Diagnose and solve problems" has dedicated detectors (Web App Down, Web App Slow, SNAT, CPU/Memory, Application Changes). Kudu/SCM (`https://<app>.scm.azurewebsites.net`) is the shell into the sandbox. Functions need their storage account reachable.

```bash
az webapp log tail -g RG -n app
az webapp config appsettings list -g RG -n app -o table
az webapp restart -g RG -n app
az webapp deployment list-publishing-profiles -g RG -n app      # SCM creds
```
**Portal:** App → **Diagnose and solve problems** → Web App Down / Slow / SNAT Port Exhaustion / Application Changes · Monitoring → **Log stream** · Development Tools → **Advanced Tools (Kudu)** → Debug console (`tcpping host:port`, `nameresolver host`) · Deployment Center → Logs · Application Insights → Failures / Dependencies.

| Symptom | Likely cause | Look |
|---|---|---|
| 503 / down | startup crash, bad app setting, plan exhausted | Web App Down detector; Log stream |
| 502 | worker crashed / cold start | restart; scale out; Always On |
| Slow / intermittent timeouts | SNAT exhaustion, dependency, CPU cap | SNAT detector; App Insights dependencies |
| Can't reach SQL/Storage | VNet integration, PE DNS, Route All, firewall | Network troubleshooter; Kudu `nameresolver` |
| Deploy fails | lock, RBAC, `WEBSITE_RUN_FROM_PACKAGE`, storage access | Deployment Center logs; Activity log |
| Cert / domain | expired cert, CNAME vs A, managed cert delay | SSL and Domains detector |
| Functions silent | trigger connection string, storage locked down, host.json | Function detectors; Environment variables |

**Example — random outbound timeouts to a third-party API, only under load:**
Portal: App → Diagnose → **SNAT Port Exhaustion** → shows port allocation failures at peak. Fix: reuse `HttpClient` (connection pooling), and/or attach a **NAT Gateway** to the integration subnet. Verify: detector shows zero SNAT failures; App Insights dependency duration normal.

**Trap:** Enable App Service logs and App Insights *before* the next outage — the blade can't detect what wasn't logged. Restart really does fix a lot. Linux apps: Development Tools → SSH.

---

## 6. Storage & Key Vault

**Concept:** Storage 403 has four buckets: (1) **identity** — key, SAS expiry, wrong data-plane role; (2) **firewall** — "Selected networks" missing your IP/VNet/resource instance; (3) **private endpoint + DNS** still resolving public; (4) **state** — soft-deleted, lease, immutability, failover. Read the HTTP status + `x-ms-error-code`.

```bash
az storage account show -g RG -n sa --query '{net:networkRuleSet.defaultAction,pub:publicNetworkAccess}'
az storage account network-rule list -g RG -n sa
az storage blob list --account-name sa -c container --auth-mode login -o table
az keyvault show -n kv --query '{rbac:properties.enableRbacAuthorization,net:properties.networkAcls.defaultAction}'
az keyvault secret list-deleted --vault-name kv
```
**Portal:** Storage → **Networking** (firewall, Private endpoint connections) · **Access keys** / **Shared access signature** · IAM (data-plane roles) · Monitoring → Metrics (Availability, E2E latency, Throttling). Key Vault → Settings → **Access configuration** (RBAC vs policy) · Networking · Objects → **Manage deleted** · Diagnostic settings → `AuditEvent`.

**Example — 403 from a VM to a storage account with "Selected networks":**
```bash
az storage account network-rule list -g RG -n sa --query virtualNetworkRules   # → VNet rule for app-subnet only
# VM is in db-subnet
az storage account network-rule add -g RG -n sa --vnet-name vnet --subnet db-subnet
```
Verify: `az storage blob list … --auth-mode login` from the VM returns rows. (Subnet also needs the `Microsoft.Storage` service endpoint, which `network-rule add` prompts for.)

**Trap:** 404 on a blob can mean **soft-deleted** — check Show deleted. Throttling returns **503** — fix with exponential backoff and key/partition spread, not a bigger account. Key Vault firewall + "trusted services" still blocks many resource types; use a private endpoint.

---

## 7. Azure SQL

**Concept:** Three walls: server-level firewall, VNet rules, private endpoint. Two auth modes: **Entra** vs **SQL auth**. DTU/vCore cap produces timeouts that look like network.

```bash
az sql server firewall-rule list -g RG -s sqlsrv -o table
az sql server firewall-rule create -g RG -s sqlsrv -n myip --start-ip-address <ip> --end-ip-address <ip>
az sql db show -g RG -s sqlsrv -n db --query '{sku:currentSku.name,status:status}'
az monitor metrics list --resource <db-id> --metric dtu_consumption_percent
```
**Portal:** SQL server → **Networking** (firewall, "Allow Azure services", private endpoint) · Database → **Connection strings** · **Query Performance Insight** · **Diagnose and solve problems** · Metrics (DTU %, deadlocks, workers).

**Example — "Login failed for user" from a new app, works from SSMS:**
The app connection string uses SQL auth for a user created on **master** only. Fix: `CREATE USER app FROM LOGIN app` inside the *target* database and grant a role. Verify: app connects; Entra sign-in logs stay clean (it was never Entra).

**Trap:** Firewall rules are per **server**; VNet rules need the `Microsoft.Sql` service endpoint on the subnet. SQL on VM: it's a VM problem first (disk IOPS, IaaS agent), then a SQL problem.

---

## 8. AKS

**Concept:** Cluster/node-pool provisioning failures show up in the Activity log of the managed cluster **and** the node-pool VMSS (`MC_` resource group). The classic cause is **blocked egress** — nodes can't reach the cluster API / MCR / ACR.

```bash
az aks show -g RG -n aks --query '{s:provisioningState,pools:agentPoolProfiles[].{n:name,s:provisioningState}}'
az aks get-credentials -g RG -n aks
kubectl get nodes ; kubectl get pods -A
kubectl describe pod <p> -n <ns>
kubectl logs <p> -n <ns> --previous
kubectl get events -n <ns> --sort-by='.lastTimestamp'
az aks check-acr -g RG -n aks --acr myacr.azurecr.io           # image pull auth + network test
```
**Portal:** AKS → **Resource health** · Activity log · **Diagnose and solve problems** (run before recreating anything) · **Kubernetes resources** (Nodes, Pods, Events, Workloads) · Insights (Container insights) · node pool → VMSS → Activity log / instance view.

| Symptom | Cause | Fix |
|---|---|---|
| Node pool Failed, `OutboundConnFailVMExtensionError` | egress blocked (NSG/firewall/UDR) | allow required outbound; `az aks update` to reconcile |
| `ImagePullBackOff` | ACR firewall / kubelet identity lacks AcrPull | `az aks check-acr`; attach ACR |
| `CrashLoopBackOff` | app crash, bad env | `describe` + `logs --previous` |
| `OOMKilled` | limits too low | raise memory limits |
| Pending | requests vs allocatable, taints, PVC zone, quota | events; scale; Usage + quotas |
| Cluster Failed | CSE, subnet full, private DNS, LB `InvalidResourceReference` | Activity log innermost error; Diagnose blade |

**Example — new node pool stuck Failed:**
```bash
az monitor activity-log list -g MC_RG_aks_eastus --offset 1h --status Failed --query "[].properties.statusMessage" -o tsv
# → OutboundConnFailVMExtensionError … could not reach mcr.microsoft.com:443
```
Fix: the subnet's UDR sends 0.0.0.0/0 to a firewall missing the AKS egress rules — add them. Verify: `az aks nodepool show … --query provisioningState` → Succeeded; `kubectl get nodes` Ready.

**Trap:** Don't recreate the cluster before running Diagnose and solve problems — most Failed states reconcile once the cause is fixed. Subnet IP exhaustion with Azure CNI is common and silent.

---

## 9. Deployments (ARM / Bicep / Terraform)

**Concept:** The outer error is generic; the **inner** error is the truth. Validate before deploying.

```bash
az deployment group show -g RG -n <deployment> --query properties.error
az deployment group validate -g RG --template-file main.bicep -p …
az deployment operation group list -g RG -n <deployment> --query "[?properties.provisioningState=='Failed'].properties.statusMessage"
```
**Portal:** Resource group → **Deployments** → failed deployment → Operation details → error JSON.

| Code | Means | Fix |
|---|---|---|
| `QuotaExceeded` / `OperationNotAllowed` | cores/IPs/disks/vCPU family | Usage + quotas → request |
| `SkuNotAvailable` | SKU retired or not in region/zone | `az vm list-skus --location …` |
| `RequestDisallowedByPolicy` | Azure Policy deny | Activity log → properties → policy definition link |
| `Conflict` / 409 / `AnotherOperationInProgress` | lock, concurrent op, disk lease | wait; check Locks; retry |
| `InvalidTemplate` | syntax / reference | `validate` first |
| `ParentResourceNotFound` | wrong RG/name/region or deleted dependency | fix resource ID |
| `InternalOperationError` | platform, often transient | retry; if persistent → Resource Health + support |

**Trap:** Same correlation ID on retry helps support trace it. A `CanNotDelete` lock fails deletes with a permissions-looking error even for Owner.

---

## 10. Monitoring Baseline

| Tool | Use it for | Latency |
|---|---|---|
| Azure Monitor **Metrics** | CPU, requests, errors, throttling | near-instant |
| **Log Analytics (KQL)** | deep cross-resource queries | minutes of ingestion delay |
| **Application Insights** | traces, dependencies, exceptions, failed requests | minutes |
| **Diagnostic settings** | must be enabled **per resource** to ship logs | — |
| Resource Health / Service Health **alerts** | platform events → Action Group | — |
| NSG / VNet **flow logs** | allow/deny after the fact → Traffic Analytics | — |

```kusto
AzureDiagnostics
| where TimeGenerated > ago(1h) and Level == "Error"
| summarize count() by ResourceProvider, Resource

requests
| where timestamp > ago(1h) and success == false
| summarize count() by resultCode, name
```

---

## 11. Error Codes on Repeat

| Code | Means | First move |
|---|---|---|
| `AuthorizationFailed` / 403 | missing RBAC, deny assignment, data-plane role | IAM at exact scope + Deny assignments |
| `InvalidAuthenticationToken` / `AADSTS*` | expired, wrong audience/tenant, CA block | `az account get-access-token`; Entra sign-in logs |
| `AllocationFailed` / `ZonalAllocationFailed` | no capacity for size/zone | other size/zone/region |
| `QuotaExceeded` | subscription quota | Usage + quotas |
| `RequestDisallowedByPolicy` | Azure Policy | Activity log → policy definition |
| `PublicNetworkAccessDenied` / 403 (data) | firewall / PE / public access off | Networking blade + private DNS |
| `SkuNotAvailable` / `SubscriptionNotFound` | SKU retired or wrong context | `az account show`; `list-skus` |
| `VMExtensionProvisioningError` | guest agent / outbound 443 / CSE | Extensions instance view |
| SNAT port exhaustion | too many outbound from PaaS | NAT Gateway; connection reuse |
| `Conflict` / 409 | lock, concurrent op, lease | Locks; wait; retry |
| `InternalOperationError` | platform-side | retry; Resource Health; support |

---

## 12. Escalation Packet

Open a request when Resource Health says platform-unavailable, it's Sev-A, a detector says "contact support," or layers 1–5 still look like platform (persistent `InternalOperationError`, allocation failing in every zone, control-plane 500s). Portal: Resource → **Resource health → New support request** (no paid plan needed for platform issues) or Help + support → Create a support request. Copilot's Troubleshooting agent can pre-fill it.

Attach: subscription ID · resource ID · region · UTC timestamps · **correlation ID(s)** · Resource Health screenshot + Service Health incident ID · Diagnose detector output · innermost error code + full JSON · what you already tried · impact and workaround.

---

## Quick Wins Checklist

- Right tenant / subscription? (`az account show`, top-right directory filter)
- Did a cert, key, SAS, or secret expire?
- NSG changed recently — at **both** subnet and NIC?
- Quota or throttling (429/503)?
- Restart it. Then Redeploy to a fresh host if state looks corrupt.
- Role added in the last 10 minutes? Wait, then retry.

## Pocket Checklist

Status page → Service Health → Resource Health → Activity log (Failed + correlation ID) → Diagnose and solve problems → Network Watcher / IAM / firewall → guest or app logs → support packet.

**Portal chrome:** global search accepts resource names and IDs · **Directories + subscriptions** filter top-right · resource left menu: Overview, Activity log, IAM, Diagnose and solve problems, Resource health, Networking · pin Service Health, Monitor, Network Watcher, VMs, App Services, Key vaults, Help + support.
