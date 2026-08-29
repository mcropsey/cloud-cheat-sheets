# Azure Study Guide — Working in the Portal

A practical, portal-first guide, mirroring the AWS guide. Each section covers: **what it is**, **where it lives in the Portal**, **what you actually do**, and **what breaks**.

---

## 0. Portal Basics (read first)

- **Hierarchy:** Tenant (Entra ID directory) → **Subscription** (billing boundary) → **Resource Group** (folder for related resources) → Resource. Every resource lives in exactly one resource group. Delete the group, delete everything in it — great for labs, dangerous in prod.
- **Region** is chosen per resource (not a portal-wide switch like AWS). Resources in a resource group can be in different regions, but keep them together for sanity.
- **Search bar** (top): type the resource type or name. `Ctrl+/` focuses it.
- **RBAC** (Identity & Access): roles are assigned at a **scope** (subscription, RG, or resource) and inherit downward. "You do not have authorization" = RBAC, not the service. Common roles: Owner, Contributor, Reader, Network Contributor, Virtual Machine Contributor.
- **Entra ID** (formerly Azure AD): users, groups, service principals, **managed identities**. Separate from RBAC — Entra says *who you are*, RBAC says *what you can do on resources*.
- **Resource IDs** are long paths: `/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{name}`. Errors reference these. The provider (`Microsoft.Compute`, `Microsoft.Network`) tells you which service.
- **Tags**: key/value on any resource; apply at RG level and use Azure Policy to inherit.
- **Cloud Shell** (`>_` icon at top): browser terminal with `az` CLI and PowerShell already logged in. Use it constantly.
- **Activity Log** on every resource: who changed what, when, and any deployment errors.

### AWS → Azure name map

| AWS | Azure |
|---|---|
| Account | Subscription |
| Region / AZ | Region / Availability Zone |
| VPC | Virtual Network (VNet) |
| Subnet | Subnet |
| Security Group | Network Security Group (NSG) + Application Security Group (ASG) |
| NACL | (no direct equivalent — NSGs attach to subnets too) |
| Route table | Route table (UDR — user-defined routes) |
| Internet Gateway | Built in — every VNet can reach the internet by default |
| NAT Gateway | NAT Gateway |
| Elastic IP | Public IP address (Static SKU) |
| ALB / NLB | Application Gateway / Load Balancer |
| EC2 | Virtual Machine |
| AMI | Image (Marketplace, Compute Gallery, or managed image) |
| Auto Scaling Group | Virtual Machine Scale Set (VMSS) |
| EBS / S3 / EFS | Managed Disk / Blob Storage / Azure Files |
| ECR / ECS / EKS | Container Registry (ACR) / Container Apps or ACI / AKS |
| API Gateway | API Management (APIM) |
| WAF | WAF on Application Gateway or Front Door |
| IAM role for EC2 | Managed Identity |
| CloudFormation | ARM templates / Bicep |
| CloudWatch | Azure Monitor + Log Analytics |
| CloudTrail | Activity Log |
| Session Manager | Azure Bastion / Run Command / Serial Console |
| VPC Endpoint | Private Endpoint / Service Endpoint |
| Peering / Transit GW | VNet Peering / Virtual WAN |

---

## 1. Networking — Virtual Networks

### Mental model
A **VNet** is your private network (address space like `10.0.0.0/16`) with **subnets**. Unlike AWS, **outbound internet works by default** (via a shared SNAT unless you add a NAT Gateway or Load Balancer — note Microsoft is phasing out default outbound access for new VNets, so plan on adding a NAT Gateway). Inbound is controlled by **public IPs** on the resource and **NSGs**.

```
Internet
   │
[Public IP] ── on a VM NIC, Load Balancer, App Gateway, or Bastion
   │
VNet 10.0.0.0/16
 ├─ AppGatewaySubnet   (Application Gateway / WAF)
 ├─ AzureBastionSubnet (Bastion — exact name required)
 ├─ web-subnet         (VMs, NSG allows 80/443 from AppGW)
 ├─ app-subnet         (NSG allows app port from web-subnet)
 └─ data-subnet        (Private Endpoints to SQL/Storage)
        │
   [NAT Gateway] attached to subnets for predictable outbound IP
```

### Key components

| Component | What it does | Portal location |
|---|---|---|
| Virtual Network | Address space + subnets | Virtual networks |
| Subnet | Slice of the VNet; NSGs, route tables, NAT GW attach here | VNet → Subnets |
| Network Security Group | Stateful allow/deny rules, on subnet and/or NIC | Network security groups |
| Application Security Group | Named group of NICs you can reference in NSG rules | Application security groups |
| Route table (UDR) | Override default routing (e.g. force traffic through a firewall) | Route tables |
| NAT Gateway | Static outbound IP(s), scalable SNAT | NAT gateways |
| Public IP address | Static or dynamic public IPv4/v6 | Public IP addresses |
| Private Endpoint | Private IP in your subnet for a PaaS service (Storage, SQL, Key Vault) | Private endpoints |
| Service Endpoint | Routes PaaS traffic over the backbone; service still has a public IP | VNet → Subnets → Service endpoints |
| VNet Peering | Connect two VNets (same or different region) | VNet → Peerings |
| VPN Gateway / ExpressRoute | On-prem connectivity | Virtual network gateways |
| Azure Firewall | Managed L3–L7 firewall, hub-and-spoke | Firewalls |
| Bastion | Browser RDP/SSH with no public IP on the VM | Bastions |

### Subnet gotchas
- Azure reserves **5 IPs per subnet** (first 4 + last). A `/29` gives you 3 usable addresses.
- Some services require **dedicated, exact-name subnets**: `AzureBastionSubnet` (/26 min), `GatewaySubnet` (/27 min), `AzureFirewallSubnet` (/26), and Application Gateway wants its own subnet.
- **Subnet delegation** hands a subnet to a service (Container Apps, App Service VNet integration, Flexible Server DBs). A delegated subnet can't host VMs.
- Can't resize a subnet with resources in it.

### Network Security Groups (NSG)

- **Stateful**, like AWS SGs. Reply traffic auto-allowed.
- Rules have a **priority (100–4096, lower wins)**, direction, source/destination (IP, CIDR, service tag, ASG), port, Allow/Deny.
- Attach to a **subnet**, a **NIC**, or both. Both are evaluated: inbound = subnet NSG then NIC NSG; outbound = NIC then subnet. A Deny in either blocks.
- **Default rules** (priority 65000+): allow VNet-to-VNet, allow Azure Load Balancer inbound, allow all outbound, deny everything else inbound. You can't delete them, only override with lower numbers.
- **Service tags** replace IP lists: `Internet`, `VirtualNetwork`, `AzureLoadBalancer`, `Storage`, `Sql`, `AzureCloud`, `GatewayManager`.
- **Application Security Groups**: tag NICs as `asg-web`, `asg-db`; write a rule "allow 1433 from asg-web to asg-db". Equivalent to AWS's SG-referencing-SG.
- **Effective security rules** (NIC → Networking → Effective security rules) shows the merged result — first stop when debugging.
- Rule changes apply within seconds, no reboot.
- Portal creates NSGs per NIC by default when you create a VM — you end up with dozens of `vmname-nsg`. Prefer subnet-level NSGs plus ASGs.

### Public IPs
- SKU **Standard** (zone-redundant, static, secure by default — needs an NSG to allow inbound) vs **Basic** (retiring). Use Standard.
- **Static** allocation = your Elastic IP equivalent. Survives deallocate/start. Dynamic IPs change on deallocate.
- Attach to: VM NIC (NIC → IP configurations), Load Balancer frontend, App Gateway, NAT Gateway, Bastion, VPN Gateway.
- Charged per hour while it exists, associated or not. Delete unused ones.

### NAT Gateway
- Create → assign one or more static **Public IPs** or a prefix → **associate with subnets** (NAT gateway → Subnets tab).
- Once attached, all outbound from those subnets uses those IPs (good for allow-listing at a partner). Takes precedence over Load Balancer outbound rules and default SNAT.
- Zonal — one NAT GW serves subnets in one VNet; it can't span VNets.
- No inbound. Not a firewall.

### Routing
- System routes: VNet, peered VNets, and `0.0.0.0/0 → Internet` exist automatically.
- **User-defined route table**: create → add routes (`0.0.0.0/0 → Virtual appliance 10.0.1.4`, i.e. the firewall) → associate with subnets.
- **Effective routes** (NIC → Networking → Effective routes) shows what a VM actually uses. Second stop when debugging.
- Peering is **not transitive**: A↔B and B↔C doesn't give A↔C. Use hub-and-spoke with a firewall/NVA, or Virtual WAN.

### Load balancing options

| Service | Layer | Use for |
|---|---|---|
| **Load Balancer** | L4 TCP/UDP | Regional, VM/VMSS backends, internal or public. Like NLB |
| **Application Gateway** | L7 HTTP/S | Path/host routing, TLS termination, **WAF**. Like ALB. Regional |
| **Front Door** | L7 global | Global anycast CDN + LB + WAF across regions |
| **Traffic Manager** | DNS | Geographic/priority DNS routing |

**Application Gateway structure:** Frontend IP → **Listener** (port, hostname, cert) → **Routing rule** → **Backend pool** (VMs, IPs, FQDNs, VMSS, App Service) + **Backend settings** (port, protocol, timeout) + **Health probe**. Needs its own subnet, and the backend NSG must allow traffic from the App Gateway subnet, plus the `GatewayManager` service tag on ports 65200–65535 inbound on the AppGW's subnet NSG (v2 SKU).

Backend health: App Gateway → **Backend health** blade. Unhealthy = 502 to users.

---

## 2. Virtual Machines

### Creating (Virtual machines → Create → Azure virtual machine)
1. **Basics:** subscription, resource group, name, region, **availability options** (zone / availability set / none), **image** (Ubuntu, Windows Server, etc.), **size** (`B2s`/`B1s` for testing; B = burstable, D = general, E = memory, F = compute, N = GPU), **auth** (SSH key or password), inbound ports (this creates the NIC NSG).
2. **Disks:** OS disk type (Standard SSD for test, Premium SSD for prod). Add data disks here or later.
3. **Networking:** VNet, subnet, **public IP (set to None for private VMs)**, NIC NSG, load balancing.
4. **Management:** auto-shutdown (turn on for labs), backup, patching, **system-assigned managed identity** (turn on).
5. **Advanced:** **custom data / user data** (cloud-init or script), extensions.
6. **Review + create** → it validates first. Validation failures show here before anything is built.

Downloading the SSH private key: Azure generates it and prompts you to download **once**. Or paste your own public key.

### Connecting
- **Bastion** (VM → Connect → Bastion): browser RDP/SSH, VM needs no public IP. Requires `AzureBastionSubnet` in the VNet. Best practice; costs per hour.
- **SSH/RDP via public IP**: NSG must allow 22/3389 from your IP (VM → Networking → add inbound rule → Source: My IP address).
- **Run Command** (VM → Operations → Run command): execute a script inside the VM through the Azure fabric, no network needed. Lifesaver for fixing sshd/firewall.
- **Serial Console** (VM → Help → Serial console): text console for boot problems. Requires boot diagnostics enabled.
- **Windows:** RDP from VM → Connect → Native RDP downloads an `.rdp` file.

### Managed Identity (the "IAM role for EC2")
VM → **Identity** → System assigned → On. Then give that identity RBAC on the target (e.g. Storage Blob Data Reader on a storage account, or Key Vault Secrets User). Code uses `DefaultAzureCredential` — no keys anywhere. **User-assigned** identities are reusable across many resources.

### VM lifecycle
- **Stop (from OS)** = still billed, "Stopped" state. **Stop (from Portal)** = **Deallocated**, not billed for compute, releases dynamic IP, may move host.
- **Restart**: same host.
- **Delete**: by default the OS disk, NIC, and public IP are **kept** (checkboxes at delete time / VM creation). Orphaned disks and NICs cost money — clean up.
- **Resize**: VM → Size. Requires deallocate for most changes.
- **Extensions** (VM → Extensions + applications): Custom Script Extension, Azure Monitor Agent, Entra login, etc.

### Scale and availability
- **Availability Zone**: pin VM to zone 1/2/3.
- **Availability Set**: spread across fault/update domains in one datacenter (older pattern).
- **Virtual Machine Scale Set (VMSS)**: image + size + instance count + autoscale rules + load balancer. Flexible orchestration is the current default.
- **Boot diagnostics** (VM → Help → Boot diagnostics): screenshot + serial log. Enable it.

---

## 3. Storage

Almost everything goes through a **Storage Account** (Blob, Files, Queues, Tables) except **Managed Disks**, which are their own resource type.

| Service | Type | Use it for | AWS equivalent |
|---|---|---|---|
| **Managed Disks** | Block | VM OS and data disks | EBS |
| **Blob Storage** | Object | Files, backups, static sites, logs, data lake | S3 |
| **Azure Files** | SMB/NFS file share | Shared drives, lift-and-shift, container volumes | EFS |
| **Queue / Table** | Messaging / NoSQL key-value | Simple apps | SQS / DynamoDB-lite |

### Managed Disks
- Types: **Standard HDD, Standard SSD, Premium SSD, Premium SSD v2, Ultra**. Premium SSD is the normal prod choice; Premium requires a size that supports it (most D/E/F "s" sizes, e.g. `D2s_v5`).
- Attach: VM → Disks → Attach new/existing → save. Then in the OS: `lsblk`, partition, `mkfs`, mount, `/etc/fstab` (use UUID). Windows: Disk Management → Initialize → New volume.
- Resize: deallocate VM (or use online resize for supported types) → Disks → disk → Size + performance → then extend the partition in the OS.
- **Snapshots**: Disks → disk → Create snapshot. Create a new disk from a snapshot, even in another zone/region.
- **Azure Backup**: VM → Backup → policy. Simpler than DIY snapshots.
- Disks are zonal; to move zones, snapshot → create disk in target zone.

### Storage Accounts
- Name is **globally unique**, lowercase, 3–24 chars, no dashes.
- **Redundancy**: LRS (3 copies one DC) → ZRS (across zones) → GRS/GZRS (paired region). Start with LRS for labs, ZRS/GRS for prod.
- **Access tiers** for blobs: Hot → Cool → Cold → Archive. **Lifecycle management** rules move/delete automatically.
- **Auth**: prefer Entra ID + RBAC (`Storage Blob Data Contributor`) over **account keys**. **SAS tokens** = presigned URLs (time-limited, scoped).
- **Networking tab**: Public access from all networks / selected VNets (service endpoint) / disabled + **Private Endpoint**. If you flip this and suddenly get 403 from the Portal, it's because your own IP is now blocked — add it to the firewall or the "allow trusted services" exception.
- **Soft delete** and **versioning** for blobs — enable on anything important.
- **Static website** hosting for blobs; put Front Door/CDN in front.
- Blob URL format: `https://<account>.blob.core.windows.net/<container>/<blob>`.

### Azure Files
- Create a **file share** in a storage account (SMB by default; NFS requires Premium FileStorage account).
- Mount via the **Connect** button — it generates the exact script for Windows/Linux/macOS.
- SMB needs port **445** outbound; many ISPs block it. Use a VPN or Private Endpoint. Entra/Kerberos auth is available for domain-joined VMs.

---

## 4. Containers

### Azure Container Registry (ACR)
- Create registry (name globally unique). SKUs Basic/Standard/Premium (Premium adds private endpoints, geo-replication).
- Push: `az acr login --name <registry>` then `docker tag app <registry>.azurecr.io/app:v1` and `docker push`. Or `az acr build` builds in the cloud with no local Docker.
- Pull access: **managed identity + `AcrPull` role** (preferred), or the **admin user** (Access keys blade — fine for labs, avoid in prod).
- Repositories blade shows images and tags; can enable **tag immutability** and vulnerability scanning via Defender.

### Choosing a container service

| Service | When |
|---|---|
| **Container Apps** | Default for web apps/APIs/microservices. Serverless, scale-to-zero, revisions, ingress, Dapr. Like ECS Fargate + a bit more |
| **Container Instances (ACI)** | Run one container/group on demand, no orchestrator. Jobs, quick tests. Like a single Fargate task |
| **App Service (containers)** | Web app with custom container on the App Service platform |
| **AKS** | Full Kubernetes. When you need it, you know |

### Container Apps
- **Environment** = the isolated boundary (its own VNet subnet if you want, delegated `/23` minimum for workload profiles, `/27` for consumption). Apps in the same environment can talk by name.
- **Container App**: image (from ACR with managed identity), CPU/memory, **environment variables/secrets**, **ingress** (external/internal, target port), **scale rules** (HTTP concurrency, CPU, KEDA triggers), **revisions** (each deploy = new revision; split traffic between them).
- Logs: Container App → **Log stream** (live) or **Logs** (Log Analytics, table `ContainerAppConsoleLogs_CL` / `ContainerAppSystemLogs_CL`).
- Common failures:
  - Revision stuck **Provisioning/Failed** → image pull error (wrong tag, missing `AcrPull` on the identity, registry public access disabled).
  - App runs but 404/502 from ingress → **target port** doesn't match what the container listens on, or app binds to `127.0.0.1`.
  - Scale to zero cold start → set min replicas to 1.

### AKS
- Create: cluster → **node pools** (system + user), VM size, autoscaling, **networking** (Azure CNI overlay is the current default; Kubenet is legacy), integrate with ACR at creation (attaches `AcrPull`).
- Get credentials: `az aks get-credentials -g <rg> -n <cluster>` in Cloud Shell, then `kubectl`.
- Portal shows Workloads, Services/Ingress, Nodes, and lets you apply YAML, but it's a viewer — use `kubectl`.
- **Entra integration + Azure RBAC for Kubernetes** replaces managing kubeconfig secrets. "Forbidden" = your Entra user lacks an Azure Kubernetes RBAC role.
- Ingress: **Application Routing add-on** (managed NGINX) or **Application Gateway for Containers**.
- Node issues: Node pools → **Upgrade/scale**; nodes are a VMSS in the managed `MC_*` resource group — don't edit that RG by hand.

---

## 5. API Management (APIM)

The API Gateway equivalent. Heavier than AWS API Gateway; it's a full gateway + developer portal + product/subscription model.

**Tiers:** Consumption (serverless, pay-per-call — start here), Basic v2 / Standard v2 (fast-provisioning), Developer (non-prod, all features), Premium (VNet, multi-region). Classic tiers take **30–60 minutes** to provision.

**Structure:** APIM instance → **APIs** (import from OpenAPI, Function App, Container App, App Service, or blank) → **Operations** (GET /users) → **Policies** (XML applied inbound/backend/outbound/on-error) → **Products** (bundle of APIs) → **Subscriptions** (keys) → **Developer portal**.

**Backend**: the real service URL. Set at API level (Settings → Web service URL) or as a named **Backend** resource.

**Policies** do the heavy lifting: `rate-limit`, `validate-jwt`, `set-header`, `rewrite-uri`, `cors`, `cache-lookup`, `mock-response`. Edit in the API → Design → Inbound processing → `</>` code view. Test in the **Test** tab — it shows the full trace including which policy ran and what the backend returned.

**Gotchas**
- 401 `Access denied due to missing subscription key`: the API/product requires a subscription key. Pass `Ocp-Apim-Subscription-Key` header or untick **Subscription required** on the API settings.
- 404 `Resource not found`: API URL suffix + operation template doesn't match. Check the full URL in the Test tab.
- 500/502 from backend: use **Test → Trace** to see the actual backend response; often the backend URL is wrong or a private backend isn't reachable (needs VNet-integrated tier or a public backend).
- CORS: add the `cors` policy at **All APIs** level, list allowed origins.
- Reaching private backends requires Developer/Premium (VNet injection) or Standard v2 (VNet integration).
- Simpler alternative for a single Function App: Functions have built-in HTTP triggers with keys — you may not need APIM at all.

---

## 6. WAF

Azure WAF is not standalone; it's a feature of **Application Gateway (regional)** or **Front Door (global)**.

**Structure:** **WAF Policy** (a separate resource) → rules → associate with an App Gateway (whole gateway, a listener, or a route) or Front Door profile.

**Rule types**
- **Managed rule sets**: OWASP CRS 3.2 or the newer **Microsoft Default Rule Set (DRS 2.1)** for Front Door; **Bot Manager** rule set. Start here.
- **Custom rules**: match on IP/CIDR, geo, headers, query, body, request URI, size; actions Allow/Block/Log; priority order.
- **Rate limit** custom rules: N requests per 1 or 5 minutes per client IP.
- **Exclusions**: exclude a specific header/cookie/arg from specific rules (fixes false positives without disabling the rule).

**Workflow**
1. WAF policies → Create → choose **Regional (Application Gateway)** or **Global (Front Door)**.
2. Policy settings: mode **Detection** first, then **Prevention**.
3. Managed rules tab: enable rule set, optionally lower the **anomaly score threshold** (CRS 3.2 uses scoring; a request blocks at score ≥ 5 by default).
4. Associate with the App Gateway / Front Door.
5. Turn on **diagnostic settings** → Log Analytics → query `AzureDiagnostics | where Category == "ApplicationGatewayFirewallLog"` to see what matched.
6. Tune with exclusions or rule disables, then switch to Prevention.

**Gotchas**
- App Gateway must be **WAF_v2** SKU to attach a policy; you can't add WAF to a Standard_v2 gateway without recreating it.
- Default max request body is 128 KB and file upload limit 100 MB — large uploads get blocked until you raise them in policy settings.
- Common false positives: rule 942xxx (SQLi) on JSON bodies with quotes, 920xxx protocol rules on unusual headers. Use exclusions, not blanket disables.
- WAF on Front Door doesn't protect the origin if someone hits it directly — lock the origin (App Gateway/App Service) to the `AzureFrontDoor.Backend` service tag and validate the `X-Azure-FDID` header.

---

## 7. ARM Templates & Bicep — Deployments and Troubleshooting

### Basics
- **ARM template** = JSON declaring resources. **Bicep** = cleaner DSL that compiles to ARM; **use Bicep** for anything you write by hand. Both deploy the same way, and the errors are identical.
- **Deployment** is scoped to a **resource group** (usual), subscription, management group, or tenant.
- Modes: **Incremental** (default — adds/updates, leaves other resources alone) vs **Complete** (deletes resources in the RG not in the template — dangerous).
- Portal paths: **Deploy a custom template** (search that phrase) → Build your own / load file → fill parameters → Review + create. Or Resource group → **Deployments** to see history, inputs, outputs, and errors of every past deployment (including ones the Portal itself did when you clicked Create).
- **Export template** on any resource/RG gives you ARM JSON of what exists — verbose but a great starting point.
- Bicep structure: `param`, `var`, `resource <symbolicName> '<type>@<apiVersion>' = { ... }`, `module`, `output`. Reference with `symbolicName.id` / `symbolicName.properties.x`; dependencies are implicit when you reference another resource.

### Where to find the error
1. Resource group → **Deployments** → click the failed deployment.
2. **Operation details** / the red error link → shows the failing resource and an **error code + message**, often with an inner error that's the real one.
3. Also check the **Activity Log** filtered by Failed for the same timestamp.
4. In CLI: `az deployment group show -g <rg> -n <name> --query properties.error` or `az deployment operation group list`.

### Deployment states
| State | Meaning |
|---|---|
| Running | Working |
| Succeeded | Good |
| **Failed** | One or more resources failed. **Nothing is rolled back** — successful resources stay (unlike CloudFormation). Fix and redeploy; incremental mode makes redeploy safe |
| Canceled | Stopped by a user or a dependent failure |

Because there's no rollback, you can end up with a **half-built** environment. Redeploying the same template is the fix — ARM is idempotent for most resources.

### Common error codes and fixes
| Error code / message | Cause | Fix |
|---|---|---|
| `AuthorizationFailed` | Your identity lacks RBAC at that scope, or the deployment identity does | Assign Contributor (or the specific role) on the RG/subscription |
| `InvalidTemplate` / `InvalidTemplateDeployment` | Syntax, bad expression, wrong parameter type | Read the inner message — it gives line/position for ARM, or the Bicep linter flags it in VS Code before you deploy |
| `ResourceNotFound` | Referencing a resource that doesn't exist (typo, wrong RG, wrong API version, or it's being created in the same deployment without a dependency) | Fix name; add `dependsOn` or reference the symbolic name so Bicep infers it |
| `NoRegisteredProviderFound` / `MissingSubscriptionRegistration` | Resource provider not registered in the subscription | Subscription → Resource providers → Register (e.g. `Microsoft.ContainerService`) |
| `SkuNotAvailable` / `ZonalAllocationFailed` / `AllocationFailed` | VM size not available in that region/zone, or capacity exhausted | Change size, zone, or region; check `az vm list-skus` |
| `QuotaExceeded` / `OperationNotAllowed ... exceeding approved ... quota` | vCPU/core or resource quota | Subscription → Usage + quotas → request increase |
| `StorageAccountAlreadyTaken` / `DnsRecordInUse` / `already exists` | Globally unique name collision | Use `uniqueString(resourceGroup().id)` in the name |
| `InvalidResourceLocation` | Resource region doesn't match a required location (e.g. must match RG or parent) | Align locations |
| `PropertyChangeNotAllowed` | Trying to change an immutable property in place (VM zone, subnet address on a used subnet, storage kind) | Delete/recreate or don't change it |
| `SubnetIsFull` / `InUseSubnetCannotBeDeleted` / `InUseNetworkSecurityGroupCannotBeDeleted` | Networking resource still referenced | Remove the reference first |
| `DeploymentQuotaExceeded` (800 deployments per RG) | Deployment history full | Delete old deployments (RG → Deployments → select → delete) or enable auto-cleanup |
| `RequestDisallowedByPolicy` | Azure Policy blocks it (region, SKU, missing tag, public IP not allowed) | Read the policy name in the message; comply or get an exemption |
| `Conflict` / `AnotherOperationInProgress` | Resource busy from a previous operation | Wait, retry |
| `LinkedInvalidPropertyId` / `BadRequest` on a `resourceId()` | Malformed resource ID expression | Check the segments: type, name, and that it's the right RG |
| `DeploymentCanceled` on a resource | Not the real error — another resource failed first | Find the actual `Failed` operation |

### Other useful concepts
- **What-if** (`az deployment group what-if`): previews create/modify/delete before you deploy. Equivalent of a CloudFormation change set. Use it in prod.
- **Deployment stacks** (`az stack group create`): groups resources so deleting the stack deletes them, and can deny manual changes — closest thing to a CFN stack.
- **Azure Policy**: the guardrail layer; if deployments fail mysteriously, check Policy → Compliance.
- **Resource locks** (`CanNotDelete`, `ReadOnly`): a lock on an RG will fail deletions/updates with `ScopeLocked`.
- **Template specs**: publish a versioned template for others to deploy from the Portal.
- **Terraform** works fine on Azure too; error messages come from the same ARM API, so this table still applies.

---

## 8. Troubleshooting Cheat Sheet

### "I can't SSH / RDP to my VM"
1. VM **running** (not deallocated)? Check Overview.
2. Does the NIC have a **public IP**? (VM → Networking.) If not, use Bastion.
3. **Effective security rules** on the NIC: is 22/3389 allowed from your IP, and is there a subnet-level NSG denying it?
4. OS-level firewall (ufw / Windows Firewall)? Use **Run Command** to check/fix.
5. Wrong username/key? Reset via VM → Help → **Reset password** (works for SSH keys too).
6. VM won't boot → Boot diagnostics screenshot + **Serial Console**.
7. **Connection troubleshoot** (VM → Help → Connection troubleshoot / Network Watcher) tests the exact path and names the blocking NSG rule.

### "VM has no internet / can't reach a public endpoint"
1. NSG outbound Deny rule? (Default allows all.) Check effective rules.
2. **Effective routes**: is `0.0.0.0/0` pointing at a firewall/NVA (UDR) that's dropping it?
3. New VNet with default outbound access disabled → add a NAT Gateway or a public IP/Load Balancer outbound rule.
4. Azure Firewall in the path → check its rule collections and logs.
5. DNS: VM → Networking → DNS servers; custom DNS that's down breaks everything.

### "Application Gateway returns 502 / backend unhealthy"
1. App Gateway → **Backend health** → read the reason.
2. Backend NSG allows the backend port from the **App Gateway subnet**.
3. App Gateway subnet NSG allows `GatewayManager` 65200–65535 and `AzureLoadBalancer` inbound (v2 requirement).
4. Health probe path returns 200–399; hostname/`Pick host name from backend target` set right for App Service backends.
5. Backend cert trusted if using HTTPS to backend (or upload the root cert).
6. Backend settings timeout (default 20 s) — raise for slow APIs (504).

### "Containers keep failing"
Container Apps: Revision → System logs → look for image pull / probe failures. Check managed identity has `AcrPull`, target port, and health probes. AKS: `kubectl describe pod` → Events (ImagePullBackOff, CrashLoopBackOff), `kubectl logs`.

### "AuthorizationFailed / You do not have permission"
- Which identity? Your user, a managed identity, or a service principal.
- Resource → **Access control (IAM)** → **Check access** → type the identity → see effective roles at that scope.
- Data-plane vs control-plane: `Contributor` lets you manage a storage account but **not read blobs** — you need `Storage Blob Data Reader` (data roles). Same for Key Vault (`Key Vault Secrets User`) and Service Bus.
- Deny assignments / Azure Policy / locks can block even Owners.
- Role assignments take up to ~10 minutes to propagate.

### "Two resources in Azure can't talk"
1. Same VNet or **peered** (both directions show Connected)? Peering isn't transitive.
2. NSGs on **both** NICs and **both** subnets. Use **Effective security rules**.
3. UDRs sending traffic somewhere unexpected — Effective routes.
4. **Network Watcher → Connection troubleshoot / NSG diagnostics** — tells you exactly which rule blocks.
5. For PaaS (Storage/SQL/Key Vault): is the service's firewall set to selected networks? Private Endpoint DNS resolving to the private IP? (`nslookup <account>.blob.core.windows.net` should return `10.x` from inside the VNet; if it returns a public IP, the **Private DNS zone** isn't linked to the VNet.)
6. **NSG flow logs / VNet flow logs** → Log Analytics for accept/deny per flow.

### "Deployment failed"
See Section 7. RG → Deployments → error code → inner message. Redeploy after fixing (no rollback to clean up).

### "Where do I find logs?"
- Everything → **Azure Monitor** + **Log Analytics workspace** (KQL queries). Enable **Diagnostic settings** on each resource to send logs there — it's off by default for most PaaS.
- VM guest OS → Azure Monitor Agent + Data Collection Rule → `Syslog` / `Event` tables.
- Who did what → **Activity Log** (control plane, 90 days).
- Network → NSG/VNet flow logs, Network Watcher.
- App Gateway → `AzureDiagnostics` access/firewall/performance logs.
- APIM → Application Insights + gateway logs.
- Container Apps / AKS → Log Analytics (`ContainerAppConsoleLogs_CL`, `ContainerLogV2`).
- Cost surprises → **Cost Management → Cost analysis** grouped by resource.

---

## 9. Security Quick Rules

- Enable **MFA** and **Conditional Access**; no standing Owner assignments — use **PIM** for elevation.
- **Managed identities** everywhere; never store keys/secrets in config. Put secrets in **Key Vault** and reference them.
- Assign RBAC at **resource group** scope, to **groups**, not individual users.
- No public IPs on VMs; **Bastion** for admin access, **Private Endpoints** for PaaS, disable public network access on storage/SQL/Key Vault.
- Subnet-level NSGs with ASGs; deny-by-default inbound from `Internet`.
- Turn on **Microsoft Defender for Cloud** (free tier at least) and look at Secure Score weekly.
- Resource **locks** on production RGs; **Azure Policy** for allowed regions, required tags, no public IPs.
- **Auto-shutdown** on dev VMs and budgets/alerts in Cost Management.

---

## 10. Self-Check — can you do these in the Portal?

- [ ] Create a resource group, VNet with three subnets, a subnet-level NSG with ASG-based rules, and explain the default NSG rules.
- [ ] Deploy a Linux VM with no public IP, system-assigned identity, and connect via Bastion and via Run Command.
- [ ] Attach a Static Public IP to a VM and explain what changes on deallocate/start with static vs dynamic.
- [ ] Attach a NAT Gateway to a subnet and verify the outbound IP from inside the VM (`curl ifconfig.me`).
- [ ] Add and grow a data disk, take a snapshot, create a disk from it in another zone.
- [ ] Create a storage account with a Private Endpoint, disable public access, give a VM's managed identity blob read access, and prove `nslookup` returns the private IP.
- [ ] Build an Application Gateway (WAF_v2) → backend pool of two VMs, break a health probe, and diagnose it from Backend health.
- [ ] Push an image to ACR with `az acr build` and run it as a Container App with external ingress using managed identity pull.
- [ ] Put APIM (Consumption) in front of that Container App, add a rate-limit policy, and use Test → Trace.
- [ ] Attach a WAF policy in Detection, find a false positive in Log Analytics, add an exclusion, switch to Prevention.
- [ ] Write a Bicep file for the VNet + VM, run what-if, deploy it, break it on purpose (bad SKU), and read the error from RG → Deployments.
- [ ] Use Network Watcher → Connection troubleshoot to explain why two VMs can't talk.

---

## Glossary

**Tenant** Entra ID directory · **Subscription** billing/permission boundary · **RG** resource group · **RBAC** role-based access control · **Entra ID** identity service (formerly Azure AD) · **MI** managed identity · **SP** service principal · **VNet** virtual network · **NSG / ASG** network / application security group · **UDR** user-defined route · **NVA** network virtual appliance · **PIP** public IP · **PE** private endpoint · **App GW** Application Gateway · **AFD** Front Door · **VMSS** VM scale set · **ACR / ACA / ACI / AKS** container registry / container apps / container instances / Kubernetes service · **APIM** API Management · **ARM** Azure Resource Manager (the deployment API) · **Bicep** ARM's DSL · **KQL** Kusto Query Language (Log Analytics) · **PIM** Privileged Identity Management · **SAS** shared access signature · **LRS/ZRS/GRS** storage redundancy levels.
