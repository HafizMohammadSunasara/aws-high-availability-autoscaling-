# Engineering Deep Dive — AWS High Availability, Load Balancing & EC2 Auto Scaling

**A zero-to-advanced field guide for understanding why a multi-instance AWS web architecture can remain usable during changing demand—and why having an Auto Scaling Group does not automatically make a system reliable.**

**Case study:** Original August 2026 AWS project write-up describing an EC2 Launch Template, Application Load Balancer, an Auto Scaling Group across `us-east-1a` and `us-east-1b`, and CloudWatch-driven target tracking. `t3.micro` is the reported instance type; “up to 10 instances” is a *reported maximum configuration*, **not evidence of ten deployed servers**.

> **Mental model:** ALB decides **where a request goes**. Auto Scaling decides **how much compute capacity should exist**. A Launch Template decides **what a newly launched EC2 instance looks like**. CloudWatch provides **measurements** used for decisions. None of these by itself proves availability or end-user success.

## Reading map

| Level | Sections | You should be able to explain |
|---|---|---|
| Beginner | 1–7 | VPC, EC2, ALB, ASG, target groups, AZs, health checks |
| Intermediate | 8–18 | Networking, bootstrap, scaling, traffic, capacity, validation |
| Advanced | 19–29 | Failure modes, warmup, security, observability, cost, trade-offs |
| Interview-ready | 30–33 | Scenarios, checks, design review, technical questions |

---

## 1 — The actual problem: one server is not an availability strategy

Suppose a small business website runs on one EC2 instance. It works while that machine remains healthy and the traffic fits into its compute and memory limits. The weaknesses are obvious: if the host fails, the application may disappear; if a sudden burst of requests arrives, the server can saturate; and if you deploy a new version incorrectly, every request can fail at once.

You could manually create three more EC2 instances. That may increase capacity, but now you face new questions: How do clients choose a server? Which servers are healthy? How do you discover them? What happens when one is replaced? Should you always pay for all four? Who removes old targets from service?

The AWS architecture in this case study explores the corresponding building blocks: **ALB + target group + EC2 Auto Scaling Group + Launch Template + CloudWatch metrics**. Those components solve different pieces of the problem, so we must study them separately before connecting them.

**Key distinction:** *High availability* is a **service outcome** measured from the user's perspective. An ASG or ALB is only a **mechanism that may help achieve that outcome**.

## 2 — Six essential networking terms before touching AWS

**VPC:** A logically isolated IP address space. Think of it as a private cloud network boundary. It contains subnets, route tables and controls; it is not itself a single EC2 instance.

**Subnet:** A range of IPs inside a VPC tied to one Availability Zone. An individual subnet does not span AZs. A multi-AZ application requires subnets in separate AZs.

**Availability Zone (AZ):** An independently designed failure domain within an AWS Region. Spreading workload across AZs can reduce exposure to some hardware or infrastructure failures; the application can still fail because of a common database, DNS issue, bad software release or insufficient remaining capacity.

**Region:** A geographical area grouping AZs. This project reports `us-east-1` (N. Virginia). In this region the original description names `us-east-1a` and `us-east-1b`.

**Route table:** Determines where packets travel next based on their destination. A public subnet typically has an internet-gateway route. A private subnet often does not allow unsolicited inbound internet connections. Route tables alone do not guarantee application access.

**Security group:** A stateful virtual firewall attached to resources such as EC2 and ALB. ALB SG inbound can allow approved client HTTP(S); EC2 SG inbound can allow only the ALB SG on the app port. Avoid making every instance publicly reachable merely because an ALB sits in front.

**Micro-exercise:** Draw a VPC with two AZ-specific subnets, write one CIDR per subnet, then trace the route a public client request would take to a backend instance through an ALB.

## 3 — What EC2 does—and what it doesn't

Amazon EC2 runs virtual machines. An instance has an operating system, virtual CPU, memory, network interface, storage and lifecycle state. A web application may be installed on the instance directly, inside a container or via a process manager.

`Running` means the EC2 instance is running. It does not mean your app returns `HTTP 200`, the database is reachable or that requests are completing within a reasonable latency. Your application could be misconfigured while EC2 status checks still pass.

The original project uses `t3.micro` in the write-up. This is an instance class choice, not an optimization benchmark. Burstable `t3` performance depends on CPU credits and workload patterns, so “my group has more instances” does not answer whether it has enough sustained throughput.

**Never infer service reliability from an EC2 instance state alone.** Look for an actual web response and health metrics.

## 4 — Launch Template: a recipe, not a running application

An EC2 Launch Template contains *configuration used to launch instances*. Depending on how you build it, this may include:

- AMI ID (a specific machine image),
- EC2 instance type,
- subnet/ENI settings as appropriate,
- security groups,
- IAM instance profile (when required),
- SSH key choice or safer management access design,
- block-device mappings,
- tags and optional bootstrap user data.

A template can have versions. The ASG references a template and version. Changing the default version does not magically update every instance already running. Updating existing machines generally requires a deliberate instance replacement/refresh/deployment strategy.

**Example:** Template v1 uses an old image with Nginx. Template v2 uses a newer image. If the ASG still references v1, it can keep launching the old software even after you have created v2.

**Do not store access keys in user data.** Use a limited IAM role attached through an instance profile when AWS API access is required.

## 5 — Bootstrapping: the hidden point of failure

A machine may launch successfully but fail to serve traffic because the web server was never installed, the app listens only on `127.0.0.1`, an S3 download failed, or packages cannot be reached from a private subnet.

A minimal reproducible bootstrap sequence should establish preconditions, install required packages, configure the service, start it under a supervisor, verify that it listens on the intended interface and port, and log failures.

**Lesson:** If bootstrap takes 90 seconds, an ASG scaling event cannot produce a *ready* web server instantly. The readiness process must be integrated with ALB health checks and reasonable health-check grace periods.

## 6 — Application Load Balancer: the HTTP traffic director

An **Application Load Balancer** operates at layer 7 for HTTP/HTTPS-oriented traffic. Its listeners accept connections on configured ports and evaluate rules to choose target groups. ALB is not just a DNS alias for all EC2 instances—it performs actual request forwarding, target checks and connection handling.

A typical production internet-facing design has the ALB enabled in at least two AZs and app instances placed behind it. Whether those app instances are in public or private subnets depends on the chosen design. A common hardened approach uses **private app subnets** and allows backend inbound traffic only from the ALB security group.

The ALB is a service that has its own cost and scaling behavior. It is *not free because the Auto Scaling Group scales in*. And adding an ALB does not remove single points of failure in your own database or external dependencies.

## 7 — Target groups and health checks: more than a list of IPs

A **target group** defines the pool of destination servers and the health check behavior. For an EC2 instance target type, each registered instance must answer appropriately at the chosen protocol/port/health path to become healthy.

Imagine the health check path is `/health` and expects status `200`. An instance can be running yet **unhealthy** if:

- no server is listening on that port;
- the port is blocked by security groups/NACLs/local firewall;
- `/health` returns 500;
- startup takes longer than your thresholds;
- app readiness depends on a database that is unavailable.

**Readiness is not just liveness.** `/health` might report “process is alive” while `/ready` means “can serve real user requests.” Carefully decide which signal the load balancer should depend upon.

**Nuance:** ALB target-group documentation includes special **fail-open behavior** under conditions where every registered target is unhealthy. Do not reduce the behavior to an unconditional claim that the ALB *never* forwards to an unhealthy target; understand target health and observed responses in context.

## 8 — Auto Scaling Group: keeping the requested capacity alive

An **ASG** manages instances from a Launch Template (or other supported configuration) and aims to maintain a requested number of healthy compute instances across selected subnets/AZs. It can add and remove capacity as policies ask for changes.

Three numbers matter:

| Setting | Practical meaning |
|---|---|
| `MinSize` | Lowest ordinary allowed group size |
| `DesiredCapacity` | Number of instances the group currently aims to have |
| `MaxSize` | Highest ordinary capacity the group should scale to |

**Example only (not historical settings):** `min=1`, `desired=2`, `max=3`. Under load, a policy might increase desired capacity to 3. If desired is already 3, the policy cannot scale out normally because max has been reached. At night, a policy may reduce desired toward 1. One instance is not enough to demonstrate persistent two-AZ redundancy; a cost-minimum is a design trade-off.

The original project write-up says **configured to scale up to 10 instances**. That means a *reported ceiling*—not ten instances observed under traffic. A credible validation requires ASG instance listings, scaling activity history and related metric timestamps.

## 9 — ALB and ASG cooperation: who registers targets?

For ALB integration, the ASG is attached to an **ALB target group**. As the ASG launches instances, it registers them; when instances leave service, it deregisters them from the target group. The ALB routes based on its own target-health observations.

A common misconception is “attach the ALB to the ASG.” In the actual AWS configuration, for an Application Load Balancer you attach the **target group ARN** to the ASG. The listener on the ALB forwards to that target group. These are related, but they are separate resources.

Troubleshooting order:

1. Does the ALB listener forward to the expected target group?
2. Is that target group attached to the ASG?
3. Did the ASG launch instances?
4. Are the instances registered to the target group?
5. Are any targets healthy and serving actual user requests?

A single incorrect port or health path can make every layer appear broken even when EC2 instances are running.

## 10 — EC2 checks versus ALB checks versus ASG decisions

EC2 status checks focus on instance/system-level health. ALB target health checks focus on the HTTP/TCP-facing service you configured. ASG replacement is based on which health-check types are enabled for the group.

**Critical operational nuance:** An ASG does **not automatically** use ALB target health as a replacement trigger simply because a target group is attached. You should explicitly configure Elastic Load Balancing health checks if you intend the group to replace instances that the ALB identifies as unhealthy.

**Scenario:** Your Nginx service stops, but EC2 remains Running and passes system checks. ALB stops considering that target healthy. If ASG is configured only for EC2 checks, it may **not replace the instance** based solely on the failing HTTP health check.

That difference is a frequent interview question and a real root cause of “self-healing not working.”

## 11 — Target tracking scaling: the thermostat analogy

A target tracking policy attempts to keep a metric near a configured target by changing desired capacity. AWS describes it like a thermostat. It does not mean “launch one server for every new request.”

**Hypothetical example:** With `ASGAverageCPUUtilization` target 50%, sustained load lifts group-average CPU above target, so scaling may increase capacity. As more instances share demand, the average may decrease, and the policy can later reduce excess capacity subject to safeguards.

Target tracking policies also support ALB request count per target where applicable. Choosing the wrong metric can cause scaling that is too slow, too noisy or unrelated to actual application pain.

**Never hardcode 50% as “what was used in the original project.”** The screenshot does not show the actual target value. Use it only as an explanatory model.

AWS manages the associated CloudWatch alarms for target tracking; manual editing of those policy-managed alarms is discouraged.

## 12 — Choosing the correct metric

**CPU utilization** is useful for CPU-bound work such as calculations or compression. It may be misleading for a system waiting on a database while using little CPU.

**ALB request count per target** is appropriate when request rate is connected to required instance capacity, although response time and processing cost per request may vary.

**Memory utilization** often needs a custom metric because EC2's basic host telemetry doesn't automatically expose every OS/application memory measurement as a standard EC2 metric.

**Queue length / backlog** is meaningful for asynchronous workers. But a simple web ALB/ASG architecture doesn't automatically include SQS or a worker queue.

**Metric checklist:** Is it measurable? Does it respond reliably to load? Does adding instances reduce the per-instance metric? Is its collection interval fast enough? Can it saturate independent of user traffic?

## 13 — Metric sampling and reaction times

Real scaling operates on a timeline:

1. An increased workload begins.
2. A CloudWatch metric reflects the change.
3. A policy-managed alarm moves state when thresholds/time periods are satisfied.
4. The ASG increases desired capacity.
5. AWS provisions a new EC2 instance.
6. User data or application startup completes.
7. Target health checks mark it healthy.
8. The ALB starts directing appropriate traffic to it.

This chain may take minutes, depending on policy, metric and initialization. Therefore a system can experience latency or errors during a rapid spike even though Auto Scaling is configured correctly. Headroom, pre-warming strategy and workload design matter.

**Do not report a scaling response time without measuring timestamps for these stages.**

## 14 — Instance warmup and grace periods

A **warmup period** tells scaling mechanisms that freshly started capacity needs time to become stable. A **health check grace period** gives new instances a temporary window before ASG health checks lead to replacement decisions. They serve related but different purposes.

Set warmup and health checks based on **actual application boot behavior**, not arbitrary universal defaults. If a service takes longer than the configured grace period to become healthy, the ASG might repeatedly replace instances before they finish starting.

This creates a launch/terminate loop that looks like an AWS outage but is actually a timing/configuration bug.

## 15 — Why scaling down is harder than scaling up

Scale-out decisions typically prioritize adding capacity when demand is high. Scale-in can disrupt active users, especially when sessions are stored on one instance or long-running requests are being processed.

Consider:

- ALB deregistration delay / draining;
- application shutdown hooks;
- existing network connections;
- websocket sessions or long-running uploads;
- request retries;
- instance termination policy;
- safe rollout of in-flight jobs.

A scale-in event that reduces the EC2 count without visible 5xx errors can be useful—but the repository needs **measured HTTP outcomes** to claim it. ASG activity alone is not proof of uninterrupted user experience.

## 16 — Multi-AZ availability: redundancy must be real

Having subnets in two AZs is a good start, but a system is not genuinely redundant if its ASG desired capacity is one instance. That one server can exist in only one AZ at a time.

When aiming for multi-AZ availability, consider minimum desired capacity, healthy targets per AZ, remaining capacity if an AZ fails, cross-zone load balancing behavior, shared state and database dependencies.

**Illustrative thought experiment:** If every AZ has one backend and one AZ fails, the remaining backend must handle whatever traffic is still directed to it. If it was already operating near its CPU limit, the surviving instance may overload before Auto Scaling can add replacement capacity.

**Conclusion:** Multi-AZ is a resilience design property you must validate under expected workload and failure conditions—not a guarantee of zero downtime.

## 17 — Control plane versus data plane: an essential distinction

**Data plane:** the actual client request path: client → ALB listener → target group → EC2 application → response.

**Control plane:** Launch Template configuration, ASG desired-capacity updates, CloudWatch metrics, scaling policies, EC2 creation/termination, target registration.

An AWS console showing an **Active** policy or healthy ASG control plane does not prove the data plane serves the user. Conversely, a website can respond while its Auto Scaling policy is misconfigured, leaving it unable to cope with later demand.

Test the planes separately and correlate them: **healthy target group** + **HTTP status** + **metric movement** + **ASG activity**.

## 18 — Observability: turn claims into time-stamped measurements

For a meaningful case study, capture a minimum operational dashboard with:

- request rate (`RequestCount` where applicable),
- ALB 4xx/5xx responses and target-side 5xx,
- ALB target response time,
- healthy/unhealthy host counts,
- EC2 CPU utilization,
- ASG desired/in-service instance count,
- scaling activities with status and reason,
- request-latency observations from a client probe.

When describing a failure, use precise language:

- **Incident observed:** 14:02 UTC, user requests intermittently failing.
- **Detection:** target group health changes and 5xx rate rise.
- **Hypothesis:** bootstrap failure in new EC2 instances.
- **Action:** fix startup/service configuration and replace test nodes.
- **Evidence:** target group returns healthy; client receives stable responses.
- **Limitations:** no 24-hour uptime monitoring collected.

This story is credible; “production-grade zero downtime achieved” without metrics is not.

## 19 — Fault scenario A: EC2 host fails

**Expected goal:** Remove an unhealthy target from serving traffic and replace failed compute capacity.

**Investigate:** EC2 system/instance status checks, ASG instance health, ALB target reason codes, other targets' spare capacity and request error metrics.

**Why it may still hurt users:** Detection isn't instantaneous, in-flight requests can fail, new capacity takes time to become healthy, and one remaining instance may be underpowered.

**Good evidence:** pre-failure healthy target list → failure timestamp → unhealthy status → replacement launch → target healthy → client response probe. This project lacks that evidence at present; it belongs on the future validation roadmap.

## 20 — Fault scenario B: application process stops

**Symptom:** EC2 is Running, but ALB health becomes `unhealthy`.

**Likely layers:** web server process, app dependency, misconfigured listener port, ALB target group path, EC2 SG, NACL or local OS firewall.

**Investigate with the narrowest test first:** local `curl http://127.0.0.1:PORT/health`, then an in-VPC test to private IP, then target group health status, then actual ALB DNS endpoint. This isolates app failure from networking problems.

**Important:** If the ASG uses only EC2 health checks, you may still need to enable ELB health checks for automatic replacement of an app-unhealthy but system-healthy instance.

## 21 — Fault scenario C: load spike

**Symptom:** CPU/latency increases; target tracking may be too slow to add enough healthy capacity.

**Response:** Measure baseline first, generate controlled load on a lab endpoint, record CloudWatch metrics and ASG activities, wait for new targets to pass health checks, then measure whether client latency and error rate improve.

**Stop conditions:** Halt traffic if you exceed the intended cost/scale ceiling, see uncontrolled errors or impact any real users. Do not run high-volume load generation against public services that you do not own or lack authorization to test.

## 22 — Fault scenario D: rollout/bootstrap error

**Symptom:** A new Launch Template/AMI version creates instances that fail to become healthy; ASG may repeatedly replace them.

**Root causes:** Missing dependencies, failed package repository access, wrong health port, invalid IAM policy, bad file path, application binding to localhost, user-data script exiting early.

**Better process:** Bake and test a known-good image; validate bootstrap on one disposable instance; configure health checks before scaling; version Launch Templates; plan instance refresh carefully with a maintenance policy and rollback strategy.

A changed Launch Template setting is not the same as a successful full-fleet deployment.

## 23 — Security: where to place the trust boundaries

- Put public endpoints behind a correctly configured ALB, preferably using TLS/HTTPS for real internet-facing traffic.
- In a typical hardened design, place application EC2s in private subnets and permit inbound app ports only from the ALB security group.
- Restrict management access: use appropriate IAM/SSM patterns rather than unnecessarily opening SSH/RDP to the internet.
- Use IAM roles for instances instead of static AWS access keys.
- Limit outbound permissions where practical, while keeping known dependencies reachable.
- Protect AMIs, logs, user data and secrets. Secrets belong in an appropriate AWS secrets solution, not committed Markdown or bootstrap output.
- Patch base AMIs and audit security-group changes.

**Design ≠ validation:** Without actual SG/subnet/role exports, do not claim all these controls were implemented in the historical project.

## 24 — Stateful applications: why scaling needs more than instances

A stateless web tier is easier to scale horizontally because any healthy instance can serve a request. A stateful system might keep sessions, uploaded files, local caches or database writes on one instance.

If an ASG terminates that instance, local state may be lost or users may be logged out. For scalable design, consider centralized session storage, shared durable object storage, external databases, idempotent endpoints and proper cache invalidation. These are *design extensions*, not evidence they were part of this project.

**Reliability thought:** You can have five perfectly healthy frontend EC2 instances and still have a total outage if they all depend on one unavailable database.

## 25 — The truth about “zero downtime”

“Zero downtime” is an outcome claim. It needs a defined time window, uptime/service-level indicator (SLI), sampling interval, alert thresholds and real user request results. A design diagram cannot establish it.

A strong junior engineer says: “I configured an ALB/ASG architecture to reduce single-instance risk. I still need a controlled failure drill and request-level monitoring before claiming failover behavior.”

That is more credible than claiming 100% availability from a console screenshot.

A real experiment might measure:

- total failed requests during a node termination,
- maximum/average HTTP latency during recovery,
- time until a replacement instance becomes healthy,
- time to recover normal error rates.

## 26 — Cost engineering: the difference between auto scaling and cost optimization

Auto Scaling can reduce EC2 runtime usage relative to running peak capacity continuously—but **actual savings require a baseline and a comparable workload**. An ALB has its own cost, as do EC2, EBS, data transfer, CloudWatch extras, network egress services and possibly public IPv4 addresses.

Reported `MaxSize=10` is a cost *risk ceiling*, not a bill estimate or usage result. An overly generous maximum can incur unexpected expense if a test creates repeated scale-out.

For a disposable lab, set a budget alert, limit scale boundaries, avoid unnecessary NAT resources, stop after tests and verify the resources were removed. A cost comparison requires actual billing/cost data; none was provided for the original project.

## 27 — ASG versus ECS, EKS, Lambda and Transit Gateway

This project scales **EC2 instances**. It is not an ECS Fargate deployment, a Kubernetes HPA, a serverless Lambda application or a multi-VPC transit routing project.

| Technology | Primary scaling focus | When it fits |
|---|---|---|
| EC2 Auto Scaling | EC2 instance capacity | VM-hosted or self-managed web/services |
| ECS Service Auto Scaling | Container service task counts | Containerized workloads on ECS |
| Kubernetes HPA | Pod replicas (with node capacity handled separately) | Kubernetes application scaling |
| AWS Lambda | Function concurrency managed by service controls | Event-driven/serverless use cases |

Know the difference before attributing one service's feature to another.

## 28 — Build a validation matrix you can defend in an interview

| Objective | Minimum test | Evidence to save | What would falsify success? |
|---|---|---|---|
| Serve requests via ALB | HTTP request to ALB DNS | Timestamped `curl`, status & header | No response / 5xx |
| Register backend instances | Describe target health | Healthy target list, AZs | Empty/unhealthy target group |
| Distribute across AZs | List ASG instances/AZs | Instances in both AZs | All instances in one AZ |
| Scale out | Controlled load | CloudWatch metric + scaling activity + new healthy target | Metric rises but no capacity change |
| Scale in | Remove load | Scaling activity + target drain + error rate | Terminated in-flight requests cause sustained failures |
| Replace unhealthy server | Controlled instance failure | Before/after target health and replacement | No replacement or app remains unhealthy |
| Stay available | Client probe during scenario | Request counts, latency, errors | Significant user-visible disruption |
| Reduce cost | Cost trend before/after | Bills with comparable workload | Savings not observed |

These are **proposed validation tests**. They are not presented as tests completed in the earlier project.

## 29 — Read-only AWS CLI toolkit

Use the following only with your own AWS account and actual names/ARNs; all identifiers here are placeholders. The commands are examples for a rerun, not copied historical execution logs.

```bash
# ASG configuration and currently managed instance state
aws autoscaling describe-auto-scaling-groups \
  --region us-east-1 \
  --auto-scaling-group-names YOUR_ASG_NAME

# Scaling policy and actual scaling activity history
aws autoscaling describe-policies \
  --region us-east-1 \
  --auto-scaling-group-name YOUR_ASG_NAME
aws autoscaling describe-scaling-activities \
  --region us-east-1 \
  --auto-scaling-group-name YOUR_ASG_NAME \
  --max-items 20

# ALB details and target group target health
aws elbv2 describe-load-balancers --region us-east-1 --names YOUR_ALB_NAME
aws elbv2 describe-target-health \
  --region us-east-1 \
  --target-group-arn YOUR_TARGET_GROUP_ARN

# Launch Template version information
aws ec2 describe-launch-template-versions \
  --region us-east-1 \
  --launch-template-name YOUR_TEMPLATE_NAME
```

Be mindful that detailed launch template information may contain user data and other fields that should not be publicly posted without review.

## 30 — A practical troubleshooting algorithm

When someone says **“my Auto Scaling app is down”**, separate the problem into progressively smaller questions:

1. **Client entry:** Does ALB DNS resolve? Does the HTTP(S) listener accept connections?
2. **Forwarding:** Does the listener route to the correct target group?
3. **Health:** Are registered targets healthy? What reason codes are reported?
4. **Network:** Does ALB SG permit clients and EC2 SG permit target traffic from ALB?
5. **Application:** Does the process listen at the expected port and return 200 locally?
6. **Capacity:** Is ASG desired capacity above zero? Are instances `InService`?
7. **Scaling:** Are policies attached, limits sensible and CloudWatch metrics arriving?
8. **Startup:** Do instance logs show bootstrap failures or failed service starts?
9. **Dependencies:** Is the database/cache/third-party API actually available?
10. **Proof:** What did the user observe before, during and after your fix?

A disciplined debugger does not increase desired capacity blindly or open `0.0.0.0/0` on every instance to “fix” access.

## 31 — Practice exercises to move from knowledge to skill

**Exercise A — Basic architecture:** Draw two AZs and explain the difference between ALB listener, target group, ASG, and Launch Template in one sentence each.

**Exercise B — Health checks:** Deliberately break `/health` on one *test* backend while EC2 remains Running. Observe target health. Then verify whether your ASG is configured for ALB health-based replacement. Document the difference.

**Exercise C — Scaling:** Choose a safe max capacity, apply authorized controlled load, and create a 5-point timeline: load started, metric rose, policy reacted, instance entered service, latency improved or did not improve.

**Exercise D — AZ handling:** Verify actual targets across AZs. Simulate removing one set of lab targets from service and watch how traffic and remaining capacity react. Do not equate this with a real AZ outage test.

**Exercise E — Cost:** Estimate baseline running costs using current AWS pricing, then compare with a genuine billing statement after the lab. Explicitly note excluded network/storage costs.

## 32 — Interview questions with model reasoning

### Q1. Why use an ALB in front of EC2 instances?
To provide an HTTP-facing entry point that forwards traffic according to listener rules and target-group health. It is different from the compute-capacity management provided by ASG.

### Q2. What does an Auto Scaling Group actually do?
Maintains desired EC2 capacity and can automatically adjust it based on configured policy and health status, subject to configured limits and resource availability.

### Q3. Is an EC2 `Running` state proof that the website works?
No. It says the VM runs; application process, port, dependencies and health endpoint still require separate verification.

### Q4. What is the difference between a target group and an ASG?
A target group is where the load balancer forwards requests; the ASG manages EC2 instances and can attach to the target group so its instance lifecycle updates registration.

### Q5. How does a newly created instance start receiving traffic?
It must launch, bootstrap, be registered into the target group, and pass the configured health check. Only then is it eligible as a healthy target under normal conditions.

### Q6. Does an ASG automatically replace an instance whose web process fails?
Not necessarily. If EC2 checks pass and ALB health checks are not enabled for ASG replacement, the application-unhealthy instance may remain running in the group.

### Q7. What do min, desired and max mean?
Minimum, currently requested, and maximum ordinary capacities. `Max=10` does not prove ten instances ever existed simultaneously.

### Q8. What is target tracking?
A dynamic scaling approach that adjusts ASG capacity toward a target value of an appropriate CloudWatch metric. The scaling response has delays and safeguards, so it is not instantaneous.

### Q9. Which is better: CPU target tracking or request-count target tracking?
It depends on what predicts workload saturation. CPU often suits CPU-heavy work; requests per target can suit predictable per-request processing. Profile the application and validate the metric.

### Q10. Why is a two-AZ design not automatically highly available?
It could still have one active instance, insufficient standby capacity, a single-zone database or a failed shared dependency. Measured request behavior during failure is required.

### Q11. What is instance warmup used for?
It prevents newly launched instances from prematurely distorting scaling decisions before they are ready/steady. It differs from health check grace periods.

### Q12. What happens if target tracking wants to scale out and the ASG is at max?
Ordinary scale-out is capped at the configured maximum. The application may remain saturated if existing capacity is insufficient.

### Q13. What's the difference between horizontal and vertical scaling?
Horizontal scaling adds/removes instances; vertical scaling changes an instance's resources. ASG is commonly used for horizontal instance count management.

### Q14. How would you prove actual autoscaling occurred?
Use a time-correlated metric graph, ASG scaling activity history, changed desired capacity, launched instance IDs and healthy target registration—not only a policy settings screenshot.

### Q15. How would you prove zero downtime?
Define a measurement interval, send repeated synthetic or real-user probes through the ALB, capture errors/latency and verify availability objectives during actual failure and scaling events. Without that, don't make the claim.

### Q16. Why might the application still fail after ASG adds servers?
Because the bottleneck might be a database, shared session state, downstream dependency, networking or broken bootstrap; increasing web VM count doesn't solve all bottlenecks.

### Q17. Why could scale-in break users?
Existing connections, sessions or long requests may be on a server that is draining/terminating. Shutdown and deregistration policies need to handle active work.

### Q18. What costs remain if scale-in reduces EC2 count?
Potentially ALB, EBS, CloudWatch, network and other charges, plus any always-on minimum EC2 capacity; savings need actual cost comparisons.

### Q19. Is AWS CloudWatch itself serving traffic?
No. It collects metrics, logs and alarms. The request path is client to ALB to EC2 backend; CloudWatch is observational/control-plane support.

### Q20. What is the most honest summary of this project?
“I documented an AWS architecture using EC2 Launch Templates, ALB, multi-AZ Auto Scaling and CloudWatch target tracking. My current evidence includes a prior project record, but to claim demonstrated failover, exact scaling behavior or no downtime I would need fresh time-stamped target-health, scaling and HTTP test results.”

## 33 — Engineering reflection and next milestone

This architecture is valuable because it teaches an engineer to reason across **networking, service health, fleet management, observability and customer-facing behavior**. The biggest lesson is not that AWS can launch VMs automatically; it is that reliability emerges only when health checks, placement, bootstrapping, capacity and application dependencies are designed and verified together.

A mature project story should answer:

- What demand or failure motivated the architecture?
- What exact resources were configured and why?
- Which signals tell us the web app is really healthy?
- How does an unhealthy instance stop taking traffic and get replaced?
- How quickly does new capacity become usable after demand changes?
- What did users experience during a controlled failure?
- What evidence supports every technical outcome claimed?

**Next improvement:** Capture a fresh, budget-bounded rerun with healthy targets in two AZs, a measured scale-out/scale-in sequence, a single-instance failure drill and a timestamped report. Only then promote “reported setup” to “measured behavior” in the recruiter landing page.

---

## Official AWS documentation used for verification

- [Amazon EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/)
- [Target tracking scaling policies](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html)
- [Create target tracking policy](https://docs.aws.amazon.com/autoscaling/ec2/userguide/policy_creating.html)
- [Attach an ALB target group to an Auto Scaling group](https://docs.aws.amazon.com/autoscaling/ec2/userguide/attach-load-balancer-asg.html)
- [ALB target-group health checks and status reasons](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html)
- [EC2 Auto Scaling health check types](https://docs.aws.amazon.com/autoscaling/ec2/userguide/health-checks-overview.html)
- [Using load balancing with Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/autoscaling-load-balancer.html)

**Last authored:** October 2026. **Scope:** Educational case study with limitations of archived project evidence stated explicitly. Commands and drills are suggested rerun procedures, not historical run logs.
