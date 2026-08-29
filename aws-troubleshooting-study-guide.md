# AWS Troubleshooting Study Guide

Each topic: **Concept** (why it breaks), **Core** (CLI and console paths), **Example** (a realistic incident, start to finish, ending in a verify), **Trap** (what wastes an hour). Console labels are as of Aug 2026.

**The one rule:** work the five layers top-down and don't skip. **Platform → Identity → Network → Data plane → App.** Most "AWS is down" tickets are a quota, an IAM deny, a VPC path, or a change someone made 20 minutes ago. Prove layers 1–4 before you read application code.

| Layer | Smells like | First tool |
|---|---|---|
| 1 Platform / quota | `InsufficientInstanceCapacity`, `LimitExceeded`, Health event | AWS Health, Service Quotas, CloudTrail |
| 2 Identity | `AccessDenied`, `UnauthorizedOperation`, failed AssumeRole | CloudTrail errorMessage, Policy Simulator |
| 3 Network | timeouts, SSH/RDP fail, PrivateLink 403, DNS wrong IP | Reachability Analyzer, SG/NACL/routes, Flow Logs |
| 4 Data plane | S3 403/301, KMS deny, RDS login fail, throttling | Resource policy, Block Public Access, key policy |
| 5 App / guest OS | 5xx, crash loop, failed status check | Serial console, CloudWatch Logs, X-Ray |

---

## 1. First 60 Seconds

**Concept:** Three health surfaces answer three different questions. Don't confuse them. Then find out what changed.

| Surface | Question | Where |
|---|---|---|
| Service health | Is AWS down for everyone? | health.aws.amazon.com/health/status (no login) |
| Your account health | Did an AWS event hit *my* resources? | Console → **AWS Health** |
| Resource signals | Is *this* resource sick? | EC2 Status checks; RDS/ELB/Lambda console + CloudWatch |

```bash
aws sts get-caller-identity                                  # who am I, which account — ALWAYS first
aws health describe-events                                   # account events (Business+ support)
aws cloudtrail lookup-events --max-items 20                  # what just happened (90 days, per Region)
aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=i-0123
aws cloudtrail lookup-events --lookup-attributes AttributeKey=Username,AttributeValue=alice
aws service-quotas list-service-quotas --service-code ec2
```
**Console:** CloudTrail → Event history → filter **Error code = AccessDenied / Client.Failed** → open event → copy *Event name, User identity, Source IP, Request ID, Error message*. Systems Manager → Automation → **Owned by Amazon** → search `AWSSupport-` for a runbook that matches the symptom.

**Decision rule:** Health lists your resource on an AWS event → stop changing config, follow the event's remediation, open a case. Health clean and CloudTrail shows no failed API → it's IAM, VPC, quota, or your code.

**Example — "The app broke at 14:05, nobody touched anything":**
```bash
aws cloudtrail lookup-events --start-time 2026-08-29T13:45:00Z --end-time 2026-08-29T14:10:00Z \
  --query 'Events[].{t:EventTime,n:EventName,u:Username}' --output table
# → 13:58 AuthorizeSecurityGroupIngress / RevokeSecurityGroupIngress by "deploy-role"
```
Verify: the offending event's *Request parameters* show exactly which rule changed. Somebody always touched something.

**Trap:** CloudTrail is **per Region**; IAM/STS events land in **us-east-1**. Event history is management events only — S3 `GetObject` and Lambda `Invoke` need a trail with data events. Wrong Region = empty list, not an error.

---

## 2. IAM — AccessDenied Playbook

**Concept:** Evaluation order: explicit Deny anywhere wins → SCP (Organizations) → permissions boundary → session policy → identity policy → resource policy. Cross-account needs **both** sides to allow. Many services (S3, KMS, SQS, SNS, Lambda, Secrets Manager) need a **resource policy** in addition to IAM.

```bash
aws sts get-caller-identity                                    # wrong profile is the #1 cause
aws sts decode-authorization-message --encoded-message <msg>   # EC2 encoded denials
aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names s3:GetObject --resource-arns <arn>
aws iam get-role --role-name X ; aws iam list-attached-role-policies --role-name X
```
**Console:** IAM → **Policy simulator** (pick principal, action, resource ARN, Run). Role → **Permissions boundary** tab. Organizations → Policies → SCPs. Resource → Permissions / Bucket policy / Key policy tab.

**Read the error literally:** "with an explicit deny in a *service control policy*" vs "because no *identity-based policy* allows" — the message names the layer.

**Example — "Lambda works from my laptop but the deploy pipeline gets AccessDenied creating it":**
```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateFunction
# errorMessage: "...is not authorized to perform: iam:PassRole on resource: arn:aws:iam::123:role/lambda-exec"
aws iam simulate-principal-policy --policy-source-arn arn:aws:iam::123:role/pipeline \
  --action-names iam:PassRole --resource-arns arn:aws:iam::123:role/lambda-exec     # → implicitDeny
```
Fix: add `iam:PassRole` on that role ARN to the pipeline role. Verify: re-run simulate → `allowed`; re-run pipeline.

**Trap:** `iam:PassRole` bites every EC2/Lambda/ECS launch. A brand-new role or policy takes seconds-to-minutes to propagate — retry before rewriting. Check the *execution role* for Lambda/ECS errors, not your own user. Condition keys (`aws:SourceIp`, `aws:RequestedRegion`, MFA) deny silently.

---

## 3. VPC Networking

**Concept:** Security groups are **stateful** (return traffic auto-allowed) and attach to ENIs; NACLs are **stateless** (need both directions incl. ephemeral 1024–65535) and attach to subnets, first-match-wins by rule number. Route table decides IGW vs NAT vs TGW vs blackhole.

Decision tree — walk in order:
1. ENI / public IP / EIP attached? Instance running with passing status checks?
2. DNS resolves to the IP you think? (Route 53 private hosted zone associated to *this* VPC?)
3. Route table on the subnet: `0.0.0.0/0` → IGW (public), NAT (private), TGW, or nothing?
4. NACL both directions + every SG on the ENI + guest OS firewall.
5. Is the destination listening? (`ss -lnt`, ALB target health, RDS "available" ≠ reachable)
6. Cross-VPC/account: peering/TGW routes on **both** sides + SG referenced-group rules.
7. VPC Block Public Access or Network Firewall can deny what SGs allow.

```bash
aws ec2 create-network-insights-path --source <eni-or-igw> --destination <eni> --protocol tcp --destination-port 22
aws ec2 start-network-insights-analysis --network-insights-path-id nip-…   # then describe-network-insights-analyses
aws ec2 describe-route-tables --filters Name=vpc-id,Values=vpc-…
aws ec2 describe-security-groups --group-ids sg-…
aws ec2 describe-network-acls --filters Name=vpc-id,Values=vpc-…
```
**Console:** VPC → **Reachability Analyzer** → Create and analyze path → open analysis → **Explanations** names the blocking component. VPC → Your VPCs → **Flow logs** tab → read in CloudWatch Logs Insights (`REJECT` = SG or NACL; *no entry at all* = routing/DNS). VPC → NAT gateways → Monitoring (`PacketsDropCount`).

**Example — "Private-subnet app can't reach the internet":**
```bash
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=subnet-priv
# → 0.0.0.0/0 → nat-0abc   (good)
aws ec2 describe-nat-gateways --nat-gateway-ids nat-0abc --query 'NatGateways[].SubnetId'
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<that subnet>
# → no 0.0.0.0/0 → igw   ← NAT is sitting in a private subnet
```
Fix: NAT must live in a subnet whose route table sends `0.0.0.0/0` to the IGW. Verify: `curl -m5 https://example.com` from the instance via Session Manager.

**Trap:** NAT Gateway port exhaustion looks like *random* outbound timeouts. Interface VPC endpoints have their own **endpoint policy** (a second IAM layer) and need private DNS on. Ephemeral-port outbound rule missing on the NACL = SYN gets in, SYN-ACK never leaves.

---

## 4. EC2

**Concept:** Two status checks. **System** = AWS host/hypervisor → stop/start migrates you to a new host. **Instance** = guest OS not responding → your problem (fstab, sshd, disk full, kernel). Don't SSH-debug a host with a failed system check.

```bash
aws ec2 describe-instance-status --instance-ids i-…
aws ec2 get-console-output --instance-id i-… --latest          # boot log
aws ssm start-session --target i-…                             # shell with no inbound 22
aws ssm send-command --document-name AWS-RunShellScript --instance-ids i-… --parameters commands="ss -lnt"
aws ec2 describe-instance-attribute --instance-id i-… --attribute userData
aws cloudwatch get-metric-statistics … --metric-name CPUCreditBalance   # T-family throttling
```
**Console:** Instance → **Status checks** tab · Actions → Monitor and troubleshoot → **Get system log / Get instance screenshot** · Connect → **EC2 serial console** (out-of-band, works when network is dead) · Connect → **Session Manager** · Systems Manager → Automation → `AWSSupport-TroubleshootSSH` / `TroubleshootRDP` / `TroubleshootSessionManager` / `ExecuteEC2Rescue` · EC2 → **Events** for scheduled retirements.

**Won't launch:** CloudTrail `RunInstances` innermost error. `InsufficientInstanceCapacity` → other AZ/family. `VcpuLimitExceeded` → Service Quotas → EC2. Encrypted AMI → KMS key access. SCP deny.

**Can't SSH:** SG allows 22 from your IP? Public IP + IGW route? Right username (`ec2-user` / `ubuntu` / `admin`)? Then Reachability Analyzer IGW → ENI :22. Prefer Session Manager anyway (needs SSM agent + instance profile + `ssm`/`ssmmessages`/`ec2messages` endpoints or NAT).

**Example — instance status check failed after a reboot, no SSH:**
```bash
aws ec2 get-console-output --instance-id i-… --latest --output text | tail -40
# → "mount: /data: can't find UUID=…"  → dropped to emergency shell; bad /etc/fstab
```
Fix path: Connect → EC2 serial console → log in → fix fstab → reboot. If no serial console: stop → snapshot root volume → detach → attach to rescue instance → edit → reattach. Verify: status checks 2/2 passing, `aws ssm start-session` works.

**Trap:** Stop/start moves hosts and **changes the public IP** unless it's an Elastic IP. Snapshot before you write to a detached root volume. T-family: `CPUCreditBalance` at 0 = "mysteriously slow"; gp2 `BurstBalance` drained → move to gp3.

---

## 5. Lambda & Async Pipelines

**Concept:** Lambda never has a public IP. In a VPC it needs NAT or VPC endpoints to reach anything. Errors are the *execution role's* permissions. Every invoke writes `START / END / REPORT` to `/aws/lambda/<fn>`.

```bash
aws logs tail /aws/lambda/my-fn --follow
aws lambda get-function-configuration --function-name my-fn      # Timeout, MemorySize, VpcConfig, Role
aws lambda get-policy --function-name my-fn                       # resource-based policy (who may invoke)
aws lambda list-event-source-mappings --function-name my-fn       # trigger State: Enabled?
aws cloudwatch get-metric-statistics … --metric-name Throttles
```
**Console:** Function → **Monitor** (Errors / Duration / Throttles / Concurrent executions) · Configuration → General (Timeout), VPC, Permissions (role link + resource-based policy), Concurrency, Triggers · X-Ray traces tab · Systems Manager Automation → `AWSSupport-TroubleshootLambdaS3Event`.

| Symptom | Likely cause | Look |
|---|---|---|
| Timeout | default 3 s too low; VPC with no NAT; downstream hang | Duration vs Timeout; VpcConfig; X-Ray |
| AccessDenied / KMS | execution role, resource policy, key policy | role + downstream CloudTrail |
| 429 Throttles | reserved concurrency 0 or account cap (1,000/Region) | Throttles metric; Concurrency config; Service Quotas |
| Never invoked | trigger disabled, resource policy missing, event filter | Triggers tab; `get-policy` |
| Deploy fails | zip too big; S3 bucket in another Region; `iam:PassRole` | CloudTrail `UpdateFunctionCode` |
| Async fails silently | no DLQ / destination | Configure on-failure destination |

**Example — function in a VPC times out calling DynamoDB:**
```bash
aws lambda get-function-configuration --function-name fn --query 'VpcConfig.SubnetIds'
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet>   # → no NAT route
```
Fix: add a DynamoDB **gateway endpoint** to the route table (free, no NAT needed). Verify: `aws logs tail` shows `REPORT Duration: 120 ms` instead of `Task timed out`.

**Trap:** Memory maxed in the `REPORT` line → raise memory (also raises CPU). Package bucket must be in the function's Region. Reserved concurrency = 0 is a silent "off switch."

---

## 6. S3 & KMS

**Concept:** S3 403 has four buckets of causes: (1) IAM identity vs bucket policy vs ACLs/Object Ownership; (2) **Block Public Access** (bucket *and* account) + SCP; (3) wrong Region → **301 PermanentRedirect**; (4) VPC gateway endpoint policy / private DNS. KMS: the **key policy** is the root of trust — IAM alone can't grant use of a key.

```bash
aws s3 ls s3://bucket/prefix/ --recursive            # exact key? case-sensitive, trailing slash/space
aws s3api get-bucket-location --bucket b
aws s3api get-bucket-policy --bucket b ; aws s3api get-public-access-block --bucket b
aws s3api get-bucket-cors --bucket b                 # CORS is browser-only; CLI never sees it
aws kms get-key-policy --key-id … --policy-name default
```
**Console:** S3 bucket → **Permissions** tab (policy, BPA, Object Ownership, ACLs) · Overview shows **AWS Region** · IAM → **Access Analyzer for S3** shows who can really reach it. KMS → Customer managed keys → key → **Key policy** + **Grants**.

**Example — "403 on GetObject, but the IAM policy clearly allows s3:GetObject":**
```bash
aws s3api head-object --bucket b --key k --query ServerSideEncryption   # → aws:kms
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=Decrypt
# → AccessDenied on kms:Decrypt for the role
```
Fix: add the role to the key policy (or a grant) with `kms:Decrypt`. Verify: `aws s3 cp s3://b/k -` succeeds.

**Trap:** 403 on a *missing* object usually means you lack `s3:ListBucket` (bucket-level), not `GetObject`. Multi-Region key replicas are different ARNs. `kms:ViaService` conditions restrict which service may use the key. Object-level `GetObject` won't show in Event history without a data-events trail.

---

## 7. RDS / Aurora

**Concept:** "Available" ≠ reachable. Connectivity is SG on the DB **and** on the client, subnet group placement, `Publicly accessible` flag, and the right endpoint (Aurora **writer** vs **reader**). Storage-full and max-connections *look* like network failures.

```bash
aws rds describe-db-instances --db-instance-identifier db --query 'DBInstances[].{s:DBInstanceStatus,pub:PubliclyAccessible,sg:VpcSecurityGroups,ep:Endpoint}'
aws rds describe-events --duration 1440                    # failovers, maintenance, storage-full
aws rds describe-db-log-files --db-instance-identifier db   # then download-db-log-file-portion
```
**Console:** RDS → Databases → **Connectivity & security** (endpoint, VPC, subnet group, SGs, port) · **Logs & events** · **Monitoring** (`FreeableMemory`, `DatabaseConnections`, `FreeStorageSpace`, Read/WriteIOPS) · **Performance Insights** (top SQL by load).

**Example — "connections started timing out at 3 pm, DB shows available":**
```bash
aws rds describe-events --duration 180 --source-identifier db --source-type db-instance
# → "The free storage capacity for DB Instance: db is low" then "storage-full"
```
Fix: increase allocated storage / enable storage autoscaling; find the runaway (temp tables, logs). Verify: status back to `available`, writes succeed, `FreeStorageSpace` climbing.

**Trap:** `Publicly accessible = No` → connect from inside the VPC (bastion, SSM port-forward, VPN). Maxed `DatabaseConnections` → connection pooling / RDS Proxy, not a bigger instance. Password expired vs IAM DB auth — check the auth mode before blaming the SG.

---

## 8. ECS / EKS

**Concept:** Control plane first (CloudTrail on `CreateNodegroup` / `RunTask`), then Kubernetes/ECS events. Node/task can't start = capacity, subnet IPs, IAM on the node/task role, or no route to ECR/S3/cluster API.

```bash
# ECS
aws ecs describe-tasks --cluster c --tasks <id> --query 'tasks[].{s:lastStatus,r:stoppedReason,c:containers[].reason}'
aws logs tail /ecs/<task-family> --follow
# EKS
aws eks update-kubeconfig --name cluster && aws sts get-caller-identity   # this identity must be in access entries / aws-auth
kubectl get nodes ; kubectl get pods -A
kubectl describe pod <p> -n <ns>
kubectl logs <p> -n <ns> --previous
kubectl get events -A --sort-by=.metadata.creationTimestamp
```
**Console:** ECS → cluster → Tasks → **Stopped reason**. EKS → cluster → **Observe / Health issues**, **Access** tab (access entries), **Resources** tab; CloudWatch Container Insights; Automation → `AWSPremiumSupport-TroubleshootEKSCluster`.

| Symptom | Cause | Fix |
|---|---|---|
| Task PENDING forever | no capacity, image pull fail | Stopped reason; instance CPU/mem |
| `CannotPullContainerError` | task *execution* role lacks ECR; private subnet has no NAT/endpoints | add `ecr.api`, `ecr.dkr`, `s3` endpoints |
| Container exits at once | bad entrypoint, missing env var | awslogs driver → CloudWatch |
| EKS `kubectl` Unauthorized | caller not mapped | Access tab / aws-auth ConfigMap |
| Node NotReady | node role missing `AmazonEKSWorkerNodePolicy`/CNI/ECR ReadOnly, subnet IPs exhausted | IAM; VPC → Subnets → Available IPv4 |

**Example — pod `ImagePullBackOff` on a private-subnet node:**
```bash
kubectl describe pod p -n app | tail -5   # → "dial tcp …ecr… i/o timeout"
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=vpc-… --query 'VpcEndpoints[].ServiceName'
# → only s3; no ecr.api / ecr.dkr
```
Fix: create interface endpoints `com.amazonaws.<region>.ecr.api` and `.ecr.dkr` with private DNS + SG allowing 443 from nodes. Verify: `kubectl get pods` → Running.

**Trap:** ECS has two roles — **task role** (what the app can do) vs **task execution role** (pull image, write logs). `OOMKilled` = limits too low, not a crash bug. `Pending` with "Insufficient cpu" = requests vs allocatable, taints, or PVC in the wrong AZ.

---

## 9. CloudFormation

**Concept:** Read Events **bottom-up**: the *first* `CREATE_FAILED` is the cause; everything after is rollback noise.

```bash
aws cloudformation describe-stack-events --stack-name s --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`].[LogicalResourceId,ResourceStatusReason]'
aws cloudformation continue-update-rollback --stack-name s --resources-to-skip <LogicalId>
aws cloudformation detect-stack-drift --stack-name s
```
**Console:** Stack → **Events** (sort by time, scroll to the bottom) · Stack actions → Continue update rollback · Drift → Detect.

**Trap:** `UPDATE_ROLLBACK_FAILED` → "Continue update rollback," skipping the broken resource. Delete stuck → non-empty S3 bucket, ENIs held by Lambda-in-VPC, deletion protection on RDS/DynamoDB.

---

## 10. Error Codes on Repeat

| Code | Means | First move |
|---|---|---|
| `AccessDenied` / `UnauthorizedOperation` | IAM/SCP/resource-policy deny | CloudTrail errorMessage → Policy Simulator → `get-caller-identity` |
| `UnrecognizedClientException` / `AuthFailure` | bad/expired keys, wrong Region, clock skew | rotate keys; check system time; Region |
| `ExpiredToken` | assumed-role / instance-profile creds expired | re-assume; IMDS hop limit on containers |
| `InsufficientInstanceCapacity` | AZ has none of that family | other AZ / family |
| `VcpuLimitExceeded` / `InstanceLimitExceeded` | account quota | Service Quotas → request |
| `InvalidInstanceID.NotFound` | wrong Region or terminated | Region picker |
| `OptInRequired` | service/Region not enabled | Account settings |
| `Throttling` / `RequestLimitExceeded` / 429 | API rate | backoff + jitter; Service Quotas |
| `KMS.AccessDeniedException` | key policy / grant | key policy principal; `kms:ViaService` |
| `301 PermanentRedirect` (S3) | wrong Region endpoint | `get-bucket-location` |
| `TargetNotConnected` (SSM) | agent offline, no endpoint, bad profile | `AWSSupport-TroubleshootSessionManager` |
| `RequestDisallowedByPolicy`-style SCP text | Organizations SCP | Organizations → Policies |

---

## 11. Logging You Should Already Have On

| Signal | Catches | Send to |
|---|---|---|
| CloudTrail org trail | every API call incl. failed auth, >90 d, data events | log-archive account + CloudTrail Lake |
| CloudWatch metrics + alarms | CPU, 5xx, throttles, status checks | SNS / Chatbot / Incident Manager |
| CloudWatch Logs | Lambda, ECS, RDS, OS (agent) | set retention per group |
| VPC Flow Logs | ACCEPT/REJECT per ENI | Logs Insights or S3 + Athena |
| AWS Health → EventBridge | account events, scheduled changes | rule → SNS / ticket |
| ALB / WAF / CloudFront logs | 4xx/5xx, TLS, WAF blocks | S3 + Athena |
| GuardDuty + Security Hub | compromised creds, unusual API | findings → ticket (not a CloudTrail substitute) |

---

## 12. Cost Spikes

Cost Explorer → group by **service** → then **usage type** → filter to the spike day. Usual suspects: NAT Gateway data processing, cross-AZ/Region transfer, orphaned EBS volumes/snapshots, idle load balancers, CloudWatch Logs ingestion, hot S3 request patterns. Set **AWS Budgets** + billing alarms so you learn in hours, not at month-end.

---

## 13. Escalation Packet

Open a case when Health lists your resource on an AWS event, it's a Sev-1, a runbook says to, or layers 1–5 still look like platform (capacity in every AZ, control-plane 500s, impaired system status that stop/start won't clear). Console: Support Center → Create case, or straight from the Health event.

Attach: account ID · Region · ARNs · UTC timestamps · CloudTrail **eventID + requestID + full errorMessage JSON** · Health event ARN · Reachability Analyzer analysis ID / SSM execution ID · system log or screenshot · what you already tried · impact and workaround. Reproduce with `aws … --debug` to rule out console weirdness. Health API and full Trusted Advisor need Business+ support; Basic still gets Service health, Event history, runbooks, and Service Quotas.

---

## Pocket Checklist

Service health → Your account health → resource status checks / CloudWatch → CloudTrail (error code + request ID, right Region) → `AWSSupport-*` runbook → Reachability Analyzer / Policy Simulator / Service Quotas → guest or app logs → support packet.

**Console chrome:** top search bar takes resource IDs directly (`i-…`, `vpc-…`, bucket name) · Region picker top-right (flip to us-east-1 for IAM events) · account menu top-right confirms who you are before "impossible" AccessDenied · pin Health, CloudTrail, VPC, IAM, EC2, CloudWatch, Systems Manager, Service Quotas.
