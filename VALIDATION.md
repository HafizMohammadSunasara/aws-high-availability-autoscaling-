# Validation Matrix — What is known, and what remains to be proven

> **Evidence policy:** Do not convert described intentions into verified outcomes. This repository currently contains a LinkedIn write-up with an embedded ASG thumbnail, **not** a set of complete AWS console exports.

## Available source

**File:** [`evidence/01-linkedin-project-record.png`](../evidence/01-linkedin-project-record.png)

The submitted LinkedIn project entry, titled **“Highly Available & Auto-Scalable Web Architecture on AWS”**, reports a build in **August 2026** with:

- EC2 Launch Template standardization using `t3.micro` instances.
- Application Load Balancer in `us-east-1a` and `us-east-1b`.
- ASG association with ALB target groups.
- Target tracking based on CloudWatch and a reported capacity ceiling of 10 instances.

These are author-reported statements, not independently authenticated live resource exports.

## Claim-by-claim matrix

| Statement | Source strength | Safe description for README |
|---|---|---|
| A prior AWS autoscaling project was recorded | Screenshot of LinkedIn project record | **Documented** |
| The record includes an ASG console thumbnail | Visible in supplied screenshot | **Visible**, details too small to audit |
| Launch Template was configured with t3.micro | LinkedIn description | **Reported** |
| ALB spanned two AZs | LinkedIn description | **Reported**, no independent target-group details |
| ASG integrates with target group | LinkedIn description | **Reported**, not independently replayed |
| Max configured up to 10 | LinkedIn description | **Reported limit**, not 10 running instances |
| Actual scale-out occurred under measured load | No scaling activity / metric captures | **Not independently verified** |
| Failed instance was automatically replaced | No failure experiment record | **Not independently verified** |
| Users experienced zero downtime | No uptime/latency/error logs | **Not verified; do not claim** |
| Money was saved | No before/after AWS cost analysis | **Not measured** |

## What would constitute stronger proof?

| Test | Evidence to capture | Pass signal |
|---|---|---|
| ALB integration | Target group + health tab | Correct ASG targets are healthy |
| Multi-AZ placement | ASG instances with `AvailabilityZone` | Instances in at least two AZs |
| Web functionality | `curl` output / HTTP status | Repeated successful client requests |
| Real scale-out | Time-stamped scaling activities and CloudWatch graph | Desired capacity rises and instances become healthy |
| Real scale-in | Activity history + metric decrease | Desired capacity decreases without unexpected errors |
| Single-instance failure | Failure timestamp + target status + replacement activity | Unhealthy target drained/replaced; client behavior recorded |
| Cost effect | AWS Cost Explorer / billing before/after | Observed cost change with comparable workload |

## Minimum read-only AWS CLI evidence bundle (for a future recreation)

```bash
aws autoscaling describe-auto-scaling-groups --region us-east-1 --auto-scaling-group-names YOUR_ASG_NAME
aws autoscaling describe-scaling-activities --region us-east-1 --auto-scaling-group-name YOUR_ASG_NAME
aws autoscaling describe-policies --region us-east-1 --auto-scaling-group-name YOUR_ASG_NAME
aws elbv2 describe-target-health --region us-east-1 --target-group-arn YOUR_TARGET_GROUP_ARN
```

**Never commit:** AWS keys, secrets, IAM tokens, session credentials or private production logs. Review screenshots for account IDs, public addresses and other exposure risks.
