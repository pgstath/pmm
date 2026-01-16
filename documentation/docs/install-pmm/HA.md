# Install PMM in High Availability (HA) mode

When your database monitoring goes down, you lose visibility into critical performance issues just when you need it most. HA ensures your PMM monitoring stays online even when servers fail, networks disconnect, or hardware breaks.

Implement HA to build a resilient PMM deployment that keeps monitoring your databases no matter what happens to individual components.

## Understand what PMM HA can and can't do

Before you invest time in setting up HA for PMM, evaluate whether its benefits justify the added complexity for your specific use case.

Critical systems requiring sub-second failover gain the most value from PMM HA, while environments that can tolerate brief monitoring gaps (seconds to minutes) may find simpler solutions more appropriate. Consider your RTO requirements and incident response processes when deciding whether HA justifies the operational investment:

### What PMM HA provides

- Continuous monitoring visibility during server failures, preventing blind spots when you need observability most.
- Automatic failover that restarts services or switches to backup systems without manual intervention.
- Zero metric loss during brief outages, thanks to PMM's client-side caching that preserves data until connectivity resumes.
- Reduced operational risk by maintaining monitoring coverage during critical incidents.

### What PMM HA cannot solve

- Even with perfect HA, you'll still only detect issues after PMM's minimum one-minute alerting interval.
- Complete network partitions that isolate entire segments of your infrastructure from monitoring.
- Increased operational overhead since HA introduces additional complexity in deployment, maintenance, and troubleshooting.

### HA deployment options

Choose the option that best fits your infrastructure and requirements:

**Note:** All options prevent data loss (PMM Clients cache metrics).


| Feature / Option        | Docker HA (basic) | Kubernetes HA (single-inst.) | Kubernetes HA (clustered) |
|-------------------------|-------------------|-------------------------------|----------------------------|
| Status                  | ✅ Production     | ✅ GA ⭐                       | ⚠️ Tech Preview            |
| Failover Time           | 1–3 min           | 2–5 min                       | < 30 sec                   |
| Platform Requirements   | No K8s needed     | K8s required                  | K8s + operators            |
| Complexity              | Very Low          | Low                           | High                       |
| Best for                | • Dev/test<br>• Small scale<br>• No K8s | • Production ✓<br>• Most users ✓<br>• K8s teams ✓ | • Zero-downtime requirements  • Multi-instance Load balancing Testing only<br>• Feedback<br>• Evaluation |

