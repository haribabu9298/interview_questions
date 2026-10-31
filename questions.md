# DevOps Topics — Interview & Reference Guide

A cleaned-up, structured version of the raw notes in [`new_topics`](./new_topics).
Covers AWS identity, databases, Kubernetes autoscaling, compliance, Terraform, Vault,
multi-cluster `kubectl`, GitLab CI/CD, and Linux disk troubleshooting.

---

## Table of Contents

**Cloud & Identity**
1. [Alternatives to IAM Users](#1-alternatives-to-iam-users)

**Data**
2. [SQL vs NoSQL Databases](#2-sql-vs-nosql-databases)

**Kubernetes**
3. [KEDA — Event-Driven Autoscaling](#3-keda--event-driven-autoscaling)
4. [VPA — Vertical Pod Autoscaler](#4-vpa--vertical-pod-autoscaler)
5. [HPA vs VPA vs KEDA — When to Use What](#5-hpa-vs-vpa-vs-keda--when-to-use-what)
6. [Managing 100+ Kubernetes Clusters](#6-managing-100-kubernetes-clusters)

**Compliance**
7. [ISO 27001 vs SOC 2 vs HIPAA](#7-iso-27001-vs-soc-2-vs-hipaa)

**Infrastructure as Code**
8. [Terraform Dynamic Blocks](#8-terraform-dynamic-blocks)
9. [Terraform + HashiCorp Vault](#9-terraform--hashicorp-vault)

**CI/CD**
10. [GitLab CI/CD — workflow, include, rules](#10-gitlab-cicd--workflow-include-rules)

**Troubleshooting**
11. ["Disk Full" But Space Is Available](#11-disk-full-but-space-is-available)

**Reference**
- [Glossary](#glossary)
- [Key Takeaways](#key-takeaways)

---

## 1. Alternatives to IAM Users

**The problem:** an IAM user comes with long-lived access keys (`AKIA...`). If those keys
leak — in a Git commit, a CI log, a laptop backup — an attacker has permanent access
until someone notices and rotates them.

**The fix:** every modern option below issues **temporary credentials** that expire
automatically (typically 1–12 hours), so there is no permanent secret to steal.

| Method | Best use case | How authentication works |
| --- | --- | --- |
| **IAM roles + STS** | EC2, Lambda, AWS services, cross-account access | Workload assumes a role and receives temporary credentials |
| **IAM Identity Center (SSO)** | Engineers accessing AWS from a laptop | Sign in through your organization's identity provider |
| **OIDC federation** | Jenkins, GitHub Actions, other CI/CD | Exchange a trusted identity token for temporary AWS credentials |
| **EKS Pod Identity / IRSA** | Applications running inside EKS | Kubernetes ServiceAccount is mapped to an IAM role |
| **SAML federation** | Enterprise IdPs (Okta, ADFS, Azure AD) | Sign in through the corporate identity system |

> **Interview answer:** "All of these provide AWS access without a dedicated IAM user and
> without permanent access keys. Credentials are short-lived and automatically rotated."

### Quick decision guide

```
Running on an EC2 instance?          → Instance profile (IAM role + STS)
Running as a pod in EKS?             → EKS Pod Identity (or IRSA on older clusters)
Running in GitHub Actions / Jenkins? → OIDC federation
A human at a keyboard?               → IAM Identity Center (SSO) or SAML
Another AWS account needs access?    → Cross-account IAM role with sts:AssumeRole
```

### Example: GitHub Actions via OIDC (no secrets stored)

```yaml
permissions:
  id-token: write      # required to request the OIDC token
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-deploy
          aws-region: us-east-1
```

No `AWS_ACCESS_KEY_ID` secret exists anywhere. GitHub presents a signed token, AWS trusts
the issuer, and STS returns credentials valid for the job only.

---

## 2. SQL vs NoSQL Databases

### Common databases by type

| Database | Type | Data model |
| --- | --- | --- |
| PostgreSQL | SQL | Relational tables |
| MySQL | SQL | Relational tables |
| MongoDB | NoSQL | Documents |
| DynamoDB | NoSQL | Key-value and documents |
| Neo4j | NoSQL | Graph — nodes and relationships |
| Cassandra | NoSQL | Wide-column |
| Amazon Neptune | NoSQL | Graph |

### Key differences

| Feature | SQL | NoSQL |
| --- | --- | --- |
| **Structure** | Tables | Documents, key-value, graphs, wide-column |
| **Schema** | Usually predefined | Often flexible / schema-on-read |
| **Relationships** | Joins and foreign keys | Embedding, references, graph edges |
| **Transactions** | Strong, mature support | Varies by engine; many now support them |
| **Scaling** | Often vertical; horizontal options exist | Usually designed for horizontal scaling |
| **Consistency** | Depends on engine and configuration | Depends on database and consistency settings |
| **Best suited for** | Banking, payments, structured records | High-volume, flexible data, distributed workloads |

> **Caution:** "NoSQL = no transactions" and "SQL = cannot scale" are both outdated.
> DynamoDB supports transactions; PostgreSQL scales horizontally with read replicas and
> sharding. Choose based on **access patterns**, not slogans.

### Choosing in practice

```
Strong relational integrity, reporting, ad-hoc queries   → PostgreSQL / MySQL
Flexible or evolving document shape                      → MongoDB
Predictable key-based lookups at massive scale           → DynamoDB
Relationship traversal (fraud rings, recommendations)    → Neo4j / Neptune
Write-heavy, multi-region, time-series style data        → Cassandra
```

---

## 3. KEDA — Event-Driven Autoscaling

**KEDA** (Kubernetes Event-Driven Autoscaling) scales workloads based on **external
event sources** — queue depth, Prometheus metrics, Kafka lag, cron schedules — rather
than only CPU and memory.

Its headline feature: **scale to zero**. Standard HPA cannot go below 1 replica; KEDA can.

### ScaledObject example

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-app-scaler
  namespace: default
spec:
  # 1. The deployment you want to scale
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-web-app

  # 2. General scaling parameters (defaults apply if omitted)
  pollingInterval: 15    # seconds between trigger checks (default: 15)
  cooldownPeriod:  300   # seconds to wait before scaling back to 0 (default: 300)
  minReplicaCount: 0     # KEDA allows scaling down to zero
  maxReplicaCount: 10    # upper scaling limit

  # 3. Event triggers
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://cluster.local
        metricName: http_requests_total
        threshold: '100'   # scale up when the query result exceeds this per pod
        query: sum(rate(http_requests_total{job="my-web-app"}[2m]))
```

### How it works

```
External source (Prometheus / SQS / Kafka)
        │  KEDA polls every pollingInterval
        ▼
   KEDA operator
        │  creates and manages an HPA under the hood
        ▼
   HPA scales the Deployment  (1 → N)
        │
   KEDA handles 0 ↔ 1 directly (HPA cannot)
```

### Common triggers

| Trigger type | Scales on |
| --- | --- |
| `prometheus` | Any PromQL query result |
| `aws-sqs-queue` | Number of messages in an SQS queue |
| `kafka` | Consumer group lag |
| `rabbitmq` | Queue length |
| `cron` | Time windows (e.g. scale up 9am–6pm) |
| `azure-servicebus` / `gcp-pubsub` | Cloud queue depth |

---

## 4. VPA — Vertical Pod Autoscaler

While HPA and KEDA add **more pods**, VPA makes **each pod bigger or smaller** by
adjusting CPU and memory requests/limits based on observed usage.

### VerticalPodAutoscaler example

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
  namespace: default
spec:
  # 1. Target workload
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: my-web-service

  # 2. How resource updates are applied
  updatePolicy:
    updateMode: "Auto"     # Auto | Recreate | Initial | Off

  # 3. Resource boundaries — crucial for production safety
  resourcePolicy:
    containerPolicies:
      - containerName: '*'   # all containers, or name one explicitly
        controlledResources: ["cpu", "memory"]
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
        maxAllowed:
          cpu: "2"
          memory: "4Gi"
```

### Update modes

| Mode | Behaviour |
| --- | --- |
| `Off` | Only generates recommendations — safest starting point |
| `Initial` | Applies recommendations to **new** pods only |
| `Recreate` | Evicts and recreates pods when recommendations drift |
| `Auto` | Currently behaves like `Recreate`; may use in-place resize in future versions |

> **Production warning:** `Auto` and `Recreate` **restart pods**. Always set `minAllowed`
> and `maxAllowed`, and pair with a PodDisruptionBudget. Many teams run VPA in `Off` mode
> and apply the recommendations manually or through their CI pipeline.

> **Do not** run VPA and HPA on the **same metric** (CPU or memory) for the same workload —
> they will fight each other. HPA on custom/external metrics + VPA on resources is fine.

---

## 5. HPA vs VPA vs KEDA — When to Use What

| | **HPA** | **VPA** | **KEDA** |
| --- | --- | --- | --- |
| **Scales** | Pod count (out) | Pod size (up) | Pod count (out), incl. zero |
| **Driven by** | CPU / memory / custom metrics | Observed resource usage | External events and metrics |
| **Scale to zero** | No (min 1) | N/A | Yes |
| **Restarts pods** | No | Yes (`Auto`/`Recreate`) | No |
| **Typical use** | Web APIs under variable traffic | Right-sizing unknown workloads | Queue workers, batch jobs, bursty traffic |

```
Steady web traffic, CPU-bound        → HPA
Unsure what resources a pod needs    → VPA in "Off" mode, read recommendations
Queue/Kafka consumers, idle at night → KEDA (scale to zero saves real money)
```

---

## 6. Managing 100+ Kubernetes Clusters

### The kubeconfig model

A kubeconfig file holds three lists plus a pointer:

| Section | Holds |
| --- | --- |
| `clusters` | API server URL and CA certificate |
| `users` | Credentials — certs, tokens, or exec plugins |
| `contexts` | A named pairing of **cluster + user + namespace** |
| `current-context` | Which context `kubectl` uses right now |

With 100 clusters you do **not** want 100 separate files you edit by hand. You want one
merged config (or a merge at runtime) and fast context switching.

```yaml
apiVersion: v1
kind: Config
clusters:
  - name: prod-us-east
    cluster:
      server: https://prod-use1.example.com:6443
      certificate-authority: /etc/k8s/prod-use1-ca.crt
  - name: prod-eu-west
    cluster:
      server: https://prod-euw1.example.com:6443
      certificate-authority: /etc/k8s/prod-euw1-ca.crt

users:
  - name: hari
    user:
      client-certificate: /etc/k8s/hari.crt
      client-key: /etc/k8s/hari.key

contexts:
  - name: prod-use1
    context:
      cluster: prod-us-east
      user: hari
      namespace: payments
  - name: prod-euw1
    context:
      cluster: prod-eu-west
      user: hari
      namespace: payments

current-context: prod-use1
```

### Merging and switching

```bash
# Merge multiple kubeconfigs at runtime (colon-separated, like PATH)
export KUBECONFIG=~/.kube/config:~/.kube/prod-use1:~/.kube/prod-euw1

# Flatten the merged view into one real file
kubectl config view --flatten > ~/.kube/merged && mv ~/.kube/merged ~/.kube/config

# Inspect
kubectl config get-contexts
kubectl config current-context

# Switch
kubectl config use-context prod-euw1

# Pin a default namespace for the current context
kubectl config set-context --current --namespace=payments

# One-off override without switching context
kubectl get pods --context prod-use1 -n payments
```

> **Tip:** `kubectx` / `kubens` (or `kubectl config use-context` with shell completion)
> make this far faster at scale. For 100 clusters, generate the kubeconfig from automation
> — `aws eks update-kubeconfig`, Terraform output, or a cluster registry — never by hand.

### Creating a user and binding a role

Kubernetes has no `User` object. A user is simply a name presented by a client
certificate, token, or OIDC claim, which RBAC then authorizes.

**Step 1 — generate a client certificate**

```bash
openssl genrsa -out hari.key 2048
openssl req -new -key hari.key -out hari.csr -subj "/CN=hari/O=dev-team"
# CN becomes the username, O becomes the group
```

**Step 2 — get it signed by the cluster CA** (via a `CertificateSigningRequest`)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: hari
spec:
  request: $(base64 < hari.csr | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages: ["client auth"]
EOF

kubectl certificate approve hari
kubectl get csr hari -o jsonpath='{.status.certificate}' | base64 -d > hari.crt
```

**Step 3 — define the Role and RoleBinding**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: hari-pod-reader
  namespace: default
subjects:
  - kind: User
    name: hari            # must match the certificate CN
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Step 4 — register the credentials and use them**

```bash
kubectl config set-credentials hari \
  --client-certificate=hari.crt \
  --client-key=hari.key \
  --embed-certs=true

kubectl get pods --user hari

# Verify permissions without trial and error
kubectl auth can-i list pods --as hari -n default     # yes
kubectl auth can-i delete pods --as hari -n default   # no
```

> `Role`/`RoleBinding` are namespace-scoped. Use `ClusterRole`/`ClusterRoleBinding` for
> cluster-wide permissions or for non-namespaced resources like nodes and PVs.

---

## 7. ISO 27001 vs SOC 2 vs HIPAA

### Side-by-side comparison

| Framework | Primary focus | Target audience | Validation method |
| --- | --- | --- | --- |
| **ISO 27001** | Information Security Management System (ISMS) | Global organizations, all sectors | Third-party **certification audit** |
| **SOC 2** | Operational security controls and trust criteria | SaaS, cloud, and technology service providers | **Attestation report** (Type I or Type II) from a CPA firm |
| **HIPAA** | Protection of Protected Health Information (PHI) | US healthcare providers, clearinghouses, business associates | Compliance assessment / audit — there is **no official "HIPAA certification"** |

### 7.1 ISO 27001 — *How do we manage security?*

Think of it as a **security management system** for the whole company. You must identify
risks and establish policies and processes to manage them.

**Your DevOps work:**
- Restrict AWS access using IAM and least privilege.
- Scan infrastructure for security vulnerabilities.
- Encrypt sensitive data at rest and in transit.
- Maintain security policies and run periodic access reviews.

**Goal:** establish and maintain a *systematic* approach to information security.

### 7.2 SOC 2 — *Can we prove our controls work?*

Think of it as an **independent auditor checking your controls**. It is not enough to say
your infrastructure is secure — you need evidence that controls are properly designed
and, for **Type II**, that they operated effectively over a period (typically 3–12 months).

**Your DevOps work:**
- Enable AWS CloudTrail logging (and keep logs immutable).
- Maintain Jenkins/GitLab deployment approvals and change records.
- Keep vulnerability scan reports.
- Record who accessed infrastructure and when.

**Goal:** produce an independent report about the relevant controls.

| | **Type I** | **Type II** |
| --- | --- | --- |
| Question answered | Are controls *designed* correctly? | Did controls *operate* effectively? |
| Time dimension | Point in time | Over a period (3–12 months) |
| Effort | Lower | Higher — evidence must be collected continuously |

### 7.3 HIPAA — *How do we protect patient data?*

US healthcare privacy and security requirements. Applies to **covered entities** and
their **business associates**.

**Your DevOps work:**
- Encrypt electronic Protected Health Information (ePHI).
- Restrict access to patient records on a need-to-know basis.
- Audit all access to sensitive data.
- Protect backups and maintain incident response procedures.

**Goal:** protect healthcare information under applicable HIPAA requirements.

> **Note:** on AWS you must sign a **Business Associate Addendum (BAA)** and use only
> HIPAA-eligible services for workloads that touch ePHI.

### What they share

Despite different audiences, the DevOps controls overlap heavily:

```
Encryption (at rest + in transit)   ✔ ISO 27001   ✔ SOC 2   ✔ HIPAA
Least-privilege access / RBAC       ✔ ISO 27001   ✔ SOC 2   ✔ HIPAA
Audit logging (CloudTrail)          ✔ ISO 27001   ✔ SOC 2   ✔ HIPAA
Change management / approvals       ✔ ISO 27001   ✔ SOC 2   ✔ HIPAA
Vulnerability management            ✔ ISO 27001   ✔ SOC 2   ✔ HIPAA
Incident response plan              ✔ ISO 27001   ✔ SOC 2   ✔ HIPAA
```

Build these once and you are most of the way to all three.

---

## 8. Terraform Dynamic Blocks

A `dynamic` block acts like a **for loop inside a single resource**, programmatically
generating repeated nested blocks from a list or map.

### Without dynamic — repetitive

```hcl
resource "aws_security_group" "web_sg" {
  name = "web-server-sg"

  ingress {
    description = "SSH Access"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP Access"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS Access"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### With dynamic — data-driven

```hcl
locals {
  inbound_rules = [
    { port = 22,  desc = "SSH Access" },
    { port = 80,  desc = "HTTP Access" },
    { port = 443, desc = "HTTPS Access" }
  ]
}

resource "aws_security_group" "web_sg" {
  name        = "web-server-sg"
  description = "Managed dynamically via Terraform"

  # The dynamic keyword creates multiple "ingress" blocks
  dynamic "ingress" {
    for_each = local.inbound_rules

    content {
      description = ingress.value.desc
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

### Anatomy

| Part | Meaning |
| --- | --- |
| `dynamic "ingress"` | The **nested block type** to generate |
| `for_each` | The list or map to iterate over |
| `content { }` | The body of each generated block |
| `ingress.value` | The current element (named after the block label) |
| `ingress.key` | The index (list) or key (map) |
| `iterator = rule` | Optional — rename the iterator to `rule.value` etc. |

> **Caveat:** `dynamic` only works for **nested blocks**, not for top-level arguments or
> whole resources. To create multiple *resources*, use resource-level `for_each` or `count`.

> **Style note:** HashiCorp advises using `dynamic` sparingly — overusing it makes
> configurations hard to read. Prefer it when the data genuinely comes from a variable.

---

## 9. Terraform + HashiCorp Vault

Read secrets from Vault at apply time instead of hardcoding them or storing them in
`.tfvars`.

```hcl
# Read the secret from the KV v2 engine
data "vault_kv_secret_v2" "db" {
  mount = "secret"
  name  = "db-creds"
}

# Use the retrieved secret in another resource
resource "aws_db_instance" "default" {
  allocated_storage   = 10
  engine              = "mysql"
  instance_class      = "db.t3.micro"
  username            = data.vault_kv_secret_v2.db.data["username"]
  password            = data.vault_kv_secret_v2.db.data["password"]
  skip_final_snapshot = true
}
```

### Provider configuration

```hcl
provider "vault" {
  address = "https://vault.example.com:8200"
  # Token comes from VAULT_TOKEN env var, or use an auth method block:
  # auth_login { path = "auth/approle/login" ... }
}
```

| Auth method | Use case |
| --- | --- |
| Token (`VAULT_TOKEN`) | Local development |
| AppRole | CI/CD pipelines |
| AWS IAM | Terraform running on EC2 / EKS |
| JWT / OIDC | GitHub Actions, GitLab CI |

> **Critical warning:** Terraform writes every data source result into **state in plain
> text**. Reading a secret from Vault does *not* keep it out of state. You must:
> - store state in an encrypted, access-controlled backend (S3 + KMS + DynamoDB lock),
> - restrict who can run `terraform state pull`,
> - prefer **dynamic secrets** (Vault's database engine) with short TTLs so a leaked
>   state value expires quickly.

---

## 10. GitLab CI/CD — workflow, include, rules

Three concepts control *when* pipelines run, *how* configuration is reused, and *which*
jobs execute.

| Keyword | Interview answer |
| --- | --- |
| `workflow` | Controls whether the **pipeline** is created at all |
| `include` | **Reuses/imports** CI/CD configuration from other files |
| `rules` | Controls whether an **individual job** runs, and under what conditions |

### 10.1 `workflow` — should the pipeline run?

`workflow:rules` is the **pipeline-level gate**.

```yaml
workflow:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - when: never
```

What happens:
- Merge request created or updated → pipeline runs.
- Push to `main` → pipeline runs.
- Push to any other branch → **no pipeline at all**.

### 10.2 `include` — reuse YAML from other files

Instead of one giant `.gitlab-ci.yml`, split by concern:

```
my-project/
├── .gitlab-ci.yml
├── build.yml
├── test.yml
├── security.yml
└── deploy.yml
```

```yaml
include:
  - local: 'build.yml'
  - local: 'test.yml'
  - local: 'security.yml'
  - local: 'deploy.yml'

stages:
  - build
  - test
  - security
  - deploy
```

GitLab merges the included files with the main file when creating the pipeline. Jobs then
run according to their stages and rules.

**Include types**

| Type | Purpose |
| --- | --- |
| `local` | Include a YAML file from the same repository |
| `project` | Reuse YAML from another GitLab project |
| `remote` | Include YAML from a remote URL |
| `template` | Use a GitLab-provided CI/CD template |
| `component` | Use a versioned CI/CD component (newer GitLab) |

Reusing a centralized DevOps template:

```yaml
include:
  - project: 'platform/devops-templates'
    ref: main
    file: '/templates/terraform.yml'
```

> **Production tip:** pin the template to a **protected tag or commit SHA** instead of
> `main`, so an upstream change cannot silently alter your pipeline.

### 10.3 `rules` — should this job run?

Unlike `workflow`, job-level `rules` control individual jobs.

```yaml
build:
  stage: build
  script:
    - echo "Building application"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy:
  stage: deploy
  script:
    - echo "Deploying application"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

Result:
- On `main`, `build` runs automatically.
- `deploy` waits for a manual trigger.
- On other branches, neither job is added.

GitLab evaluates rules **in order and stops at the first match**.

### 10.4 Common rules patterns

**A. Only on the main branch**
```yaml
rules:
  - if: '$CI_COMMIT_BRANCH == "main"'
```

**B. Only for merge requests**
```yaml
rules:
  - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

**C. Only when Terraform files change**
```yaml
rules:
  - changes:
      - 'terraform/**/*.tf'
      - 'terraform/**/*.tfvars'
```

**D. Manual job**
```yaml
rules:
  - when: manual
```

**E. Never run**
```yaml
rules:
  - when: never
```

**F. Combine condition + changes + allow failure**
```yaml
rules:
  - if: '$CI_COMMIT_BRANCH == "main"'
    changes: ['src/**/*']
    when: on_success
    allow_failure: false
  - when: never
```

> **Gotcha:** `rules:changes` behaves differently for pipelines without a push event
> (scheduled or manually started), where it can evaluate to always-true. Use `compare_to`
> when you need an explicit comparison baseline.

### 10.5 `include` with `rules` — conditional configuration

You can decide whether to include an entire file based on a condition:

```yaml
include:
  - local: 'terraform.yml'
    rules:
      - if: '$RUN_TERRAFORM == "true"'

  - local: 'application.yml'
    rules:
      - if: '$RUN_APPLICATION == "true"'
```

`include:rules` decides which **configuration** enters the pipeline;
job-level `rules` decides which **jobs** enter it.

### 10.6 Complete working example

```yaml
workflow:
  rules:
    # Run pipeline for merge requests
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    # Run pipeline for the main branch
    - if: '$CI_COMMIT_BRANCH == "main"'
    # Don't create a pipeline for anything else
    - when: never

stages:
  - build
  - test
  - deploy

build:
  stage: build
  script:
    - echo "Building application"

test:
  stage: test
  script:
    - echo "Running tests"

deploy:
  stage: deploy
  script:
    - echo "Deploying application"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

---

## 11. "Disk Full" But Space Is Available

When a system reports a full disk despite showing available space, the cause is usually
**metadata exhaustion**, **hidden application limits**, or **filesystem constraints**.

### Triage order

```
1. df -h        → is the filesystem actually full?
2. df -i        → are inodes exhausted?
3. lsof | grep deleted → are deleted files still held open?
4. tune2fs -l   → is root-reserved space the culprit?
5. du -xh / | sort -h | tail   → where is the space actually going?
```

### 11.1 Out of inodes (most common Linux cause)

- **Problem:** every file and directory consumes an **inode** — an entry in the filesystem
  table. Millions of tiny files exhaust inodes even with hundreds of GB free.
- **Check:** `df -i`
- **Fix:** if `IUse%` is 100%, find and delete directories holding huge numbers of tiny
  files — session caches, temp files, mail queues, old log fragments.

```bash
df -i

# Find the directories with the most files
sudo find / -xdev -type f 2>/dev/null \
  | sed 's|/[^/]*$||' | sort | uniq -c | sort -rn | head -20
```

### 11.2 Deleted files still held open by processes

- **Problem:** if a large file is deleted while a process still has it open, the space is
  **not** freed until that process closes the handle or exits. `df` shows the space as
  used; `du` does not account for it.
- **Check:** `lsof +L1` or `lsof | grep deleted`
- **Fix:** restart the process or service identified by `lsof`.

```bash
sudo lsof +L1                      # files with link count 0 (deleted but open)
sudo lsof -nP | grep '(deleted)'

# Emergency: truncate the file in place without restarting the process
sudo truncate -s 0 /proc/<PID>/fd/<FD>
```

Classic culprit: an application writing to a log file that `logrotate` deleted without
sending a reopen signal.

### 11.3 Reserved space for root

- **Problem:** `ext4` reserves ~5% of total blocks for the root user. A regular user hits
  "disk full" while `df` still reports 5% free.
- **Check:** `sudo tune2fs -l /dev/sdX | grep -i reserved`
- **Fix:** reduce the reservation on **data** volumes (never on `/`).

```bash
sudo tune2fs -m 1 /dev/sdX    # drop reservation from 5% to 1%
```

> Keep the reservation on the root filesystem — it exists so root can still log in and
> system daemons can keep running when the disk fills.

### 11.4 Other causes worth checking

| Cause | Check | Fix |
| --- | --- | --- |
| Mount shadowing (files written under a mountpoint) | `du` on the unmounted path | Unmount, clean, remount |
| Docker image/layer/volume bloat | `docker system df` | `docker system prune -a --volumes` |
| Kubernetes node ephemeral storage | `kubectl describe node` → `DiskPressure` | Prune images, add `emptyDir` limits |
| LVM thin pool exhausted | `lvs` → `Data%` | Extend the thin pool |
| Snapshots consuming space | `btrfs subvolume list` / `zfs list -t snapshot` | Delete old snapshots |
| Quota limit, not disk limit | `quota -u <user>`, `repquota -a` | Raise or clear the quota |

---

## Glossary

| Term | Meaning |
| --- | --- |
| **STS** | AWS Security Token Service — issues temporary credentials |
| **OIDC** | OpenID Connect — token-based federated identity protocol |
| **IRSA** | IAM Roles for Service Accounts — maps an EKS ServiceAccount to an IAM role |
| **SAML** | XML-based federation standard used by enterprise IdPs |
| **KEDA** | Kubernetes Event-Driven Autoscaling |
| **HPA / VPA** | Horizontal / Vertical Pod Autoscaler |
| **ScaledObject** | KEDA CRD that defines what to scale and on which triggers |
| **ISMS** | Information Security Management System (the core of ISO 27001) |
| **PHI / ePHI** | (Electronic) Protected Health Information under HIPAA |
| **BAA** | Business Associate Addendum — required HIPAA contract with a vendor |
| **Attestation** | An auditor's report on controls (SOC 2), distinct from certification |
| **Dynamic block** | Terraform construct that generates repeated nested blocks |
| **KV v2** | Vault's versioned key-value secrets engine |
| **kubeconfig** | File defining clusters, users, and contexts for `kubectl` |
| **Context** | A named cluster + user + namespace triple |
| **RBAC** | Role-Based Access Control |
| **Inode** | Filesystem metadata entry; one per file or directory |

---

## Key Takeaways

1. **Stop creating IAM users.** Roles, SSO, and OIDC federation all give short-lived
   credentials — there is no permanent key to leak.
2. **Pick a database by access pattern,** not by the SQL/NoSQL label. Both camps now
   support transactions and horizontal scaling to varying degrees.
3. **KEDA scales on events and can reach zero**; HPA cannot. Use KEDA for queue consumers
   and bursty workloads.
4. **VPA resizes pods and restarts them.** Start in `Off` mode, always set
   `minAllowed`/`maxAllowed`, and never share a metric with HPA.
5. **ISO 27001 certifies a system, SOC 2 attests to controls, HIPAA is a law.**
   The underlying DevOps controls — encryption, least privilege, audit logging,
   change management — are largely the same.
6. **Terraform `dynamic` blocks** generate nested blocks only; use resource-level
   `for_each` for multiple resources. Use sparingly for readability.
7. **Vault secrets still land in Terraform state in plain text.** Encrypt the backend and
   prefer short-TTL dynamic secrets.
8. **For many clusters, generate kubeconfigs from automation** and switch with contexts.
   Kubernetes has no `User` object — identity comes from certs, tokens, or OIDC; RBAC
   authorizes it.
9. **GitLab CI:** `workflow` gates the pipeline, `include` reuses config, `rules` gates
   jobs. Rules evaluate top-down and stop at the first match.
10. **"Disk full" with free space** is almost always inodes, deleted-but-open files, or
    root-reserved blocks. Check `df -i` and `lsof +L1` before anything else.
