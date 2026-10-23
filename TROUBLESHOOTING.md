# Troubleshooting — Practical incident decisions

## 1. ALB exists but browser gets `503 Service Unavailable`

**Check:** Are any registered targets healthy? Verify listener forward action, correct target group, health check path and status codes. `503` is often consistent with no available targets, though always inspect ALB logs/metrics and specific response context.

**Verify:** Target health reason codes; curl backend locally or from an approved network path.

## 2. EC2 reports `Running` but ALB marks it unhealthy

**Likely reasons:** Web process not started, wrong service port, local firewall rules, restrictive EC2 security group, unhealthy `/health`, health check success matcher mismatch or bootstrap errors.

**Procedure:** EC2 status → process logs → local HTTP response → EC2 SG/NACL → target group health reason.

**Lesson:** infrastructure health is not application readiness.

## 3. ASG creates instances but does not replace unhealthy application targets

**Check:** Was additional **Elastic Load Balancing health checks** enabled on the ASG? By default EC2 health checks and ELB health checks are not interchangeable. Also check grace period and warmup settings.

## 4. Scaling never occurs

- Is the metric being published and is the correct policy attached?
- Is the target value actually breached long enough?
- Is `DesiredCapacity` already equal to `MaxSize`?
- Is scaling suspended?
- Are new instances still warming up?
- Are CloudWatch alarm states reflecting recent data?
- Is the chosen metric suitable for the workload (e.g. CPU for CPU-bound load versus request count for HTTP-bound load)?

## 5. Repeated launch/terminate loop

**Look for:** incorrect AMI, failed user-data scripts, insufficient IAM permissions, target health failures, port mismatch, app warmup longer than health grace period, quotas / capacity limitations.

**Inspect:** ASG scaling activity causes and instance system/service logs. Do not solve this simply by giving the server overly broad IAM permissions.

## 6. Scaling increased, but latency barely improved

Possible causes: central database or third-party API bottleneck, session state tied to one instance, CPU is not dominant bottleneck, targets have not passed health checks, connection/queue limits, traffic is uneven or caching behavior changed.

## 7. Instance terminated during load, some users saw errors

This can still happen in an ASG + ALB architecture. Investigate connection draining/deregistration delay, target health interval, application readiness, retries, in-flight sessions, spare capacity and response-code metrics. Availability must be measured at the **user-facing request level**.

## 8. One AZ fails but the service does not hold steady

Check remaining AZ capacity, AZ-specific subnet sizing, health target distribution, instance type capacity, quotas, dependencies pinned to the impaired AZ, database resilience and load assumptions. Two AZs on a console page do not guarantee failover resilience.

## 9. Cost is unexpectedly high

ALB hourly and capacity charges, EC2 usage, EBS, NAT, data transfer, CloudWatch custom metrics, unused public IPv4 allocations and always-on baseline capacity can dominate a small lab budget. Verify via AWS billing tools and delete test resources on completion.

## Incident note template

```text
Incident ID:
Date/time (UTC):
Observed user symptom:
ALB status / target health:
ASG min / desired / max:
CloudWatch metric and alarm:
ASG scaling activity reason:
Root cause hypothesis:
Change made:
Evidence after change:
Remaining uncertainty:
```

## Official references

- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html
- https://docs.aws.amazon.com/autoscaling/ec2/userguide/health-checks-overview.html
- https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html
