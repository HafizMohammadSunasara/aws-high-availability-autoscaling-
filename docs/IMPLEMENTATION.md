# Implementation — Reproduction runbook

**Purpose:** Teach a reader to rebuild a similar AWS architecture from scratch and validate each dependency.

**This is a guided reproduction, NOT a log of commands collected during the August 2026 project.** Always use your own VPC IDs, subnet IDs, security groups, AMIs and resource names. Do not publish secrets.

## Before starting: cost and IAM guardrails

- Confirm your AWS region and account. Running EC2 instances, ALBs, CloudWatch, NAT gateways and data transfer may incur charges.
- Set a budget alert, choose the smallest suitable instance, and keep the ASG **minimum and maximum deliberately low for a disposable lab**. The project's reported max=10 is **not** a recommendation to launch ten instances.
- Use least-privilege IAM for your own workload. Never embed AWS credentials in EC2 user data or a README.
- Remove all resources at the end of the learning session, and verify deletion.

## Stage 1 — Network groundwork

1. Select region `us-east-1`.
2. Use a suitable VPC with subnets in *two* distinct AZs (for example `us-east-1a` and `us-east-1b`).
3. For an internet-facing ALB, configure appropriate subnet internet access.
4. Create an **ALB security group** allowing intended inbound HTTP(S) from approved clients.
5. Create an **EC2 security group** that permits the application port **from the ALB security group**, not the world.
6. Decide whether backend instances need outbound access for package installation/updates and how that egress is supplied (for private subnets, NAT or appropriate endpoints may be relevant).

**Gate:** Record VPC, subnet and security-group relationships before creating compute resources.

## Stage 2 — Application bootstrapping

1. Choose an appropriate, region-specific AMI and an instance type such as `t3.micro` *where available and appropriate*.
2. Prepare a web app or small HTTP endpoint that returns 200 for `/health`.
3. Configure launch-time user data to install/start the service, or prebuild a known-good AMI.
4. Test a single instance in isolation first, including reboot behavior.
5. Verify a successful HTTP response from the instance in an allowed network context.

**Common failure:** Security groups are correct, but the web app only listens on `127.0.0.1` instead of its instance interface. Ensure it binds to the intended interface/port.

## Stage 3 — Launch Template

EC2 console → Launch Templates → Create launch template:

- AMI: your tested build
- Instance type: chosen deliberately
- Security group: backend EC2 SG
- User data: tested bootstrapping script (optional if baked AMI)
- IAM instance profile: only when required; prefer short-lived instance-role credentials
- Tags: identify the lab and cost owner

**Gate:** Confirm the Launch Template version and its fields; avoid silently using the wrong version.

## Stage 4 — Load balancer and target group

EC2 console → Target Groups:

1. Create an **instance-type target group** (if using EC2 instance targets).
2. Configure HTTP/HTTPS protocol and intended port.
3. Set health check path `/health` (or your verified equivalent).
4. Choose a valid matcher, success thresholds, interval and timeout.

EC2 console → Load Balancers:

1. Create an Application Load Balancer.
2. Select subnets from at least two intended AZs.
3. Choose internet-facing or internal deliberately.
4. Attach ALB security group.
5. Configure a listener forwarding to the target group.
6. For publicly exposed real services, prefer HTTPS with a certificate and security-reviewed listener configuration.

**Gate:** ALB and target group exist. Do not confuse their IDs/names with the ASG.

## Stage 5 — Auto Scaling Group and attachment

EC2 console → Auto Scaling Groups → Create:

1. Select the Launch Template and version.
2. Specify the VPC and two distinct AZ subnets.
3. Set **minimum, desired and maximum** to safe values for this lab; e.g. 1 / 2 / 3 *as a lab example*, not the claimed historical setting.
4. Select **Attach to an existing load balancer → existing ALB target group**.
5. Consider enabling **Elastic Load Balancing health checks** if application-level health should trigger replacement.
6. Select suitable health check grace period and instance warmup.
7. Create the ASG and observe instance lifecycle until `InService`.

**Gate:** ASG instances appear in target group with at least one **healthy** target. Ideally demonstrate one healthy target in each AZ.

## Stage 6 — Target tracking policy

ASG → Automatic scaling → Create dynamic scaling policy:

1. Use **Target tracking scaling**.
2. Choose metric such as `ASGAverageCPUUtilization` for a CPU-bound demonstration; consider `ALBRequestCountPerTarget` for request-based applications.
3. Pick a target appropriate for load characteristics (e.g. **50% CPU is an illustrative example, not an archived project value**).
4. Specify instance warmup accurately.
5. Ensure group max capacity exceeds its desired capacity if you want to observe scale-out.

**Gate:** Policy exists and associated CloudWatch alarms are managed by the service. No scaling claim until activity history shows it.

## Stage 7 — Read-only CLI inspections

Run from a configured AWS CLI environment. **Replace all placeholders**; these are generic reference commands, not output from this project.

```bash
aws autoscaling describe-auto-scaling-groups \
  --region us-east-1 \
  --auto-scaling-group-names YOUR_ASG_NAME \
  --query 'AutoScalingGroups[*].{Name:AutoScalingGroupName,Min:MinSize,Desired:DesiredCapacity,Max:MaxSize,Instances:Instances[*].[InstanceId,LifecycleState,HealthStatus]}'

aws autoscaling describe-policies \
  --region us-east-1 \
  --auto-scaling-group-name YOUR_ASG_NAME

aws autoscaling describe-scaling-activities \
  --region us-east-1 \
  --auto-scaling-group-name YOUR_ASG_NAME \
  --max-items 20

aws elbv2 describe-target-health \
  --region us-east-1 \
  --target-group-arn YOUR_TARGET_GROUP_ARN

aws elbv2 describe-load-balancers \
  --region us-east-1 \
  --names YOUR_ALB_NAME
```

Read output carefully: `Healthy` target health and `InService` ASG lifecycle are **different fields**. Capture timestamps and settings but redact account identifiers and any confidential values before publication.

## Stage 8 — Real validation experiments

### Test A: User traffic

Call the ALB DNS endpoint with `curl -I`, record HTTP status, target health and timestamps. This verifies client-facing application behavior rather than just AWS console status.

### Test B: Scaling behavior

Use a **controlled, authorized** load generator in the test environment. Record baseline, load start, CPU/ALB metric, desired capacity change, EC2 startup and target health. Do not claim 10 instances launched just because `Max=10`.

### Test C: Single-instance recovery

In a non-production test group with adequate spare capacity, intentionally stop/terminate a **test** instance in a controlled manner. Observe target deregistration, replacement and end-user error rate. Do not perform on real customer workloads.

### Test D: Multi-AZ behavior

Demonstrate target distribution across AZs, verify routing if one AZ's **test targets** are marked unhealthy, and record the observed error/recovery window. A real AZ failure is not required for a safe lab drill.

## Stage 9 — Clean up and validate cost

Typical dependency-aware teardown:

1. Stop load generator and remove test traffic.
2. Scale down or delete the ASG, allowing instances to terminate.
3. Delete ALB listeners / ALB and target groups after detaching.
4. Delete unneeded Launch Template versions, role/test configuration, stray volumes, log groups and security groups when no longer used.
5. Check billing/cost tools and verify running instances and ALBs are gone.

**Outcome:** A reproducible procedure for a *future rerun*. Without fresh console captures, this document is a guide, not a certificate of historical execution.

## References

- https://docs.aws.amazon.com/autoscaling/ec2/userguide/attach-load-balancer-asg.html
- https://docs.aws.amazon.com/autoscaling/ec2/userguide/policy_creating.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html
