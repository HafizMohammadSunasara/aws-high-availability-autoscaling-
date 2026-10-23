# Evidence Index — Project 3

## Current evidence inventory

| File | Original source | What it establishes | What it cannot establish |
|---|---|---|---|
| [`01-linkedin-project-record.png`](../evidence/01-linkedin-project-record.png) | Screenshot of the original LinkedIn project entry (August 2026 project) | The author documented a design with EC2 Launch Templates, ALB, multi-AZ ASG integration and target tracking; a small ASG console screenshot is embedded | That scaling, ALB target health, instance replacement, zero downtime, exact capacity or actual cost savings were independently measured |

## Image processing performed

The supplied original was **upscaled to 2× and gently sharpened** for readability. No AWS status, values or topology were synthesized. The small AWS screenshot embedded in the LinkedIn card remains limited by its source resolution.

There are **no duplicate images**, no `raw/enhanced` hierarchy, and no manufactured verification screenshots.

## Architecture diagram provenance

[`diagrams/architecture.png`](../diagrams/architecture.png) is an **illustrative reconstruction** from the project description and standard AWS architecture principles. It is not original AWS console evidence.

## Recommended additions after a legitimate rerun

1. `02-launch-template-settings.png` — redact IDs, capture AMI/type/profile/security settings.
2. `03-alb-target-health.png` — actual target health and two AZs.
3. `04-asg-capacity-and-az.png` — min/desired/max and instance placements.
4. `05-target-tracking-policy.png` — metric, target and instance warmup.
5. `06-cloudwatch-and-scaling-activities.png` — time-aligned metrics and successful scaling event.
6. `07-instance-failure-recovery.png` — controlled replacement sequence and user-visible request outcome.

Only add any of these if you genuinely produce the corresponding record. None are currently claimed as part of the project.
