Most Asked DevOps Interview Questions
Question 1. When would you use depends_on in Terraform? Explain with a real-time
Answer:

"depends_on is used to create an explicit dependency between Terraform resources when Terraform cannot infer the dependency automatically.

Question 2. If an entire AWS Region goes down, how would you design a Disaster Recovery solution?
Answer:

How would you handle database backups, snapshots, and S3 Cross-Region Replication as part of a DR strategy?

Production
              ap-south-1
                  |
           PostgreSQL / RDS
                  |
       +----------+----------+
       |                     |
    Snapshot              Backup
       |                     |
       |                     v
       |                  S3 Bucket
       |                     |
       |              S3 Replication
       |                     |
       |                     v
       |                  S3 Bucket
       |                  us-east-1
       |
       v
  DB Snapshot
Question 3. How corss region replication is done
Answer:

Yes. Let's walk through how S3 Cross-Region Replication (CRR) is actually configured, first manually and then how you'd do it with Terraform.

1. Architecture
Assume your primary application is in Mumbai:

Primary Region
ap-south-1
    |
    v
S3: prod-backups
    |
    | Cross-Region Replication
    |
    v
DR Region
us-east-1
    |
    v
S3: prod-backups-dr
When you upload:

database-backup-2026-10-02.sql.gz

to prod-backups, S3 replicates it to prod-backups-dr.

2. Create the two buckets
Create:

prod-backups

in:

ap-south-1

and:

prod-backups-dr

in:

us-east-1

Important
Enable versioning on both buckets.

Primary bucket
Versioning: ENABLED
        ↓
DR bucket
Versioning: ENABLED
S3 replication requires versioning.

3. Create IAM role for S3 replication
S3 needs permission to read objects from the source bucket and write replicas to the destination.

Conceptually:

S3
|
| AssumeRole
v
IAM Replication Role
|
+---- Read source objects
|
+---- Replicate objects
    to destination
The role gets permissions such as:

s3:GetObjectVersion
s3:GetObjectVersionAcl
s3:GetObjectVersionForReplication
s3:ReplicateObject
s3:ReplicateDelete
The exact permissions depend on your replication configuration.

4. Configure the replication rule
In the source bucket:

prod-backups
    ↓
Management
    ↓
Replication rules
    ↓
Create replication rule
alt text

You configure:

Rule name:
database-backups

Status:
Enabled

Destination:
prod-backups-dr

You can replicate:

Everything
/
or only a particular prefix:

database/

or use object tags.

For example:

s3://prod-backups/database/

gets replicated, while unrelated objects don't.

5. What happens internally?
Suppose you upload:

s3://prod-backups/database/db-001.sql.gz

S3 sees the replication rule:

Source:

s3://prod-backups/database/*
                |
                | matches rule
                v
    Replication process
                |
                v
Destination:
s3://prod-backups-dr/database/db-001.sql.gz

You don't need a Lambda function to copy every file yourself.

S3 handles the replication.

Question 4. How can CloudFront serve different content based on the user's country/location?
Answer:

"CloudFront uses edge locations to serve cached content close to users, but edge location selection itself doesn't provide country-specific content. For geo-based content, I can use the viewer's country information, such as the CloudFront-Viewer-Country header, and use CloudFront Functions or Lambda@Edge to modify the request or route it to the appropriate path or origin. For example, requests from India can be routed to /india/, while requests from the US can be routed to /us/. If I only want to restrict access by country, I would use CloudFront geographic restrictions instead."

CloudFront automatically chooses the nearest/appropriate edge location for the user, but it doesn't automatically serve different content by country. You configure country-based behavior.

Real-time example
Suppose you have an e-commerce application:

                CloudFront
                    |
        +-----------+-----------+
        |                       |
    India                    USA
        |                       |
Indian website           US website
/in/index.html           /us/index.html
You want:

User from India
    ↓
CloudFront
    ↓
India-specific content
User from USA
    ↓
CloudFront
    ↓
US-specific content
CloudFront can determine the viewer's country using the CloudFront-Viewer-Country header.

How do you actually implement it?
There are a few approaches.

Option 1 — CloudFront Functions / Lambda@Edge
You can inspect the country and modify the request.

Conceptually:

User
|
v
CloudFront Edge
|
| CloudFront-Viewer-Country
|
+---- IN ---> /india/*
|
+---- US ---> /usa/*
|
+---- GB ---> /uk/*
|
v
Origin
For example: implement this in cloudfrontfunction (js)

if (country === "IN") {
    request.uri = "/india" + request.uri;
}
alt text

So:

User requests:

/index.html

Country = IN
CloudFront changes request to:

/india/index.html

Option 2 — Different origins
You can also configure CloudFront with different origins:

            CloudFront
                |
    Country-based routing
        /        |        \
    /         |         \
    IN          US        EU
    |           |          |
    v           v          v
S3-IN       S3-US       S3-EU
For example:

India → S3 bucket in India
USA   → S3 bucket in US
Europe → S3 bucket in Europe
The routing logic can be implemented using CloudFront Functions/Lambda@Edge or an application/origin routing layer, depending on the architecture.

Option 3 — CloudFront geographic restrictions
Don't confuse this with geo restriction.

Geo restriction answers:

"Should this country be allowed to access my content?"

For example:

India → Allow
USA   → Allow
North Korea → Block
It does not mean:

India → Indian content
USA → American content
For different content, you need request-based routing/logic.

Question 5. How would you design a multi-cloud environment?
Answer:

Your answer describes one possible scenario, but for the interview question “How would you design a multi-cloud environment?”, the interviewer is looking for a broader architecture: networking, workload placement, identity, DNS, data, security, observability, and DR.

Also, I wouldn't frame it as “create a GCP Kubernetes cluster and maintain it from AWS.” The management plane doesn't necessarily need to live in AWS.

A stronger approach is:

                Users
                  |
                  v
          Global DNS / Traffic
                  |
      +-----------+-----------+
      |                       |
      v                       v
     AWS                     GCP
      |                       |
     ALB                     LB
      |                       |
     EKS                     GKE
      |                       |
Application A           Application A
      |                       |
      +----------+------------+
                 |
           Data / Services
1. Decide why you're using multiple clouds
This is actually the first design decision.

For example:

AWS
 ├── EKS
 ├── RDS
 └── S3
GCP
 ├── GKE
 ├── Cloud SQL
 └── Cloud Storage
You might use multi-cloud for DR, regulatory/data-residency requirements, avoiding dependency on a single provider, acquisitions, or because a particular workload benefits from a service in another cloud.

You shouldn't make every component multi-cloud just because you can; that adds significant operational complexity.

2. Establish connectivity
This is the part you mentioned, and it's important.

At a basic level:

AWS VPC
   |
VPN / private connectivity
   |
GCP VPC
For production, depending on requirements, you might use dedicated connectivity through AWS Direct Connect and Google Cloud Interconnect, often with a connectivity provider.

You'd also plan:

Non-overlapping CIDRs
Routing
Firewall rules
Private DNS
Encryption
Network segmentation
For example, don't accidentally design:

AWS VPC: 10.0.0.0/16
GCP VPC: 10.0.0.0/16
if those networks need straightforward routed connectivity.

3. Kubernetes
This is where your example fits.

You could have:

   GitHub / GitLab
          |
       CI/CD
          |
   GitOps / Argo CD
      /       \
     /         \
    v           v
AWS EKS       GCP GKE
   |             |
App v1         App v1
Rather than managing GKE from AWS, I'd use a centralized deployment/management approach such as GitOps.

For example:

Git
 |
 | desired Kubernetes state
 v
Argo CD
 |
 +------> EKS
 |
 +------> GKE
Now the same deployment standards can be applied across both clouds.

4. Infrastructure as Code
Terraform can give you a common provisioning workflow:

Terraform
   |
   +---- AWS Provider
   |       |
   |       +-- VPC
   |       +-- EKS
   |       +-- RDS
   |
   +---- Google Provider
           |
           +-- VPC
           +-- GKE
           +-- Cloud SQL
You would generally separate modules/state appropriately rather than having one enormous Terraform state for the entire multi-cloud estate.

5. Identity and secrets
Avoid distributing permanent AWS/GCP credentials everywhere.

Conceptually:

Central Identity
      |
      +----> AWS IAM
      |
      +----> GCP IAM
For application secrets, you could use cloud-native secret stores or a centralized system such as Vault, depending on the requirements.

Applications
     |
     v
HashiCorp Vault
     |
     +-- DB credentials
     +-- API keys
     +-- Certificates
6. Centralized observability
This connects directly to what we discussed earlier.

EKS ──┐
      |
      +----> OpenTelemetry ----> Datadog
      |
GKE ──┘
Now you can have centralized:

Logs
Metrics
Traces
Alerts
Dashboards
instead of separately troubleshooting AWS and GCP.

7. Traffic management and DR
Suppose your goal is multi-cloud disaster recovery.

Normally:

       Global DNS
           |
   Health checking
     /           \
    v             v
 AWS             GCP
Primary           DR
  |               |
 EKS             GKE
If the AWS environment becomes unavailable, your documented failover process can direct traffic to the GCP environment. The exact DNS/global traffic product and whether failover is automatic depend on your architecture.

The difficult part isn't actually Kubernetes. Data consistency is usually much harder.

You need to decide:

AWS Database
     |
     | replication / backup strategy
     v
GCP Database
and define:

RPO = How much data can we lose?
RTO = How long can recovery take?
That determines whether you need active-active, active-passive, asynchronous replication, backup/restore, etc.

Interview answer
I'd structure your answer like this:

“I would first understand why we need multi-cloud and define the availability, RPO and RTO requirements. Then I'd establish secure connectivity between the cloud networks with non-overlapping CIDRs and appropriate routing and firewall policies. For workloads, for example, we could run EKS in AWS and GKE in GCP, with Terraform provisioning both environments and GitOps providing consistent Kubernetes deployments. I'd centralize identity/secrets and observability, using something like OpenTelemetry with Datadog for metrics, logs and traces. For DR, I'd design global traffic management and, most importantly, a cross-cloud data replication or backup strategy. I'd also avoid tightly coupling one cloud to the other so that a failure of AWS doesn't prevent us from operating the GCP environment.”

1. "A particular workload benefits from a service in another cloud"
It means:

You use Cloud A for most of your infrastructure, but Cloud B has a service that is particularly useful for one specific workload.

Example
Imagine your company primarily runs on AWS:

AWS
├── EKS
├── RDS
├── S3
└── Application
But your ML team needs a workload that benefits from Google Cloud's Vertex AI / TPU ecosystem.

You could have:

        Application
            |
 +----------+----------+
 |                     |
 v                     v
AWS                   GCP
 |                     |
EKS               Vertex AI
 |                     |
 +----------+----------+
            |
      ML predictions
So you're not moving everything to GCP. You're using GCP specifically because that workload benefits from its ML capabilities.

Another example could be:

AWS → Main application
GCP → BigQuery analytics
Your transactional application stays in AWS, while large-scale analytics is performed using GCP's analytics services.

2. What does "regulatory/data-residency requirements" mean?
Data residency
Data residency means:

Certain data must be stored/processed in a particular geographic location because of laws, regulations, contracts, or company policies.

For example, imagine a company operates in India and Europe.

It might have a requirement like:

Indian customer data
        ↓
Must remain in India
        ↓
AWS ap-south-1
while European customer data might be:

European customer data
        ↓
Must remain within approved EU locations
        ↓
EU cloud region
So your architecture could be:

           Global Application
                 |
      +----------+----------+
      |                     |
 Indian users          EU users
      |                     |
      v                     v
 AWS India              Cloud EU
ap-south-1             EU region
      |                     |
 India data             EU data
The application may be globally accessible, but the data storage location is controlled based on where the customer/data is subject to the applicable requirements.

Question 6. How can Kyverno be used to strengthen Kubernetes security policies?
Answer:

How it works
               Kubernetes Cluster
                      |
                 Kyverno
               Policy Engine
                      |
        +-------------+-------------+
        |             |             |
     Deployment      Pod          Service
        |
        v
  Policy evaluation
        |
  +-----+------+
  |            |
PASS          FAIL
  |            |
Allow        Reject
For example, you may want to enforce:

"Containers must not run as root."

You create a Kyverno policy:

apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-root
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-run-as-non-root
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Containers must run as non-root."
        pattern:
          spec:
            containers:
              - securityContext:
                  runAsNonRoot: true
Then someone tries:

apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  containers:
    - name: app
      image: nginx
Kyverno evaluates it:

Pod submitted
     |
     v
Kyverno
     |
     | runAsNonRoot missing
     v
   DENY
The deployment doesn't get admitted when the policy is configured in Enforce mode.

How Kyverno itself is installed
You can install Kyverno using Helm:

helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace
That installs Kyverno's components into the cluster.

Then you create your own policies as Kubernetes YAML resources.

So distinguish:

Helm
  ↓
Installs Kyverno
Kyverno
  ↓
Evaluates Kubernetes resources
Kyverno Policies
  ↓
Define your security/compliance rules
What can you enforce?
For example:

❌ Containers running as root
❌ Privileged containers
❌ HostNetwork enabled
❌ HostPath volumes
❌ Images without approved registries
❌ Missing resource limits
❌ Missing required labels
❌ Unapproved image tags
You can also use Kyverno for mutation and generation, not just validation.

Better interview answer
"We use Kyverno as a Kubernetes-native policy engine to strengthen security and compliance. We install Kyverno into the cluster using Helm and define ClusterPolicies that validate or mutate Kubernetes resources. For example, we can enforce that containers run as non-root, don't use privileged mode, only pull images from approved registries, and have required security contexts and resource limits. In Enforce mode, Kyverno can reject resources that violate the policy before they're admitted to the cluster."

Question 7. How would you structure Terraform code for a large-scale project? How would you create reusable Terraform modules? How would you use the same Terraform modules across Dev, Stage, and Production with different configurations?
Answer:

files for dev, stage, and prod just to change configuration. Keep the module code reusable and keep environment-specific configuration separate.

1. Large-scale Terraform structure
A common structure is:

terraform-infra/
│
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── eks/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── rds/
│   └── iam/
│
└── environments/
    ├── dev/
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── terraform.tfvars
    │   └── backend.tf
    │
    ├── stage/
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── terraform.tfvars
    │   └── backend.tf
    │
    └── prod/
        ├── main.tf
        ├── variables.tf
        ├── terraform.tfvars
        └── backend.tf
The important separation is:

Reusable logic
      ↓
   modules/
Environment-specific configuration
      ↓
environments/dev
environments/stage
environments/prod
Separate Terraform state
This is very important in a large-scale project.

Don't use one state file for:

dev + stage + prod

Instead:

Terraform State
      |
      +---- dev.tfstate
      |
      +---- stage.tfstate
      |
      +---- prod.tfstate
For example, with an S3 backend:

S3 Terraform State
│
├── dev/terraform.tfstate
├── stage/terraform.tfstate
└── prod/terraform.tfstate
This gives you isolation.

If someone runs:

terraform apply
Question 8. What happens when someone manually changes an AWS resource that is managed by Terraform?
Answer:

Terraform can detect the manual change by refreshing its state against the real AWS resource.

The flow is:

Terraform Configuration
        |
        v
terraform.tfstate
        |
        | Terraform refresh/read
        v
Actual AWS Resource
        |
        | someone manually changed it
        v
Drift detected
Example
Terraform originally manages:

resource "aws_instance" "app" {
  instance_type = "t3.medium"
}
Terraform state says:

instance_type = t3.medium
Someone manually changes the EC2 instance in AWS Console:

t3.medium → t3.large
Now you have configuration vs real infrastructure drift:

Terraform configuration → t3.medium
Terraform state         → t3.medium
AWS actual resource     → t3.large
When you run:

terraform plan
Terraform refreshes the resource information from AWS and can detect the difference:

~ aws_instance.app
instance_type:

"t3.large" → "t3.medium"
Terraform is essentially saying:

"AWS currently has t3.large, but your desired configuration says t3.medium."

If you apply the plan, Terraform will generally try to bring AWS back to the configuration:

AWS:

t3.large
    ↓
terraform apply
    ↓
t3.medium
What about terraform refresh?
Your answer mentions:

terraform refresh
Historically, this command was used to update state from real infrastructure, but you generally shouldn't use terraform refresh as your normal drift-detection workflow now.

Instead, use:

terraform plan
Terraform refreshes/read current resource state as part of planning.

You can also use:

terraform plan -refresh-only
when your specific goal is to update the state to reflect changes made outside Terraform without proposing configuration changes to infrastructure.

Very important distinction
There are three things:

              Terraform
                 |
     +-----------+-----------+
     |           |           |
Configuration   State      AWS
   desired     recorded    actual
Manual AWS change:

Configuration: t3.medium
State:         t3.medium
AWS:           t3.large
                         ↑
                       DRIFT
terraform plan detects the drift.
Strong interview answer
"If someone manually changes an AWS resource managed by Terraform, we have configuration drift. For example, if Terraform specifies an EC2 instance as t3.medium but someone changes it to t3.large through the AWS console, Terraform can detect the difference during the refresh phase of terraform plan. The plan will show that the actual resource differs from the desired configuration and, depending on the resource, Terraform will propose changing it back. I wouldn't normally use the old terraform refresh command; I'd use terraform plan, or terraform plan -refresh-only when I specifically want to reconcile the state with the real infrastructure."

Yeah, when someone changed it manually in the AWS, then we have run this Terraform plan and we identified that there is a drift and something has been changed. If we want to prevent that change, do we need to manage from Terraform or we can directly edit that one from AWS?

If the resource is managed by Terraform, the preferred approach is to make the desired change in Terraform, not directly in AWS.

Think of Terraform as the source of truth:

Git / Terraform code
        |
        v
   Desired state
        |
        v
       AWS
Example
Terraform says:

resource "aws_instance" "app" {
  instance_type = "t3.medium"
}
Someone changes AWS manually:

AWS:

t3.medium → t3.large
Now:

Terraform code = t3.medium
AWS            = t3.large
                  ↑
                DRIFT
You run:

terraform plan
Terraform shows:

~ instance_type = "t3.large" -> "t3.medium"
If the t3.large change was accidental, you can:

terraform apply
and Terraform brings it back to:

t3.medium

What if you actually want t3.large?
Don't change it manually in AWS and leave Terraform saying t3.medium.

Instead, change Terraform:

resource "aws_instance" "app" {
  instance_type = "t3.large"
}
Then:

git commit
terraform plan
terraform apply
Now:

Terraform code = t3.large
AWS            = t3.large
                 ↓
               No drift
So remember
Situation	What to do
Accidental AWS change	Terraform apply can restore it
Intentional change	Change Terraform code → plan → apply
Emergency AWS change	Make the emergency change, then reconcile Terraform afterward
Permanent manual management	Don't manage that resource with Terraform
Question 9. A new application version has been deployed, but customers cannot access the application. How would you troubleshoot it?
Answer:

Troubleshooting flow
Customer
   |
   v
DNS
   |
   v
Load Balancer
   |
   v
Ingress
   |
   v
Service
   |
   v
Endpoints / EndpointSlices
   |
   v
Pod
   |
   v
Application
   |
   v
Database / External dependency
I'd troubleshoot each layer in that order.

1. Confirm the symptom
First determine:

HTTP 4xx?
HTTP 5xx?
Timeout?
Connection refused?
DNS failure?
For example:

curl -v https://myapp.example.com
This immediately tells you whether you're dealing with DNS, TLS, HTTP, or connectivity.

2. Check DNS
dig myapp.example.com
Verify that the hostname resolves to the expected load balancer.

DNS
 ↓
Correct LB address?
3. Check Load Balancer
Check:

Listener
Target/backend health
Security groups/firewalls
TLS certificate
Listener rules
For AWS:

aws elbv2 describe-target-health ...
You want:

Load Balancer
      |
      v
Healthy target?
If targets are unhealthy, investigate why.

4. Check Kubernetes Ingress
kubectl get ingress -n <namespace>
kubectl describe ingress <name> -n <namespace>
Check:

Host
Path
Backend service
Ingress controller
TLS
A common deployment problem is:

Ingress
   |
   v
service: payment-service
but the new deployment actually created:

payment-service-v2

or the service selector doesn't match the new Pods.

5. Check Service
kubectl get svc -n <namespace>
kubectl describe svc <service> -n <namespace>
Then check endpoints:

kubectl get endpoints <service> -n <namespace>
or preferably:

kubectl get endpointslice -n <namespace>
You want:

Service
   |
   +---- selector
   |
   v
EndpointSlice
   |
   +---- Pod IP
   +---- Pod IP
If there are zero endpoints, the Service isn't selecting your Pods.

For example:

selector:
  app: payment
but your new Pods have:

labels:
  app: payment-v2
Then:

Service selector
      ↓
No matching Pods
      ↓
0 endpoints
      ↓
Customers get errors
6. Check Pods
kubectl get pods -n <namespace>
Look for:

CrashLoopBackOff
ImagePullBackOff
Pending
0/1 Ready
Then:

kubectl describe pod <pod> -n <namespace>
Pay attention to:

Events
Readiness probe
Liveness probe
Image
Environment variables
Mounts
Readiness is particularly important.

A Pod can be:

Running

but still:

Not Ready

In that case Kubernetes won't normally put it into the Service's ready endpoints.

7. Check application logs
Now check:

kubectl logs <pod> -n <namespace>
If the container restarted:

kubectl logs <pod> -n <namespace> --previous
Look for:

Database connection failure
Configuration missing
Authentication failure
Port binding failure
External API timeout
8. Check dependencies
If the application itself is healthy but requests fail:

Application
    |
    +---- Database
    |
    +---- Redis
    |
    +---- Kafka
    |
    +---- External API
Check connectivity and credentials.

One very important thing: compare with the previous version
Because the problem started after a new deployment, I'd immediately compare:

Old version
vs
New version
Check:

Image
Environment variables
ConfigMap
Secrets
Service port
Container port
Readiness probe
Liveness probe
Resource limits
Ingress configuration
Database migrations
For example, a classic deployment issue:

Service:
targetPort: 8080

New application:
listens on 8081

Everything can look healthy at the Kubernetes level, but traffic won't reach the application correctly.

Strong interview answer
"Since the issue started immediately after a new deployment, I'd troubleshoot from the client toward the application and compare the new version with the previous working version. First I'd verify DNS and the load balancer, including listeners, target health and TLS. Then I'd check the Kubernetes Ingress and verify that it routes to the correct Service. I'd inspect the Service selectors and EndpointSlices to make sure the new Pods are actually registered as ready endpoints. Then I'd check Pod status, events, readiness and liveness probes, and container logs. Finally, I'd check application dependencies such as the database, Redis or external APIs. I'd also compare the new image, configuration, secrets, ports and probes against the previous working version. If the new deployment is clearly responsible, I'd consider rolling back while continuing the investigation."

The key sequence to remember for interviews is:

DNS → LB → Ingress → Service → Endpoints → Pod → Application → Dependencies.
HTTP 4xx? HTTP 5xx? Timeout? Connection refused?
when will we get this errors
These errors tell you where the request is failing. A useful way to remember them is:

4xx → Client/request problem
5xx → Server/application/infrastructure problem
Timeout → Nobody responded in time
Connection refused → Something actively rejected the TCP connection
1. HTTP 4xx — client/request problem
The server received the request, but the request isn't acceptable or the client isn't authorized.

Common examples:

Code	Meaning	Example
400	Bad Request	Invalid JSON/body
401	Unauthorized	Missing/invalid authentication
403	Forbidden	User authenticated but not allowed
404	Not Found	Wrong URL/path
405	Method Not Allowed	GET sent to POST-only endpoint
429	Too Many Requests	Rate limit exceeded
Example
Client
  |
  | GET /api/payment/abc
  v
Application
  |
  +--> 404 Not Found
The network connection worked. The application received the request and returned a response.

2. HTTP 5xx — server-side problem
The request reached the server, but something went wrong while processing it.

Common ones:

Code	Meaning	Example
500	Internal Server Error	Application exception
502	Bad Gateway	Proxy/LB got invalid response from backend
503	Service Unavailable	No healthy backend / service unavailable
504	Gateway Timeout	Upstream didn't respond in time
Example: 500
Client
  |
  v
Load Balancer
  |
  v
Application
  |
  X
Exception
  |
  v
HTTP 500
Application logs might show:

NullPointerException
Database connection failed
3. 502 Bad Gateway
This is particularly important for Kubernetes.

Suppose:

Client
   |
   v
Load Balancer
   |
   v
Ingress
   |
   v
Service
   |
   v
Pod
The proxy/Ingress successfully received the request but couldn't get a valid response from its upstream.

Possible reasons:

Ingress
   |
   X----> Service
          |
          X----> Pod
Examples:

Backend connection failed
Wrong Service port
Application isn't listening on expected port
Invalid upstream response
Backend closed the connection unexpectedly
4. 503 Service Unavailable
Usually means:

There is currently no usable backend available to serve the request.

For Kubernetes:

Ingress
   |
   v
Service
   |
   v
Endpoints
   |
   X
0 ready endpoints
For example:

kubectl get endpoints payment-service
returns no addresses.

Why could that happen?
Pod
 ├── CrashLoopBackOff
 ├── Readiness probe failing
 └── Wrong Service selector
Then the Service has no ready endpoints, and the ingress/load balancer may return 503.

5. 504 Gateway Timeout
This means the gateway/proxy waited for the upstream but didn't receive a response within its timeout.

Example
Client
  |
  v
Load Balancer
  |
  v
Ingress
  |
  v
Application
  |
  v
Database
  |
  X---- very slow
Maybe the application is waiting 60 seconds for a database query.

Then:

Ingress waits
    ↓
Timeout
    ↓
504
Typical causes:

Slow application
Slow database
External API hanging
Network problem
Timeout configured too aggressively
6. Connection refused
This is different from an HTTP error.

You may see:

curl: (7) Failed to connect ... Connection refused
This means the TCP connection reached the destination, but nothing is accepting connections on that IP:port, or the connection is actively rejected.

Example
Client
  |
  | TCP connection :8080
  v
Pod IP
  |
  X
Nothing listening on 8080
For example, your Kubernetes configuration says:

containerPort: 8080
but the application actually listens on:

8081

You could get connection refused.

7. Timeout vs Connection Refused
This is a very important distinction.

Connection refused
Client
  |
  | SYN
  v
Server
  |
  X---- RST / rejected
Usually:

"I reached the destination, but nobody is accepting connections on that port."

Timeout
Client
  |
  | SYN
  v
      ????
      |
      X---- No response
Usually:

"I couldn't establish communication or get a response within the timeout."

Possible causes:

Security group
Network ACL
Firewall
Network routing
Broken network path
Application hanging
8. How I use these during troubleshooting
If customers say "application is down", first:

curl -v https://myapp.example.com
Suppose you get:

404

I'd investigate:

URL
Ingress path
Application routes
401/403

I'd investigate:

Authentication
Authorization
IAM/token
Ingress auth
502

I'd investigate:

Ingress
Service
Service port
Pod port
Backend connectivity
503

I'd immediately check:

kubectl get endpoints <service>
kubectl get endpointslice -n <namespace>
kubectl get pods -n <namespace>
504

I'd investigate:

Application latency
Database
External APIs
Ingress/LB timeout
Connection refused

I'd check:

Is application listening?
Correct containerPort?
Correct Service targetPort?
Is the process running?
Timeout

I'd investigate:

DNS
Routing
Security groups/firewalls
Network policies
Load balancer
Application responsiveness
So during your interview, don't just say "I'll check the logs." The HTTP status itself gives you a very useful first clue about which layer to investigate next.

Question 10. What are Jenkins Controllers and Agents, and how do they work?
Answer:

"The Jenkins controller is the central orchestration component. It receives triggers, loads the pipeline configuration, schedules jobs, manages Jenkins configuration and credentials, and decides which agent should execute each stage. Jenkins agents are worker nodes where the actual build, test, Docker, security scanning, and deployment commands execute. The controller assigns a job to an agent based on labels or the pipeline's agent configuration. In a production setup, I would avoid running resource-intensive builds on the controller and use dedicated or ephemeral agents, often Kubernetes-based, to scale build workloads."

Question 11. Declarative vs Scripted Pipeline?
Answer:

1. Declarative vs Scripted Jenkins Pipeline
Your current answer is a little mixed up.

Both are written in Groovy, but the structure and philosophy are different.

Declarative Pipeline
Declarative is more structured and opinionated:

pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Deploy') {
            steps {
                sh './deploy.sh'
            }
        }
    }
}
It follows a predefined structure:

pipeline
 ├── agent
 ├── stages
 │    ├── stage
 │    │    └── steps
 │    └── stage
 └── post
It's easier to read, validate and maintain.

You can also define:

environment {}
parameters {}
options {}
triggers {}
when {}
post {}
Scripted Pipeline
Scripted Pipeline gives you much more programming flexibility:

node {
    stage('Build') {
        sh 'mvn clean package'
    }

    if (env.BRANCH_NAME == 'main') {
        stage('Deploy') {
            sh './deploy.sh'
        }
    }
}
You can use normal Groovy programming constructs:

if/else
for loops
try/catch
variables
functions
dynamic logic
So the simple distinction is:

Declarative	Scripted
Structured	Flexible
Opinionated syntax	More programming-oriented
Easier to maintain	More complex
Easier for teams	Useful for complex logic
Uses pipeline {}	Usually uses node {}
Question 12. CMD vs ENTRYPOINT?
Answer:

1. CMD vs ENTRYPOINT
Your understanding is partially correct.

CMD
CMD provides the default command or default arguments for a container.

FROM ubuntu:24.04

CMD ["sleep", "100"]
Run:

docker run myimage
It executes:

sleep 100

But you can override it:

docker run myimage sleep 500
Now:

sleep 500

is executed.

ENTRYPOINT
ENTRYPOINT defines the main executable of the container.

FROM ubuntu:24.04

ENTRYPOINT ["sleep"]
Now:

docker run myimage 100
results in:

sleep 100

Here 100 becomes an argument to the entrypoint.

Using both
This is a very common pattern:

ENTRYPOINT ["python3"]
CMD ["app.py"]
Docker effectively executes:

python3 app.py

If you run:

docker run myimage test.py
you get:

python3 test.py

So:

ENTRYPOINT = executable
CMD        = default arguments
Important correction to your answer
You said:

"In the case of entry point, we can't override."

That's not completely correct.

You can override ENTRYPOINT using:

docker run --entrypoint /bin/bash myimage
The difference is that CMD is normally easier to override because runtime arguments replace it.

What happens if both CMD and ENTRYPOINT are specified?
They are combined:

ENTRYPOINT + CMD
Example
ENTRYPOINT ["echo"]
CMD ["hello"]
Result:

echo hello

2. Multiple CMD instructions
You were correct here.

If you have:

CMD ["echo", "first"]

CMD ["echo", "second"]
only the last CMD takes effect.

Result:

echo second

Same idea applies to ENTRYPOINT: you should normally have one effective ENTRYPOINT; if multiple are specified, the last one takes effect.

Interview answer
"ENTRYPOINT defines the main executable of the container, while CMD provides default arguments or a default command. When both are used, CMD is appended as arguments to ENTRYPOINT. CMD is easily overridden at runtime, while ENTRYPOINT can also be overridden explicitly using --entrypoint."

Question 13. COPY vs ADD?
Answer:

COPY vs ADD
This is another area where I'd correct your answer.

You said:

"ADD can add a remote repository URL."

That's not the main distinction.

COPY
COPY simply copies files/directories from the build context into the image.

COPY app.py /app/app.py
or:

COPY . /app
It's predictable and is generally preferred when you simply need to copy files.

ADD
ADD has additional functionality.

It can:

Copy local files
Copy directories
Automatically extract local tar archives
Support URL sources in Docker's build mechanisms, though remote URL use has important limitations and is generally not preferred for reproducible builds

Example
ADD app.tar.gz /app/
Docker can automatically extract the archive.

Remote URLs: Downloads a file from a URL directly into the image filesystem. (Note: Remote tarballs are not automatically unpacked).dockerfile
ADD https://example.com /app/config.json
Question 14. How do you reduce Docker image size?
Answer:

Technique 1 — Use a minimal base image
Instead of:

FROM ubuntu
you might use:

FROM alpine
or for Java:

FROM eclipse-temurin:21-jre
rather than a full JDK image for the runtime.

Technique 2 — Multi-stage builds
This is one of the most important techniques.

Instead of putting build tools into your production image:

Maven
JDK
source code
dependencies
build artifacts
you build in one image and copy only the artifact into a smaller runtime image.

Technique 3 — .dockerignore
Don't send unnecessary files into the Docker build context.

.git
node_modules
target
*.log
.env
Technique 4 — Don't install unnecessary packages
Avoid:

RUN apt-get install \
    package1 \
    package2 \
    package3 \
    package4 \
    package5
if the application doesn't need them at runtime.

Technique 5 — Combine package installation and cleanup
For Debian/Ubuntu:

RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
Interview answer
"I reduce Docker image size by using minimal runtime base images, multi-stage builds, .dockerignore, installing only required packages, removing package-manager caches, and copying only the required runtime artifacts into the final image."

Question 15. What are Docker multi-stage builds?
Answer:

5. Multi-stage Docker builds
This is the biggest correction needed in your answer.

You described it as:

Dockerfile 1 → image 1 → Dockerfile 2 → image 2 → Dockerfile 3...
That's not how multi-stage builds normally work.

A multi-stage build uses multiple FROM statements in the same Dockerfile.

Example
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-21 AS builder

WORKDIR /app

COPY pom.xml .
COPY src ./src

RUN mvn clean package
Then:

# Stage 2: Runtime
FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=builder /app/target/app.jar app.jar

ENTRYPOINT ["java", "-jar", "app.jar"]
Architecture:

            Dockerfile
                │
     ┌──────────┴──────────┐
     │                     │
Stage 1: Builder      Stage 2: Runtime
     │                     │
  Maven/JDK             JRE only
     │                     │
  Build JAR                │
     │                     │
     └──── app.jar ───────►│
                           │
                     Final image
The final image contains:

JRE
application.jar
It doesn't contain:

Maven
source code
build cache
compiler
That's why the final image can be much smaller.

Interview answer
"A multi-stage Docker build uses multiple FROM stages in a single Dockerfile. I use one stage to compile and build the application, then copy only the required artifact into a minimal runtime image using COPY --from. This keeps build tools and source code out of the production image and significantly reduces the attack surface and image size."

Question 16. How do you troubleshoot a restarting container?
Answer:

Troubleshooting a restarting Docker container
Your answer is correct but too short for a senior interview.

I'd follow:

Container restarting
       ↓
docker ps
       ↓
docker logs
       ↓
docker inspect
       ↓
Check exit code
       ↓
Check configuration/env/volumes/network
       ↓
Run interactively if necessary
Step 1 — Check status
docker ps -a
Look for:

Restarting
Exited (1)
Exited (137)
Exited (139)
Step 2 — Logs
docker logs <container>
For recent logs:

docker logs --tail 100 <container>
Step 3 — Inspect
docker inspect <container>
Look at:

ExitCode
OOMKilled
RestartCount
State
Environment
Mounts
For example:

ExitCode: 137
OOMKilled: true
would strongly indicate the container was killed due to memory pressure.

Step 4 — Check the image command
Sometimes the application simply starts and immediately exits.

docker inspect <container>
Check:

Entrypoint
Cmd
Step 5 — Reproduce interactively
docker run -it --entrypoint /bin/sh myimage
Then investigate inside the container.

Interview answer
"First I check docker ps -a and docker logs to understand why the container exits. Then I use docker inspect to check the exit code, OOMKilled status, restart count, environment, mounts and entrypoint. If necessary, I run the image interactively and reproduce the startup command. Common causes include application crashes, incorrect CMD/ENTRYPOINT, missing environment variables or secrets, permission issues, missing files, dependency connectivity and OOM kills."

Question 17. How do you scan Docker images for vulnerabilities?
Answer:

A common tool is Trivy.

trivy image myapp:1.0
You might get:

Library       Vulnerability     Severity
openssl       CVE-XXXX          HIGH
curl          CVE-YYYY          MEDIUM
You can make the CI pipeline fail for serious vulnerabilities:

trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  myapp:1.0
Pipeline:

Build image
     ↓
Trivy scan
     ↓
HIGH/CRITICAL?
   ↙       ↘
 YES       NO
  ↓         ↓
FAIL      Push image
            ↓
          Deploy
You can also scan before building:

trivy config .
for IaC/Kubernetes/Terraform configuration.

Interview answer
"I use Trivy to scan Docker images for known CVEs. In CI, I configure the pipeline to fail when vulnerabilities above an agreed severity threshold, such as HIGH or CRITICAL, are detected. I also scan the Dockerfile or IaC configuration separately and keep the base images patched."

Question 18. What is KEDA and when would you use it?
Answer:

KEDA (Kubernetes Event-driven Autoscaling) is a Kubernetes autoscaling component that scales workloads based on external events or event-source metrics, rather than only CPU or memory.

Simple example
Suppose you have a payment-processing worker consuming messages from Kafka:

Kafka
  │
  │ 1000 messages waiting
  ▼
KEDA
  │
  │ scales based on queue lag
  ▼
Kubernetes Deployment
  │
  ├── Pod
  ├── Pod
  ├── Pod
  ├── ...
  └── Pod
If the Kafka lag increases:

Kafka lag = 10
      ↓
2 Pods
Kafka lag = 1,000
      ↓
10 Pods
Kafka lag = 0
      ↓
0 Pods
That's where KEDA is very useful.

Example KEDA configuration
Conceptually:

apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-worker
spec:
  scaleTargetRef:
    name: order-worker

  minReplicaCount: 0
  maxReplicaCount: 20

  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.ap-south-1.amazonaws.com/123/orders
        queueLength: "50"
The important pieces are:

scaleTargetRef
      ↓
Which Deployment to scale
minReplicaCount
      ↓
Minimum Pods
maxReplicaCount
      ↓
Maximum Pods
trigger
      ↓
What event/metric controls scaling
Question 19. How do variables, locals and outputs differ?
Answer:

In Terraform, the easiest way to remember them is:

variable  → Input
local     → Internal calculation/reuse
output    → Result exposed to outside
1. Variables — Input to a module
Variables allow you to pass values into your Terraform configuration.

variable "instance_type" {
  type    = string
  default = "t3.medium"
}
resource "aws_instance" "app" {
  instance_type = var.instance_type
}
You can provide a different value:

instance_type = "t3.large"
Think:

terraform.tfvars → variable → resource
2. Locals — Internal reusable values
locals are used to avoid repeating expressions or to create calculated values.

locals {
  environment = "prod"
  name        = "app-${local.environment}"
}
Then:

resource "aws_instance" "app" {
  tags = {
    Name        = local.name
    Environment = local.environment
  }
}
Unlike variables, locals aren't intended to be supplied by the user.

3. Outputs — Values exposed after Terraform creates resources
Outputs let you expose useful values from your Terraform configuration or module.

output "instance_ip" {
  value = aws_instance.app.public_ip
}
After:

terraform apply
Terraform displays:

instance_ip = "13.x.x.x"
Outputs are especially important when one Terraform module needs to expose information to another module.

For example:

VPC module
   │
   └── output: vpc_id
            ↓
EKS module
   │
   └── uses var.vpc_id
Question 20. What is the difference between count and for_each?
Answer:

count is index based , for_each uses key,value or maping based

resource "aws_instance" "test"{
  for_each=var.name
  name=each.value
}
resource "aws_instance" "test"{
  count=len(var.name)
  name=var.name[count.index]
}
Question 21. iam user, group and roles
Answer:

You're close, but there's one important correction: IAM roles are not only for AWS services. Roles are identities that can be assumed by AWS services, users, workloads, or accounts.

Also, I think you mean IAM user vs group vs role, rather than "access group."

1. IAM User
An IAM user represents a specific identity, typically a human or application that needs long-term AWS credentials.

Example
Haribabu
   ↓
IAM User
   ↓
Permissions
   ↓
S3 / EC2 / RDS
You can attach policies directly to the user, although for human access, groups/roles are generally preferable.

2. IAM Group
A group is a collection of IAM users.

For example:

DevOps Group
 ├── User A
 ├── User B
 └── User C
        ↓
  Group Policy
        ↓
EC2 + EKS + S3 permissions
Instead of giving the same permissions individually to 20 users, you attach the policy to the group.

Example
Developers
   → Developer permissions
Operations
   → Operations permissions
ReadOnly
   → Read-only permissions
Important: Groups contain users; groups don't contain roles.

3. IAM Role
A role is an AWS identity that doesn't have permanent username/password credentials.

Something can assume the role and receive temporary credentials.

For example, an EC2 instance needs to access S3:

EC2
 │
 │ AssumeRole
 ▼
IAM Role
 │
 ▼
S3 permissions
The application doesn't need an AWS access key stored inside the EC2 instance.

Similarly with EKS:

Kubernetes Pod
      ↓
IRSA / Pod Identity
      ↓
IAM Role
      ↓
S3 / DynamoDB / Secrets Manager
And AWS services can assume roles too:

Lambda
   ↓
IAM Role
   ↓
S3 / DynamoDB / CloudWatch
Roles are also heavily used for cross-account access:

AWS Account A
   User
     ↓
AssumeRole
     ↓
AWS Account B
   IAM Role
     ↓
Resources
The easiest way to remember
IAM concept	Think of it as
User	A specific identity
Group	Collection of users
Role	Identity that can be assumed
Policy	Rules defining permissions
And:

USER
  │
  └── belongs to → GROUP
                       │
                       └── policies → permissions
EC2 / Lambda / Pod / User
            │
            └── Assume → ROLE
                           │
                           └── policies → permissions
Question 22. Security Group vs NACL? Public vs Private Subnet? NAT Gateway vs Internet Gateway?
Answer:

1. Security Group vs NACL
Your core understanding is correct.

Security Group
A Security Group is a stateful virtual firewall associated with resources such as EC2 network interfaces.

Example
Internet
   │
   ▼
Security Group
   │
   ▼
EC2
Suppose you allow:

Inbound TCP 443
Source: 0.0.0.0/0
When the client sends HTTPS traffic to EC2, the return traffic is automatically allowed because the Security Group is stateful.

You don't need a separate outbound rule specifically for that response, assuming the outbound rules permit the traffic generally.

NACL
A Network ACL operates at the subnet level.

VPC
 │
 ├── Public Subnet
 │      └── NACL
 │           └── EC2
 │
 └── Private Subnet
        └── NACL
             └── EC2
NACLs are stateless.

Therefore, if you allow inbound traffic, you must also explicitly allow the corresponding outbound/return traffic.

For example:

Client
  │
  │ TCP 443
  ▼
NACL
  │
  ▼
EC2
  │
  │ Return traffic
  ▼
NACL
  │
  ▼
Client
Both directions need appropriate rules.

Important correction
You said:

"NACL doesn't allow outbound traffic to reach that client."

Not necessarily. You have to explicitly configure the outbound rule that permits the return traffic.

Interview answer
"Security Groups are stateful firewalls associated with resources such as EC2 network interfaces, while NACLs are stateless firewalls applied at the subnet level. With a Security Group, return traffic is automatically allowed for an established connection. With a NACL, inbound and outbound traffic are evaluated independently, so both directions must be explicitly allowed."

2. Public vs Private Subnet
Your idea is correct, but there's an important definition.

A subnet is considered public when its route table has a route to an Internet Gateway.

Example
Public Subnet
      │
      ▼
Route Table
      │
0.0.0.0/0 → Internet Gateway
      │
      ▼
Internet
A private subnet does not have a direct route to an Internet Gateway.

But this doesn't necessarily mean it has no Internet access.

A private subnet can access the Internet through a NAT Gateway:

Private Subnet
      │
      ▼
NAT Gateway
      │
      ▼
Internet Gateway
      │
      ▼
Internet
The important difference is:

Public subnet
→ resources can have direct inbound/outbound Internet connectivity
Private subnet
→ no direct inbound Internet connectivity
→ can have outbound Internet access through NAT Gateway
3. Internet Gateway vs NAT Gateway
This is where your answer needs more correction.

Internet Gateway
An Internet Gateway provides the VPC's connectivity to the public Internet.

Internet
   │
   ▼
Internet Gateway
   │
   ▼
VPC
A public subnet's route table might contain:

0.0.0.0/0 → igw-xxxx
A resource also generally needs a public IPv4 address (or appropriate public addressing) for direct IPv4 Internet communication.

NAT Gateway
NAT Gateway allows resources in private subnets to initiate outbound Internet connections without exposing those resources to unsolicited inbound Internet connections.

Architecture:

    Internet
       │
       ▼
Internet Gateway
       │
       ▼
 NAT Gateway
 Public Subnet
       │
       ▼
Private Subnet
       │
       ▼
     EC2
The NAT Gateway itself should be placed in a public subnet and typically uses an Elastic IP for public IPv4 connectivity.

The private subnet route table points to the NAT Gateway:

0.0.0.0/0 → nat-xxxx
The complete picture
                   INTERNET
                       │
                       ▼
              ┌────────────────┐
              │ Internet GW    │
              └───────┬────────┘
                      │
                   VPC
       ┌──────────────┴──────────────┐
       │                             │
       ▼                             ▼
PUBLIC SUBNET                 PRIVATE SUBNET
       │                             │
  ┌────┴────┐                        │
  │         │                        │
 ALB    NAT Gateway ◄───────────────┘
  │         │
  ▼         │
 EC2        │
            ▼
         INTERNET
Interview-ready answer
"A Security Group is a stateful firewall associated with resources such as EC2 network interfaces, whereas a NACL is a stateless firewall applied at the subnet level. A public subnet has a route to an Internet Gateway, while a private subnet doesn't have direct Internet Gateway access. A NAT Gateway is deployed in a public subnet and allows resources in private subnets to initiate outbound Internet connections without making those private resources directly reachable from the Internet."

And remember this very useful interview distinction:

Internet Gateway = VPC ↔ Internet connectivity
NAT Gateway = Private subnet → Internet outbound connectivity
Question 23. How would a Kubernetes Pod securely access an S3 bucket?
Answer:

EKS Pod → ServiceAccount → IAM Role → S3
                         AWS ACCOUNT
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│                         ┌─────────────┐                       │
│                         │  S3 Bucket  │                       │
│                         │             │                       │
│                         │ app/*       │                       │
│                         └──────▲──────┘                       │
│                                │                              │
│                         S3 API request                        │
│                                │                              │
│                         ┌──────┴──────┐                       │
│                         │  IAM Role   │                       │
│                         │             │                       │
│                         │ s3:GetObject│                       │
│                         │ s3:PutObject│                       │
│                         └──────▲──────┘                       │
│                                │                              │
│                   IAM Trust / Pod Identity                    │
│                                │                              │
│ ┌──────────────────────────────┴────────────────────────────┐ │
│ │                         EKS Cluster                        │ │
│ │                                                           │ │
│ │   Namespace: payments                                     │ │
│ │                                                           │ │
│ │   ┌─────────────────────┐                                 │ │
│ │   │ ServiceAccount      │                                 │ │
│ │   │ payment-sa          │                                 │ │
│ │   └──────────┬──────────┘                                 │ │
│ │              │                                             │ │
│ │              │ serviceAccountName: payment-sa              │ │
│ │              ▼                                             │ │
│ │   ┌─────────────────────┐                                 │ │
│ │   │       Pod           │                                 │ │
│ │   │                     │                                 │ │
│ │   │  Payment App        │                                 │ │
│ │   │       │             │                                 │ │
│ │   │       └── AWS SDK ──┼──────────────► S3               │ │
│ │   └─────────────────────┘                                 │ │
│ │                                                           │ │
│ └───────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
How we set it up
There are 3 main pieces:

1. Create IAM policy
Give only the required S3 permissions:

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-app-bucket/app/*"
    }
  ]
}
Notice we're not giving:

s3:*

This is least privilege.

2. Create IAM Role
The IAM role has two important parts:

IAM Role
   │
   ├── Permission Policy
   │       └── S3 Get/Put
   │
   └── Trust Policy
           └── Allows EKS workload to assume role
With IRSA, the trust relationship allows the specific Kubernetes ServiceAccount to assume the role.

Conceptually:

payments namespace
       │
       ▼
payment-sa
       │
       ▼
IAM Role
       │
       ▼
S3 permissions
3. Create Kubernetes ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-sa
  namespace: payments
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/payment-s3-role
Apply it:

kubectl apply -f serviceaccount.yaml
Check:

kubectl get serviceaccount payment-sa -n payments
4. Tell the Pod to use that ServiceAccount
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment
  namespace: payments

spec:
  replicas: 2

  selector:
    matchLabels:
      app: payment

  template:
    metadata:
      labels:
        app: payment

    spec:
      serviceAccountName: payment-sa

      containers:
        - name: payment
          image: myregistry/payment:1.0
The critical line is:

serviceAccountName: payment-sa
This connects:

Pod
 ↓
payment-sa
 ↓
IAM Role
 ↓
S3
What happens at runtime?
The application doesn't contain AWS credentials.

Instead:

1. Pod starts
       ↓
2. Pod uses payment-sa
       ↓
3. EKS identity mechanism associates the IAM role
       ↓
4. AWS SDK obtains temporary credentials
       ↓
5. Application calls S3
       ↓
6. IAM evaluates the role policy
       ↓
7. S3 allows/denies the request
For example:

Payment Pod
     │
     │ s3.PutObject()
     ▼
Temporary AWS credentials
     │
     ▼
payment-s3-role
     │
     ├── s3:PutObject ✓
     ├── s3:GetObject ✓
     ├── s3:DeleteObject ✗
     └── ec2:* ✗
So even if the application only needs to upload files, it doesn't automatically get permission to delete objects or manage EC2.

Question 24
Answer:
