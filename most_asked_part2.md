# Accenture Interview Prep — DevOps, Kubernetes & AWS

Interview question bank with detailed explanations, diagrams, commands and
ready-to-speak answers. Source notes: [`accenture_questions`](./accenture_questions).

Each topic follows the same shape: **what it is → how it works → how to
troubleshoot → what to say in the interview.**

---

## Table of Contents

**Git**
1. [Undoing a Bad Commit Pushed to `main`](#1-undoing-a-bad-commit-pushed-to-main)

**Kubernetes Operations**
2. [Node `DiskPressure`](#2-node-diskpressure)
3. [Pod Error States — Quick Reference](#3-pod-error-states--quick-reference)

**AWS**
4. [How AWS Pre-Signed URLs Work Internally](#4-how-aws-pre-signed-urls-work-internally)
5. [Secure Three-Tier AWS Architecture](#5-secure-three-tier-aws-architecture)

**Autoscaling**
6. [Cluster Autoscaler — Scale Up and Scale Down](#6-cluster-autoscaler--scale-up-and-scale-down)

**Kubernetes Security**
7. [Service Accounts vs Users](#7-service-accounts-vs-users)
8. [Pod Security Policy (PSP) vs Pod Security Standards (PSS)](#8-pod-security-policy-psp-vs-pod-security-standards-pss)
9. [Securing etcd](#9-securing-etcd)
10. [Secrets: Base64, Encryption at Rest, and Secrets Manager](#10-secrets-base64-encryption-at-rest-and-secrets-manager)

**Observability**
11. [How Do You Monitor Kubernetes?](#11-how-do-you-monitor-kubernetes)

**Reference**
- [Command Cheat Sheet](#command-cheat-sheet)
- [Key Takeaways](#key-takeaways)

---

# 1. Undoing a Bad Commit Pushed to `main`

> **Question:** How do you undo a bad commit that has already been pushed to the
> main branch?

**Short answer:** use `git revert`, not `git reset`. Revert undoes the *changes*
without rewriting *shared history*.

### 1.1 Find the bad commit

```bash
git log --oneline
```

```
a8f31c2 Fix payment timeout      <- bad commit
72bc991 Add monitoring
31de452 Initial setup
```

### 1.2 Revert it

```bash
git revert a8f31c2
git push origin main
```

Git creates a **new commit** that reverses the changes introduced by `a8f31c2`:

```
Before:                 After:
A → B → C → D           A → B → C → D → E
            ↑                       ↑   ↑
          bad                     bad  revert
```

The bad commit stays in history, but its changes are undone.

**Multiple bad commits:**

```bash
git revert <oldest-commit>^..<newest-commit>
```

Or revert them individually, depending on the situation.

> **Secrets warning:** if the bad commit contains a password or API key,
> `git revert` is **not enough** — the secret still exists in history. You must
> **rotate the secret** and, if required, purge it from history.

---

### 1.3 What `git reset` actually does

Suppose `main` looks like this and everyone has already pulled `D`:

```
A --- B --- C --- D
                  ↑
              bad commit
```

| Commit | Change |
| --- | --- |
| `A` | Initial code |
| `B` | Add login |
| `C` | Add database |
| `D` | **Bad database configuration** |

Now you run:

```bash
git reset --hard HEAD~1
```

`HEAD~1` means *move HEAD one commit backward*:

```
Before:                      After:
A --- B --- C --- D          A --- B --- C --- D
                  ↑                      ↑
                 HEAD                  HEAD
```

Your **local branch pointer** moved backward. The remote did not:

```
Local:                       Remote:
A --- B --- C                A --- B --- C --- D
            ↑                                  ↑
           main                           origin/main
```

Local and remote history now diverge.

---

### 1.4 Why you then need `git push --force`

```bash
git push origin main
```

Git rejects it:

```
! [rejected] main -> main (non-fast-forward)
```

Why? The remote contains `D`, which your local branch no longer has. A normal
push is designed to **add** commits, not **remove** them.

```bash
git push --force origin main
```

This tells Git: *"I know the remote has commits mine doesn't. Point the remote
branch where my local branch points."*

```
A --- B --- C
            ↑
      origin/main
       local main
```

Commit `D` is no longer reachable from `main`. **That is rewriting history.**

---

### 1.5 Why rewriting shared history is dangerous

Another developer, Alice, already pulled `D`:

```
Alice:                       Remote after your force push:
A --- B --- C --- D          A --- B --- C
                  ↑                      ↑
                main                origin/main
```

Two different realities now exist. Your repo says `main → C`; Alice's says
`main → D`.

When Alice commits `E` and pushes:

```
Alice:  A → B → C → D → E
Remote: A → B → C
```

Git rejects the push — the histories no longer line up. She has to recover with
something like:

```bash
git fetch
git rebase origin/main
```

This is the core problem with rewriting history on a shared branch.

---

### 1.6 `reset` vs `revert` side by side

```
reset                        revert
-----                        ------
A → B → C → D                A → B → C → D → E
       ↓ RESET                           bad   undo
A → B → C
```

| | `git reset` | `git revert` |
| --- | --- | --- |
| What it changes | Moves the branch pointer backward | Adds a new commit |
| History | Rewritten | Preserved |
| Push type | Requires `--force` | Normal push |
| Safe on shared branches | **No** | **Yes** |
| Best for | Local, unshared commits | Anything already pushed |

Say it this way:

- **reset** → *"Pretend the branch never moved to D."*
- **revert** → *"D happened, but here's a commit that undoes D."*

---

### 1.7 Why Git cares so much about history

Each commit stores a reference to its parent:

```
D
│
parent → C
         │
         parent → B
                  │
                  parent → A
```

So the chain is `A ← B ← C ← D`. Creating a new commit `E` whose parent is `D`
gives you `A → B → C → D → E`.

Git is fundamentally an **append-only** history model. `git revert` follows that
model — it *adds* `E` rather than *modifying* `D`.

A branch such as `main` is just a **pointer to a commit**. `git reset` moves
that pointer; the commits themselves aren't immediately destroyed. That's why we
say reset *rewrites the branch's history* while revert keeps `main` moving
forward.

---

### 1.8 Real-world DevOps scenario

You manage Kubernetes infrastructure. Commit `D` changes:

```yaml
replicas: 3     ->     replicas: 100
```

You merge it to production. Kubernetes starts creating 100 pods. You must undo
it immediately.

You *could* do:

```bash
git reset --hard HEAD~1
git push --force
```

But 10 engineers may already have `D` locally. Instead:

```bash
git revert D
git push origin main
```

Result: `A → B → C → D → E`, where `E` restores `replicas: 3`. Everyone just
runs `git pull` and their history stays consistent.

---

### 1.9 Why force push is especially dangerous on `main`

```
                 Developer A
                      ↓
A → B → C → D → E → F
              ↑
          Developer B
```

If Developer A force-pushes a reset back to `C`, commits `D`, `E` and `F` may
become unreachable from `main` — and other developers may have branched off
them:

```
D
├── E
│   └── F
└── G
```

You risk disconnecting an entire section of collaborative history.

**That's why organizations protect `main`:**

```
main
 ├── Pull request required
 ├── Code review required
 ├── CI must pass
 ├── No direct push
 └── Force push disabled
```

Even if you try, the server rejects it:

```
remote: You are not allowed to force push to protected branch 'main'
```

---

### 1.10 The production workflow

With a protected `main`, the real-world flow is:

```
Create revert branch
       ↓
git revert <bad-commit>
       ↓
Run tests
       ↓
Create Pull Request
       ↓
CI/CD
       ↓
Code review
       ↓
Merge into main
```

---

### 1.11 Is `git reset` always bad?

**No** — and this nuance matters in interviews.

`git reset` is perfectly fine on a **local branch whose commits you haven't
shared**:

```
local:  A → B → C → D
                    ↑ unwanted, nobody pulled it
```

```bash
git reset --hard HEAD~1     # now: A → B → C — no problem
```

You don't need to preserve history nobody else depends on.

**The real rule — don't memorize "reset is bad". Memorize:**

> `git reset` is for rewriting **local** history; `git revert` is for undoing
> changes **already shared** with others.

---

### 1.12 `--force` vs `--force-with-lease`

| Flag | Meaning |
| --- | --- |
| `--force` | "Overwrite the remote branch with my version." |
| `--force-with-lease` | "Overwrite the remote **only if it is still in the state I expect**." |

Example: you last fetched `A → B → C` and prepare to force push. Meanwhile
another developer pushes `D`.

- `git push --force` → you silently **overwrite `D`**.
- `git push --force-with-lease` → Git detects the remote changed and **rejects**
  your push.

---

### 1.13 Interview-ready answer

> Since the commit is already pushed to a shared `main` branch, I would normally
> use `git revert <commit-sha>`. Revert creates a new commit that reverses the
> changes while preserving the existing history. I would avoid `git reset --hard`
> followed by a force push, because reset moves the branch pointer backward and
> force push rewrites the shared branch history — other developers may already
> have based work on that commit, which creates divergence and lost commits.
> If the bad commit is only on my local branch and hasn't been shared, then
> reset is appropriate. For exceptional cases where history absolutely must be
> rewritten, I would use `--force-with-lease` rather than an unconditional
> `--force`, and coordinate with the team.

That answer demonstrates Git internals + collaboration + production practice,
rather than memorized commands.

---

# 2. Node `DiskPressure`

> **Question:** How does Kubernetes detect `DiskPressure`, how do you
> troubleshoot it, and how do you prevent it?

### 2.1 What is DiskPressure?

Kubernetes reports `DiskPressure=True` when a node's available **disk space** or
**inodes** fall below configured eviction thresholds.

Common causes:

- Container logs consuming excessive disk space
- Old or unused container images
- High container filesystem usage (temporary files)
- Deleted files still held open by running processes
- Insufficient disk capacity or inode exhaustion

Check the node condition:

```bash
kubectl get nodes
kubectl describe node <node-name>
```

Look for:

```
Conditions:
  Type          Status
  DiskPressure  True
```

---

### 2.2 Step-by-step troubleshooting

#### Step 1 — Check disk and inode usage

SSH into the node, or use your approved node-debugging method.

```bash
df -h      # full filesystems and mount points
df -i      # inode exhaustion
lsblk      # block devices and sizes
```

#### Step 2 — Find directories consuming the most space

```bash
sudo du -xhd1 /var      2>/dev/null | sort -h
sudo du -xhd1 /var/lib  2>/dev/null | sort -h
sudo du -ahx /var/log   2>/dev/null | sort -rh | head -20
```

Common locations to investigate:

| Path | Contents |
| --- | --- |
| `/var/log/` | System logs |
| `/var/log/pods/` | Pod logs |
| `/var/log/containers/` | Container log links |
| `/var/lib/containerd/` | Container runtime data |
| `/var/lib/kubelet/` | Pod and volume-related data |

Exact paths depend on the node OS and container runtime.

#### Step 3 — Check deleted files still consuming disk

A file may be deleted from the directory but still held open by a process. Its
disk blocks are **not released** until the final open file descriptor is closed.

```bash
sudo lsof +L1
```

Identify the process and file descriptor. If the application still needs the
file, use a safe log-rotation or reopen procedure. Restart the relevant service
only when appropriate and approved.

> **Important:** `lsof` *identifies* the issue; it does not itself release disk
> space. Avoid blindly killing processes, especially critical Kubernetes
> services.

#### Step 4 — Clean up safely

```bash
sudo crictl images
sudo crictl ps -a
sudo crictl rmi --help
```

Remove only **confirmed-unused** images using supported runtime cleanup
mechanisms. Investigate oversized logs and configure log rotation.

> **Do not** manually delete files inside containerd or kubelet internal
> directories — this can corrupt runtime state or pod data.

#### Step 5 — Verify recovery

```bash
df -h
df -i
kubectl describe node <node-name>
kubectl get nodes
```

Confirm disk availability has recovered and `DiskPressure` becomes `False`. The
kubelet may take some time to clear the condition.

---

### 2.3 What if disk usage is still high?

If cleanup is insufficient, investigate whether the underlying disk needs
expansion:

- **Cloud VM:** expand the attached disk, then extend the partition and
  filesystem if required.
- **Kubernetes worker node:** follow the managed node group's or platform's
  disk-resizing procedure.
- **Persistent volume:** expand the PVC and storage backend where supported.
  *This is different from expanding the node's root filesystem.*

Also consider adding worker nodes or redistributing workloads if node-local
ephemeral storage demand is consistently too high.

---

### 2.4 Preventing recurrence

| Area | Preventive action |
| --- | --- |
| Container logs | Configure rotation and retention limits |
| Ephemeral storage | Set pod requests and limits for `ephemeral-storage` |
| Container images | Remove unused images through supported garbage collection |
| Monitoring | Alert on disk space, inode usage and eviction events |
| Capacity planning | Size node disks for expected workload and log volume |
| Workload behavior | Investigate applications writing excessive temporary files |

Example ephemeral storage limits in a container spec:

```yaml
resources:
  requests:
    ephemeral-storage: "1Gi"
  limits:
    ephemeral-storage: "2Gi"
```

These values are examples; size them according to the workload.

---

### 2.5 Interview-ready answer

> First, I check the node condition using `kubectl describe node` and confirm
> whether `DiskPressure` is true. Then I log into the worker node and use `df -h`
> and `df -i` to identify disk or inode exhaustion.
>
> Next, I use `du` to find directories consuming the most space, especially
> container logs, runtime storage and temporary files. I also use `lsof +L1` to
> identify deleted files still held open by processes.
>
> Based on the findings, I safely clean up unused images, address excessive logs,
> or restart the relevant application through an approved procedure. If the
> underlying disk capacity is insufficient, I expand the disk or replace the node
> according to the platform's procedure.
>
> Finally, I verify that disk utilization has decreased and the node's
> `DiskPressure` condition has returned to false. To prevent recurrence, I
> configure log rotation, ephemeral storage requests and limits, monitoring
> alerts, and appropriate node capacity.

> **One important interview point:** Kubernetes can **evict pods** when a node
> experiences disk pressure. So I would also check pod eviction events and
> kubelet eviction thresholds, rather than treating it only as a Linux
> disk-cleanup issue.

---

# 3. Pod Error States — Quick Reference

| Kubernetes status | Typical cause | What to check |
| --- | --- | --- |
| `ErrImagePull` | Image missing, invalid tag, authentication failure | Registry, image name, `imagePullSecrets` |
| `ImagePullBackOff` | Repeated image-pull failures | Pod events and registry access |
| `CreateContainerConfigError` | Missing Secret or ConfigMap, invalid container configuration | Environment references and configuration |
| `CreateContainerError` | Container creation failed at runtime | Mounts, runtime errors, security settings |
| `ContainerCreating` | Container setup still in progress or blocked | Volumes, CNI, image pulling, events |
| `CrashLoopBackOff` | Container starts and repeatedly exits | Application logs, command and exit code |

---

# 4. How AWS Pre-Signed URLs Work Internally

**Scenario:** an application must let a user download a private file from Amazon
S3 **without giving the user AWS credentials**.

### 4.1 The flow

```
1. Client requests a download
   Application asks its backend for access to a private S3 object
                 ↓
2. Backend signs the request
   Uses AWS credentials and Signature Version 4 (SigV4)
                 ↓
3. Backend returns a pre-signed URL
   Contains signature, credential scope, expiry and signing parameters
                 ↓
4. Client sends HTTPS request to S3
   The client presents the URL; it never needs the secret access key
                 ↓
5. S3 validates the signature
   Checks request integrity, expiry, credentials and permissions
```

If the checks pass, S3 allows the requested operation — provided all other
authorization conditions are satisfied.

---

### 4.2 Internals: Signature Version 4

Suppose the URL grants permission to download:

```
s3://company-documents/reports/report.pdf
```

**Step 1 — Create a canonical request.** AWS builds a standardized
representation of the HTTP request including:

- HTTP method (e.g. `GET`)
- Canonical URI identifying the object
- Canonical query parameters
- Relevant headers
- A payload hash, or the appropriate unsigned-payload representation

**Step 2 — Create the string to sign.** The canonical request is hashed and
combined with the signing algorithm, timestamp and credential scope.

**Step 3 — Derive the signing key.** SigV4 derives a signing key from the AWS
secret access key using HMAC-SHA256 plus the date, Region, service name and
signing terminator:

```
AWS secret access key
        |
        v
Date-based signing key
        |
        v
Region-based signing key
        |
        v
Service-based signing key
        |
        v
Final signing key
```

**Step 4 — Generate the signature.** AWS calculates an HMAC-SHA256 signature
over the string to sign. The resulting URL contains parameters such as:

```
X-Amz-Algorithm
X-Amz-Credential
X-Amz-Date
X-Amz-Expires
X-Amz-SignedHeaders
X-Amz-Signature
```

If temporary credentials are used, the URL also needs the **session token**.

**Step 5 — Validate the request.** When the client uses the URL, S3 validates
the signature against the request and checks that the signing credentials and
permissions are still valid, and that the URL is within its validity period.

> **The secret access key is never included in the URL.**

---

### 4.3 Pre-signed URL vs HTTPS/TLS

| Feature | AWS pre-signed URL | HTTPS/TLS |
| --- | --- | --- |
| Main purpose | Authorize a specific AWS request temporarily | Secure communication over the network |
| Uses | AWS credentials + cryptographic signature | Server certificate + TLS session keys |
| Client certificate required? | No, not normally | No, except with mutual TLS |
| Validates | Signature, expiry and authorization | Server identity, secure session establishment |
| Protects data in transit? | Not by itself | Yes, when properly configured |

The two work **together**: TLS protects the connection, the pre-signed URL
provides temporary request authorization.

---

### 4.4 Expiry and security

A pre-signed URL is a **bearer credential** — anyone who obtains it may be able
to perform the permitted operation until it expires or becomes invalid.

- URLs created with **IAM user credentials** can be valid for up to **seven days**
  under SigV4, subject to other constraints.
- URLs created with **temporary role credentials** cannot outlive those
  credentials' expiry.
- Bucket policies, IAM permissions, explicit denies and network restrictions can
  block access **even if the signature is valid**.
- For uploads, the signed request must match the operation and relevant signed
  parameters.

---

### 4.5 Interview-ready answer

> AWS pre-signed URLs work using Signature Version 4, rather than a
> client-certificate handshake. First, the backend authenticates to AWS and uses
> its credentials to construct a canonical request and calculate a cryptographic
> signature.
>
> The backend returns a URL containing the signing information, expiry and
> signature, without exposing the secret access key. The client sends an HTTPS
> request using that URL. S3 validates the signature, expiry, credentials and
> applicable permissions before allowing the operation.
>
> HTTPS independently establishes a secure TLS connection, so the pre-signed URL
> handles temporary authorization while TLS protects the communication in
> transit.

> **Useful follow-up question:** *What happens if someone modifies the object key
> or changes a signed query parameter?* In general, signature validation fails,
> because the request no longer matches the signed request.

---

# 5. Secure Three-Tier AWS Architecture

> **Two corrections to the common answer:**
> 1. Frontend, application backend and database should be separated into
>    **three tiers with proper network boundaries**.
> 2. A **bastion host is for administrative access**, not for
>    application-to-backend communication. The frontend/load balancer reaches
>    the backend via **security group rules**. **NAT Gateways provide outbound**
>    internet access, not inbound.

### 5.1 The architecture

```
Internet users
   │  HTTPS :443
   ▼
Public tier — across multiple AZs
   AWS WAF → Internet-facing Application Load Balancer
   │
   ▼
Private application tier
   EC2 Auto Scaling / ECS / EKS workloads
   Only accepts traffic from the ALB security group
   │
   ▼
Isolated private database tier
   Amazon RDS / Aurora in DB subnets
   Accepts connections only from the application security group

Administrative access
   AWS Systems Manager Session Manager preferred; bastion host as alternative

Outbound connectivity
   Private app subnet → NAT Gateway → Internet, when required
```

For high availability, deploy the load balancer and application/database
resources across **multiple Availability Zones**. Each tier gets its own subnet
group and security boundaries.

---

### 5.2 VPC and subnet design

Example VPC CIDR: `10.0.0.0/16`.

| Subnet | AZ A | AZ B | Purpose |
| --- | --- | --- | --- |
| Public | `10.0.1.0/24` | `10.0.2.0/24` | ALB and NAT Gateways |
| Private application | `10.0.11.0/24` | `10.0.12.0/24` | Backend workloads |
| Private database | `10.0.21.0/24` | `10.0.22.0/24` | RDS / Aurora |

*(Illustrative CIDRs.)*

**Route table design:**

- **Public subnets** — default route `0.0.0.0/0` to an **Internet Gateway**.
- **Private application subnets** — default route to a **NAT Gateway** when
  workloads need outbound internet access.
- **Database subnets** — **no** direct route to the public internet; add only
  routes needed for internal dependencies and administration.

A NAT Gateway does **not** allow unsolicited inbound connections. For
resilience, use a NAT Gateway in each AZ and route each private subnet through
its local NAT Gateway.

---

### 5.3 Security groups — the most important part

Security groups should **reference one another**, not allow broad access from
the entire VPC.

**ALB security group** *(public entry)*

- Inbound: TCP `443` from approved internet sources — often `0.0.0.0/0` for a
  public website
- Outbound: application port to the **backend security group**

**Application security group** *(private)*

- Inbound: application port (e.g. TCP `8080`) from the **ALB security group only**
- Outbound: database port to the **database security group**, plus other
  explicitly required destinations

**Database security group** *(restricted)*

- Inbound: database port (e.g. TCP `5432` for PostgreSQL) from the
  **application security group only**
- **No public inbound access**

The ALB security group controls who can reach the frontend entry point; the
backend and database security groups limit **lateral movement** if a workload is
compromised.

---

### 5.4 Administrative access

**Preferred — AWS Systems Manager Session Manager**

- Administer private EC2 instances **without exposing SSH to the internet**
- Use IAM permissions, audit logging and session controls
- Provide connectivity through SSM VPC endpoints or permitted outbound access

**Alternative — bastion host**

- Place the bastion in a **public subnet**
- Restrict SSH inbound to approved administrator IPs, preferably through a VPN
- Permit the bastion to reach only the required private instances and ports
- Apply key management, patching, logging and strong authentication

> Neither method should be used for normal frontend-to-backend traffic.

---

### 5.5 Additional production security controls

| Area | Recommended control |
| --- | --- |
| Edge security | AWS WAF managed rules, rate limiting, AWS Shield where appropriate |
| Encryption | HTTPS/TLS in transit, encryption at rest with AWS KMS |
| Secrets | AWS Secrets Manager or SSM Parameter Store |
| IAM | Least-privilege roles; avoid static access keys on EC2 |
| Database | Private subnets, encryption, backups, restricted inbound rules |
| Monitoring | CloudTrail, VPC Flow Logs, CloudWatch, security findings |
| Availability | Multi-AZ design, health checks, tested recovery procedures |

---

### 5.6 Interview-ready answer

> I would create a VPC spanning at least two Availability Zones and divide it
> into public, private application, and private database subnets.
>
> The public tier contains an internet-facing Application Load Balancer,
> optionally protected by AWS WAF. The application tier runs on private EC2
> instances, ECS or EKS workloads. The database tier uses RDS or Aurora in
> private DB subnets with public access disabled.
>
> I configure security groups so that the ALB accepts HTTPS traffic, the backend
> accepts requests only from the ALB security group, and the database accepts
> connections only from the application security group.
>
> For outbound internet access I use NAT Gateways where required. For
> administrative access I prefer Systems Manager Session Manager, with a tightly
> restricted bastion host as an alternative.
>
> Finally, I implement least-privilege IAM roles, encryption, Secrets Manager,
> logging, monitoring, backups and multi-AZ availability.

> **Key interview distinction:** A **public subnet is defined by its route to an
> Internet Gateway**, not simply by containing a public IP address. A private
> subnet can still have outbound internet access through a NAT Gateway without
> allowing unsolicited inbound connections.

---

# 6. Cluster Autoscaler — Scale Up and Scale Down

> **Question:** Does the Cluster Autoscaler monitor metrics, and does it need AWS
> access to add nodes?

**Important correction:** Cluster Autoscaler does **not** primarily monitor CPU
and memory metrics to decide when to add nodes. It mainly reacts to **pods that
cannot be scheduled**.

### 6.1 How it works on EKS

```
Application pods
   CPU/memory requests, affinity, taints and tolerations
          ↓
Kubernetes Scheduler
   Cannot find a suitable node for a pod
          ↓
Cluster Autoscaler
   Evaluates whether adding a node would make the pod schedulable
          ↓
AWS Auto Scaling / EKS managed node group
   Launches additional EC2 worker nodes within configured limits
          ↓
New node joins the cluster
   Scheduler places the pending pod when feasible
```

---

### 6.2 Does it monitor metrics?

Depends which autoscaler you mean:

| Component | What it monitors / responds to | Main purpose |
| --- | --- | --- |
| **HPA** | CPU, memory, or custom/external metrics | Increase or decrease **pod replicas** |
| **Cluster Autoscaler** | Unschedulable pods; whether nodes can be safely removed | Increase or decrease **worker nodes** |
| **Karpenter** | Unschedulable pods and scheduling requirements | Provision and consolidate compute capacity |
| **CloudWatch** | Infrastructure and application metrics | Monitoring, alarms, dashboards |

**Example:** a Deployment has 3 replicas and CPU rises.

1. HPA observes the metric and scales the Deployment from 3 → 6 replicas.
2. The scheduler tries to place the additional pods.
3. If some pods remain **Pending** due to insufficient suitable capacity,
   Cluster Autoscaler may add worker nodes.
4. The scheduler places those pods once capacity is available.

> High CPU **alone** does not cause Cluster Autoscaler to add nodes if all pods
> are already schedulable.

---

### 6.3 Does it need AWS access?

**Yes.** On EKS it needs AWS permissions to discover and modify node-group
capacity, plus Kubernetes permissions to inspect pods, nodes and scheduling
information.

Typical AWS permissions:

```
autoscaling:DescribeAutoScalingGroups
autoscaling:DescribeAutoScalingInstances
autoscaling:DescribeLaunchConfigurations
autoscaling:DescribeScalingActivities
autoscaling:SetDesiredCapacity
autoscaling:TerminateInstanceInAutoScalingGroup
```

*(Exact permissions depend on deployment and version.)*

**How it gets those permissions** — IAM Roles for Service Accounts (IRSA) or EKS
Pod Identity:

```
Cluster Autoscaler pod
        ↓  IRSA / EKS Pod Identity (temporary AWS credentials)
IAM role with least-privilege permissions
        ↓
AWS Auto Scaling APIs
        ↓
Adjusts the node group's desired capacity
```

> Avoid storing **static AWS access keys** in the Cluster Autoscaler pod.

---

### 6.4 How it knows which node group to scale

You configure eligible node groups and their min/max sizes:

```
Node group: general-workers
Minimum nodes: 2
Desired nodes: 3
Maximum nodes: 10
```

If pending pods cannot fit on existing nodes and adding a node from this group
would help, Cluster Autoscaler increases desired capacity up to the maximum.

On AWS, node-group discovery is often configured using **tags** on the
underlying Auto Scaling groups:

```
k8s.io/cluster-autoscaler/enabled        = true
k8s.io/cluster-autoscaler/<cluster-name> = owned
```

---

### 6.5 How scale-**down** is decided

Scale-up is straightforward (unschedulable pods). Scale-down asks a different
question: *can this node be removed without disrupting workloads?* It does **not**
simply look at low CPU usage.

```
Cluster Autoscaler checks nodes periodically
          ↓
Is the node underutilized?
   (compare requested CPU/memory against node capacity)
   │
   ├── No  → Keep node
   │
   └── Yes → Check removability
                 Can all relevant pods be safely rescheduled?
                 (capacity, affinity, PDBs, local storage, other constraints)
                       ↓
                 Cordon and drain the node
                 Evict eligible pods, then reduce node-group capacity
```

**Utilization example** — three worker nodes, scale-down threshold 50%:

| Node | CPU capacity | CPU requested by pods | Requested utilization | Verdict |
| --- | --- | --- | --- | --- |
| Node A | 8 cores | 6 cores | 75% | Not underutilized |
| Node B | 8 cores | 2 cores | 25% | Potentially removable |
| Node C | 8 cores | 1 core | 12.5% | Potentially removable |

> These percentages use **pod CPU requests ÷ node CPU capacity**, not the actual
> CPU usage reported by `kubectl top`. Cluster Autoscaler also considers memory
> requests and its configured scale-down rules; exact behavior depends on
> version and configuration.

**What happens next?** Suppose Node C has one pod requesting 1 CPU / 1 GiB. The
autoscaler checks whether that pod fits on Node A or Node B given available
requested resources and scheduling constraints. If it can:

1. Mark the node as a **scale-down candidate**.
2. Wait for configured timing and stabilization conditions.
3. Check whether the node's pods can move safely.
4. **Drain** the node, respecting eviction rules and PodDisruptionBudgets.
5. Decrease the desired capacity of the AWS node group → EC2 instance terminates.

If the pod cannot move — e.g. strict node affinity or insufficient remaining
capacity — the node may **not** be removed.

---

### 6.6 Observing it in production

```bash
# Requested resources per node — see "Allocated resources"
kubectl describe node <node-name>

# Actual current usage (NOT what scale-down is based on)
kubectl top nodes
kubectl top pods -A

# Autoscaler decisions: unneeded nodes, candidates, blocked scale-downs
kubectl logs -n kube-system deployment/cluster-autoscaler
```

**What can prevent scale-down:**

- A **PodDisruptionBudget** prevents the required eviction
- A pod uses **local ephemeral storage** or other non-relocatable resources
- Strict **node affinity** or topology constraints prevent rescheduling
- Remaining nodes lack sufficient capacity
- The node is protected by autoscaler configuration or scale-down rules

---

### 6.7 Interview-ready answers

**Scale-up:**

> Cluster Autoscaler runs inside the cluster and watches for unschedulable pods.
> When the scheduler cannot place a pod because of insufficient resources or
> other scheduling constraints, it evaluates whether adding a node would make the
> pod schedulable. In EKS it uses AWS permissions through IRSA or EKS Pod
> Identity to interact with the underlying Auto Scaling groups and increase
> desired capacity. After a new EC2 worker joins, Kubernetes schedules the
> pending pods. HPA is different: it scales the number of application **pods**
> based on metrics, while Cluster Autoscaler scales **worker-node capacity**.

**Scale-down:**

> For scale-down, Cluster Autoscaler periodically evaluates node utilization
> based primarily on **requested** resources, not just actual CPU usage. If a
> node is below the configured utilization threshold, it checks whether all
> relevant pods can be rescheduled elsewhere while respecting scheduling
> constraints and disruption budgets. If the node is safely removable and remains
> a candidate for the configured period, it drains the node and reduces the
> desired capacity of the underlying node group.

> **The key difference:** scale-up asks *"do pending pods need more capacity?"*;
> scale-down asks *"can existing workloads fit safely without this node?"*

> **Production troubleshooting tip:** If Cluster Autoscaler does not add nodes,
> check its logs, the pending pod's `kubectl describe pod` events, node-group
> maximum size, IAM permissions, and whether the pod's resource requests,
> affinity, taints or storage constraints can actually be satisfied by the
> available node groups.

---

# 7. Service Accounts vs Users

In Kubernetes, Service Accounts and Users are both identities used for
authentication, but they serve different purposes.

- **User** — typically a human administrator, developer, or an external system.
- **Service Account** — an application, pod, or automated process that needs to
  interact with the Kubernetes API.

| Feature | User | Service Account |
| --- | --- | --- |
| Represents | Human or external identity | Application or automated workload |
| Created as a Kubernetes object | Generally **no** | **Yes** |
| Typical use | Administrator runs `kubectl` | Pod calls the Kubernetes API |
| Authentication | Certificates, OIDC, IAM integrations | Commonly short-lived Service Account tokens |
| Permissions | RBAC Role / ClusterRole bindings | RBAC Role / ClusterRole bindings |
| Namespace | Not inherently namespace-scoped | Belongs to a namespace |

Both users and Service Accounts are granted permissions through **Kubernetes
RBAC**.

---

# 8. Pod Security Policy (PSP) vs Pod Security Standards (PSS)

PSP and PSS are security mechanisms that control what a pod is allowed to do on
a node.

> **Most important interview point:** `PodSecurityPolicy` was **deprecated in
> Kubernetes 1.21 and removed in 1.25**. **Pod Security Standards** are the
> current built-in guidance, enforced through **Pod Security Admission (PSA)**.

### 8.1 What risks do they prevent?

Suppose a developer deploys a pod that:

- Runs as root
- Uses privileged mode
- Mounts the host filesystem
- Gains access to the host's network or devices
- Adds dangerous Linux capabilities

These configurations increase the impact of a compromised container and can
potentially compromise the worker node. Pod security controls restrict them.

---

### 8.2 Pod Security Policy (PSP) — historical

PSP was a Kubernetes API resource allowing cluster administrators to define
security requirements for pods (e.g. restricting privileged containers). It used
**RBAC** to determine which policies a user or Service Account could use.

**PSP was removed in Kubernetes 1.25** — never recommend it for a new cluster.

---

### 8.3 Pod Security Standards (PSS)

PSS defines three security levels:

| Level | Restriction | Description |
| --- | --- | --- |
| **Privileged** | Least restrictive | Minimal restrictions. Intended for trusted system-level workloads needing elevated privileges. |
| **Baseline** | Moderate | Prevents known privilege-escalation configurations while supporting common application workloads. |
| **Restricted** | Most restrictive | Stronger requirements: non-root execution, no privilege escalation, tighter Linux capability controls. |

> PSS **defines** the standards; Pod Security Admission **enforces** them.

---

### 8.4 Enforcing Pod Security Standards

Label a namespace:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

The three modes behave differently:

| Mode | Behavior |
| --- | --- |
| `enforce` | **Rejects** pods that violate the selected standard |
| `audit` | **Records** policy violations in audit events |
| `warn` | Returns **warnings** to the user submitting the pod |

You can also pin the policy to a Kubernetes version with version labels such as
`pod-security.kubernetes.io/enforce-version`.

```bash
kubectl apply -f namespace.yaml
```

Now a privileged pod created in that namespace is rejected by Pod Security
Admission.

---

### 8.5 Example: a pod rejected by Restricted

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: insecure-app
  namespace: production
spec:
  containers:
    - name: app
      image: nginx:1.28
      securityContext:
        privileged: true
        runAsUser: 0
```

With **Restricted** enforced, admission rejects this pod.

A simplified compliant container configuration:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  seccompProfile:
    type: RuntimeDefault
```

The image may also need an appropriate **non-root user** configured, and the
full pod must satisfy all applicable Restricted requirements.

---

### 8.6 How this differs from RBAC and NetworkPolicy

| Mechanism | What it controls |
| --- | --- |
| **RBAC** | **Who** can perform actions against the Kubernetes API |
| **Pod Security Standards / Admission** | **Which pod security configurations** are allowed |
| **NetworkPolicy** | **Which network connections** are allowed between workloads |
| **Linux security controls** | Runtime restrictions, capabilities, seccomp, process-level protections |

These controls **complement** each other; none replaces the others.

> **OpenShift note:** OpenShift also uses **Security Context Constraints (SCCs)**
> to control pod privileges, user IDs, capabilities and host access. SCCs are an
> OpenShift-specific mechanism, distinct from upstream Pod Security Admission.

---

# 9. Securing etcd

etcd stores the cluster's state — Secrets, ConfigMaps, pod specifications, RBAC
configuration and other Kubernetes resources. If an attacker gains unrestricted
access to etcd, they may be able to retrieve sensitive data or compromise the
cluster.

### 9.1 Six areas to secure

**1. Enable encryption at rest**
Encrypt Kubernetes Secrets stored in etcd using API server encryption at rest.
For stronger key management, integrate with a **KMS provider** where supported.

**2. Enable mutual TLS (mTLS)**
Configure etcd **client** and **peer** communication to use trusted
certificates. The API server authenticates to etcd using client certificates;
peer communication is also authenticated and encrypted.

**3. Restrict network access**
Keep etcd on private control-plane networks. Allow its client port (usually TCP
`2379`) only from authorized components. Restrict peer port `2380` to etcd
members.

**4. Apply least privilege**
Restrict access to etcd certificates, keys and configuration files. Only
authorized administrators and control-plane components should reach etcd
directly.

**5. Take encrypted, tested backups**
Periodically create etcd snapshots, protect them with encryption and restrictive
access controls, store copies **outside** the cluster, and **test restoration**.

**6. Monitor and patch**
Monitor etcd health, latency, disk usage, certificate expiry and suspicious
access. Keep Kubernetes and etcd updated using the supported upgrade process.

---

### 9.2 How encryption at rest works

By default, Kubernetes resource data in etcd is not necessarily encrypted at the
application level. A Kubernetes Secret is **base64-encoded** in its API
representation — **base64 is not encryption**.

You configure the API server with an encryption provider (e.g. AES-GCM) or a
supported KMS provider:

```
Kubernetes Secret
        ↓
API server encryption provider
        ↓
Encrypted data stored in etcd
```

Encryption at rest protects **stored** data. It does not replace TLS, access
controls or secure backups.

---

### 9.3 Verifying etcd security

For a self-managed control plane, inspect the API server's encryption
configuration and etcd TLS settings. Check etcd health:

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/path/to/ca.crt \
  --cert=/path/to/client.crt \
  --key=/path/to/client.key \
  endpoint health
```

> Paths above are placeholders — use the actual paths for your distribution.
> Protect the client key and **never** paste private keys into logs or chat.

Also verify:

- The API server uses the expected etcd client certificate and trusted CA
- Peer communication requires TLS
- Network rules prevent unauthorized clients from reaching etcd
- Encryption at rest is configured and existing sensitive data has been migrated
  or rewritten as required
- Snapshot backups can be **restored successfully**

---

### 9.4 Managed Kubernetes

- **Amazon EKS** — AWS manages the control plane and etcd infrastructure. You
  configure supported control-plane encryption features and permissions rather
  than directly administering etcd nodes.
- **OpenShift** — follow the platform's supported control-plane security,
  encryption and backup procedures rather than modifying etcd manually.

---

### 9.5 Interview-ready answer

> I secure etcd using encryption at rest, mutual TLS for client and peer
> communication, network isolation, least-privilege access, encrypted backups and
> monitoring.
>
> I ensure that only the Kubernetes API server and authorized control-plane
> components can connect to etcd, and that client certificates and private keys
> are tightly protected. I also configure encryption for Kubernetes Secrets,
> monitor certificate expiry and etcd health, and regularly test snapshot
> restoration.
>
> In managed services such as EKS, I use the provider's supported control-plane
> security mechanisms because the underlying etcd infrastructure is managed by
> the provider.

> **Key interview distinction:** TLS protects etcd **communication**; encryption
> at rest protects **stored data**; network isolation and access controls
> restrict **who can reach it**. You need all three.

---

### 9.6 What does "AES-GCM" mean?

**AES-GCM** = *Advanced Encryption Standard – Galois/Counter Mode*.

- **AES** — a **symmetric** encryption algorithm. The same secret key encrypts
  and decrypts.
- **GCM** — a mode of operation providing both **encryption and integrity
  verification**. It detects if encrypted data has been modified.

AES commonly uses 128-, 192- or 256-bit keys. AES-GCM commonly uses a 128-bit
authentication tag.

**In Kubernetes:**

```
Kubernetes Secret  (Password: my-db-password)
        ↓
Kubernetes API server
   Encryption provider uses AES-GCM and an encryption key
        ↓
Encrypted data stored in etcd
   Ciphertext, nonce and authentication tag
```

When an authorized request reads the Secret, the API server decrypts the stored
data before returning it. Without the encryption key, someone who obtains only
the encrypted data should not be able to recover the password.

**Why AES-GCM rather than plain encryption?**

| Protection | What it does |
| --- | --- |
| Confidentiality | Makes the original data unreadable without the key |
| Integrity / authenticity | Detects unauthorized modifications to the encrypted data |

> GCM uses a **unique nonce** for each encryption under a given key. Reusing a
> nonce with the same key can seriously compromise security.

**Configuration shape:**

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: example-key
              secret: <base64-encoded-key>
      - identity: {}
```

> **Important:** this example uses `aescbc`, **not** AES-GCM. For AES-GCM the
> provider is named `aesgcm` with an appropriate base64-encoded key. The example
> above illustrates **provider ordering** only.

In production you must manage encryption keys securely, plan **key rotation**,
and migrate existing stored data as needed. A **KMS provider** is often
preferable to keeping raw encryption keys in the configuration file.

> **Interview answer:** *"AES-GCM is a symmetric authenticated-encryption
> algorithm. In Kubernetes, the API server can use it to encrypt Secrets before
> storing them in etcd. It provides confidentiality and integrity, while TLS
> separately protects data in transit."*

---

# 10. Secrets: Base64, Encryption at Rest, and Secrets Manager

> **Question:** We already "encrypt" Secrets using base64 — will they be further
> encrypted by kube-apiserver encryption?

**Yes — but the important distinction is: Base64 is *encoding*, not
*encryption*.** If API server encryption at rest is configured, the Secret data
is encrypted before it is stored in etcd.

### 10.1 What happens when you create a Secret

```bash
kubectl create secret generic db-secret \
  --from-literal=password=myPassword123
```

```
1. Original password
   myPassword123
          ↓
2. Base64 encoding
   bXlQYXNzd29yZDEyMw==        ← encoding only; easily reversible
          ↓
3. Kubernetes API server
   Applies configured encryption-at-rest provider, if enabled
          ↓
4. etcd
   Stores encrypted Secret data when encryption at rest is configured
```

> **Nuance:** the Kubernetes API *represents* Secret values using Base64 in
> YAML/JSON, but Kubernetes does not rely on Base64 for security. Encryption at
> rest is a **separate step** performed by the API server when configured.

---

### 10.2 What happens when you retrieve it

```bash
kubectl get secret db-secret -o yaml
```

The flow reverses:

1. The API server reads the **encrypted** record from etcd.
2. It **decrypts** using the configured provider and key.
3. It returns the Secret through the Kubernetes API.
4. `kubectl` displays the data in **Base64-encoded** form.

```bash
echo 'bXlQYXNzd29yZDEyMw==' | base64 --decode
# myPassword123
```

---

### 10.3 With vs without encryption at rest

| Layer | Without encryption at rest | With encryption at rest |
| --- | --- | --- |
| Secret YAML representation | Base64-encoded data | Base64-encoded data |
| Stored data in etcd | **Not encrypted** by the API encryption provider | **Encrypted** by the configured provider |
| `kubectl get secret` | Returns Base64-encoded data | Also returns Base64-encoded data |
| Authorized application | Can obtain the original value | Can obtain the original value |

> **Interview takeaway:** Base64 makes data representable in text formats; it
> does **not** protect confidentiality. Kubernetes encryption at rest adds a
> separate cryptographic layer to protect stored Secret data in etcd.

---

### 10.4 "If someone has access, they can still decode it — right?"

**Correct.** Encryption at rest protects Secrets **stored in etcd**; it does not
prevent an authorized Kubernetes user — or a compromised application — from
retrieving the decrypted Secret.

```
Kubernetes Secret (database password)
        ↓
API server encrypts data before persistence
        ↓
etcd stores encrypted data

Threat A: someone reads the raw etcd data
   → Secret remains encrypted if the attacker lacks the decryption key

Threat B: someone has Kubernetes API permission to read the Secret
   → API server returns the value (Base64-encoded, trivially decoded)

Threat C: compromised pod with a mounted Secret
   → Attacker may obtain that application's credential
```

So the threat depends on **access level**:

| Access level | Does encryption at rest help? |
| --- | --- |
| Raw etcd access | **Yes** — data is encrypted |
| Kubernetes API access with Secret-read RBAC | **No** — the value is returned decrypted |
| Compromised pod with mounted Secret | **No** — the credential is available to the app |

---

### 10.5 How AWS Secrets Manager helps

Instead of storing the database password directly in a Kubernetes Secret, store
it in AWS Secrets Manager. For EKS, a common architecture uses **EKS Pod
Identity or IRSA**:

```
Application pod
        ↓
Kubernetes Service Account
        ↓  mapped to an IAM role via Pod Identity or IRSA
AWS IAM authorization
        ↓  allows access only to the required secret ARN
AWS Secrets Manager
        ↓
Encrypted storage, access control and optional rotation
```

The application retrieves the credential at runtime using the AWS SDK, or you
use the **AWS Secrets and Configuration Provider (ASCP)** with the **Secrets
Store CSI Driver** to mount the secret into the pod.

**Why is this more secure?**

| Kubernetes Secret alone | AWS Secrets Manager |
| --- | --- |
| Stored as a Kubernetes resource | Stored in a dedicated AWS service |
| Access controlled through Kubernetes RBAC | Access controlled through IAM and resource policies |
| Encryption at rest must be configured appropriately | Encryption at rest uses AWS KMS integration |
| Rotation needs an additional mechanism | Supports managed rotation workflows |
| Users with Secret-read permission can retrieve the value | Only authorized IAM principals can retrieve it |

> **But remember:** if the application retrieves the password, the password must
> exist in memory or otherwise be available to the application. A compromised
> application may still expose it.
>
> Also, if you **synchronize** Secrets Manager values into Kubernetes Secret
> objects, those copies are still subject to Kubernetes Secret access controls
> and encryption-at-rest configuration.

---

### 10.6 Recommended for EKS production

- Store database credentials in **AWS Secrets Manager**
- Use **EKS Pod Identity or IRSA** to grant each workload narrowly scoped IAM
  permissions
- Restrict secret access to the **specific secret ARN**
- Enable and **test credential rotation** where supported
- Avoid unnecessarily copying credentials into Kubernetes Secrets
- Apply least-privilege Kubernetes RBAC, encrypt Kubernetes Secrets at rest, and
  audit access through **CloudTrail** and **Kubernetes audit logging**

---

### 10.7 Interview-ready answer

> Kubernetes encryption at rest protects Secret data stored in etcd, but
> authorized users can still retrieve and decode the Secret through the
> Kubernetes API. To strengthen secret management in EKS, I would use AWS Secrets
> Manager with IRSA or EKS Pod Identity. Each application gets a dedicated IAM
> role with permission to retrieve only the required secret. This centralizes
> secret storage, supports rotation and auditing, and reduces unnecessary
> exposure in Kubernetes. However, I would still enforce Kubernetes RBAC and
> secure the application itself, because a compromised application can expose
> credentials it is authorized to use.

---

# 11. How Do You Monitor Kubernetes?

| Component | Role |
| --- | --- |
| **Prometheus + Grafana** | The production standard. The `kube-prometheus-stack` Helm chart installs everything. |
| **metrics-server** | Lightweight in-memory metrics for `kubectl top` and HPA. **Not** for long-term storage. |
| **kube-state-metrics** | Exports cluster **state** metrics — pod restarts, deployment replicas, PVC status. |
| **Node Exporter** | OS-level metrics per node — CPU, memory, disk, network, filesystem. |
| **Alertmanager** | Routes alerts to PagerDuty (P1), Slack (P2/P3), email. Define SLO-based alerts. |
| **Grafana dashboards** | Import K8s community dashboards (IDs **315**, **6417**) for instant cluster visibility. |

---

# Command Cheat Sheet

**Git**

```bash
git log --oneline                              # find the bad commit
git revert <sha>                               # safe undo on shared branches
git revert <oldest>^..<newest>                 # revert a range
git reset --hard HEAD~1                        # LOCAL, unshared commits only
git push --force-with-lease origin main        # safer than --force
```

**Node disk troubleshooting**

```bash
kubectl get nodes
kubectl describe node <node>                   # check DiskPressure condition
df -h ; df -i ; lsblk                          # space and inodes
sudo du -xhd1 /var | sort -h                   # biggest directories
sudo du -ahx /var/log | sort -rh | head -20
sudo lsof +L1                                  # deleted-but-open files
sudo crictl images ; sudo crictl ps -a         # runtime storage
```

**Autoscaling**

```bash
kubectl describe node <node>                   # "Allocated resources" = requests
kubectl top nodes ; kubectl top pods -A        # actual usage
kubectl logs -n kube-system deployment/cluster-autoscaler
kubectl describe pod <pending-pod>             # why it can't schedule
```

**etcd**

```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=<ca> --cert=<crt> --key=<key> endpoint health
```

**Secrets**

```bash
kubectl create secret generic db-secret --from-literal=password=myPassword123
kubectl get secret db-secret -o yaml
echo '<base64>' | base64 --decode
```

---

# Key Takeaways

1. **`revert` undoes changes; `reset` rewrites history.** Use revert on anything
   already pushed; reset only on local, unshared commits. Prefer
   `--force-with-lease` over `--force`.
2. **`DiskPressure` is an eviction condition, not just a Linux disk problem.**
   Always check pod eviction events and kubelet thresholds alongside `df`, `du`
   and `lsof`.
3. **Pre-signed URLs are SigV4 signatures, not TLS handshakes.** They provide
   temporary *authorization*; TLS provides *confidentiality in transit*. They are
   bearer credentials — treat them as secrets.
4. **A public subnet is defined by its route to an Internet Gateway**, not by
   having a public IP. NAT Gateways are outbound-only. Bastions are for
   administration, not app traffic.
5. **Cluster Autoscaler reacts to unschedulable pods, not CPU metrics.** HPA
   scales pods on metrics; Cluster Autoscaler scales nodes on schedulability.
   Scale-down is based on **requested** resources plus rescheduling feasibility.
6. **PSP is gone (removed in 1.25).** Use Pod Security Standards enforced by Pod
   Security Admission via namespace labels. OpenShift adds SCCs.
7. **etcd needs all three:** TLS for communication, encryption at rest for
   stored data, network isolation and least privilege for access.
8. **Base64 is encoding, not encryption.** Encryption at rest protects etcd
   contents only — anyone with Secret-read RBAC still gets the plaintext. Use a
   dedicated secret manager with IRSA/Pod Identity for stronger control.
9. **Monitoring stack:** Prometheus + Grafana for the long term, metrics-server
   for HPA/`kubectl top`, kube-state-metrics for cluster state, Node Exporter for
   OS metrics, Alertmanager for routing.
