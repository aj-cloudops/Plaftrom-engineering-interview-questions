AWS ECS
50 Real-World Scenario-Based Interview Questions & Answers
Covering: Incident Response  |  Architecture  |  Security  |  Cost Optimization  |  CI/CD  |  Scaling  |  Networking  |  Multi-Region  |  Advanced Operations

🚨 Incident & Troubleshooting Scenarios

Scenario 1
Your ECS service suddenly shows 0 running tasks in production during peak hours. CloudWatch alarms are firing. The on-call alert just woke you up. What do you do?
Answer
Immediate triage (first 2 minutes):
1. Check ECS Service Events tab — look for 'service was unable to place tasks' or 'task failed to start' messages.
2. Check recently stopped tasks — inspect stoppedReason and container exit codes. Exit code 137 = OOM kill. Exit code 1 = app crash.
3. Check CloudWatch Logs for the last tasks that ran — look for errors right before shutdown.
 
Common root causes and fixes:
- OOM Kill (exit 137): Increase task memory. Rollback task definition to previous revision immediately.
- Image pull failure: Check ECR permissions, verify the image tag still exists (was it deleted?), verify VPC endpoint for ECR is healthy.
- Health check failure: Check if the app port is listening. Tune startPeriod in the health check.
- Capacity issues (EC2): Is the cluster out of CPU/memory? Scale up the Auto Scaling Group manually.
- Bad deployment: If this followed a release, immediately update the service to the previous task definition revision.
 
Recovery: If a bad deployment caused it, run: aws ecs update-service --cluster prod --service my-svc --task-definition my-task:PREVIOUS_REVISION
Post-incident: Enable deployment circuit breakers with rollback so future bad deployments auto-revert.
 
Scenario 2
An ECS task is running but the application inside is unresponsive. The ALB health checks are passing, but users are getting timeouts. How do you investigate?
Answer
This is a 'lying healthy' scenario — the health check endpoint is shallow while the app is actually broken.
 
Investigation steps:
1. Use ECS Exec to get into the running container: aws ecs execute-command --cluster prod --task <task-id> --container app --interactive --command '/bin/sh'
2. Inside the container, check if the app process is running (ps aux), check app-level logs, and try curling the service locally (curl localhost:8080/api/health).
3. Check if the health check endpoint is actually testing real app functionality or just returning 200 blindly.
4. Look at CloudWatch Container Insights — is CPU pegged at 100%? Is the app deadlocked?
5. Check downstream dependencies — if the app depends on a DB or cache, is that reachable? (curl the DB endpoint from inside the container).
 
Fix: Improve the health check to be a 'deep health check' that validates real dependencies. Consider adding a /ready endpoint that checks DB connectivity, cache connectivity, and critical service availability before returning 200.
Short term: Force a new deployment to restart the task: aws ecs update-service --cluster prod --service my-svc --force-new-deployment
 
Scenario 3
Your ECS deployment is stuck. The service shows 'deployment in progress' for 45 minutes, half the tasks are on the new version, half on the old. Nothing is moving. What happened and how do you fix it?
Answer
This is a stuck rolling deployment, typically caused by one of these:
 
Diagnosis:
1. Check the new tasks — are they healthy? Go to ECS > Service > Tasks, filter by RUNNING, check which task definition they are on.
2. Check CloudWatch Logs for the new task version — is the app crashing or hanging during startup?
3. Check ALB Target Group — are new tasks registering but failing health checks?
4. Check the deployment configuration: if minimumHealthyPercent=100 and maximumPercent=200 with desired count=2, ECS launched 2 new tasks (now at 200%) but won't stop old ones until new ones pass health checks.
 
Root cause scenarios:
- New app version has a startup bug — it launches but health check URL doesn't respond during warmup. Fix: Increase startPeriod in the container health check.
- Database migration is locking tables — new version waits for migration that never completes. Fix: Roll back.
- Resource exhaustion — no capacity to place more tasks. Check cluster CPU/memory.
 
Resolution:
- If bad code: aws ecs update-service --task-definition my-task:PREVIOUS_REVISION to roll back.
- If resource issue: Scale up the cluster or increase max percent.
- Going forward: Enable deployment circuit breakers so the deployment auto-fails after a timeout instead of getting stuck forever.
 
Scenario 4
Tasks in your ECS cluster are randomly getting OOM (Out of Memory) killed every few hours, but your memory metrics look fine in CloudWatch. What could be happening?
Answer
This is a subtle memory issue. 'Fine metrics' in CloudWatch don't tell the whole story.
 
Investigation:
1. Check container exit codes in stopped tasks — exit code 137 confirms OOM kill (SIGKILL from kernel).
2. Distinguish between ECS OOM kill vs OS-level OOM kill: ECS kills a container when it exceeds its hard memory limit (memoryReservation is a soft limit, memory is the hard limit). The OS kernel kills processes when the host runs out of RAM.
3. CloudWatch metrics show average memory — look at the maximum/p99 metrics. Spiky workloads can look fine on average but hit limits momentarily.
4. Look for memory leaks: does memory usage trend upward over time in the Container Insights metrics? If yes, the app has a memory leak.
 
Solutions:
- Increase the container hard memory limit (memory field in task definition) by 20-30% as buffer.
- If memory grows over time, it's a leak — work with devs to fix it. As a stop-gap, set a container restart policy or schedule regular task replacements.
- Use Container Insights with memory reservation vs usage metrics to right-size your tasks.
- Enable swap space as a buffer (available in ECS task definitions for Linux containers on EC2).
- For JVM apps: Explicitly set heap size flags (-Xmx) to prevent the JVM from using more memory than the container limit.
 
Scenario 5
After a routine ECR image push, your ECS service fails to pull the new image and all tasks fail with 'CannotPullContainerError'. The image is confirmed to be in ECR. What do you check?
Answer
CannotPullContainerError when the image exists usually means a permissions, network, or tag issue.
 
Checklist in order:
1. Image tag mismatch: Verify the exact tag in the task definition matches what was pushed (case-sensitive). Did someone push 'latest' but the task definition uses a specific SHA?
2. Wrong region: Is the ECR repo in the same region as the ECS cluster? Cross-region ECR pulls require explicit cross-region configuration.
3. Execution role permissions: Does the ECS Task Execution Role have ecr:GetDownloadUrlForLayer, ecr:BatchGetImage, ecr:BatchCheckLayerAvailability, and ecr:GetAuthorizationToken?
4. ECR VPC endpoint: If using PrivateLink (no NAT Gateway), check that the ECR VPC endpoint is healthy and its security group allows traffic from the ECS tasks.
5. ECR repository policy: If pulling from another account, has the cross-account repository policy been updated?
6. Image architecture: Did someone push an arm64 image but the cluster is x86_64 (or vice versa)? ECS Fargate will fail to pull incompatible architectures.
7. ECR lifecycle policy: Did a lifecycle policy delete the image tag that was just referenced?
 
Quick fix: If it's a tag issue, update the task definition to use the full image digest (@sha256:...) instead of a tag for immutable image references. Pin critical deployments to digests, not tags.
 
Scenario 6
Your ECS service is running on Fargate and suddenly tasks are being SIGTERM'd and replaced every 20 minutes. No code change was made. What's going on?
Answer
Periodic task replacement without a deployment is unusual. Key suspects:
 
Investigation:
1. Check stoppedReason on stopped tasks — is it 'Essential container exited', 'Task failed ELB health checks', or 'Fargate service maintenance'?
2. Essential container exiting: Something inside the container is crashing silently (a cron job, a background goroutine). Check all process logs, not just the main app logs.
3. ECS service health check failures: If the ALB health check is marginal (failing sometimes, passing others), ECS replaces tasks when health checks fail.
4. Fargate maintenance: AWS occasionally replaces Fargate tasks for infrastructure maintenance. This is normal and should be handled with graceful shutdown.
5. Memory pressure: If the task has a soft memory limit and a companion process is growing, the container could be getting OOM killed.
6. Scheduled task termination: Check if any Lambda, EventBridge, or automation script is calling StopTask on a schedule.
 
Solutions:
- Add SIGTERM handler in your application so it drains gracefully and logs why it is shutting down.
- Check the task's CPU and memory metrics just before termination — spikes reveal the cause.
- For health check flapping: increase the ALB health check unhealthyThreshold or tune the check interval.
- Ensure your app logs process exits at the WARNING/ERROR level.
 
⚙️ Architecture & Design Scenarios

Scenario 7
You need to migrate a monolithic application running on EC2 into ECS containers without downtime. How do you approach this migration?
Answer
A phased, risk-controlled migration strategy:
 
Phase 1 — Containerize (no traffic change):
- Write a Dockerfile for the monolith. Build and test locally.
- Push to ECR. Create an ECS Task Definition mirroring the EC2 config (env vars, IAM role, volumes).
- Run the container in a staging ECS cluster. Validate all functionality.
 
Phase 2 — Parallel run (shadow traffic):
- Create an ECS service with the containerized app in production.
- Keep the EC2 monolith running. Register both as targets in the same ALB Target Group.
- Monitor error rates, latency, and logs side-by-side.
 
Phase 3 — Traffic shifting:
- Gradually shift ALB weighted target group traffic to ECS (10% → 25% → 50% → 100%).
- Monitor after each shift. Roll back by adjusting weights if issues arise.
 
Phase 4 — Decommission:
- Once ECS handles 100% of traffic and is stable for 24-48 hours, remove EC2 from the target group.
- Terminate the EC2 instance (after a final backup).
 
Key considerations: Externalize state (sessions to Redis, uploads to S3), handle startup time differences, replicate all environment variables and IAM permissions exactly, and plan for EFS mounts if the monolith uses local file storage.
 
Scenario 8
Your company processes financial transactions. Compliance requires that no data leaves a specific AWS region, all traffic must be encrypted in transit, and audit logs must be immutable. Design an ECS architecture to meet these requirements.
Answer
Compliance-first ECS architecture:
 
Data residency:
- Deploy ECS cluster strictly in the required region. Disable ECR cross-region replication.
- Use VPC endpoints (PrivateLink) for all AWS service access (ECR, Secrets Manager, S3, CloudWatch) — no traffic leaves the AWS backbone.
- Use S3 bucket policies with aws:RequestedRegion condition to prevent cross-region data transfer.
 
Encryption in transit:
- Enforce HTTPS on ALB (HTTP -> HTTPS redirect, TLS 1.2+ only).
- Enable App Mesh mTLS for service-to-service communication using ACM Private CA.
- Enable EFS encryption in transit for any shared storage mounts.
- Use SSL-enabled RDS connections (enforce SSL at DB parameter group level).
 
Immutable audit logs:
- Enable CloudTrail with log file validation (SHA-256 hash chain) — detects tampering.
- Stream CloudTrail logs to S3 with Object Lock (WORM — Write Once Read Many) in Compliance mode.
- Enable VPC Flow Logs → S3 with Object Lock for network audit trail.
- Use CloudWatch Logs with log retention policies and export to S3 with Object Lock.
 
Additional controls: Least-privilege IAM task roles, ECR image scanning, Config rules for compliance drift detection, GuardDuty for threat detection, and Macie for sensitive data discovery in S3.
 
Scenario 9
You run 200 microservices on ECS. Service-to-service communication is chaos — hardcoded IPs, no retries, no circuit breaking, and debugging failures takes days. How do you fix this?
Answer
This calls for a service mesh or managed service connectivity layer.
 
Option A — ECS Service Connect (simpler, native):
- Enable Service Connect on each ECS service. Each service gets a DNS name (e.g., payment-service.prod).
- ECS automatically injects an Envoy proxy sidecar and handles traffic routing.
- Built-in: retries, timeouts, and CloudWatch metrics per service-to-service connection.
- No App Mesh setup required. Best for teams that want simplicity.
 
Option B — AWS App Mesh (more control):
- Define a mesh with Virtual Services, Virtual Nodes (one per ECS service), and Virtual Routers.
- Add Envoy sidecar to every Task Definition.
- Configure retry policies, circuit breakers, and traffic shifting at the mesh level.
- Integrate with X-Ray for distributed tracing across all 200 services.
 
Migration approach:
- Start with the highest-traffic or most-critical service pairs.
- Migrate service by service. Service Connect is easier to adopt incrementally.
- Use AWS Cloud Map for service discovery as the foundation.
 
Immediate wins: Service discovery via DNS (no more hardcoded IPs), automatic retries on transient failures, centralized timeout configuration, and a service dependency map in X-Ray.
 
Scenario 10
Your e-commerce platform needs to handle 10x traffic spikes on Black Friday. Currently your ECS service takes 8 minutes to scale from 10 to 100 tasks. How do you reduce this?
Answer
The 8-minute scale-out time is a compounded delay from multiple steps. Here's how to attack each:
 
Step 1 — Reduce container startup time:
- Profile your container's startup sequence. Is the app doing slow initializations on startup?
- Use lightweight base images (Alpine, distroless) to reduce image pull time.
- Cache layers in ECR — ensure large unchanged layers are cached on the instance.
- For Fargate: image pulls are the biggest time sink. Use small images or pull-through cache.
 
Step 2 — Pre-warm via Scheduled Scaling:
- Create a Scheduled Scaling action to pre-scale to 80 tasks at 7:55 AM on Black Friday before traffic hits.
- This is the single biggest win — scale before you need it.
 
Step 3 — Faster EC2 capacity (if EC2 launch type):
- Pre-warm EC2 instances via scheduled scaling on the ASG to ensure instances are ready before tasks need to be placed.
- Use Warm Pools on the ASG (instances are pre-initialized and waiting) to cut instance startup from 3 min to 15 sec.
 
Step 4 — Use Fargate for burst capacity:
- Set capacity provider strategy: base=10 on EC2, overflow on FARGATE. Fargate can absorb bursts without waiting for EC2 instances.
 
Step 5 — Tune scaling policy:
- Reduce scale-out cooldown to 60 seconds.
- Use a more aggressive target tracking policy (target 40% CPU instead of 70%).
- Add predictive scaling based on historical Black Friday traffic patterns.
 
Scenario 11
You need to run nightly data processing jobs that take 2-4 hours and process large datasets. These jobs should not impact production ECS services. How do you architect this?
Answer
Architecture for isolated batch processing on ECS:
 
Isolation strategy:
- Create a separate ECS cluster for batch jobs (never share with production services).
- Use Fargate Spot for cost savings — batch jobs can handle interruptions with retry logic.
- Give batch tasks their own VPC subnets and security groups — no overlap with prod.
 
Orchestration:
- Use AWS Step Functions to orchestrate multi-step pipelines (extract → transform → validate → load).
- Each step runs as an ECS Fargate task via the EcsRunTask state with .sync:2 integration.
- Step Functions handles retries, error handling, and parallel processing automatically.
 
Scheduling:
- Use EventBridge Scheduler to trigger the Step Functions state machine at 2 AM nightly.
- EventBridge rule: cron(0 2 * * ? *) → Step Functions StartExecution.
 
Cost optimization:
- Fargate Spot saves 60-70% vs regular Fargate. For a 3-hour job, this is significant.
- Tasks scale to 0 when not running — no idle compute costs.
- Use EFS or S3 for checkpoint storage so interrupted Spot tasks can resume from last checkpoint.
 
Monitoring:
- CloudWatch alarms on Step Functions ExecutionsFailed metric.
- SNS notification on failure for on-call alerts.
- Set task CPU and memory based on profiling actual job resource usage.
 
Scenario 12
Your startup is growing fast. You started with a simple ECS setup but now have 3 teams, 30 services, and everything is in one AWS account with no separation. How do you restructure?
Answer
Time to implement AWS Organizations with multi-account strategy:
 
Account structure (AWS Organizations):
- Management Account: Billing, SSO, SCPs only. No workloads.
- Shared Services Account: ECR (shared image registry), networking (Transit Gateway), CI/CD tooling.
- Per-environment accounts: dev, staging, production — each team gets isolated AWS accounts.
- Security account: CloudTrail aggregation, GuardDuty master, Security Hub.
 
ECS restructuring:
- One ECS cluster per team per environment (e.g., team-a-prod, team-b-prod).
- Each team owns their ECR repos in the Shared Services account with cross-account pull permissions.
- Use SCPs (Service Control Policies) to enforce guardrails (e.g., deny launching tasks without task roles, deny disabling CloudTrail).
 
Networking:
- Each account has its own VPC. Use VPC sharing or Transit Gateway for controlled cross-VPC communication.
- Shared ALBs or per-team ALBs based on team needs.
 
CI/CD:
- Central CodePipeline in the Shared Services account deploys to each team's account using cross-account IAM roles.
- Each team has a separate pipeline for their services.
 
IaC: Migrate to Terraform with one state file per team per environment. Teams own their own modules.
 
Scenario 13
You need to expose a legacy SOAP/XML service running in ECS to modern REST API consumers, without modifying the legacy app. How do you architect this?
Answer
Build an API facade/adapter layer in front of the legacy ECS service:
 
Architecture:
- Deploy the legacy SOAP service as an ECS service on an internal ALB (not internet-facing).
- Build a lightweight REST adapter service (Node.js, Python Flask, or Go) that: accepts REST/JSON requests, translates to SOAP/XML, calls the legacy service, transforms the SOAP response back to JSON.
- Deploy the REST adapter as a separate ECS Fargate service on a public-facing ALB.
- Route all external traffic through API Gateway → REST Adapter ECS → Legacy ECS (SOAP).
 
Benefits: Legacy app is untouched and isolated. REST clients get a clean API. You can version the REST adapter independently. Legacy service is in a private subnet — not directly exposed.
 
Resilience:
- Add retry logic in the adapter for SOAP call failures.
- Cache common SOAP responses in ElastiCache to reduce load on the legacy service.
- Implement circuit breaker in the adapter (using a library like resilience4j or opossum).
 
Monitoring:
- Separate CloudWatch dashboards for the adapter and the legacy service.
- Alert on SOAP response errors separately from REST API errors.
- Use X-Ray to trace requests end-to-end through the adapter into the legacy service.
 
Scenario 14
A client requires you to process medical imaging files (up to 50 GB each) in ECS containers. The processing takes 30-90 minutes per file. How do you design this system?
Answer
Design for large-file, long-running processing on ECS:
 
Compute:
- Use ECS EC2 launch type (not Fargate) — Fargate max storage is 200 GB ephemeral and Fargate Spot interruptions are a risk for 90-minute jobs.
- Use compute-optimized (C5) or memory-optimized (R5) EC2 instances based on the algorithm's profile.
- Consider GPU instances (G4dn, P3) if the imaging algorithm uses GPU acceleration.
 
Storage strategy:
- Ingest files to S3 (not EFS) — S3 handles 50 GB files natively with multipart upload.
- The ECS task downloads the file from S3 to the container's ephemeral storage or a bound EBS volume at startup.
- For shared intermediate data, use EFS. For final outputs, write back to S3.
 
Workflow:
- SQS queue receives file processing jobs (S3 object key as the message).
- ECS service (REPLICA, scaled based on SQS queue depth) picks up messages and processes files.
- Set SQS visibility timeout > 90 minutes so the message doesn't become visible while being processed.
- On success, delete the SQS message. On failure, let it go to a Dead Letter Queue (DLQ).
 
HIPAA considerations: Encrypt EBS volumes at rest, use S3 SSE-KMS, enforce TLS in transit, log all access via CloudTrail, and isolate the processing cluster in a private subnet with no internet access.
 
💰 Cost Optimization Scenarios

Scenario 15
Your AWS bill for ECS/Fargate jumped 40% this month. You're tasked with reducing it by 25% without degrading performance. What's your approach?
Answer
Systematic cost reduction for ECS/Fargate:
 
Step 1 — Right-size tasks (biggest impact):
- Pull Container Insights metrics for all services. Look for CPU and memory average utilization over 30 days.
- If average CPU is <30%, reduce the task CPU allocation. If average memory is <40%, reduce memory.
- Example: Cutting a task from 1 vCPU/2 GB to 0.5 vCPU/1 GB halves the Fargate cost for that service.
- Use AWS Compute Optimizer — it provides specific Fargate right-sizing recommendations.
 
Step 2 — Introduce Fargate Spot for eligible workloads:
- Identify stateless, interruptible services (background workers, async processors, dev/test).
- Add FARGATE_SPOT capacity provider with appropriate weight. A 50/50 mix saves ~35%.
- Ensure apps handle SIGTERM gracefully for Spot interruptions.
 
Step 3 — Purchase Compute Savings Plans:
- Analyze consistent Fargate usage (the base load that's always running).
- Purchase a 1-year Compute Savings Plan for that baseline — saves up to 50% vs on-demand.
 
Step 4 — Schedule non-production scale-down:
- Use EventBridge Scheduler + Lambda to set desired count to 0 for dev/staging ECS services outside business hours (evenings and weekends).
- A dev environment running 16 hours/day vs 24 saves 33% on that environment.
 
Step 5 — Review data transfer costs:
- Are ECS tasks calling S3, ECR, or other AWS services over NAT Gateway? VPC endpoints eliminate NAT Gateway data processing charges.
 
Scenario 16
You have a machine learning inference service on ECS that gets 95% of its traffic between 9 AM and 6 PM. At 2 AM it still runs 20 tasks. How do you optimize this?
Answer
This is a classic scheduled scaling opportunity combined with predictive scaling:
 
Scheduled Scaling Actions:
- 8:45 AM (pre-warm): Scale out to 25 tasks ahead of traffic arrival.
- 6:15 PM (post-traffic): Scale in to 5 tasks.
- 11:00 PM: Scale in to 2 tasks (minimum for availability).
- Actions are set via Application Auto Scaling ScheduledAction.
 
Target Tracking for intra-day variability:
- Between the scheduled actions, let Target Tracking Auto Scaling handle real-time adjustments based on CPU or custom latency metric.
- This handles lunch peaks, Monday morning spikes, etc.
 
Capacity Provider Strategy:
- base=2 on FARGATE (always guaranteed, handles overnight load).
- Remainder (burst tasks) on FARGATE_SPOT for daytime scaling — 60% cost savings on the burst portion.
 
Expected savings calculation:
- Before: 20 tasks x 24 hours = 480 task-hours/day.
- After: 2 tasks x 15 hours (night) + 20 tasks x 9 hours (day) = 30 + 180 = 210 task-hours/day.
- That's a 56% reduction in compute hours, plus ~60% savings on Spot for daytime burst tasks.
- With Savings Plans on the base 2 tasks, total savings approach 65-70%.
 
Scenario 17
Your company runs 50 ECS services across dev, staging, and production. The dev and staging environments cost almost as much as production. How do you cut this?
Answer
Non-production environments are a classic area for savings. Strategy:
 
Scale-to-zero for off-hours:
- Create a Lambda function that calls UpdateService with desiredCount=0 for all dev/staging services.
- EventBridge rule: Run at 7 PM weekdays and all day weekends.
- Create a complementary Lambda to scale back up at 8 AM weekdays.
- Estimated savings: dev/staging runs 11 hours/day vs 24 — saves 54%.
 
Right-size aggressively for non-prod:
- Non-production services don't need the same task size as production. Use the minimum viable config.
- Example: Prod uses 1 vCPU/2 GB. Dev can use 0.25 vCPU/512 MB — 8x cost reduction per task.
 
Consolidate onto fewer tasks:
- In dev, run 1 task per service instead of 2. Availability doesn't matter in dev.
- Merge related dev services onto a single ECS cluster with EC2 using binpack placement.
 
Use FARGATE_SPOT exclusively for dev/staging:
- Dev and staging can handle interruptions — use 100% Fargate Spot. Saves 60-70%.
 
Share infrastructure where safe:
- Dev and staging can share an ALB (using different listener rules/paths).
- Share a single RDS instance (with separate databases) for non-prod.
 
Combined, these measures typically reduce non-prod costs by 70-80%.
 
🔐 Security Scenarios

Scenario 18
A security audit found that several ECS tasks are running as root, and some containers have access to the Docker socket. How do you remediate this?
Answer
This is a critical security finding — running as root and Docker socket access are container escape vectors.
 
Immediate remediation:
1. Remove Docker socket mounts: Search all Task Definitions for volume mounts of /var/run/docker.sock. Remove immediately unless there's a documented operational need (if needed, restrict to specific monitoring agents only).
2. Set non-root user in Dockerfiles: Add USER 1000:1000 (or a named non-root user) as the last USER instruction before CMD/ENTRYPOINT.
3. If Dockerfile can't be changed immediately: Set user field in the ECS container definition to a non-root UID.
 
Additional hardening in Task Definitions:
- Set readonlyRootFilesystem: true (app writes go to specific volume mounts, not root FS).
- Use linuxParameters.capabilities.drop: ['ALL'] and add back only what's needed.
- Set privileged: false (should already be false, but make it explicit).
- Set noNewPrivileges: true to prevent privilege escalation.
 
Preventive controls going forward:
- Add a CI/CD gate that scans Dockerfiles and task definitions for root user, privileged mode, and Docker socket mounts. Fail the build if found.
- Use AWS Config custom rule to detect ECS task definitions with privileged: true.
- Enable ECR image scanning — it flags images built to run as root.
- Use Fargate instead of EC2 where possible — Fargate's VM isolation limits the blast radius of a container compromise.
 
Scenario 19
You suspect one of your ECS containers may have been compromised. It's currently running in production. What do you do?
Answer
This is a security incident response scenario. Act fast but methodically.
 
Immediate containment (within minutes):
1. Isolate the task: Update the task's security group to deny all inbound and outbound traffic. This stops exfiltration and C2 communication without immediately killing the task (preserving forensic state).
2. Deregister from ALB: Remove the task from its target group so no user traffic reaches it.
3. DO NOT immediately stop the task — you want to preserve memory state for forensics.
4. Start a new task from a known-good image to restore service capacity.
 
Forensic collection (while task is still running):
- Use ECS Exec to collect: running processes (ps auxf), network connections (netstat -antp), bash history, cron jobs, and any suspicious files.
- Capture VPC Flow Logs around the task's ENI to see what connections were made.
- Export CloudWatch Logs for the container's full output.
- Check CloudTrail for any API calls made by the task's IAM role — did it access S3, Secrets Manager, or other services it shouldn't have?
 
After forensics:
- Stop and terminate the compromised task.
- Rotate all secrets and IAM credentials the task had access to.
- Revoke and reissue any API keys the task could have accessed.
- Review the compromised image for backdoors or malicious dependencies.
 
Post-incident: Enable GuardDuty (detects anomalous IAM calls from containers), and Runtime Security tools (Falco, Sysdig) for behavioral anomaly detection.
 
Scenario 20
Your ECS tasks need to access an on-premises database over a direct connection, but you cannot expose the database to the internet. How do you securely connect them?
Answer
Secure on-premises connectivity options for ECS:
 
Option A — AWS Direct Connect (production-grade):
- Establish a Direct Connect link from on-premises to AWS.
- ECS tasks in a private subnet route traffic to the on-premises database through the Direct Connect virtual interface.
- No internet exposure — traffic stays on a dedicated private link.
- Add encryption over Direct Connect using MACsec (Layer 2) or site-to-site VPN over Direct Connect for extra security.
 
Option B — AWS Site-to-Site VPN (simpler, lower cost):
- Create a VPN connection between the on-premises network and your AWS VPC.
- ECS tasks route database traffic through the VPN tunnel.
- VPN traffic is encrypted (IPSec). Works well for lower bandwidth needs.
 
Network configuration:
- ECS tasks use awsvpc mode with security groups that allow only port 5432/3306 (DB port) to the on-premises CIDR range.
- On-premises firewall allows only the ECS subnet CIDRs to reach the DB server.
- No 0.0.0.0/0 anywhere.
 
Authentication:
- Use long-lived DB credentials stored in AWS Secrets Manager with automatic rotation.
- ECS tasks retrieve credentials via the Secrets Manager VPC endpoint (no internet path for secret retrieval either).
- Enable SSL/TLS on the database connection and verify the DB server certificate.
 
Scenario 21
A developer accidentally hardcoded an AWS access key in a Docker image that was pushed to ECR and deployed to production ECS. How do you respond?
Answer
This is a credentials exposure incident — treat it as a breach until proven otherwise.
 
Immediate response (first 5 minutes):
1. Deactivate the leaked key immediately: aws iam update-access-key --access-key-id AKIAXXXXXX --status Inactive
2. Check CloudTrail NOW: aws cloudtrail lookup-events --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIAXXXXXX — look for any calls made with this key that you didn't make.
3. If suspicious activity found: this is now a security incident — escalate and engage your incident response process.
 
Cleanup:
- Delete the ECR image (and all tags pointing to it): aws ecr batch-delete-image ...
- If the key was in a Dockerfile layer, it may exist in intermediate layers too — delete the repository entirely if needed and start fresh.
- Delete the IAM access key: aws iam delete-access-key --access-key-id AKIAXXXXXX
- Review and revoke any permissions associated with the key's IAM user. Were any resources created or modified?
 
Fix the root cause:
- Remove the hardcoded key from the Dockerfile and rebuild/push a clean image.
- Deploy the clean image immediately.
- Use IAM Task Roles instead — ECS tasks should never use static IAM access keys. Task roles provide temporary credentials via the Instance Metadata Service with automatic rotation.
 
Prevention:
- Add git-secrets or truffleHog to CI/CD to block commits with hardcoded credentials.
- Enable ECR Enhanced Scanning — it flags secrets in image layers.
- Enable AWS Config rule: iam-no-inline-policy, and train devs on IAM task roles.
 
Scenario 22
Your ECS service handles PII data and must comply with GDPR's right to erasure. A user requests data deletion, and it must be completed within 72 hours. How do ECS architectures factor into this?
Answer
GDPR right to erasure in an ECS microservices context:
 
The challenge: PII can be scattered across multiple services' datastores, logs, caches, and backups.
 
Architecture design for erasure-friendly ECS services:
1. Data inventory: Each ECS service must document what PII it stores and where (RDS, DynamoDB, S3, ElastiCache, CloudWatch Logs).
2. Deletion API: Each service exposes a /users/{id}/delete internal API endpoint that deletes that user's data from its own datastore.
3. Orchestration service: A dedicated 'erasure' ECS service receives the deletion request and calls each microservice's deletion API, tracking completion.
4. Event-driven fan-out: Publish a 'UserDeletionRequested' event to SNS/EventBridge. Each service subscribes and handles its own deletion asynchronously, reporting completion back.
 
For logs (CloudWatch, S3):
- Avoid logging raw PII. Log user IDs instead of emails/names.
- Use tokenization or encryption so log entries are unreadable without the encryption key.
- Erasing a user's key makes all log entries for that user unintelligible — equivalent to deletion.
 
For ECS ephemeral storage:
- Container ephemeral storage is wiped when a task stops — not a concern.
- EFS volumes or EBS volumes with PII must be explicitly cleaned.
 
Audit trail: Maintain a separate audit log (outside of erasure scope) that records 'user X was deleted on date Y' without any PII content.
 
🚀 CI/CD & Deployment Scenarios

Scenario 23
You need to deploy 30 ECS microservices as part of a single release. Some services must deploy in a specific order due to database schema dependencies. How do you manage this?
Answer
Ordered multi-service deployment orchestration:
 
Strategy: Decompose into deployment waves with dependency gates.
 
Wave 1 — Database migrations (no traffic yet):
- Run DB migration as a one-off ECS task (not a service) using RunTask API.
- Wait for the migration task to complete successfully before proceeding.
- Step Functions: use EcsRunTask state with .sync:2 and a failure transition to abort the whole pipeline.
 
Wave 2 — Backend services (after migrations):
- Deploy backend services (APIs, data processors) that depend on the new schema.
- These can deploy in parallel within the wave.
- Gate: Wait for each service to reach steady state (all tasks healthy) before proceeding.
 
Wave 3 — Frontend/gateway services:
- Deploy API gateways and frontend-facing services that call the new backend APIs.
 
Implementation options:
A. AWS Step Functions: Each wave is a parallel state that deploys services concurrently. State machine waits for ECS service stable state between waves using DescribeServices polling.
B. CodePipeline with multiple stages: Stage 1 = migrations, Stage 2 = backends (parallel actions), Stage 3 = frontends.
C. Deployment orchestration tool: Spinnaker or ArgoCD for complex multi-service pipelines.
 
Rollback: If any wave fails, Step Functions error handler calls UpdateService to roll back all deployed services to previous task definition revisions in reverse order (Wave 3 → Wave 2 → Wave 1 undo migration).
 
Scenario 24
A bad deployment went to production and you need to roll back 15 ECS services to their previous versions in under 5 minutes. How do you do this at speed?
Answer
Emergency mass rollback procedure:
 
Preparation (should be done before incidents):
- Maintain a 'last known good' deployment manifest: a JSON/YAML file in S3 or DynamoDB recording the task definition ARN for each service that was last confirmed stable.
- Store this after every successful deployment.
 
Emergency rollback execution:
1. Trigger rollback pipeline: A pre-built CodePipeline or Lambda reads the 'last known good' manifest.
2. Lambda calls UpdateService in parallel for all 15 services simultaneously (Python boto3 with ThreadPoolExecutor or Step Functions parallel states).
3. Each UpdateService call references the previous task definition revision.
4. ECS begins rolling back each service concurrently — all 15 services start rolling back at the same time.
 
Sample parallel rollback code concept:
- Read manifest from DynamoDB: {service-a: task-def-arn:42, service-b: task-def-arn:17, ...}
- Call UpdateService for all services concurrently using thread pool.
- Monitor until all services reach steady state.
 
Time estimate: With parallel updates, all 15 services start rolling back within 30 seconds. With 2 tasks per service and fast health checks, each service completes rollback in 2-4 minutes.
 
Going forward: Enable deployment circuit breakers on all services for automatic rollback on bad deployments, so manual intervention is rarely needed.
 
Scenario 25
Your team wants to implement feature flags so that new code can be deployed to ECS but features can be toggled on/off without redeployment. How do you build this?
Answer
Feature flags in ECS without redeployment:
 
Option A — AWS AppConfig (recommended):
- Store feature flag configuration in AWS AppConfig (JSON/YAML: {featureX: true, featureY: false}).
- Your application polls AppConfig via SDK. AppConfig supports real-time push via Lambda extensions.
- AppConfig has built-in gradual rollout (deploy to 10% of calls, validate, then 100%).
- No ECS task restart needed — the SDK polls for new config at runtime.
- Integrate the AppConfig agent as a sidecar container in ECS for efficient polling.
 
Option B — Parameter Store with application polling:
- Store flags in SSM Parameter Store.
- Application checks Parameter Store every 30 seconds (with caching) for flag values.
- Simple to implement. Add AWS SDK call with a TTL-based local cache.
 
Option C — Dedicated feature flag service:
- Use LaunchDarkly, Split.io, or Flagsmith — managed feature flag platforms.
- Provide SDKs for all languages. Support user-targeting (roll out to specific user segments).
- Best for complex use cases: A/B testing, percentage rollouts, user-based targeting.
 
Implementation pattern:
- Deploy new code to ECS with the feature flag set to OFF. Verify deployment is healthy.
- Enable flag in AppConfig/LaunchDarkly. Monitor metrics (error rate, latency).
- If metrics degrade, disable the flag instantly — no redeployment, no rollback pipeline needed.
- This is called a 'dark launch' or 'dark deployment' pattern.
 
Scenario 26
Your ECS CI/CD pipeline takes 45 minutes from code commit to production deployment, but the team wants it under 10 minutes. How do you speed it up?
Answer
Pipeline acceleration — attack each phase:
 
Phase 1 — Build optimization (typically 15-20 min → 3-4 min):
- Use Docker layer caching in CodeBuild. Cache the FROM and RUN apt-get layers — only rebuild changed layers.
- Use CodeBuild cache (S3 or local cache) for package managers (npm, pip, maven).
- Use multi-stage builds to reduce what's built in CI.
- Parallelize unit tests across CodeBuild containers.
- Use a faster CodeBuild instance type (compute.large instead of compute.small).
 
Phase 2 — Image push optimization (5-10 min → 1-2 min):
- Enable ECR pull-through cache — if base images are cached, they don't need to be pulled from Docker Hub.
- Use BuildKit's inline cache (--cache-from the previous image) for incremental builds.
 
Phase 3 — Test/staging gate (10-15 min → 2-3 min):
- Run integration tests against the new container in parallel with the build.
- Use lightweight smoke tests instead of full regression in the CD pipeline (full regression in a separate async pipeline).
 
Phase 4 — ECS deployment (5-10 min → 2-3 min):
- Optimize container startup time (faster JVM startup, pre-built assets).
- Tune ALB health check to be faster for new tasks (lower interval, lower threshold).
- Increase maximumPercent to 200% so new tasks start before old ones stop.
 
Result: Build (4 min) + Push (2 min) + Tests (3 min) + Deploy (3 min) = 12 min. With further tuning, sub-10 min is achievable.
 
📊 Scaling & Performance Scenarios

Scenario 27
Your ECS service's p99 latency spikes to 30 seconds every day at exactly 3 PM, even though traffic doesn't spike at that time. How do you investigate this?
Answer
A time-correlated latency spike that isn't traffic-driven points to internal scheduled behavior.
 
Investigation checklist:
1. Correlate with deployments: Is there a scheduled deployment, Lambda, or cron job at 3 PM? Check CodePipeline execution history.
2. Check ECS task lifecycle events: Are tasks being replaced at 3 PM? (Check ECS service events — maybe a scheduled scaling action is replacing tasks.)
3. Check application-level cron jobs: Does the app have a background job at 3 PM? (DB cleanup, report generation, cache warm-up?) Check CloudWatch Logs for log volume spike at 3 PM.
4. Check garbage collection: JVM-based apps can have GC pauses. Look for 'GC pause' log entries. At 3 PM, maybe a memory-heavy scheduled job triggers a full GC cycle.
5. Check downstream dependencies: Does a DB backup, snapshot, or maintenance window run at 3 PM? DB snapshot can cause I/O contention.
6. Check Auto Scaling scale-in: Is ECS scaling in at 3 PM (scheduled action or low CPU after lunch), causing connection draining to be slow?
 
Diagnosis tools:
- X-Ray service map: Shows which downstream call is responsible for the latency at 3 PM.
- CloudWatch Container Insights: Check CPU, memory, and network metrics at 3 PM exactly.
- RDS Performance Insights: Check for long-running queries at 3 PM.
 
Fix depends on cause: If GC pause → tune JVM heap settings. If DB backup → reschedule. If scale-in → tune cooldown. If internal cron → run it asynchronously in a separate Fargate task.
 
Scenario 28
Your ECS auto scaling is thrashing — tasks are constantly scaling out and immediately scaling back in, causing instability. How do you fix this?
Answer
Scaling thrash (hunting) is caused by misconfigured scaling policies or metric selection. Fixes:
 
Root cause identification:
- Is the metric too sensitive? If tracking CPU and it fluctuates between 45% and 55% around a 50% target, Auto Scaling will constantly add and remove tasks.
- Are cooldown periods too short? If scale-in cooldown is 60 seconds and tasks take 90 seconds to start, new tasks register before the old metric settles.
 
Fixes:
1. Increase cooldown periods: Set scale-out cooldown to 120-180 seconds, scale-in cooldown to 300-600 seconds. Scale-in should always be more conservative than scale-out.
2. Add metric smoothing: Instead of scaling on the raw CPU metric, scale on a 3-minute average or a custom metric that's less spiky.
3. Use target tracking with appropriate target: If CPU fluctuates 40-60%, set target to 70% so normal operation stays below the threshold.
4. Add a scale-in protection buffer: Target tracking already has built-in scale-in protection. Ensure it's enabled.
5. Use step scaling with dead bands: Instead of target tracking, use step scaling with a dead band (e.g., only scale out at >75% CPU, only scale in at <30% CPU — no action in the 30-75% band).
6. Disable scale-in during business hours if appropriate: Some teams disable scale-in during the day and only allow it at night.
 
Monitoring: Create a CloudWatch dashboard tracking desired count and running count over time. Thrashing is immediately visible as a saw-tooth pattern.
 
Scenario 29
Your ECS service on EC2 can't scale out because tasks fail to place — the cluster shows available memory and CPU but tasks still fail placement. Why?
Answer
This is a common and confusing situation. Available aggregate resources don't mean any single instance can fit the task.
 
Root cause: Fragmentation. Imagine 4 instances each with 0.4 vCPU and 0.5 GB free. Aggregate: 1.6 vCPU and 2 GB — looks like plenty. But a task needing 1 vCPU / 1 GB can't fit on any single instance.
 
Diagnosis:
- Check ECS placement failure reason in the service events: 'no container instances found that satisfy all placement constraints'.
- Check each instance's available resources: aws ecs describe-container-instances — look for instances where remaining CPU and remaining memory can together accommodate the task.
 
Solutions:
1. Add more instances: The fastest fix. The ASG adds a new clean instance with full resources.
2. Enable Cluster Auto Scaling: With a managed capacity provider, ECS automatically scales the ASG when it can't place tasks — this should have handled it automatically.
3. Enable Cluster Auto Scaling with targetCapacity: Set it to scale proactively (e.g., keep 20% buffer) rather than reactively.
4. Reduce task size: If tasks can be made smaller, fragmentation is less of a problem.
5. Restart fragmented instances: Instances with fragmented resources are often old instances with small tasks accumulated. Rotate them (scale out + scale in) periodically.
6. Switch to Fargate: Fargate never has placement failures due to fragmentation — AWS manages the underlying bin-packing.
 
Scenario 30
You're seeing high CPU utilization on your ECS tasks during business hours but low memory usage. Adding more tasks doesn't improve throughput. What's happening?
Answer
When more tasks don't improve throughput despite high CPU, the bottleneck is not compute capacity — it's something else.
 
Investigation:
1. Check if CPU is genuinely the bottleneck or if tasks are waiting: Use X-Ray to see where time is spent. Are tasks waiting on I/O (DB calls, external API calls) rather than computing?
2. Check thread pool exhaustion: If the app has a fixed thread pool (e.g., 10 threads) and each thread is blocked on a DB call, adding tasks just adds more blocked threads. The DB is the bottleneck.
3. Check database connection limits: Is the RDS instance at max connections? New ECS tasks can't get DB connections and queue up — appearing as CPU wait.
4. Check external API rate limits: If the service calls a third-party API with rate limits, more tasks mean more rate limit hits and more retrying — wasting CPU.
5. Check for a lock contention in the app: If tasks compete for a shared lock (in-memory, DB-level, or distributed lock), adding more tasks increases contention.
 
Solutions by root cause:
- DB bottleneck: Scale up RDS (more IOPS, larger instance), add read replicas, add RDS Proxy to pool connections.
- Thread pool exhaustion: Switch to async/non-blocking I/O (e.g., reactive frameworks). Increase thread pool size as a stop-gap.
- Rate limits: Implement a token bucket or leaky bucket rate limiter. Add SQS in front to smooth request rates.
- Lock contention: Redesign to reduce shared state. Use optimistic locking or CQRS patterns.
 
🌍 Networking & Connectivity Scenarios

Scenario 31
ECS tasks in your private subnet can't reach the internet to call a third-party API. The NAT Gateway exists and routes look correct. How do you troubleshoot?
Answer
Systematic networking troubleshooting for ECS private subnet internet access:
 
Step 1 — Verify the task's subnet is truly private:
- Is the task's subnet route table pointing 0.0.0.0/0 to a NAT Gateway (not an Internet Gateway)?
- If it points to an Internet Gateway, it's a public subnet and the task needs a public IP (or it's misconfigured).
 
Step 2 — Verify NAT Gateway is in a public subnet:
- The NAT Gateway must be in a public subnet (one with a route to an Internet Gateway).
- Check the NAT Gateway status — is it 'available'?
 
Step 3 — Security group check:
- Does the ECS task's security group have an outbound rule allowing TCP 443 (HTTPS) to 0.0.0.0/0?
- Many setups block all outbound traffic. Add: egress TCP 443 → 0.0.0.0/0.
 
Step 4 — Network ACL check:
- Unlike security groups, NACLs are stateless. Check both the task's subnet NACL and the NAT Gateway's public subnet NACL.
- Ensure outbound ephemeral ports (1024-65535) are allowed back in (return traffic).
 
Step 5 — VPC endpoints override:
- If you have a VPC endpoint for the service being called, it may be routing traffic internally and failing. Remove or check the endpoint policy.
 
Step 6 — Test from inside the container:
- Use ECS Exec: curl -v https://api.third-party.com — if it times out, the routing issue is confirmed. If you get a TLS error, routing works but there's a cert/proxy issue.
- Check: curl https://checkip.amazonaws.com — this should return the NAT Gateway's public IP if routing is correct.
 
Scenario 32
Two ECS services need to communicate with each other. Service A calls Service B's API. After deploying Service B with a new task definition, Service A gets intermittent connection refused errors. Why?
Answer
Intermittent connection refused after a deployment is a classic service discovery / rolling update timing issue.
 
Root cause scenarios:
 
Scenario A — Stale DNS records (most common):
- Service A caches the IP of Service B (from DNS lookup or hardcoded Service B's old task IP).
- Service B's old tasks are stopped and new ones start with new IPs.
- Service A still has the old IP cached → connection refused.
- Fix: Use Cloud Map or Service Connect DNS for Service B. Reduce DNS TTL (Cloud Map default is 60 seconds). Ensure Service A's HTTP client respects DNS TTL and re-resolves on each connection (disable connection pooling to stale IPs or use short pool TTLs).
 
Scenario B — ALB connection draining too fast:
- Service A calls Service B via an ALB. Old tasks deregister from ALB before in-flight requests complete.
- Fix: Increase the Target Group deregistration delay to 30-60 seconds to allow in-flight requests to complete before old tasks are removed.
 
Scenario C — New tasks not healthy yet:
- During deployment, new Service B tasks are starting but haven't passed health checks. Old tasks are gone. Brief period with zero healthy targets.
- Fix: Set minimumHealthyPercent=100 so old tasks aren't stopped until new ones are healthy.
 
Scenario D — Service B's startPeriod too short:
- New tasks receive traffic from ALB before they're fully initialized.
- Fix: Increase startPeriod in the health check, or implement a /ready endpoint that only returns 200 when the app is fully initialized.
 
Scenario 33
Your ECS Fargate tasks are exhausting the IP addresses in your VPC subnet. You're running out of private IPs. How do you solve this without re-architecting the VPC?
Answer
IP exhaustion in Fargate (awsvpc mode) is a scaling pain point. Solutions:
 
Option A — Add more subnets (quickest fix):
- Create new /24 or /22 subnets in the same VPC using unused CIDR ranges.
- Add these subnets to the ECS service's network configuration.
- ECS will spread tasks across all configured subnets, using IPs from all of them.
 
Option B — VPC secondary CIDR expansion:
- Attach an additional CIDR block to your existing VPC (AWS allows up to 5 CIDRs per VPC).
- Example: If VPC is 10.0.0.0/16 (65,536 IPs) and it's exhausted, add 100.64.0.0/16 (RFC 6598 shared address space — valid for VPCs).
- Create new subnets in the new CIDR and add to ECS services.
 
Option C — Reduce IP usage:
- Audit whether all tasks truly need awsvpc mode. Services that don't need task-level security groups could use bridge mode (multiple tasks per ENI IP).
- Reduce task count through right-sizing — fewer, more capable tasks use fewer IPs.
- Shorter task lifetimes (ephemeral batch tasks) mean IPs are returned sooner.
 
Option D — IPv6 dual-stack:
- Enable IPv6 on the VPC and subnets. Use IPv6 addresses for Fargate tasks.
- IPv6 provides a /56 prefix per subnet — effectively unlimited addresses.
- Requires application support for IPv6 (most modern apps handle this fine).
 
Long-term: Plan VPC CIDR sizes during the initial VPC design. A /16 VPC with /20 subnets supports ~4,000 IPs per subnet — sufficient for most workloads.
 
📦 Migration & Modernization Scenarios

Scenario 34
Your team is migrating from Docker Swarm to ECS. What are the key differences, and how do you map Swarm concepts to ECS?
Answer
Docker Swarm to ECS concept mapping:
 
Concept mapping:
- Swarm Stack → ECS Cluster + multiple ECS Services
- Swarm Service → ECS Service
- Swarm Task/Container → ECS Task
- Swarm docker-compose.yml → ECS Task Definition (+ CloudFormation/Terraform)
- Swarm overlay network → ECS awsvpc mode VPC networking
- Swarm service discovery (DNS) → AWS Cloud Map or ECS Service Connect
- Swarm secrets → AWS Secrets Manager / SSM Parameter Store
- Swarm configs → AWS Parameter Store / S3
 
Key differences:
1. Networking: Swarm overlay network is automatic. ECS requires explicit VPC, subnet, and security group configuration per task.
2. Secrets: Swarm secrets are native. ECS uses AWS Secrets Manager or SSM — more powerful but requires IAM permission setup.
3. Scaling: Swarm auto-scaling is manual. ECS has native Application Auto Scaling with CloudWatch integration.
4. Health checks: Both support health checks. ECS integrates with ALB health checks natively.
5. No docker-compose in prod: ECS doesn't use compose files. Migrate to Task Definitions (JSON) and manage with CloudFormation, Terraform, or CDK.
 
Migration approach:
- Use the 'AWS Copilot CLI' — it understands compose-like patterns and generates ECS resources. Translate each Swarm service to a Copilot service configuration.
- Migrate service by service, using weighted ALB routing to shift traffic gradually from Swarm to ECS.
 
Scenario 35
You're re-platforming a 10-year-old Java EE monolith to microservices on ECS. The monolith has a single Oracle database shared by all modules. How do you approach the data layer?
Answer
The shared database is the hardest part of microservices decomposition. A phased approach:
 
Phase 1 — Strangler Fig (don't break the DB yet):
- Extract one module at a time as an ECS microservice but still read/write from the monolith's Oracle database.
- The new service and old monolith share the DB temporarily.
- This validates the service in production without a big-bang migration.
 
Phase 2 — Schema isolation (while still sharing the DB):
- Assign dedicated Oracle schemas per service (schema-per-service pattern).
- Services only access their own schema. Cross-schema joins are replaced with API calls or async events.
- Remove foreign keys across service boundaries — reference by ID only.
 
Phase 3 — Database per service:
- Migrate each service's schema to its own appropriate database on AWS (e.g., PostgreSQL on Aurora, DynamoDB for simple key-value access, etc.).
- Use AWS DMS (Database Migration Service) to migrate data from Oracle to the new database.
- Run dual writes during cutover: the service writes to both Oracle and the new DB, then switches reads to the new DB, then removes Oracle writes.
 
Event-driven data synchronization:
- Use the Outbox Pattern: services write events to a local 'outbox' table. A CDC (Change Data Capture) tool like Debezium or AWS DMS CDC mode streams these events to other services via EventBridge or Kinesis.
 
This is a multi-year journey for a large monolith. Don't rush Phase 3 — the service isolation in Phase 1 and 2 provides value long before full DB separation.
 
Scenario 36
Your company acquired another company. Their application runs on Kubernetes (EKS) but yours runs on ECS. Leadership wants both running in the same platform within 6 months. Which direction do you migrate and how?
Answer
Platform consolidation decision framework:
 
Decision factors: ECS vs EKS
- Team expertise: Does either team know Kubernetes? EKS requires deeper knowledge.
- Workload requirements: Does the acquired company use Kubernetes-specific features (CRDs, operators, Helm charts)?
- Scale: EKS is better for very large deployments with complex scheduling needs.
- Timeline: ECS migration of K8s workloads is faster (6 months is tight for either direction).
 
If migrating K8s to ECS:
- Map K8s Deployments → ECS Services, Pods → Tasks, ConfigMaps → SSM Parameter Store, Kubernetes Secrets → Secrets Manager.
- Use Kompose tool to convert Kubernetes manifests to Docker Compose, then adapt to ECS Task Definitions.
- Rebuild CI/CD pipelines around CodePipeline or GitHub Actions deploying to ECS.
- Migrate services one by one, using Route 53 weighted routing to shift traffic gradually.
 
If migrating ECS to EKS:
- Use AWS App2Container to automatically convert ECS workloads to Kubernetes YAML.
- Stand up an EKS cluster with the same VPC and networking.
- Migrate services using blue/green at the Route 53 level.
- Invest in Kubernetes training for the ECS team.
 
Hybrid option (realistic for 6 months): Keep both running. Standardize on shared infrastructure (same VPC, same ECR, same CI/CD pipelines) and consolidate over 12-18 months based on measured value.
 
📈 Observability & Monitoring Scenarios

Scenario 37
You have 50 ECS services but no centralized observability. Error rates and latency issues are discovered by customers, not your team. How do you build a proper observability platform?
Answer
Building ECS observability from scratch — three pillars: Metrics, Logs, Traces.
 
Pillar 1 — Metrics (CloudWatch Container Insights):
- Enable Container Insights on all ECS clusters: provides CPU, memory, network, and disk metrics per cluster, service, and task.
- Create CloudWatch alarms for: high CPU (>80%), high memory (>85%), and low task count (below desired).
- Build a CloudWatch dashboard showing all 50 services' health at a glance — use metric math to compute error rates from ALB metrics.
 
Pillar 2 — Logs (centralized):
- Standardize all services to use structured JSON logging (log level, request ID, service name, timestamp as fields).
- Use awslogs driver to send all container logs to CloudWatch Logs with consistent log group naming (/ecs/{service-name}).
- Create CloudWatch Log Insights queries for common investigations (errors in the last hour, slow requests, etc.).
- Set up CloudWatch metric filters to create error count metrics from logs → alarm on error rate > 1%.
 
Pillar 3 — Distributed Tracing (X-Ray):
- Add X-Ray SDK to all services. Add X-Ray daemon sidecar to all task definitions.
- X-Ray service map automatically shows service dependencies and highlights latency/error hot spots.
- Trace requests end-to-end across all 50 services.
 
SLOs and alerting:
- Define SLOs for each service (e.g., p99 latency < 500ms, error rate < 0.1%).
- Create CloudWatch Composite Alarms that combine multiple signals.
- PagerDuty/OpsGenie integration for on-call routing.
 
Scenario 38
A specific ECS service is causing cascading failures. When it slows down, 5 other services that call it also slow down, then timeout, and eventually all 6 services are down. How do you prevent this?
Answer
This is the classic cascading failure / service avalanche problem. The solution is resilience patterns.
 
Pattern 1 — Circuit Breaker:
- Implement a circuit breaker in each calling service (Service A, B, C that call the problematic Service X).
- If Service X fails > threshold (e.g., >50% of calls in 10 seconds), the circuit 'opens' and callers immediately return a fallback response without calling Service X.
- Service X gets time to recover. After a timeout (e.g., 30 seconds), the circuit half-opens and tries one request. If successful, circuit closes.
- Libraries: resilience4j (Java), pybreaker (Python), opossum (Node.js), or use ECS Service Connect which has built-in outlier detection.
 
Pattern 2 — Timeout + Retry with Exponential Backoff:
- Set aggressive timeouts on all inter-service calls (e.g., 2-second timeout, not the default infinite wait).
- Retry with exponential backoff and jitter to prevent thundering herd on recovery.
 
Pattern 3 — Bulkhead:
- Isolate thread pools per downstream service. Service A uses a dedicated pool of 10 threads for Service X calls. If Service X is slow, only those 10 threads are blocked — the rest of Service A continues normally.
 
Pattern 4 — Graceful degradation:
- When Service X is unavailable, return cached data, default values, or a degraded response rather than an error. 'Best effort' rather than 'all or nothing'.
 
Infrastructure: Use App Mesh or Service Connect — both have built-in outlier detection and retry policies at the proxy level.
 
Scenario 39
Your ECS service handles millions of requests per day. You want to implement request tracing so you can investigate a specific customer complaint and trace their exact request through all 10 services it touches. How?
Answer
End-to-end request tracing with correlation IDs:
 
Step 1 — Generate and propagate a Trace ID:
- At the entry point (API Gateway or ALB), generate a unique trace ID (UUID) for each request.
- Include it in the response header (X-Trace-Id) so the customer can provide it in a support ticket.
- Pass the trace ID in all downstream service calls as a header (X-Trace-Id or the W3C traceparent header).
 
Step 2 — AWS X-Ray (automatic instrumentation):
- X-Ray automatically propagates trace context across services using the X-Amzn-Trace-Id header.
- Add X-Ray SDK to each of the 10 services. Add X-Ray daemon sidecar to each task definition.
- X-Ray stitches all segments together into a complete trace: Service 1 → Service 2 → ... → Service 10.
- Given a trace ID from the customer, search X-Ray: aws xray get-trace-summaries or use the X-Ray console to find the full trace.
 
Step 3 — Structured logging with the trace ID:
- Include the trace ID in every log line (JSON format: {trace_id: 'xxx', level: 'ERROR', message: '...'}).
- CloudWatch Logs Insights query: filter trace_id = 'customer-provided-id' | sort @timestamp asc — gives you every log line from every service for that request.
 
Step 4 — Correlation dashboard:
- Build a CloudWatch dashboard or use AWS Managed Grafana to search by trace ID across metrics, logs, and traces simultaneously.
- For high-value customers, store trace IDs in DynamoDB indexed by customer ID for quick lookup.
 
🌐 Multi-Region & Disaster Recovery Scenarios

Scenario 40
Your ECS-based e-commerce platform needs an RTO of 15 minutes and RPO of 5 minutes in case of a full AWS region failure. How do you design for this?
Answer
15-minute RTO / 5-minute RPO requires an active-passive warm standby architecture:
 
Infrastructure (IaC everything):
- Deploy identical ECS infrastructure in two regions (e.g., us-east-1 primary, us-west-2 DR) using Terraform modules.
- The DR region runs with scaled-down capacity (warm standby — 20-30% of production scale) to save cost.
- Spin up to full capacity in minutes using Auto Scaling.
 
Data replication (5-minute RPO):
- Aurora Global Database: Asynchronous replication with typically <1 second lag. Promote the DR replica to primary in <1 minute.
- DynamoDB Global Tables: Active-active, sub-second replication. No promotion needed.
- ElastiCache: Use cross-region backup restore or Redis Global Datastore.
- S3: Enable cross-region replication (CRR) with RTC (Replication Time Control) — 99.99% of objects replicated within 15 minutes.
 
Container images:
- Enable ECR cross-region replication so all images are available in the DR region without pull time.
 
Traffic routing (Route 53):
- Route 53 health checks on the primary ALB. Failover routing policy automatically switches to DR ALB if health checks fail.
- With a 30-second health check interval and 3 failures: failover triggers in ~90 seconds.
 
Failover runbook:
- Promote Aurora global database read replica to primary (CLI: aws rds failover-global-cluster).
- Scale up DR ECS services to full capacity.
- Route 53 health check auto-switches traffic.
- Total: ~10-15 minutes. Automate with a Lambda-triggered Step Functions state machine.
 
Scenario 41
An entire AWS Availability Zone goes down while your ECS service is running. How should your architecture automatically handle this without manual intervention?
Answer
AZ failure resilience should be automatic if the architecture is designed correctly:
 
Auto-recovery requirements:
1. Multi-AZ ECS task distribution: ECS Service must have subnets in at least 3 AZs configured. ECS automatically distributes tasks across AZs using spread placement.
2. ALB multi-AZ: The ALB must have listeners in all AZs. When an AZ fails, ALB automatically stops routing to unhealthy targets in that AZ.
3. Auto Scaling rebalancing (EC2): When an AZ fails, EC2 Auto Scaling detects the capacity imbalance and launches replacement instances in healthy AZs. ECS Cluster Auto Scaling then places tasks on new instances.
 
What happens during an AZ failure (automatic):
- Tasks in the failed AZ stop (ECS detects unhealthy tasks via health checks or the task simply stops).
- ALB health checks fail for targets in the failed AZ — traffic reroutes to healthy AZs automatically within 30-60 seconds.
- ECS Service scheduler detects running task count < desired count.
- New tasks are placed in remaining healthy AZs.
- If EC2: Cluster Auto Scaling adds instances in healthy AZs if needed to accommodate the extra tasks.
 
Architectural requirements for this to work automatically:
- Desired count: At least 3 tasks (1 per AZ minimum), or more to handle AZ loss without degradation.
- Spread placement strategy: availabilityZone — ensures even distribution.
- Stateless tasks: Tasks in failed AZ can simply be restarted. No lost state.
- Database: Aurora Multi-AZ or Global Tables — auto-failover without manual intervention.
 
Scenario 42
You need to test your ECS disaster recovery plan without impacting production. How do you run a DR drill?
Answer
DR drill methodology for ECS without production impact:
 
Preparation:
- Define the drill scope: full region failover simulation, single AZ failure, or specific service failure.
- Schedule during low-traffic period (Sunday 2 AM).
- Notify stakeholders and on-call teams.
 
Full region failover drill:
1. Scale up DR region: Run the ECS scale-up runbook in the DR region. Verify all ECS services reach desired count and pass health checks.
2. Verify data access: The DR region's ECS tasks should connect to the promoted DR database. Verify read and write operations work.
3. Test traffic routing: Temporarily update Route 53 to point to the DR ALB for a test domain (not production). Run synthetic load tests against the DR endpoint.
4. Measure RTO: Time from 'failover decision' to 'DR region serving traffic' — this is your actual RTO.
5. Measure RPO: Compare data in DR vs primary at the failover moment — this is your actual RPO.
 
Chaos Engineering approach:
- Use AWS Fault Injection Simulator (FIS) to simulate AZ failures, network disruptions, or task failures against non-production environments.
- FIS can target ECS tasks, EC2 instances, and network connectivity.
 
After the drill:
- Scale DR back to warm-standby capacity.
- Document actual RTO/RPO achieved vs targets.
- Identify gaps and create remediation tasks.
- Update runbooks with lessons learned.
- Schedule next drill in 3-6 months.
 
🏗️ Infrastructure & DevOps Scenarios

Scenario 43
A new developer on your team accidentally ran 'terraform destroy' on the production ECS cluster. How do you recover, and how do you prevent this from happening again?
Answer
Recovery and prevention plan:
 
Immediate recovery:
1. Check if Terraform state still has the resource definitions (state is the truth of what was deployed).
2. Re-run terraform apply from the same commit/version that was running in production.
3. ECS cluster, services, and task definitions are stateless infrastructure — they can be recreated from IaC in minutes.
4. The critical data (databases, S3) should be unaffected by ECS cluster destruction.
5. Monitor: verify tasks start, pass health checks, and traffic is restored.
 
RTO estimate: With well-written Terraform, ECS cluster recreation takes 5-15 minutes. Add 5-10 minutes for tasks to start and become healthy.
 
Prevention — multiple layers:
1. Remove developer IAM permissions for production: Developers should not have permission to run Terraform against production directly. Only CI/CD pipelines should have prod deployment permissions (via IAM role assumed by the pipeline).
2. Terraform workspaces + separate state files per environment: Destroy in dev never affects prod state.
3. Require PR approval for production Terraform changes: No direct applies. All changes go through peer review.
4. Enable AWS Config drift detection: Alert when ECS resources are deleted outside of the approved pipeline.
5. ECS cluster deletion protection: Enable terminationProtection on ECS services (prevents deletion of running services).
6. Restrict terraform destroy: Use Sentinel policies (Terraform Enterprise) or CI gates to block destroy commands in production workspaces.
7. Break-glass procedure: Document how to get emergency prod access for real incidents, with mandatory audit trail.
 
Scenario 44
Your team is deploying ECS infrastructure manually through the console. Different environments have drifted apart — dev looks nothing like production. How do you bring everything under IaC control?
Answer
IaC adoption strategy for existing ECS environments:
 
Step 1 — Import existing infrastructure:
- Use Terraform's import command or AWS CloudFormation's import feature to bring existing resources under IaC control.
- For ECS specifically: import the cluster, services, task definitions, and supporting resources (IAM roles, security groups, ALBs).
- Alternatively, use former2 or AWS CloudFormation IaC Generator to auto-generate Terraform/CFN from existing resources.
 
Step 2 — Establish a golden module:
- Create a Terraform module for a 'standard ECS service' that encodes your best practices (awsvpc networking, least-privilege task role, CloudWatch logging, auto scaling).
- All environments use the same module with different variable values.
 
Step 3 — Enforce parity via IaC:
- Define all environment differences in tfvars files (dev.tfvars, staging.tfvars, prod.tfvars).
- The module code is identical; only variable values differ.
- Use CI/CD to apply Terraform on every merge to main — no manual console changes allowed.
 
Step 4 — Detect and prevent drift:
- Run terraform plan in CI on a schedule (nightly). If plan shows changes (drift), alert the team.
- AWS Config: enable 'ecs-service-configured-logging' and other managed rules to detect misconfigured ECS resources.
- Consider HashiCorp Sentinel or Open Policy Agent for policy-as-code guardrails.
 
Change management: Announce that console changes will be detected and reverted automatically after a cutover date. Give teams 2-4 weeks to move changes to IaC.
 
Scenario 45
You need to implement a GitOps workflow for ECS where any change merged to the main branch automatically deploys to production, with approvals for critical services. How do you design this?
Answer
GitOps for ECS — code as the source of truth for deployments:
 
Repository structure:
- Application code repo: Source code + Dockerfile. Merges to main trigger image builds.
- Infrastructure/config repo: ECS task definitions, Terraform, service configs. Merges trigger deployments.
 
Pipeline design (CodePipeline or GitHub Actions):
Stage 1 — Build:
- On merge to main: CodeBuild builds Docker image, pushes to ECR with commit SHA as tag.
- Automatically updates the task definition JSON in the config repo with the new image tag (automated PR or direct commit).
 
Stage 2 — Deploy to staging (automatic):
- Config repo change triggers automatic deployment to staging ECS cluster.
- Run automated integration tests. If tests pass, proceed.
 
Stage 3 — Deploy to production:
- For non-critical services: Automatic deployment after staging tests pass.
- For critical services: Manual approval gate in CodePipeline (an approver reviews the staging results and approves via console or Slack integration).
- After approval: CodePipeline deploys to production ECS service.
 
Rollback mechanism:
- 'Revert' the commit in the config repo. The pipeline re-deploys the previous task definition.
- For emergency rollback: pipeline can be triggered manually with a previous commit SHA.
 
Audit trail:
- Every production deployment is traceable to a Git commit, who approved it, and what changed.
- CloudTrail records the actual ECS UpdateService API calls with timestamps.
 
Scenario 46
Your company needs to pass a SOC 2 Type II audit. The auditors want evidence that all production ECS deployments are approved, traceable, and that no unauthorized changes occur. How do you demonstrate this?
Answer
SOC 2 evidence collection for ECS deployments:
 
Change management controls:
1. All code changes require peer review: GitHub/GitLab branch protection rules — require 2 approvals for merges to main. Export PR approval history as evidence.
2. Deployment approval gates: CodePipeline manual approval steps for production. Auditors can see who approved each deployment and when in the pipeline execution history.
3. Immutable audit log: All deployments are logged in CloudTrail (ECS UpdateService, RegisterTaskDefinition events). CloudTrail with Object Lock makes logs tamper-proof.
 
Access controls:
4. Only CI/CD has prod access: IAM policy evidence showing no developer has direct ECS UpdateService permission in production. Only the CodePipeline IAM role can modify ECS services.
5. Break-glass access is logged: Any emergency console access is via a break-glass IAM role that triggers a CloudWatch alarm and is recorded in CloudTrail.
 
Configuration management:
6. All infrastructure is in IaC: Show the Git history of task definitions and Terraform. No console-only changes.
7. Drift detection: AWS Config evidence showing no unauthorized infrastructure changes occurred during the audit period.
 
Monitoring and alerting:
8. CloudWatch alarms with evidence: Show alarm configurations and any triggered alerts during the audit period.
9. Vulnerability scanning: ECR scan results for all production images — evidence of security review before deployment.
 
Deliverable to auditors: A deployment evidence package per change: PR link with approvals, CodePipeline execution log, CloudTrail event, and the task definition diff.
 
🔄 Advanced Operations Scenarios

Scenario 47
You need to run a one-time database migration as part of an ECS deployment. The migration must complete before the new app version starts taking traffic. How do you implement this?
Answer
Database migration as a deployment gate — the init container / migration task pattern:
 
Option A — ECS RunTask as a migration step in CI/CD:
- Before deploying the new ECS service version, the CI/CD pipeline runs a one-off ECS task (using RunTask API) with the migration container.
- The pipeline waits for the task to complete successfully (exit code 0).
- If migration fails (non-zero exit code), the pipeline aborts and does not deploy the new app version.
- Migration task has a separate IAM role with DB admin permissions (not given to the app's task role).
 
Implementation in CodePipeline:
- Add a CodeBuild action before the ECS deploy action that: calls aws ecs run-task --task-definition db-migration, waits for task completion using aws ecs wait tasks-stopped, checks exit code — if non-zero, fail the pipeline.
 
Option B — Init container in the task definition:
- Add a migration container to the Task Definition with dependsOn the app container with condition: SUCCESS.
- The migration container runs first, applies migrations, then exits.
- ECS only starts the app container after the migration container exits successfully.
- Risk: This runs on every task launch (not just deployments). Use a migration tool with idempotent migrations (Flyway, Liquibase) that skip already-applied migrations.
 
Safety practices:
- Expand-contract (parallel change) pattern: New columns are added as nullable. Old code ignores them. New code uses them. Migration never breaks old code.
- Never run DROP or destructive ALTER in deployment migrations — do that in a separate follow-up after the new version is stable.
- Backup the database before running migrations.
 
Scenario 48
Your ECS service needs to process exactly-once semantics for financial transactions. How do you ensure idempotency in a distributed ECS environment where tasks can restart at any time?
Answer
Exactly-once processing in distributed ECS requires idempotency by design — you can't prevent retries, but you can make retries safe.
 
Core principle: At-least-once delivery + idempotent processing = effectively exactly-once.
 
Implementation:
1. Idempotency keys: Every transaction request must include a unique client-generated idempotency key (UUID).
2. Idempotency table: Use a DynamoDB table to track processed idempotency keys. When a request arrives:
  a. Try to INSERT the idempotency key with a conditional write (fails if key already exists).
  b. If INSERT succeeds: process the transaction, store the result in the idempotency table, commit.
  c. If INSERT fails (key exists): return the stored result without re-processing.
3. This ensures even if the ECS task restarts mid-processing and the client retries, the transaction is applied only once.
 
Database transactions:
- Use database transactions (BEGIN/COMMIT) to atomically update both the business data and the idempotency key table.
- If the task crashes between processing and committing, the transaction rolls back, and the idempotency key is not stored — the retry will re-process correctly.
 
SQS exactly-once:
- Use SQS FIFO queues with message deduplication IDs — SQS deduplicates messages with the same ID within a 5-minute window.
- Set SQS visibility timeout longer than max processing time to prevent concurrent processing.
 
Idempotency key TTL: Expire idempotency keys in DynamoDB after 24-48 hours to avoid unbounded table growth.
 
Scenario 49
You manage an ECS-based SaaS platform. Each customer must be completely isolated from others, but you can't afford to run a separate cluster per customer. How do you implement multi-tenancy?
Answer
Multi-tenant ECS SaaS architecture without per-customer clusters:
 
Isolation approaches by strength (strongest to least):
 
Level 1 — Task-level isolation (strong):
- Each customer gets dedicated ECS tasks with their own task IAM role and security group.
- Each customer's tasks can only access their own DynamoDB partition, S3 prefix, or RDS schema.
- Tasks for different customers never share a process — strong isolation.
- Cost: Higher task count. Use for sensitive/enterprise customers.
 
Level 2 — Service-level isolation (medium):
- Each customer gets a dedicated ECS service (shared cluster). Services have separate task roles, security groups, and CloudWatch log groups.
- Scale services independently per customer based on their usage tier.
- Good balance of isolation and cost for mid-market customers.
 
Level 3 — Application-level multi-tenancy (lightest):
- Shared ECS tasks serve all customers. Isolation enforced in application code (customer ID in every DB query, S3 key prefix per customer).
- Lowest cost. Suitable for SMB customers with lower isolation requirements.
- Risk: Application code bugs can cause cross-tenant data leakage — requires rigorous testing and row-level security in the DB.
 
Data isolation (all levels):
- DynamoDB: Tenant ID as partition key + IAM condition keys (dynamodb:LeadingKeys) to restrict access.
- RDS: Schema-per-tenant or row-level security with tenant_id column and RLS policies.
- S3: Prefix-per-tenant (s3://bucket/tenant-id/) with bucket policies enforcing prefix access.
 
Routing: API Gateway with Lambda authorizer to extract tenant ID from JWT, route to the appropriate ECS service endpoint.
 
Scenario 50
Your CTO asks: 'Should we move our entire ECS workload to EKS?' You run 80 services, have been on ECS for 4 years, and the team is comfortable with ECS. Make the case for and against.
Answer
Structured analysis for ECS vs EKS migration decision:
 
Case FOR staying on ECS (your current situation):
- Team expertise: 4 years of ECS knowledge is valuable. EKS has a steep learning curve — plan for 3-6 months of reduced velocity during migration.
- Operational simplicity: ECS control plane is fully managed by AWS with zero configuration. EKS requires managing node groups, Kubernetes upgrades, add-ons (CoreDNS, vpc-cni, kube-proxy), and cluster autoscaler.
- Native AWS integration: ECS integrates more seamlessly with IAM, ALB, CloudWatch, and Service Discovery. EKS integrations require additional configuration.
- Cost: EKS charges $0.10/hour per cluster ($876/year) plus more complex node management. ECS has no cluster fee.
- Migration cost: Migrating 80 services is a 12-18 month project minimum. Opportunity cost is enormous.
 
Case FOR migrating to EKS:
- Ecosystem: Kubernetes has a vastly richer ecosystem (Helm, Argo CD, KEDA, Istio, Karpenter, Velero).
- Portability: EKS workloads can move to GKE/AKS or on-premises with minimal changes. ECS is AWS-only.
- Advanced features: Custom resource definitions (CRDs), operators, and flexible scheduling (affinity/anti-affinity) are more powerful in K8s.
- Talent market: More engineers know Kubernetes than ECS. Hiring is easier.
- Industry trend: Kubernetes is the de facto standard — vendor support, training, and tooling are centered on K8s.
 
Recommendation:
- If the team is productive and services are working well on ECS: stay on ECS. The migration cost outweighs the benefits for a team that doesn't need K8s-specific features.
- If you're hiring aggressively, need multi-cloud portability, or require advanced K8s operators: plan a phased EKS migration over 18-24 months, starting with new services on EKS before migrating existing ones.
- Hybrid: Run new services on EKS, maintain existing services on ECS until natural re-platforming opportunities arise.
