# Architecture — A multi-AZ load-balanced EC2 service

> **Evidence classification:** This design is reconstructed from the original August 2026 LinkedIn project description. Resource names, live settings, exact target-health records and real instance counts were not archived in the screenshot supplied for this repository.

## 1. Services and responsibilities

| Service | Responsibility | Often misunderstood as |
|---|---|---|
| Amazon VPC / subnets | Network boundaries and IP addresses | Load balancing by itself |
| Application Load Balancer | Accept HTTP(S), match rules, forward to target groups | A system that directly scales EC2 |
| Target group | Pool of candidate targets with health check behavior | A list that is guaranteed healthy |
| EC2 Launch Template | Repeatable configuration of future EC2 instances | Proof that a website works |
| Auto Scaling Group | Maintain/adjust EC2 desired capacity, distribution and lifecycle | Proof of zero downtime |
| Target tracking policy | Adjust capacity toward a metric target | Event-driven immediate replication of every incoming request |
| Amazon CloudWatch | Metrics/alarms that inform capacity decisions | Full end-to-end uptime and fault tolerance |

## 2. Reconstructed network topology

![Reconstructed AWS architecture](../diagrams/architecture.png)

- Region reported: `us-east-1`.
- AZs reported: `us-east-1a` and `us-east-1b`.
- Instance class reported: `t3.micro`.
- ASG upper capacity described as **10 instances**, which is a *configuration limit*, not proof of ten launched nodes.
- ALB and ASG target group attachment are reported in project description.
- The diagram deliberately **does not assign exact port numbers or subnet types** because the supplied image does not prove them.

## 3. Request data path

1. DNS resolves the ALB DNS name.
2. A client opens a TCP/TLS connection to the ALB listener.
3. Listener rules choose a target group.
4. ALB forwards the request to a target that passes configured health checks (subject to load balancer behavior and exceptional fail-open cases).
5. The backend responds through the ALB.

**Important:** User request traffic does not pass through CloudWatch or the ASG. Those are control/observation-plane components; CloudWatch detects metric changes that influence **capacity**, not which target handles each HTTP request.

## 4. Capacity/control path

1. Web servers publish metrics or AWS derives them from EC2/ALB.
2. The target tracking scaling policy uses a chosen metric, target value and group capacity boundaries.
3. Amazon EC2 Auto Scaling manages CloudWatch alarms and changes desired capacity when policy conditions are met.
4. The ASG launches or terminates EC2 instances based on its configuration.
5. Newly launched instances run bootstrap instructions and enter the target group.
6. ALB target health controls when a new instance can receive application traffic.

## 5. Design decisions and trade-offs

### Public ALB and backend network isolation

A common production reference pattern is an internet-facing ALB in two public subnets forwarding to application EC2 instances in private subnets, allowing inbound application traffic only from the ALB security group. This is a **recommendation**, not evidence that the original lab used exactly these subnet boundaries.

### Multi-AZ placement

Using two AZs can reduce exposure to a single AZ failure, but real fault tolerance depends on healthy capacity, ASG allocation, ALB configuration, application state, and upstream/downstream dependencies.

### Launch Template versioning

Treat AMI, IAM instance profile, bootstrap and security groups as versioned operational configuration. Changing a Launch Template does not automatically update every existing instance unless you run an appropriate instance refresh or deployment procedure.

### Health checks

EC2 status checks assess underlying host/system conditions. ALB target checks assess configured application reachability, and ASG can optionally be configured to use Elastic Load Balancing health checks for replacement.

### Capacity guardrails

- Min: floor of healthy capacity ASG tries to maintain.
- Desired: current requested instance count.
- Max: capacity ceiling for routine scaling; there are edge cases around rebalancing/maintenance.
- Warmup: how long new capacity needs before metrics and scaling judgments stabilize.

## 6. Failure analysis

| Failure | Expected control-plane reaction | Failure condition to test |
|---|---|---|
| App process fails on one instance | ALB may mark target unhealthy | Does ASG replace it if ALB checks enabled? |
| EC2 host fails | EC2 health checks may trigger ASG replacement | How long until a replacement is healthy? |
| One AZ disrupted | ASG may rebalance remaining capacity | Can other AZ serve traffic without overload? |
| Load rises | Target tracking may scale out | Is new target ready before current nodes saturate? |
| Bootstrap fails | Instance exists but app endpoint fails | Does the group repeatedly launch unhealthy instances? |
| Database dependency fails | Frontend may still be healthy at port level | Does app-level readiness reflect dependency state? |

## 7. Scope honesty

The original supplied evidence is a LinkedIn project record containing a small ASG console thumbnail. This is **not sufficient to independently verify** ALB health, full configuration, timed scaling events, or failover performance. The design above is provided as a transparent reconstructed case-study architecture.

## Official AWS source references

- https://docs.aws.amazon.com/autoscaling/ec2/userguide/
- https://docs.aws.amazon.com/autoscaling/ec2/userguide/attach-load-balancer-asg.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html
