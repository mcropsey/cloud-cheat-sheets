# Google Cloud Study Guide — Working in the Console

A practical, console-first guide, mirroring the AWS and Azure guides. Each section covers: **what it is**, **where it lives in the Console**, **what you actually do**, and **what breaks**.

---

## 0. Console Basics (read first)

- **Hierarchy:** Organization → Folders → **Project** → Resource. The **project** is the unit of billing, APIs, IAM, and quotas. Everything you create lives in a project. Delete the project, delete everything (after a 30-day grace period).
- **Project selector** (top bar): most "where did my resource go" moments are the wrong project. Each project has a name, a **project ID** (immutable, used in CLI and URLs), and a project number.
- **Regions and zones:** resources are **global** (VPC, firewall rules, images, load balancers), **regional** (subnets, Cloud Run, GKE regional clusters, regional disks), or **zonal** (VMs, persistent disks, node pools). Zone = `us-central1-a`.
- **Enable APIs first.** Nothing works until the service's API is enabled in the project (Compute Engine API, Kubernetes Engine API, Cloud Run API…). "API not enabled" errors → APIs & Services → Enable APIs and services.
- **IAM** (IAM & Admin): **principal** (user, group, service account) gets a **role** (bundle of permissions) at a **scope** (org, folder, project, or resource). Inherits downward. `PERMISSION_DENIED` names the exact permission missing — e.g. `compute.instances.create`. Search that permission in IAM → Roles to find a role that grants it.
- **Service accounts** = identities for workloads. VMs, Cloud Run, GKE pods all run **as** a service account. This is the IAM-role-for-EC2 / managed-identity equivalent.
- **Cloud Shell** (`>_` top right): browser terminal with `gcloud`, `kubectl`, `terraform` preinstalled and already authenticated. Almost every Console page has an **Equivalent command line** / **REST** link at the bottom of the create form — read it to learn `gcloud`.
- **Labels** (key/value) on resources; use for cost breakdown and filtering.
- **Logs Explorer** and **Cloud Audit Logs** (Admin Activity) record every change.

### AWS / Azure → Google Cloud name map

| AWS | Azure | Google Cloud |
|---|---|---|
| Account | Subscription | Project |
| VPC | VNet | VPC network (**global**) |
| Subnet | Subnet | Subnet (regional) |
| Security Group | NSG | VPC firewall rules / Firewall policies (+ network tags, service accounts) |
| Route table | Route table (UDR) | Routes (per VPC) |
| Internet Gateway | built-in | built-in (default route to internet gateway) |
| NAT Gateway | NAT Gateway | Cloud NAT (on a Cloud Router) |
| Elastic IP | Static Public IP | Reserved static external IP address |
| ALB / NLB | App Gateway / Load Balancer | Cloud Load Balancing (Application LB / Network LB) |
| EC2 | Virtual Machine | Compute Engine VM instance |
| AMI | Image | Image / Machine image |
| Auto Scaling Group | VMSS | Managed Instance Group (MIG) |
| EBS / S3 / EFS | Managed Disk / Blob / Files | Persistent Disk (or Hyperdisk) / Cloud Storage / Filestore |
| ECR / ECS / EKS | ACR / Container Apps / AKS | Artifact Registry / Cloud Run / GKE |
| API Gateway | API Management | API Gateway or Apigee |
| WAF | WAF policy | Cloud Armor |
| IAM role for EC2 | Managed identity | Service account attached to the resource |
| CloudFormation | ARM / Bicep | Infrastructure Manager (Terraform) — Deployment Manager is legacy |
| CloudWatch | Azure Monitor | Cloud Monitoring + Cloud Logging |
| CloudTrail | Activity Log | Cloud Audit Logs |
| Session Manager | Bastion / Run Command | IAP TCP forwarding / OS Login / Serial console |
| VPC Endpoint | Private Endpoint | Private Google Access / Private Service Connect |
| Peering / Transit GW | VNet Peering / vWAN | VPC Network Peering / Network Connectivity Center |

---

## 1. Networking — VPC

### Mental model
A Google **VPC is global**: one network spans all regions; **subnets are regional**. A VM in `us-central1` and one in `europe-west1` on the same VPC talk over internal IPs with no peering. Firewall rules are also global to the VPC.

```
Internet
   │
[External IP on VM] or [Global/Regional Load Balancer]
   │
VPC "prod-vpc" (global)
 ├─ subnet-us-central1   10.10.0.0/20   (VMs, GKE nodes)
 ├─ subnet-europe-west1  10.20.0.0/20
 └─ proxy-only subnet    (required for regional Application LBs)
        │
   [Cloud Router + Cloud NAT] per region for outbound from VMs without external IPs
   Private Google Access on subnets → reach Google APIs without external IPs
```

**Default network:** every new project gets a `default` VPC with one subnet per region and permissive firewall rules (`default-allow-ssh`, `-rdp`, `-icmp`, `-internal`). Fine for labs; delete or replace it for real work.

### Key components

| Component | What it does | Console location |
|---|---|---|
| VPC network | Global network; **auto mode** (one subnet per region auto-created) or **custom mode** (you make subnets) — use custom | VPC network → VPC networks |
| Subnet | Regional IP range; may have **secondary ranges** (GKE pods/services) | VPC network → VPC networks → subnet |
| Firewall rules | Allow/deny by direction, source/target, priority | VPC network → Firewall |
| Routes | System routes for subnets + default internet route; custom static routes | VPC network → Routes |
| Cloud Router | BGP router; hosts Cloud NAT and VPN/Interconnect routes | Network Connectivity → Cloud Routers |
| Cloud NAT | Outbound internet for VMs/pods with no external IP | Network services → Cloud NAT |
| External IP address | Ephemeral (changes on stop) or **reserved static** | VPC network → IP addresses |
| Private Google Access | Subnet setting: internal-only VMs can reach `*.googleapis.com` | Subnet → Edit |
| Private Service Connect | Private endpoint to Google APIs or a producer's service | Network services → Private Service Connect |
| VPC Network Peering | Connect two VPCs (non-transitive) | VPC network → VPC network peering |
| Cloud VPN / Interconnect | On-prem connectivity | Hybrid Connectivity |
| Shared VPC | Host project owns the network; service projects use it | VPC network → Shared VPC |

### What makes a VM reachable from the internet
Nothing about the subnet. A VM is reachable if it has an **external IP** (or sits behind an external load balancer) **and** a firewall rule allows the traffic. There's no "public subnet" concept — the default route `0.0.0.0/0 → default-internet-gateway` exists on every VPC. A VM with **no external IP** cannot reach the internet unless Cloud NAT is present.

### Firewall rules
- **Stateful**. Reply traffic auto-allowed.
- Fields: **direction** (ingress/egress), **priority** (0–65535, lower wins, default 1000), **action** (allow/deny), **targets** (all instances, **network tags**, or **service accounts**), **source/destination** (IP ranges, tags, service accounts), **protocols/ports**.
- **Implied rules** (priority 65535): deny all ingress, allow all egress. You can't delete them, only override.
- **Network tags** are the everyday targeting mechanism: tag the VM `web`, write a rule "allow tcp:80,443 from `0.0.0.0/0` to targets tagged `web`". Tag `db`, rule "allow tcp:5432 from source tag `app` to target tag `db`" — the SG-references-SG pattern.
- **Service accounts as targets** are more secure than tags (tags can be set by anyone with instance edit rights).
- Health checks and load balancer proxies come from Google ranges `35.191.0.0/16` and `130.211.0.0/22` — you **must allow these** to your backends or the LB reports everything unhealthy.
- IAP TCP forwarding comes from `35.235.240.0/20` — allow 22/3389 from that range instead of the internet.
- **Firewall policies** (hierarchical / network / regional) are the newer, org-wide layer; evaluated before VPC firewall rules. If a rule you wrote seems ignored, check for a policy above it.
- **Logging** can be enabled per rule → Logs Explorer shows every allowed/denied connection matching it.
- Debug: VM → **Network interface details** → shows firewall rules and routes that apply to this VM. First stop.

### External IPs
- **Ephemeral**: assigned at creation, released on stop or when an instance is deleted.
- **Reserved static**: VPC network → IP addresses → **Reserve external static address** → regional (for VMs, regional LBs) or **global** (only for global load balancers). Assign to a VM via Edit → Network interfaces → External IPv4 → pick the reserved address.
- Promote an ephemeral IP to static in place: IP addresses → the row → **Reserve**.
- Billed while reserved and **unused**. Release what you don't need.
- **Internal static IPs** also exist (reserve within a subnet) — useful for databases and internal LBs.

### Cloud NAT
- Requires a **Cloud Router** in the same region and VPC (create it in the NAT wizard).
- Applies to **subnets you choose** (all, or specific ranges) — only VMs **without external IPs** use it.
- NAT IPs: **automatic** or **manual** reserved static IPs (for partner allow-lists).
- Regional. One per region per VPC (per subnet set).
- Logs: enable **translation and errors** logging for debugging "why is outbound failing" (`OUT_OF_RESOURCES` = port exhaustion; raise min ports per VM).

### Routes and peering
- **System routes**: each subnet range + `default-internet-gateway`. Deleting the default route makes the VPC fully private.
- **Custom static routes**: next hop = instance (NVA), VPN tunnel, ILB, or gateway. Tagged routes apply only to VMs with that tag.
- **Peering** is non-transitive and both sides must create it; subnet ranges can't overlap. Peered VPCs share routes but **not firewall rules**.
- **Shared VPC** is the enterprise pattern: central host project, app teams in service projects deploy into its subnets.

### Load balancing
Google's load balancers are a family; the Console wizard (**Network services → Load balancing → Create**) picks the type by questions.

| Type | Scope | Layer | Use for |
|---|---|---|---|
| **Global external Application LB** | Global, single anycast IP | L7 HTTP/S | Public web apps, multi-region, CDN, Cloud Armor. Like ALB + CloudFront |
| **Regional external Application LB** | Regional | L7 | Same region only; needs a **proxy-only subnet** |
| **Internal Application LB** | Regional | L7 | Internal HTTP services; proxy-only subnet |
| **External / Internal passthrough Network LB** | Regional | L4 | TCP/UDP passthrough, preserves client IP. Like NLB |
| **Proxy Network LB** | Global/regional | L4 proxy | TCP/SSL proxy |

**Structure (Application LB):** Forwarding rule (IP + port) → Target proxy (cert) → **URL map** (host/path rules) → **Backend service** (protocol, timeout, session affinity) → **Backends** (instance groups, NEGs for GKE/Cloud Run, buckets) + **Health check**.

Backend health: Load balancing → the LB → backend service → health status per instance. Unhealthy → check firewall for the health-check ranges above, then the port/path.

---

## 2. Compute Engine

### Creating (Compute Engine → VM instances → Create instance)
1. **Name**, **region/zone**.
2. **Machine configuration**: series **E2** (cheap general), N2/N2D/N4 (general), C3 (compute), M (memory), A/G (GPU). `e2-micro`/`e2-small` for labs.
3. **OS and storage**: image (Debian default, Ubuntu, Rocky, Windows Server, Container-Optimized OS), boot disk size/type (`pd-balanced` default).
4. **Networking**: network/subnet, **external IPv4: None** for private VMs, **network tags**, firewall checkboxes (creates `default-allow-http`-style rules).
5. **Security → Identity and API access**: the **service account** the VM runs as and its **access scopes**. Set scopes to **Allow full access to all Cloud APIs** and control permissions with IAM roles on the service account instead — scopes are a legacy limiter that causes confusing 403s.
6. **Advanced → Management**: **Metadata** and **Startup script** (`startup-script` key, runs every boot), **Deletion protection**, `enable-oslogin`.
7. Bottom: **Equivalent code** → copy the `gcloud compute instances create` line.

### Connecting
- **SSH button** in Console: opens browser SSH. Works with **no external IP** if you allow ingress from `35.235.240.0/20` (IAP) on port 22 and have `IAP-secured Tunnel User` role. This is the Session Manager equivalent — learn it.
- **gcloud**: `gcloud compute ssh vm-name --zone us-central1-a` (add `--tunnel-through-iap` for private VMs). It manages SSH keys for you.
- **OS Login**: metadata `enable-oslogin=TRUE` maps Google identities to Linux users; permissions via `roles/compute.osLogin` / `osAdminLogin`. Preferred over metadata-based keys.
- **Windows**: set a Windows password from the RDP dropdown, then RDP to the external IP (allow 3389 from your IP or use IAP).
- **Serial console** (VM → Logs → Serial port 1 for read-only output; enable interactive serial console in metadata for a shell).

### Service accounts (identity for the VM)
- Default: the **Compute Engine default service account** with **Editor** role on the project — far too broad. Create a dedicated SA (IAM & Admin → Service accounts), grant only needed roles (e.g. `Storage Object Viewer` on one bucket), attach at creation or after **stopping** the VM (Edit → Service account).
- Code on the VM uses Application Default Credentials automatically — no keys. Never download SA key files unless you truly must.

### Startup scripts
Metadata key `startup-script` (inline) or `startup-script-url` (`gs://` path). Runs **every boot** as root. Logs: `sudo journalctl -u google-startup-scripts` or the serial console output.

### VM lifecycle
- **Stop**: disk kept, ephemeral external IP released, compute not billed (disk is). **Suspend**: RAM saved to disk, faster resume.
- **Reset**: hard reboot.
- **Delete**: boot disk deleted by default (checkbox "Delete boot disk when instance is deleted"). Enable **deletion protection** on important VMs.
- **Machine type change** requires stop.
- **Live migration** moves VMs off failing hosts with no reboot (host maintenance policy).
- **Spot VMs** (preemptible): up to ~90% off, can be reclaimed any time.

### Scaling and availability
- **Instance template** → **Managed Instance Group (MIG)**: autoscaling (CPU, LB utilization, schedule), **autohealing** (health check → recreate), rolling updates, zonal or **regional** (spread across zones). Backends for load balancers.
- **Unmanaged instance groups**: just a list of VMs for an LB backend.
- **Machine images** capture disks + metadata + config for cloning; **custom images** for boot disks; **Image families** give you "latest" pointers.

---

## 3. Storage

| Service | Type | Use it for | Equivalent |
|---|---|---|---|
| **Persistent Disk / Hyperdisk** | Block | VM boot and data disks | EBS / Managed Disk |
| **Cloud Storage** | Object | Files, backups, static sites, data lake | S3 / Blob |
| **Filestore** | NFS | Shared files across VMs/GKE | EFS / Azure Files |
| **Local SSD** | Ephemeral | Scratch, cache; data lost on stop | Instance store |

### Persistent Disk
- Types: `pd-standard` (HDD), `pd-balanced` (default), `pd-ssd`, `pd-extreme`; **Hyperdisk** for newer machine series with tunable IOPS.
- **Zonal** by default; **regional PD** replicates across two zones for HA.
- Attach: Compute Engine → Disks → Create → then VM → Edit → Additional disks → Attach existing (**hot-attach works, no stop**). In the OS: `lsblk`, `mkfs`, mount, `/etc/fstab` with UUID.
- Resize **online**: Disks → disk → Edit → size, then `sudo growpart` + `resize2fs`/`xfs_growfs`. Can't shrink.
- **Snapshots**: incremental, stored multi-regionally; **snapshot schedules** attach to disks for automated backups. Restore = create disk from snapshot in any zone/region.
- Multi-writer / read-only multi-attach exist but Filestore is usually the right answer for sharing.

### Cloud Storage
- **Bucket** names are globally unique. Location: region, dual-region, or multi-region (`US`, `EU`).
- **Storage classes**: Standard → Nearline (30-day min) → Coldline (90) → Archive (365). **Autoclass** moves objects automatically; **Lifecycle rules** for custom policies.
- **Access control**: **Uniform bucket-level** (IAM only, recommended) vs fine-grained (ACLs). Roles: `Storage Object Viewer/Creator/Admin`. **Public access prevention** is enforced by default on new buckets — leave it.
- **Signed URLs** = presigned URLs. **Object versioning** and **soft delete** (7-day default) protect from deletes. **Retention policies / bucket lock** for compliance.
- URL: `https://storage.googleapis.com/<bucket>/<object>`; `gs://bucket/object` in tooling; `gsutil` is legacy — use `gcloud storage cp`.
- Private VMs reach it via **Private Google Access** (no NAT needed) — enable on the subnet.
- Static websites: bucket named after the domain + LB with a bucket backend for HTTPS.

### Filestore
- Create instance: tier (Basic HDD/SSD, Zonal, Regional, Enterprise), capacity (1 TiB minimum for Basic), VPC, and optionally **reserved IP range** — it uses private services access.
- Mount: `sudo mount <ip>:/<share> /mnt/fs` (the Console shows the exact command). Firewall must allow TCP/UDP 111, 2046–2050 from clients (usually covered by `default-allow-internal`).

---

## 4. Containers

### Artifact Registry
- Create **repository** (format: Docker, plus Maven/npm/Python/apt…), region, name. Hostname pattern: `<region>-docker.pkg.dev/<project>/<repo>/<image>:<tag>`.
- Auth: `gcloud auth configure-docker <region>-docker.pkg.dev`, then `docker push`. Or **Cloud Build**: `gcloud builds submit --tag <region>-docker.pkg.dev/proj/repo/app:v1` builds in the cloud.
- Pull permissions: the runtime's **service account** needs `Artifact Registry Reader`. GKE and Cloud Run default SAs in the same project usually have it; cross-project pulls don't.
- **Container Registry (`gcr.io`) is deprecated** — use Artifact Registry.

### Choosing a container service

| Service | When |
|---|---|
| **Cloud Run** (services) | Default for HTTP apps/APIs. Serverless, scale-to-zero, revisions with traffic splitting, custom domains. Like Container Apps / Fargate |
| **Cloud Run jobs** | Run-to-completion tasks, batch, cron via Cloud Scheduler |
| **GKE Autopilot** | Kubernetes without managing nodes; pay per pod. Start here for K8s |
| **GKE Standard** | Full node-pool control |
| **Container-Optimized OS on a VM** | Single container on a VM (Console: "Deploy container" on VM create) |

### Cloud Run
- **Create service**: image URL (from Artifact Registry), region, **authentication** (allow unauthenticated for public, or require IAM invoker), **ingress** (all / internal / internal + LB), CPU/memory, **min/max instances** (min 1 kills cold starts), concurrency, **container port** (defaults to `$PORT` = 8080 — your app **must listen on `$PORT`**), env vars, **secrets** (from Secret Manager), **service account**, VPC access (**Direct VPC egress** to reach private resources / Cloud SQL).
- **Revisions**: every deploy creates one; **Traffic** tab splits/rollbacks.
- Logs: service → **Logs** tab (Cloud Logging). Startup failures show `Container failed to start and listen on the port defined by the PORT environment variable` — the #1 error.
- Common failures:
  - 403 on the URL → service requires authentication; call with an identity token or allow `allUsers` as **Cloud Run Invoker**.
  - Image pull fails → SA lacks `Artifact Registry Reader`, or region/URL typo.
  - Can't reach Cloud SQL / internal IP → enable VPC egress (Direct VPC or connector) and use the private IP; Cloud SQL also needs `Cloud SQL Client` role.
  - Request timeouts → default 300 s, raise up to 60 min.

### GKE
- **Autopilot** (recommended): create cluster → region → network. Google manages nodes, autoscaling, and security hardening; you just deploy pods with resource requests.
- **Standard**: node pools (machine type, count, autoscaling, spot), release channel, networking (VPC-native with alias IPs uses subnet **secondary ranges** for pods and services), private cluster (nodes have no external IPs — then you need Cloud NAT for pulls from outside Google, and IAP/bastion to reach a private control plane).
- Connect: `gcloud container clusters get-credentials <cluster> --region <region>` in Cloud Shell → `kubectl`. Console **Workloads**, **Services & Ingress**, **Config & Storage** pages are usable for viewing and simple deploys.
- Identity: **Workload Identity Federation for GKE** maps a Kubernetes SA to a Google SA — pods get IAM permissions without keys. Cluster RBAC is via **Kubernetes RBAC** with Google identities; `roles/container.developer` etc. control API access.
- Ingress: GKE Ingress creates a Google Application LB; **Gateway API** is the newer path. Services of type LoadBalancer create passthrough Network LBs.
- Troubleshoot: Workloads → the deployment → **Events** / pod **Logs**; `kubectl describe pod` for `ImagePullBackOff` (SA/registry), `CrashLoopBackOff` (app), `Pending` (no node capacity / requests too high — Autopilot will just add capacity, Standard needs autoscaling).

---

## 5. API Gateway / Apigee

| Product | When |
|---|---|
| **API Gateway** | Lightweight managed gateway in front of Cloud Run, Cloud Functions, App Engine, or any HTTPS backend. OpenAPI 2.0 config, API keys, JWT/Google auth. Like AWS HTTP API |
| **Apigee** | Full API management platform (analytics, monetization, developer portal, policies). Like Azure APIM / AWS REST API + more. Heavy and expensive |

**API Gateway structure:** **API** → **API config** (an OpenAPI 2.0 YAML with `x-google-backend` per path pointing at the backend URL) → **Gateway** (regional deployment of a config) → `https://<gateway>-<hash>.<region>.gateway.dev`.

Minimal config snippet:
```yaml
swagger: "2.0"
info: { title: my-api, version: 1.0.0 }
schemes: [https]
paths:
  /hello:
    get:
      operationId: hello
      x-google-backend:
        address: https://my-service-xyz-uc.a.run.app
      responses: { "200": { description: OK } }
```

**Gotchas**
- Uses **OpenAPI 2.0 (Swagger)**, not 3.0. Conversion errors are the top config failure.
- The gateway calls the backend **as its service account**: give it `Cloud Run Invoker` on the Cloud Run service (and keep Cloud Run private — that's the point).
- `x-google-backend` with `path_translation: APPEND_PATH_TRANSLATION` when your backend expects the same path.
- **API keys** require enabling the API (**APIs & Services → Enable** your new managed service) and `security: [api_key: []]` in the spec; otherwise 403 `API key not valid` or `PERMISSION_DENIED` with "API not enabled."
- Updates: create a **new API config** and update the gateway to it; configs are immutable. Deploys take a few minutes.
- CORS: set `x-google-endpoints` with `allowCors: true`, and make your backend return CORS headers too.
- Custom domain = put a global external Application LB with a serverless NEG in front.
- For a single public Cloud Run service you often don't need a gateway at all.

---

## 6. Cloud Armor (WAF)

Cloud Armor attaches **security policies** to **backend services** of a **global external Application LB** (edge policies for Cloud CDN/buckets; also works with global proxy Network LB and, with limits, regional external LBs). It does not attach to VMs directly.

**Structure:** **Security policy** → **rules** (priority, match condition, action) → default rule (allow/deny) → attach to backend service(s) (Target tab).

**Rule types**
- **Preconfigured WAF rules**: ModSecurity CRS-based expressions, e.g. `evaluatePreconfiguredWaf('sqli-v33-stable', {'sensitivity': 1})`, `xss-v33-stable`, `lfi`, `rce`, `protocolattack`, `scannerdetection`, `cve-canary`. Sensitivity 1–4; start at 1.
- **Custom rules**: basic (IP/CIDR lists) or **advanced** (CEL expressions: `request.path.matches('/admin')`, `origin.region_code == 'CN'`, `request.headers['user-agent'].contains('curl')`).
- **Rate limiting**: `throttle` (per-client limit, then 429) or `rate_based_ban` (temporary ban after threshold). Key on IP, header, cookie, etc.
- **Adaptive Protection** (Standard/Enterprise tiers): ML-based L7 DDoS detection with suggested rules.
- **Bot management** via reCAPTCHA integration.
- **Named IP lists** / threat intelligence (`evaluateThreatIntelligence('iplist-tor-exit-nodes')`) on the Enterprise tier.

**Workflow**
1. Network Security → Cloud Armor policies → **Create policy** (backend security policy).
2. Add rules with action **Preview** (equivalent of Count/Detection mode) — logs matches but doesn't act.
3. Attach to the LB backend service.
4. Logs Explorer: `resource.type="http_load_balancer" jsonPayload.enforcedSecurityPolicy.name="my-policy"` → look at `previewSecurityPolicy` / `enforcedSecurityPolicy.outcome`.
5. Turn preview off to enforce.

**Gotchas**
- Only global external Application LB (and a couple of others) — if your app is on a regional LB or a raw VM, no Cloud Armor. Move it behind a global LB.
- Rules are evaluated by **priority**; the first match wins. Put allow-lists (your office IP) at low numbers.
- Preconfigured rules fire on JSON bodies with SQL-ish text — tune with `opt_out_rule_ids` or `sensitivity`, not by disabling the whole set.
- Cloud Armor Standard is pay-per-policy/rule/request; **Enterprise** (Managed Protection Plus) is a subscription.
- Serverless backends (Cloud Run behind an LB) still accept direct requests to the `run.app` URL unless ingress is set to **internal and Cloud Load Balancing**.

---

## 7. Infrastructure as Code — Terraform / Infrastructure Manager

Google's native IaC story is **Terraform**: **Infrastructure Manager** (Infra Manager) runs Terraform for you as a managed service. **Deployment Manager** (YAML/Jinja) is legacy — read it if you inherit it, don't start new work in it. Many teams simply run Terraform themselves from Cloud Build or CI.

### Basics
- Terraform config: `provider "google" { project, region }`, `resource "google_compute_instance" "vm" { ... }`, `variable`, `output`, `module`. State is stored in a **GCS bucket backend** (`terraform { backend "gcs" { bucket = "..." } }`) so the team shares it.
- `terraform init` → `plan` → `apply`. `plan` is your what-if / change set — always read it, especially lines marked **must be replaced** (delete + recreate).
- **Infrastructure Manager** (Console: search "Infrastructure Manager"): create a **deployment** from a Git repo or GCS bucket containing Terraform, choose a service account it runs as, set input values → it runs plan/apply and stores state. Deployments page shows **revisions**, each with its own logs and errors.
- Both Terraform and Infra Manager call the same Google APIs, so the errors below apply either way.

### Where to find the error
1. Terraform output: the error names the resource address (`google_compute_instance.vm`) and the API error (`Error 403: ... PERMISSION_DENIED`, `Error 409: already exists`, `Error 400: Invalid value...`).
2. Infra Manager → deployment → revision → **Logs** (Cloud Build logs behind the scenes).
3. **Cloud Audit Logs** in Logs Explorer: `protoPayload.status.code != 0` around the same time gives the server-side view, including which principal was denied.

### Deployment behavior
| Situation | Terraform / Infra Manager |
|---|---|
| Apply fails partway | **No rollback.** Resources already created stay and are recorded in state. Fix and re-apply; Terraform converges |
| State drift (someone changed things in the Console) | `terraform plan` shows it; apply reverts to the config. Use `terraform import` to adopt hand-made resources |
| Resource must be replaced | Plan says so; expect downtime for that resource. `lifecycle { prevent_destroy = true }` blocks it |
| Locked state | Another apply is running or crashed → `terraform force-unlock` after confirming nobody else is applying |

### Common errors and fixes
| Error contains | Cause | Fix |
|---|---|---|
| `Error 403 ... PERMISSION_DENIED` / `does not have <permission>` | The principal running Terraform (you or the SA) lacks that permission at that scope | Grant a role containing the named permission (IAM → Roles → search permission) |
| `Error 403 ... API has not been used in project ... before or it is disabled` / `SERVICE_DISABLED` | Service API not enabled | Enable it in the Console or add `google_project_service` resources (with `disable_on_destroy = false`) |
| `Error 409: The resource ... already exists` / `alreadyExists` | Name collision — resource exists outside state (bucket name, static IP, SA ID) | Rename, or `terraform import` it |
| `Error 400: Invalid value for field 'resource.zone'` / `Invalid value ... machineType` | Zone/type typo, or type not offered in that zone | `gcloud compute machine-types list --zones us-central1-a` |
| `ZONE_RESOURCE_POOL_EXHAUSTED` / `The zone ... does not have enough resources` | Capacity in that zone | Try another zone, or a regional MIG spanning zones |
| `QUOTA_EXCEEDED` / `Quota 'CPUS' exceeded` / `IN_USE_ADDRESSES` | Project quota | IAM & Admin → Quotas → filter metric → Edit quota (new projects have low CPU/IP quotas) |
| `resourceInUseByAnotherResource` | Deleting a subnet/IP/firewall/disk still attached to something | Remove the dependent resource first (often not in your config) |
| `Error waiting for operation ... Operation ... failed` | Long-running operation failed after acceptance | Read the operation's inner error; Compute Engine → Operations page lists them |
| `googleapi: Error 404: ... was not found` | Wrong project ID, resource in another project/region, or deleted by hand | Check `provider` project and region; `terraform state rm` if truly gone |
| `Provider produced inconsistent result` / `inconsistent final plan` | Provider bug or race condition | Re-run apply; pin provider version |
| `dial tcp ... i/o timeout` / `oauth2: cannot fetch token` | Credentials or network for the runner | `gcloud auth application-default login`, or check the runner's SA |
| Cloud Run `Container failed to start` during `google_cloud_run_v2_service` | The image, not Terraform | Fix the app / port |
| `IAM policy ... Error 400: Role ... is not supported for this resource` | Wrong role type on a resource-level binding | Use a role valid for that resource type |
| `Deployment Manager` `RESOURCE_ERROR` | Legacy DM template error | Read `errors[].message`; consider migrating to Terraform |

### Other useful concepts
- **`google_project_iam_policy`** is **authoritative** and can wipe all IAM on a project — use `_member` / `_binding` resources instead.
- **Organization policies** (IAM & Admin → Organization policies) can block things silently: `constraints/compute.vmExternalIpAccess`, `restrictSharedVpcSubnetworks`, `iam.disableServiceAccountKeyCreation`. A 412 `PRECONDITION_FAILED` or a policy-named error means this.
- **Config Connector** manages GCP resources from Kubernetes manifests — another option in GKE-heavy shops.
- **Cloud Foundation Toolkit** modules (`terraform-google-modules`) are Google's blueprints — use them rather than raw resources for VPCs, projects, GKE.
- Terraform's `google-beta` provider is needed for some newer features.

---

## 8. Troubleshooting Cheat Sheet

### "I can't SSH / RDP to my VM"
1. VM **running**? Status column.
2. **External IP** present? If not, use the Console SSH button through IAP (needs firewall allow from `35.235.240.0/20` and `IAP-secured Tunnel User`).
3. **Firewall**: VM → Network interface details → firewall rules that apply. Is there an allow rule for 22/3389 with a **target** that matches this VM (tag/SA)? Is a higher-priority deny or a firewall policy above it?
4. OS Login vs metadata keys mismatch — if `enable-oslogin=TRUE`, you need the OS Login role; if FALSE, your key must be in metadata (`gcloud compute ssh` handles this).
5. Guest agent or sshd dead → **Serial port output** and the interactive serial console.
6. Windows password: set via the RDP dropdown.
7. **Network Intelligence Center → Connectivity Tests** simulates the packet path and reports the blocking rule.

### "Private VM has no internet"
1. Does the VM have an external IP? If no → **Cloud NAT** must exist in that **region** covering that **subnet**.
2. Cloud Router present; NAT status healthy; check NAT logs for `OUT_OF_RESOURCES`.
3. Default route `0.0.0.0/0 → default-internet-gateway` still in the VPC?
4. Egress firewall/policy deny?
5. Only need Google APIs (GCS, Artifact Registry)? Enable **Private Google Access** on the subnet instead of NAT.
6. DNS: `*.googleapis.com` resolution works by default via metadata server `169.254.169.254`; custom DNS servers can break it.

### "Load balancer returns 502 / backend unhealthy"
1. LB → backend service → **health check status**.
2. **Firewall allows `35.191.0.0/16` and `130.211.0.0/22`** to the backend port — the #1 cause.
3. Health check port/path returns 200; app listens on `0.0.0.0`.
4. Regional/internal Application LB → **proxy-only subnet** exists in that region.
5. Serverless NEG → Cloud Run ingress allows the LB; service isn't requiring auth the LB can't provide.
6. New global LBs take several minutes to propagate; wait before panicking.
7. 502 `failed_to_connect_to_backend` vs `backend_timeout` in LB logs tells you connectivity vs slowness.

### "Containers keep failing"
Cloud Run: Logs tab → port/`$PORT`, image pull SA, Secret Manager access (`Secret Manager Secret Accessor`). GKE: Workloads → Events; `kubectl describe pod`; private cluster nodes need Cloud NAT to pull non-Google images.

### "PERMISSION_DENIED / 403"
- The error names the **permission** (e.g. `storage.objects.get`) — search it in IAM → Roles.
- Which **principal**? Your user, or the resource's **service account** (VM, Cloud Run, GKE pod via Workload Identity). Grant the role to the SA, not to yourself.
- **Scope**: role granted on a different project or bucket? IAM → check scope; resource-level IAM (bucket → Permissions) is separate from project IAM.
- VM **access scopes** too narrow → still 403 even with the right role; set scopes to full access.
- **API not enabled** returns 403 too — read the full message.
- **Org policy** / **IAM deny policies** / **VPC Service Controls** perimeters override allows (`VPC_SERVICE_CONTROLS` in the error).
- IAM changes propagate in ~60 s; `gcloud auth list` to confirm which account you're using.
- **Policy Troubleshooter** (IAM & Admin) answers "does principal X have permission Y on resource Z, and why not."

### "Two resources can't talk"
1. Same VPC? (Global — region doesn't matter.) Different VPCs → peering established **both ways**, no overlapping CIDRs?
2. Ingress firewall on the **destination** allows the port from the source's IP range/tag/SA. Egress allowed on source (default yes).
3. Firewall **policies** (hierarchical/network) evaluated before VPC rules — check them.
4. **Connectivity Tests** (Network Intelligence Center) — run source → destination, it shows every hop and the exact rule that drops.
5. PaaS with private IPs (Cloud SQL, Memorystore, Filestore) live in a Google-managed VPC connected via **private services access** — the peering must exist and importing/exporting custom routes may be needed.
6. **VPC Flow Logs** (enable per subnet) → Logs Explorer for accept/reject evidence.

### "Terraform / Infra Manager failed"
See Section 7. Read the API error, check Audit Logs for the denied principal, enable the API, fix quota, re-apply (no rollback to clean).

### "Where do I find logs?"
- Everything → **Logs Explorer** (Cloud Logging). Filter by `resource.type` (`gce_instance`, `cloud_run_revision`, `k8s_container`, `http_load_balancer`).
- Who did what → **Audit logs**: `logName:"cloudaudit.googleapis.com%2Factivity"`.
- VM OS logs → **Ops Agent** installed (Console prompts you) → syslog/journal in Logging; **Serial port output** without any agent.
- Network → VPC Flow Logs, firewall rule logs, Cloud NAT logs, LB request logs.
- Cloud Armor → LB logs `enforcedSecurityPolicy`.
- Errors across services → **Error Reporting**. Metrics/alerts → **Cloud Monitoring** dashboards and alerting policies.
- Cost → **Billing → Reports** grouped by service/label; export billing to BigQuery for detail.

---

## 9. Security Quick Rules

- Org-level: enforce **2-Step Verification**, use **groups** for role assignment, apply **org policies** (no external IPs, no SA keys, restrict domains).
- Replace the default Compute/App Engine service accounts' **Editor** role; one SA per workload with minimal roles.
- **No SA key files.** Use attached service accounts, Workload Identity Federation, and impersonation (`--impersonate-service-account`).
- No external IPs on VMs; **IAP** for admin access; **Cloud NAT** for egress; **Private Google Access** and **Private Service Connect** for APIs.
- Target firewall rules by **service account** where possible; enable firewall logging on deny rules; delete the `default` network.
- Cloud Storage: uniform bucket-level access + public access prevention; **Secret Manager** for secrets; **CMEK** where required.
- **VPC Service Controls** for data-exfiltration perimeters around sensitive projects.
- **Security Command Center** (Standard tier is free) for misconfiguration findings.
- Budgets and alerts in Billing; **Recommender** for idle resources.

---

## 10. Self-Check — can you do these in the Console?

- [ ] Create a custom-mode VPC with subnets in two regions, delete the default network, and write tag- and SA-targeted firewall rules.
- [ ] Launch a VM with no external IP and a dedicated service account, and SSH via the Console button through IAP.
- [ ] Reserve a static external IP, attach it to a VM, and explain what happens to an ephemeral IP on stop.
- [ ] Set up Cloud Router + Cloud NAT, verify egress from the private VM, and read the NAT logs.
- [ ] Attach and grow a persistent disk online, create a snapshot schedule, restore into another zone.
- [ ] Create a bucket with uniform access, grant the VM's SA object viewer on that bucket only, and read it from the private VM via Private Google Access.
- [ ] Build a global Application LB → MIG backend, break the health-check firewall rule, and diagnose it.
- [ ] Push an image with `gcloud builds submit` to Artifact Registry and deploy it to Cloud Run listening on `$PORT`, with a secret from Secret Manager.
- [ ] Put API Gateway in front of a private Cloud Run service with an API key.
- [ ] Attach a Cloud Armor policy in preview mode with a preconfigured SQLi rule and a rate limit, find matches in Logs Explorer, then enforce.
- [ ] Write Terraform for the VPC + VM with GCS state, read the plan, break it (bad machine type / disabled API), fix it from the error message and Audit Logs.
- [ ] Run a Connectivity Test between two VMs and explain the result.

---

## Glossary

**Org / Folder / Project** resource hierarchy · **Project ID** immutable identifier · **IAM** identity and access management · **SA** service account · **ADC** application default credentials · **VPC** virtual private cloud (global) · **PGA** Private Google Access · **PSC** Private Service Connect · **IAP** Identity-Aware Proxy · **MIG** managed instance group · **PD** persistent disk · **GCS** Cloud Storage · **NEG** network endpoint group (LB backend for GKE/Cloud Run) · **GKE** Kubernetes Engine · **AR** Artifact Registry · **CEL** Common Expression Language (Cloud Armor rules) · **CMEK** customer-managed encryption keys · **VPC SC** VPC Service Controls · **SCC** Security Command Center · **LRO** long-running operation · **Cloud Foundation Toolkit** Google's Terraform modules.
