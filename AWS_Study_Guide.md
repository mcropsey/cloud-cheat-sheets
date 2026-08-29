# AWS Study Guide — Working in the Console

A practical, console-first guide. Each section covers: **what it is**, **where it lives in the UI**, **what you actually do**, and **what breaks**.

---

## 0. Console Basics (read first)

- **Region selector** (top right): every resource except IAM, S3 bucket names, Route 53, and CloudFront is **region-scoped**. If you "can't find" a resource, check the region first.
- **Search bar** (top): type the service name (EC2, VPC, ECS) — faster than the Services menu.
- **IAM**: users, roles, policies. You need permissions for whatever you're clicking. "Access Denied" = IAM, not the service.
- **Tags**: put a `Name` tag on everything. Untagged resources become unidentifiable in a week.
- **Resource IDs**: `vpc-…`, `subnet-…`, `sg-…`, `i-…`, `eni-…`, `eipalloc-…`, `nat-…`, `igw-…`, `rtb-…`. Learn to recognize them — errors reference them constantly.

---

## 1. Networking — VPC

### Mental model
A **VPC** is your private network (a CIDR block like `10.0.0.0/16`). Inside it: **subnets** (per Availability Zone), **route tables** (where traffic goes), and **gateways** (how traffic leaves).

```
Internet
   │
[Internet Gateway] ─── attached to VPC
   │
Public Subnet  (route 0.0.0.0/0 → IGW)  ── Load balancer, NAT Gateway, bastion
   │
[NAT Gateway] (lives in public subnet, has an Elastic IP)
   │
Private Subnet (route 0.0.0.0/0 → NAT GW) ── App servers, containers, databases
```

### Key components

| Component | What it does | UI location |
|---|---|---|
| VPC | Isolated network with a CIDR | VPC → Your VPCs |
| Subnet | Slice of the VPC in one AZ | VPC → Subnets |
| Route Table | Rules: destination → target | VPC → Route Tables |
| Internet Gateway (IGW) | Two-way internet access for public subnets | VPC → Internet Gateways |
| NAT Gateway | Outbound-only internet for private subnets | VPC → NAT Gateways |
| Elastic IP (EIP) | Static public IPv4 you own | VPC → Elastic IPs (also under EC2) |
| Security Group (SG) | Stateful firewall on the instance/ENI | VPC or EC2 → Security Groups |
| Network ACL (NACL) | Stateless firewall on the subnet | VPC → Network ACLs |
| VPC Endpoint | Private path to AWS services (S3, ECR, etc.) without NAT | VPC → Endpoints |

### What makes a subnet "public"
Nothing about the subnet itself. A subnet is public **only if** its route table has `0.0.0.0/0 → igw-xxxx`. That's it. Also enable **Auto-assign public IPv4** on the subnet (Subnet → Actions → Edit subnet settings) if instances need public IPs at launch.

### Fastest way to build a VPC
VPC → **Create VPC** → choose **"VPC and more"**. It builds VPC + public/private subnets across AZs + route tables + IGW + optional NAT in one click. Use this and then study what it created.

### NAT Gateway
- Created **in a public subnet**, requires an **Elastic IP**.
- Private subnet's route table needs `0.0.0.0/0 → nat-xxxx`.
- Costs money per hour + per GB even when idle. Delete unused ones.
- One per AZ for high availability; one total is fine for learning.

### Elastic IPs
- Allocate: VPC → Elastic IPs → **Allocate**. Associate: select → Actions → **Associate** → pick instance or network interface.
- Survives instance stop/start (a normal public IP does **not** — it changes on stop/start).
- Charged when allocated but **not associated** with a running resource. Release what you don't use.
- To attach to a NAT GW, you pick the EIP at NAT creation time.

### Security Groups vs NACLs

| | Security Group | Network ACL |
|---|---|---|
| Attaches to | Instance / ENI | Subnet |
| State | **Stateful** — reply traffic auto-allowed | **Stateless** — must allow both directions |
| Rules | Allow only | Allow and Deny |
| Evaluation | All rules evaluated | Numbered, first match wins |
| Default | Inbound: deny all. Outbound: allow all | Allow all both ways |

**Practical rule:** use Security Groups for 95% of things. Leave NACLs at default unless you need to block a specific IP range at the subnet level.

**Security Group tips**
- Source can be an IP/CIDR **or another security group**. "Allow 3306 from sg-app" is the right way to let app servers reach a database — no IPs to maintain.
- `0.0.0.0/0` on port 22 is a red flag. Use your IP (`My IP` option in the dropdown) or Session Manager instead.
- An instance can have multiple SGs; rules are **unioned** (any allow = allowed).
- Changes apply **immediately**, no reboot.
- Common ports: 22 SSH, 80 HTTP, 443 HTTPS, 3389 RDP, 3306 MySQL, 5432 Postgres, 6379 Redis.

### Load Balancers (EC2 → Load Balancers)
- **ALB** (Application) — HTTP/HTTPS, layer 7, path/host routing. Use for web apps and containers.
- **NLB** (Network) — TCP/UDP, layer 4, static IPs, very high performance.
- Structure: **Listener** (port 443) → **Rule** → **Target Group** (instances/containers/IPs) with a **health check**.
- Health check failing = ALB returns 502/503. Check the target group's Targets tab first.
- ALB lives in **public subnets** (at least 2 AZs); targets live in private subnets. ALB's SG allows 80/443 from internet; target SG allows the app port **from the ALB's SG**.

---

## 2. EC2

### Launching (EC2 → Instances → Launch instances)
1. **Name** tag
2. **AMI** — Amazon Linux 2023 or Ubuntu are safe defaults. AMIs are region-specific.
3. **Instance type** — `t3.micro`/`t3.small` for testing. Letters mean family: t = burstable general, m = general, c = compute, r = memory, g/p = GPU.
4. **Key pair** — needed for SSH. Download the `.pem` once; you cannot get it again.
5. **Network settings** → Edit: pick VPC, subnet, auto-assign public IP, security group.
6. **Storage** — root EBS volume. 8 GB default, gp3 type.
7. **Advanced** → **IAM instance profile** (role for the instance) and **User data** (startup script).

### Connecting
- **Session Manager** (Connect → Session Manager tab): no SSH port, no key, works in private subnets. Requires the `AmazonSSMManagedInstanceCore` policy on the instance role and either a NAT/IGW or SSM VPC endpoints. **Learn this — it's the modern way.**
- **EC2 Instance Connect**: browser SSH, needs port 22 open to AWS's IP range.
- **SSH**: `ssh -i key.pem ec2-user@<public-ip>` (Amazon Linux) or `ubuntu@` (Ubuntu). `chmod 400 key.pem` first.

### IAM roles for EC2
Never put access keys on an instance. Create a role (IAM → Roles → Create → AWS service → EC2), attach policies, assign to instance (Actions → Security → Modify IAM role). The instance gets temporary credentials automatically.

### User data
Shell script run **once at first boot** as root. Example:
```bash
#!/bin/bash
dnf install -y nginx
systemctl enable --now nginx
```
Logs: `/var/log/cloud-init-output.log` on the instance.

### Instance lifecycle
- **Stop/Start**: keeps EBS, loses public IP (unless EIP), may move host. Not billed for compute while stopped.
- **Reboot**: same host, keeps IP.
- **Terminate**: gone. Root volume deleted by default (change "Delete on termination" if needed). Enable **termination protection** on anything important.
- **Instance store** volumes are wiped on stop. EBS is not.

### Other EC2 things to know
- **Auto Scaling Group**: launch template + min/max/desired + target group. Replaces unhealthy instances automatically.
- **Launch Template**: saved launch config; required for ASGs.
- **Placement / Spot / Reserved**: cost and placement options — know they exist.
- **Status checks**: System (AWS hardware) vs Instance (your OS). Instance check failing = OS problem, look at the **screenshot** and **system log** (Actions → Monitor and troubleshoot).

---

## 3. Storage

| Service | Type | Use it for | Attaches to |
|---|---|---|---|
| **EBS** | Block (disk) | Boot volumes, databases | One instance at a time, same AZ |
| **S3** | Object | Files, backups, static websites, logs | Anything, via API/HTTP |
| **EFS** | Network file system (NFS) | Shared files across many instances/containers | Many instances, multi-AZ |
| **Instance store** | Ephemeral local disk | Scratch/cache | Lost on stop |

### EBS
- Volume types: **gp3** (default, pick this), io2 (high IOPS), st1/sc1 (throughput/cold HDD).
- Must be in the **same AZ** as the instance. To move AZ: snapshot → create volume in new AZ.
- **Snapshots** (EBS → Snapshots): incremental backups to S3. Create AMIs from them. Use **Data Lifecycle Manager** or AWS Backup for scheduled snapshots.
- Resize: Modify Volume (can grow, not shrink), then extend the filesystem inside the OS (`growpart` + `resize2fs`/`xfs_growfs`).
- Attach: Volumes → Actions → Attach, then in the OS: `lsblk`, `mkfs`, `mount`, add to `/etc/fstab`.

### S3
- Bucket names are **globally unique**. Buckets are regional; data doesn't leave the region unless you replicate.
- **Block Public Access** is on by default — leave it on unless hosting a public site.
- Access control: **bucket policies** (resource-based) and IAM policies. ACLs are legacy.
- **Storage classes**: Standard → Standard-IA → Glacier tiers. Use **Lifecycle rules** to move/expire objects automatically.
- **Versioning**: protects from overwrites/deletes. Enable on important buckets.
- **Presigned URLs**: temporary access to a private object.
- Instances access S3 via IAM role; add a **Gateway VPC Endpoint for S3** (free) so private subnets don't go through NAT.

### EFS
- Create file system → creates **mount targets** per subnet/AZ, each with a security group that must allow **NFS port 2049** from the clients' SG.
- Mount with `amazon-efs-utils` or NFS. Common with ECS/EKS for shared persistent storage.

---

## 4. Containers

### ECR — image registry (ECR → Repositories)
- Create repo → **View push commands** gives you the exact `docker login`, `tag`, `push` lines.
- Instances/tasks pull images with IAM permissions (`AmazonEC2ContainerRegistryReadOnly` or the ECS task execution role).
- Private subnets need NAT **or** ECR VPC endpoints (`ecr.api`, `ecr.dkr`, plus S3 gateway endpoint) to pull images.

### ECS — run containers (ECS → Clusters)
Four objects, top to bottom:

1. **Cluster** — logical grouping. Choose **Fargate** (serverless, no EC2 to manage) or **EC2** launch type.
2. **Task Definition** — the blueprint: image, CPU/memory, port mappings, env vars, log config, **task role** (what the app can call) and **task execution role** (what ECS needs to pull image / write logs).
3. **Service** — keeps N copies of a task running, hooks to a load balancer, does rolling deploys.
4. **Task** — a running instance of the task definition.

**Networking:** with `awsvpc` mode (Fargate always) every task gets its **own ENI and security group** in your subnet. Treat a task like a tiny EC2 instance: subnet, SG, public IP yes/no.

**Typical Fargate web setup:** ALB in public subnets → target group (type **IP**) → ECS service in private subnets, task SG allows app port from ALB SG.

**Logs:** enable `awslogs` driver in the task definition → CloudWatch Logs. Tasks that die immediately: check the task's **Logs** tab and the **Stopped reason** on the stopped task.

Common stop reasons:
- `CannotPullContainerError` → no NAT/endpoints, or execution role lacks ECR permissions.
- `ResourceInitializationError ... secrets` → execution role can't read Secrets Manager/SSM.
- `Essential container exited` → your app crashed; read the logs.
- Task keeps restarting behind ALB → health check path/port wrong.

### EKS — Kubernetes
- Managed control plane; you run **node groups** (EC2) or **Fargate profiles**.
- Console shows workloads and nodes, but real work is `kubectl`. Need `aws eks update-kubeconfig --name <cluster>`.
- Know: Kubernetes RBAC is mapped to IAM via **access entries** (or the older `aws-auth` ConfigMap). "Unauthorized" in kubectl = that mapping.
- Start with ECS unless you already know Kubernetes.

---

## 5. API Gateway

Managed front door for HTTP APIs. Two flavors:

| | HTTP API | REST API |
|---|---|---|
| Cost / latency | Cheaper, faster | Higher |
| Features | Basic routing, JWT auth, Lambda/HTTP integrations | Full: API keys, usage plans, request validation, caching, WAF, private APIs |
| Pick when | Simple Lambda/microservice front end | You need those extra features |

**Structure:** API → **Resources/Routes** (`/users`, `/users/{id}`) → **Methods** (GET/POST) → **Integration** (Lambda, HTTP endpoint, AWS service, VPC Link to private ALB/NLB) → **Stage** (`dev`, `prod`) → invoke URL.

**Must-know gotchas**
- Changes to a REST API do **nothing until you Deploy** to a stage. #1 reason "my change isn't showing."
- **CORS** errors from a browser: enable CORS on the resource/route, and make sure your Lambda also returns the `Access-Control-Allow-Origin` header for proxy integrations.
- Lambda integration `502 Malformed response`: Lambda must return `{ statusCode, headers, body: "<string>" }`.
- Reaching private resources: use a **VPC Link**.
- Authorization: IAM, Cognito, Lambda authorizer, or JWT (HTTP API).
- Turn on **execution logging** in Stage → Logs to debug. Throttling defaults: 10,000 rps, 5,000 burst per account per region.

---

## 6. WAF — Web Application Firewall

Filters HTTP(S) requests **before** they hit your app. Attaches to **ALB, CloudFront, API Gateway (REST), AppSync, Cognito**. It does not attach to EC2 directly.

**Structure:** **Web ACL** → **Rules** (evaluated by priority, lowest first) → default action (Allow/Block).

**Rule types**
- **AWS Managed Rule Groups** — start here. `Core rule set` (OWASP-style), `Known bad inputs`, `SQL database`, `IP reputation`, `Bot Control`. Free-ish and good coverage.
- **Rate-based** — block an IP that exceeds N requests per 5 min. Cheap DDoS/brute-force mitigation.
- **IP sets** — allow/block lists.
- **Custom rules** — match on URI, headers, body, geo, query string.

**Workflow**
1. WAF & Shield → Web ACLs → Create (pick the right region; CloudFront uses "Global").
2. Add managed rule groups, set to **Count** mode first.
3. Associate with your ALB/API.
4. Watch **Sampled requests** / CloudWatch metrics for false positives, then switch rules to **Block**.
5. Enable logging to S3/CloudWatch for investigations.

**Gotcha:** managed rules can block legitimate traffic (large POST bodies, certain user agents). Always run in Count first. Also, WAF doesn't stop someone hitting the ALB's DNS name directly if they bypass CloudFront — lock the ALB SG to CloudFront's prefix list if you front it with CloudFront.

---

## 7. CloudFormation — Templates and Troubleshooting

### Basics
- **Template** (YAML/JSON) declares resources. **Stack** = a deployed template. Update the template → update the stack → CFN computes the diff.
- Sections: `Parameters`, `Mappings`, `Conditions`, `Resources` (only required one), `Outputs`.
- Reference other resources with `!Ref` (usually the ID/name) and `!GetAtt Resource.Attribute` (e.g. `!GetAtt MyALB.DNSName`).
- Console: CloudFormation → Stacks → Create stack → upload template → parameters → **acknowledge IAM capabilities** checkbox if it creates roles.

### Stack states you'll see
| State | Meaning |
|---|---|
| `CREATE_IN_PROGRESS` / `UPDATE_IN_PROGRESS` | Working |
| `CREATE_COMPLETE` | Good |
| `ROLLBACK_IN_PROGRESS` / `ROLLBACK_COMPLETE` | Create failed; everything was torn down. Stack must be **deleted** before re-creating with the same name |
| `UPDATE_ROLLBACK_COMPLETE` | Update failed, reverted to previous good state — stack is still usable |
| `UPDATE_ROLLBACK_FAILED` | Stuck. Use **Continue update rollback**, optionally skipping the broken resource |
| `DELETE_FAILED` | Something couldn't be deleted (non-empty S3 bucket, SG still in use, ENI attached). Fix manually, delete again, or retain the resource |

### How to troubleshoot a failed stack (do this in order)
1. Stack → **Events** tab → sort by time → find the **first** `CREATE_FAILED` / `UPDATE_FAILED`. Everything after it is cascading cleanup — ignore it.
2. Read the **Status reason** column. It's usually explicit.
3. Fix, then delete (if `ROLLBACK_COMPLETE`) and re-create, or update.

**Tip:** when creating, expand **Stack failure options** → **Preserve successfully provisioned resources**. Failed resources stay so you can inspect them instead of watching everything vanish.

### Common failure reasons and fixes
| Status reason contains | Cause | Fix |
|---|---|---|
| `is not authorized to perform` / `AccessDenied` | Your IAM user or the stack's service role lacks permission | Add permission; check if a service role was set on the stack |
| `Requires capabilities : [CAPABILITY_IAM]` | Template creates IAM resources | Tick the acknowledgement checkbox |
| `already exists` | Hard-coded name collides (S3 bucket, IAM role, log group) | Remove the explicit name or make it unique with `!Sub` and the stack name |
| `Template format error` / `Unresolved resource dependencies` | Typo in `!Ref`, bad YAML indent | Validate: Stacks → Create → the designer/validate step, or `aws cloudformation validate-template` |
| `The security group 'sg-…' does not exist in VPC` | SG belongs to a different VPC | Fix the reference |
| `Invalid availability zone` / `not supported in your requested Availability Zone` | Instance type not in that AZ | Change AZ or type |
| `Resource handler returned message: ... rate exceeded` / `Throttling` | API limits | Retry; add `DependsOn` to serialize |
| `Circular dependency between resources` | A refs B refs A (classic: SG rules referencing each other inline) | Use separate `AWS::EC2::SecurityGroupIngress` resources |
| `Resource creation cancelled` | Not the real error — another resource failed first | Find the earlier failure |
| `Timeout` on `WaitCondition` / `CreationPolicy` | User data never signaled `cfn-signal` | Check user data logs on the instance |
| `Bucket ... is not empty` on delete | S3 buckets must be empty | Empty it, or set `DeletionPolicy: Retain` |
| `has a dependent object` on delete | SG/ENI/subnet still in use by something outside the stack | Detach/delete the outside resource |

### Other useful concepts
- **Change Sets**: preview exactly what an update will do (and whether it will **replace** a resource — replacement means deletion + recreation, e.g. changing an EC2 subnet). Always use change sets in prod.
- **Drift detection**: Stack actions → Detect drift. Shows resources someone changed by hand in the console. Fix by updating the template to match or reverting the manual change.
- `DeletionPolicy: Retain` / `Snapshot` on databases and buckets.
- **Stack policy**: prevents accidental updates to specific resources.
- **Nested stacks / StackSets**: reuse across accounts/regions.
- **Outputs + Exports**: share values between stacks (`Fn::ImportValue`). You can't delete a stack whose exports are in use.
- Also know that **Terraform** and **CDK** do the same job; CDK compiles to CloudFormation, so these troubleshooting steps still apply.

---

## 8. Troubleshooting Cheat Sheet

### "I can't SSH / RDP to my instance"
1. Instance running and **both status checks passed**?
2. Does it have a **public IP or EIP**? (Private subnet → no.)
3. Subnet route table has `0.0.0.0/0 → igw`?
4. **Security group** inbound allows 22/3389 from your IP?
5. NACL not blocking (check ephemeral ports 1024–65535 outbound if customized)?
6. Right key pair, right username (`ec2-user`, `ubuntu`, `admin`, `Administrator`)?
7. Still stuck → use **Session Manager** or **EC2 Serial Console**, or check the instance screenshot.

### "Instance in private subnet has no internet"
1. Route table has `0.0.0.0/0 → nat-xxxx`?
2. NAT gateway is **Available** and in a **public** subnet (whose own route table points to the IGW)?
3. NAT has an EIP?
4. Instance SG outbound allows traffic (default does)?
5. Alternative: is it only AWS services you need? Add VPC endpoints instead.

### "Load balancer returns 502 / 503 / unhealthy targets"
1. Target group → Targets → health status and reason.
2. Target SG allows the **health check port** from the **ALB SG**.
3. Health check path returns 200 (not 301/302/404). Change path or success codes.
4. App is actually listening on that port on `0.0.0.0`, not `127.0.0.1`.
5. 503 with zero healthy targets; 502 usually means the app answered badly or connection reset.
6. 504 = app too slow; raise idle timeout or fix the app.

### "Containers keep stopping"
See ECS section: **Stopped reason** + CloudWatch logs + ECR pull path (NAT/endpoints) + execution role.

### "Access Denied"
- Which principal? (User, role, task role, instance role.) IAM → the role → check policies.
- Resource policies too: S3 bucket policy, KMS key policy, SQS/SNS policies can deny even if IAM allows.
- **Explicit Deny** anywhere (SCP, permission boundary, policy) always wins.
- Use **IAM Policy Simulator** or **CloudTrail** (Event history → filter by error) to see the exact denied action.

### "Can't reach another resource inside the VPC"
1. Same VPC, or peering/Transit Gateway routes in place?
2. Source SG outbound + **destination SG inbound** allows the port from the source SG.
3. NACLs on both subnets.
4. Use **VPC Reachability Analyzer** (Network Manager → Reachability Analyzer) — it tells you exactly which hop blocks traffic. Learn this tool.
5. **VPC Flow Logs** (enable on VPC/subnet/ENI → CloudWatch) show ACCEPT/REJECT per flow.

### "CloudFormation stack stuck or failed"
See Section 7. First failed event, status reason, fix, redeploy.

### "Where do I find logs?"
- App/OS → **CloudWatch Logs** (needs CloudWatch agent on EC2; automatic with `awslogs` in ECS/Lambda).
- Who did what → **CloudTrail**.
- Network → **VPC Flow Logs**.
- ALB → access logs to S3 (enable on the LB attributes).
- WAF → WAF logs.
- API Gateway → stage execution/access logs.

---

## 9. Security Quick Rules

- MFA on the root account; don't use root day-to-day.
- Roles over access keys, always.
- Least privilege SGs; reference SGs, not CIDRs, for internal traffic.
- Private subnets for anything that isn't a load balancer or NAT.
- Encrypt EBS/S3/RDS (usually a checkbox; KMS default keys are free).
- Enable CloudTrail, GuardDuty, and Config in every account.
- Session Manager instead of open port 22.

---

## 10. Self-Check — can you do these in the console?

- [ ] Build a VPC with public + private subnets, IGW, NAT, and explain each route table.
- [ ] Launch an EC2 instance in the private subnet and connect via Session Manager.
- [ ] Write an SG that allows a web tier to reach a DB tier by SG reference.
- [ ] Allocate an Elastic IP and attach it to an instance; explain why stop/start doesn't change it.
- [ ] Create an ALB → target group → EC2, and diagnose an unhealthy target.
- [ ] Push an image to ECR and run it as a Fargate service behind that ALB.
- [ ] Expand an EBS volume and the filesystem; take a snapshot; restore it in another AZ.
- [ ] Create an S3 bucket with versioning + a lifecycle rule; give an instance read access via a role.
- [ ] Deploy a Lambda behind API Gateway, fix a CORS error, and remember to deploy the stage.
- [ ] Attach a WAF Web ACL with a managed rule set and a rate-based rule to the ALB.
- [ ] Deploy a CloudFormation template, break it on purpose, find the first failed event, fix it.
- [ ] Use Reachability Analyzer to explain why two instances can't talk.

---

## Glossary

**AZ** Availability Zone — one or more data centers in a region · **AMI** machine image · **ENI** elastic network interface (the virtual NIC; SGs attach here) · **CIDR** IP range notation · **IGW** internet gateway · **NAT GW** network address translation gateway · **EIP** elastic IP · **SG** security group · **NACL** network ACL · **ALB/NLB** application/network load balancer · **ASG** auto scaling group · **EBS/EFS/S3** block / file / object storage · **ECR/ECS/EKS** registry / container service / Kubernetes · **IAM** identity & access management · **CFN** CloudFormation · **SSM** Systems Manager (Session Manager lives here) · **WAF** web application firewall · **KMS** key management service.
