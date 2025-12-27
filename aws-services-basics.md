# AWS in Plain English — 16 Core Services You'll Actually Use (Expanded)

This guide explains 16 essential AWS services in plain English with production-oriented examples, GUI (console) walkthroughs, AWS CLI examples, practical tips, limits and pricing considerations, best-practice interview talking points, and small hands-on lab tasks you can run to demonstrate understanding.

---

## 1 — Amazon EC2 (Elastic Compute Cloud)

What it is (plain English)
- EC2 is a virtual server you run in AWS. You pick CPU, RAM, disk and OS; AWS supplies the physical host and network.
- Think: "A rented computer in the cloud that I can configure like a physical server."

When to use
- Any app that needs control over runtime OS (custom kernels, stateful services).
- Long-running services, batch workers, containers (if not using managed container services), bastion/jump hosts.

Console walkthrough (quick)
1. EC2 Console → Launch Instance.
2. Choose AMI (e.g., Amazon Linux 2023).
3. Select instance type (t3.micro for small tests).
4. Configure network/subnet, add storage.
5. Create/choose key pair, configure security group (allow SSH/HTTP).
6. Launch, note Public IP, SSH in.

AWS CLI examples
- List instances:
```bash
aws ec2 describe-instances
```
- Launch one instance (example):
```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --key-name mykey \
  --security-group-ids sg-0123456789abcdef \
  --subnet-id subnet-0123456789abcdef \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-server-01}]'
```

Tips, limits & cost considerations
- Sizes matter: CPU, memory and network bursting differ by family. Use right-sized instances.
- Stop vs terminate: stopping preserves EBS (if not ephemeral); terminate deletes if set.
- EBS volumes billed separately; consider instance savings plans or reserved instances for steady workloads.
- Use placement groups for low-latency clustering; watch AZ capacity.

Interview points / best practices
- Explain when to use EC2 vs Lambda/ECS/EKS.
- Describe EBS volume types (gp3, io2) and snapshot lifecycle.
- Security: IAM roles for instance profiles, avoid embedding secrets.

Quick lab
- Launch a t3.micro, install Nginx, upload a simple HTML page, and access via public IP.

---

## 2 — Amazon S3 (Simple Storage Service)

What it is
- Object store for files (images, backups, logs). Highly durable and scalable.
- Think: "Infinite file storage with versioning, lifecycle rules and access controls."

When to use
- Static websites, backups, data lakes, storing artifacts, hosting CDN origin.

Console walkthrough
1. S3 Console → Create bucket (globally unique name).
2. Choose region, configure public access or block public access.
3. Enable versioning and lifecycle rules as needed.
4. Upload objects or use CLI/SDK.

AWS CLI examples
- Create bucket:
```bash
aws s3 mb s3://my-bucket-name --region us-east-1
```
- Upload file:
```bash
aws s3 cp index.html s3://my-bucket-name/
```
- Sync folder:
```bash
aws s3 sync ./website s3://my-bucket-name/
```

Tips, limits & cost considerations
- Pricing: storage (per GB), requests (GET/PUT), data egress. Use lifecycle rules to move to IA/Glacier for lower cost.
- Consistency: read-after-write PUTS for new objects; eventual consistency for overwrite/delete historically — modern S3 is strongly consistent for all operations.
- Security: bucket policies, ACLs (avoid ACLs unless required), IAM conditions, encryption (SSE-S3, SSE-KMS).

Interview points
- Explain versioning, lifecycle transitions, S3 storage classes (STANDARD, INTELLIGENT_TIERING, IA, GLACIER).
- How to secure a bucket for static website hosting (CloudFront + OAI/Origin Access Control + block public bucket access).

Quick lab
- Host a static site: upload HTML, enable static website hosting, optionally set up CloudFront.

---

## 3 — Amazon RDS (Relational Database Service)

What it is
- Managed relational databases (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Aurora) where AWS handles backups, patching and basic HA.

When to use
- Traditional relational apps: transactions, joins, strong consistency.

Console walkthrough
1. RDS Console → Create database.
2. Choose engine and version.
3. Choose DB instance class, storage type (gp3).
4. Enable Multi-AZ for production; automated backups, retention.
5. Configure security group and subnet group.

AWS CLI example
```bash
aws rds create-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --allocated-storage 20 \
  --master-username admin \
  --master-user-password 'MySecurePass123!' \
  --backup-retention-period 7
```

Tips, limits & cost
- RDS manages only the database engine; you cannot access the underlying OS (except for some managed services).
- For high throughput and read scaling, use read replicas or Aurora.
- Backups and snapshots incur storage cost; Multi-AZ doubles standby replicas (higher cost but fast failover).
- Use parameter groups and option groups for fine tuning.

Interview points
- Differences between RDS and Aurora.
- When to use read replicas vs Multi-AZ vs sharding.
- Explain automated backups, point-in-time recovery, and encryption at rest (KMS).

Quick lab
- Create a small PostgreSQL RDS, connect using psql from an EC2 in the same VPC.

---

## 4 — Amazon DynamoDB

What it is
- Fully managed NoSQL key-value & document database that scales with low latency.

When to use
- Use for high-scale, low-latency workloads (gaming leaderboards, session stores, shopping carts).

Console walkthrough
1. DynamoDB Console → Create table.
2. Provide table name and primary key (partition key, optional sort key).
3. Choose capacity mode: On-Demand (simpler) or Provisioned (more cost control).

AWS CLI example
```bash
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions AttributeName=OrderId,AttributeType=S \
  --key-schema AttributeName=OrderId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

Tips, limits & cost
- Design access patterns first; DynamoDB is optimized for known key-based queries.
- Use GSIs/LSIs for alternate query patterns, but they add cost and throughput usage.
- Consider DynamoDB Accelerator (DAX) or DAX alternatives if you need microsecond reads.
- Provisioned mode allows autoscaling policies.

Interview points
- Explain partitioning and hot-partition causes.
- Tradeoffs of eventual vs strong consistency, and how to design keys to avoid hot partitions.

Quick lab
- Create a table, write/read items with aws cli or SDK, experiment with queries using partition + sort key.

---

## 5 — Amazon VPC (Virtual Private Cloud)

What it is
- A virtual network you control in AWS: CIDR ranges, subnets, route tables, NAT gateways, security groups, NACLs.

When to use
- Always — every resource lives in a VPC. Use to isolate environments and control network access.

Console walkthrough
1. VPC Console → Create VPC (e.g., 10.0.0.0/16).
2. Create public & private subnets across AZs.
3. Create and attach Internet Gateway, route tables, and NAT Gateway for private subnets.

AWS CLI example
```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16
```

Tips, limits & cost
- Subnet design: put public-facing tiers in public subnets; databases in private subnets.
- Elastic IP + NAT Gateway charges matter for egress from private subnets.
- Security groups (stateful) vs Network ACLs (stateless) — use both appropriately.
- Use VPC Flow Logs and AWS Network Firewall for monitoring and control.

Interview points
- Explain differences between security groups and NACLs.
- Describe VPC peering vs Transit Gateway vs AWS PrivateLink for cross-account connectivity.

Quick lab
- Create a VPC with a public subnet and launch an EC2 with a public IP; connect via SSH.

---

## 6 — IAM (Identity and Access Management)

What it is
- IAM controls authentication and authorization: users, groups, roles, policies, identity providers.

When to use
- Always: granular permissioning for people and AWS services.

Console walkthrough
1. IAM Console → Create user or role.
2. Attach managed or inline policies; follow least privilege.
3. For EC2/ECS/Lambda, assign roles (instance profile) rather than embedding keys.

AWS CLI examples
- Create user:
```bash
aws iam create-user --user-name devops-admin
```
- Attach policy:
```bash
aws iam attach-user-policy --user-name devops-admin --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

Tips, limits & secure practices
- Use roles and temporary credentials (STS) rather than long-lived keys.
- Enable MFA for all privileged accounts; consider AWS SSO for centralized access.
- Policy size limits and max managed policies per entity exist — use least-privilege and policy delegation.

Interview points
- Show how IAM policies are evaluated (explicit deny > allow).
- Explain cross-account access using roles and trust policies.

Quick lab
- Create a role for Lambda that allows S3 GetObject and attach to a test function.

---

## 7 — AWS Lambda

What it is
- Serverless functions: run code (short-lived) without managing servers; billed per runtime and memory.

When to use
- Event-driven workloads: processing files, API backends, CRON jobs, lightweight data transformations.

Console walkthrough
1. Lambda Console → Create function (choose runtime).
2. Attach execution role with least privilege.
3. Add triggers: S3 event, API Gateway, CloudWatch Events.

AWS CLI examples
- List functions:
```bash
aws lambda list-functions
```
- Publish simple function (zip and create):
```bash
aws lambda create-function \
  --function-name hello \
  --runtime python3.11 \
  --handler lambda_function.handler \
  --role arn:aws:iam::123456789012:role/lambda-exec-role \
  --zip-file fileb://function.zip
```

Tips, limits & cost
- Cold starts: languages like Java/.NET have longer cold start times; keep payload and dependencies small for low latency.
- Max execution time = 15 minutes; not suitable for very long jobs.
- Choose memory size to balance cost and CPU (more memory = more CPU).
- Use Provisioned Concurrency for latency-sensitive services.

Interview points
- Explain event sources, cold starts, and how to handle large dependencies.
- Use of Layers and Amazon EFS for large assets.

Quick lab
- Create a Lambda triggered by S3 PutObject that logs incoming keys.

---

## 8 — Amazon API Gateway

What it is
- Managed API front door: route HTTP/WebSocket requests to Lambda, EC2, or other services. Handles authorization, throttling and request transformation.

When to use
- Expose REST or WebSocket APIs, enforce throttling and authentication with Cognito or IAM.

Console walkthrough
1. API Gateway Console → Create API (HTTP or REST).
2. Define resources/methods → Integrate with Lambda or HTTP backend.
3. Configure CORS, usage plans, and authorizers.

AWS CLI example
- Create an API (HTTP):
```bash
aws apigatewayv2 create-api --name MyApi --protocol-type HTTP --target arn:aws:lambda:...
```

Tips & cost
- HTTP API is cheaper and sufficient for many use cases; REST API has more features (stages, API keys).
- Use caching and throttling to protect backends.
- Secure with JWT authorizers (Cognito or custom) or Lambda authorizers.

Interview points
- Differences between REST API and HTTP API (feature & pricing tradeoffs).
- How to protect APIs (WAF, authorizers, rate limits).

Quick lab
- Create HTTP API backed by a Lambda that returns "Hello".

---

## 9 — Amazon CloudFront

What it is
- Global Content Delivery Network (CDN) for caching and accelerating content from S3, EC2 or other origins.

When to use
- Static site acceleration, API edge caching, media streaming and regional failover.

Console walkthrough
1. CloudFront Console → Create distribution.
2. Set origin (S3 bucket or load balancer), configure cache behavior and SSL.
3. Optionally configure origin access control for private S3 origins.

AWS CLI example
- Create distribution via CloudFront API (more complex, usually done with IaC).

Tips & cost
- Cache behavior (TTL, query string handling) controls cache hit ratio.
- Use Lambda@Edge or CloudFront Functions for edge logic.
- Edge locations reduce latency but may increase request-based costs.

Interview points
- Explain TTLs, cache invalidation strategies, and signed URLs.
- When to use CloudFront Functions vs Lambda@Edge.

Quick lab
- Put static website in S3, create CloudFront distribution, access via CloudFront domain.

---

## 10 — Amazon SQS (Simple Queue Service)

What it is
- Managed message queue for decoupling services (Standard and FIFO queues).

When to use
- Decouple producers/consumers, buffer spikes, retry/backoff patterns.

Console walkthrough
1. SQS Console → Create queue (Standard or FIFO).
2. Configure visibility timeout, retention period, delivery delay.

AWS CLI example
- Create queue:
```bash
aws sqs create-queue --queue-name my-queue
```

Tips & cost
- Standard queues: at-least-once delivery, best-effort ordering. FIFO: exactly-once processing, strict ordering (higher cost).
- Visibility timeout must cover message processing time; use dead-letter queues for poison messages.

Interview points
- Describe visibility timeout, long polling vs short polling, DLQs and exponential backoff.

Quick lab
- Send messages to an SQS queue and process them with a simple worker running on EC2 or Lambda.

---

## 11 — Amazon SNS (Simple Notification Service)

What it is
- Pub/Sub notification service: push notifications (SMS, email), fan-out to SQS/Lambda/HTTP.

When to use
- Broadcast messages, trigger multiple subscribers, alerts, mobile push.

Console walkthrough
1. SNS Console → Create topic.
2. Subscribe endpoints (email, SMS, HTTP, SQS, Lambda).
3. Publish messages.

AWS CLI example
- Create topic and subscribe:
```bash
aws sns create-topic --name my-topic
aws sns subscribe --topic-arn arn:aws:sns:... --protocol email --notification-endpoint you@example.com
```

Tips & cost
- SMS and email have additional costs/quotas.
- Use SNS + SQS for reliable delivery and retries.

Interview points
- SNS vs SQS: push vs pull.
- Message filtering and retry behavior.

Quick lab
- Create an SNS topic and subscribe an email; publish a test message.

---

## 12 — Amazon CloudWatch

What it is
- Observability platform: metrics, logs, traces (CloudWatch Logs, Metrics, Alarms, Dashboards, RUM, Synthetics).

When to use
- Monitor app health, set alarms, collect logs and create dashboards.

Console walkthrough
1. CloudWatch Console → Create log groups, configure metrics/alarms.
2. Create dashboard and add widgets, set alarms to notify via SNS.

AWS CLI examples
- Put custom metric:
```bash
aws cloudwatch put-metric-data --namespace MyApp --metric-name RequestCount --value 1
```
- Create alarm:
```bash
aws cloudwatch put-metric-alarm --alarm-name HighCPU --metric-name CPUUtilization --namespace AWS/EC2 --statistic Average --period 300 --threshold 80 --comparison-operator GreaterThanThreshold --evaluation-periods 2 --alarm-actions arn:aws:sns:...
```

Tips & cost
- Log ingestion and retention cost are factors; set good retention/lifecycle rules.
- Use embedded X-Ray for distributed tracing when needed.

Interview points
- Explain metrics, custom metrics vs default service metrics.
- Use dashboards for capacity planning and alarms for incident response.

Quick lab
- Install CloudWatch agent on EC2, push custom metrics, create an alarm.

---

## 13 — AWS CloudTrail

What it is
- Audit log of API calls (who did what, when) across your account.

When to use
- Security auditing, compliance, incident investigation.

Console walkthrough
1. CloudTrail Console → Create trail, deliver logs to S3, enable event history and CloudWatch integration.

AWS CLI example
- Create trail:
```bash
aws cloudtrail create-trail --name my-trail --s3-bucket-name my-cloudtrail-bucket
aws cloudtrail start-logging --name my-trail
```

Tips & cost
- Store logs in a dedicated S3 bucket with proper lifecycle rules and encryption.
- Use CloudWatch Logs for near real-time alerting, aggregate multi-account trails with organization trails.

Interview points
- Difference between CloudTrail and CloudWatch Logs/Metrics.
- Trail management best practices and protecting logs (immutable storage).

Quick lab
- Enable CloudTrail and generate a test event (create an S3 bucket), then find it in the trail.

---

## 14 — AWS CloudFormation

What it is
- Infrastructure as Code (IaC) service for provisioning AWS resources via templates (YAML/JSON).

When to use
- Repeatable infra provisioning, drift detection, CI/CD for infra.

Console walkthrough
1. CloudFormation Console → Create stack from template (local or S3).
2. Follow parameters and watch stack events.

CLI examples
- Validate template:
```bash
aws cloudformation validate-template --template-body file://template.yaml
```
- Create stack:
```bash
aws cloudformation create-stack --stack-name my-stack --template-body file://template.yaml --capabilities CAPABILITY_IAM
```

Tips & cost
- StackSets for multi-account/region deployments.
- Use Change Sets to preview changes. Protect against accidental deletes (termination protection).

Interview points
- Explain drift detection and change sets, stack dependencies and nested stacks.
- Compare CloudFormation to Terraform (state management, provider portability).

Quick lab
- Deploy a small CloudFormation stack that creates an S3 bucket + IAM role.

---

## 15 — Amazon EKS (Elastic Kubernetes Service)

What it is
- Managed Kubernetes control plane. AWS manages the master nodes; you manage worker nodes (or use Fargate).

When to use
- Run containerized microservices with Kubernetes orchestration, bring-your-own Kubernetes apps.

Console walkthrough
1. EKS Console → Create cluster (control plane).
2. Create node group (Managed Node Group) or configure Fargate profiles.
3. Use kubectl to manage workloads.

AWS CLI / eksctl examples
- Create cluster with eksctl (simpler):
```bash
eksctl create cluster --name demo-cluster --region us-west-2 --nodegroup-name standard-workers --node-type t3.medium --nodes 2
```

Tips & cost
- EKS control plane has a fixed hourly charge; node costs vary.
- Use IAM Roles for Service Accounts (IRSA) to give fine-grained permissions to pods.

Interview points
- How EKS differs from ECS and Fargate.
- Networking in Kubernetes: CNI (VPC CNI), service meshes, and load balancing.

Quick lab
- Create an EKS cluster with one node and deploy a sample Nginx deployment and service.

---

## 16 — AWS Step Functions

What it is
- Serverless workflow orchestration: coordinate Lambda, ECS, and other services using state machines.

When to use
- Complex business processes, long-running workflows, retries/parallel steps, human approval tasks.

Console walkthrough
1. Step Functions Console → Create state machine using visual workflow or ASL (Amazon States Language).
2. Define steps with retries, catches and parallel flows.

AWS CLI example
- Start execution:
```bash
aws stepfunctions start-execution --state-machine-arn arn:aws:states:... --input '{"orderId":"123"}'
```

Tips & cost
- Visual debugging helps reduce logic errors. Step Functions billed per state transition.
- Use Express Workflows for high-volume, short-duration workflows and Standard for long-running durable workflows.

Interview points
- Explain retry/Catch strategies and task tokens for waiting human approvals.
- Differences between Express and Standard Workflows.

Quick lab
- Create a simple Step Function that calls two Lambda functions in sequence.

---

# Cross-cutting Best Practices (Short)

- Security first: least privilege IAM, encrypt data at rest and in transit, enable MFA, monitor with CloudTrail and GuardDuty.
- Cost control: tags, budgets, Cost Explorer, and right-sizing.
- Observability: collect metrics and logs, set alerts (CloudWatch), and use distributed tracing (X-Ray).
- Resilience: multi-AZ for RDS/EC2, S3 versioning + lifecycle, backup & restore playbooks.
- Automation & IaC: treat infrastructure like code (CloudFormation/Terraform), CI/CD pipelines for deployments.

---

# Interview Prep Cheat‑Sheet (How to present knowledge)

- Start with the problem: e.g., "We need a low-latency global website" → propose S3 static site + CloudFront.
- Explain tradeoffs: cost vs complexity vs operational overhead (managed services reduce ops but can cost more).
- Be ready to discuss: security model, scaling and availability, monitoring, typical failure modes and mitigation.
- Use concrete examples: "We reduced EC2 costs by 40% using Auto Scaling + Spot for batch workers."
- Practice whiteboard scenarios: design a 3-tier web app with autoscaling, DB high availability, and secure connectivity.

---

# Suggested Hands‑On Learning Path (Beginner → Advanced)

1. Basics: Launch EC2, create S3 bucket, upload files, and configure IAM user.
2. Serverless: Build Lambda triggered by S3 and expose via API Gateway.
3. Databases: Create RDS and DynamoDB tables; connect from an EC2.
4. Observability: Send logs to CloudWatch, create alarms and dashboards.
5. Networking: Build VPC with public/private subnets, NAT, and route tables.
6. IaC & Automation: Convert infra to CloudFormation/Terraform and deploy with a CI pipeline.
7. Advanced: EKS cluster, multi-account security, cross-region DR.

---

