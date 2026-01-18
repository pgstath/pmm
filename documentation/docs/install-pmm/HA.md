# Install PMM in High Availability (HA) mode

When your database monitoring goes down, you lose visibility into critical performance issues just when you need it most. HA ensures your PMM monitoring stays online even when servers fail, networks disconnect, or hardware breaks.

Implement HA to build a resilient PMM deployment that keeps monitoring your databases no matter what happens to individual components.

## Understand what PMM HA can and can't do

Before you invest time in setting up HA for PMM, evaluate whether its benefits justify the added complexity for your specific use case.

Critical systems requiring sub-second failover gain the most value from PMM HA, while environments that can tolerate brief monitoring gaps (seconds to minutes) may find simpler solutions more appropriate. Consider your RTO requirements and incident response processes when deciding whether HA justifies the operational investment.

### What PMM HA provides

- Continuous monitoring visibility during server failures, preventing blind spots when you need observability most
- Automatic failover that restarts services or switches to backup systems without manual intervention
- Zero metric loss during brief outages, thanks to PMM's client-side caching that preserves data until connectivity resumes
- Reduced operational risk by maintaining monitoring coverage during critical incidents

### What PMM HA cannot solve

- Even with perfect HA, you'll still only detect issues after PMM's minimum one-minute alerting interval
- Complete network partitions that isolate entire segments of your infrastructure from monitoring
- Increased operational overhead since HA introduces additional complexity in deployment, maintenance, and troubleshooting

## Feature comparison

All three options prevent data loss through PMM Client caching during outages.

| Feature | [Docker](../install-pmm/HA-docker.md) | [Kubernetes (single-instance)](../install-pmm/HA-kubernetes-single-instance.md) | [Kubernetes (clustered)](../install-pmm/HA-clustered.md) |
|---------|--------|-------------------|---------------------|
| **Status** | ✅ GA - Production ready | ✅ GA - Production-ready | ⚠️ Tech Preview - Testing only |
| **Kubernetes required** | No | Yes | Yes |
| **PMM instances** | 1 | 1 | 3 |
| **Failover time** | 1-3 min | 2-5 min | < 30 sec |
| **Zero downtime** | No | No | Yes |
| **Setup complexity** | Very Low | Low | High |
| **Resource overhead** | 1x | 1.2x | 3-5x |
| **Monitoring data preserved** | Yes - stored on clients during outage | Yes - stored on clients during outage | Yes - always available on multiple servers |

## HA deployment options

Choose the deployment option that matches your infrastructure and requirements:

=== "Docker HA (basic)"

    **Status**: ✅ **Production-ready**

    Simple automatic restart capabilities using Docker's built-in recovery features. Perfect for development, testing, and single-server deployments.

    **Key features**:
    
    - Docker automatically restarts PMM Server after crashes
    - PMM Clients buffer metrics locally during outages
    - Minimal operational overhead
    - No Kubernetes required

    **Limitations**:
    
    - 1-3 minutes downtime during container restarts
    - Single point of failure
    - Manual intervention required for host-level failures

    **When to use this option**:
    
    - You're in development or testing
    - You don't have Kubernetes
    - You want the simplest setup
    - You can tolerate 1-3 minutes of downtime

    [View Docker HA installation guide →](../install-pmm/HA-docker.md)

=== "Kubernetes HA (single-instance)" 

    **Status**: ✅ **Production-ready**

    Enterprise-grade high availability through Kubernetes orchestration. Provides automatic pod rescheduling and persistent data across failures.

    **Key features**:
    
    - Kubernetes automatically reschedules failed pods to healthy nodes
    - Persistent volumes preserve all data and configurations
    - Health probes ensure only healthy instances receive traffic
    - Production-tested and stable

    **Limitations**:
    
    - 2-5 minutes monitoring interruption during pod rescheduling
    - Single PMM instance (no load distribution)

    **When to use this option**:
    
    - You have Kubernetes infrastructure
    - You need production-ready HA
    - You can tolerate 2-5 minutes of downtime
    - You want automatic recovery without complexity
    - **Best choice for 90% of production deployments**

    [View Kubernetes HA installation guide →](../install-pmm/HA-kubernetes-single-instance.md)

=== "Kubernetes HA (clustered)"

    **Status**: ⚠️ **Technical Preview - NOT for production**

    Zero-downtime high availability with multiple active PMM instances, distributed databases, and automatic load balancing.

    !!! warning "Technical Preview"
        This deployment option is in **Technical Preview** and should **NOT be used in production environments**. It is provided for testing and feedback purposes only.

    **Key features**:
    
    - Zero-downtime failover (< 30 seconds)
    - 3 PMM server replicas with leader election
    - HAProxy load balancing
    - Distributed ClickHouse, VictoriaMetrics, and PostgreSQL clusters

    **What makes it different**:
    
    - **No monitoring blind spots**: Unlike single-instance (2-5 min gaps), clustered maintains continuous visibility
    - **Multiple active instances**: Load distribution and instant failover to followers
    - **Horizontal scalability**: Add replicas as monitoring load grows
    - **True zero-downtime**: Traffic redirects to healthy instances in < 30 seconds

    **Known limitations**:
    
    - NOT production-ready as it has known bugs
    - Complex setup requiring 3 Kubernetes operators
    - 3x resource overhead minimum
    - Subject to breaking changes
    - Node selection shows incorrect PostgreSQL instances
    - Services added via pmm-admin don't show dashboard data
    - PostgreSQL monitoring may show incorrect FAILED status

    **When to use this option**:
    
    - you understand this is NOT production-ready
    - you need continuous monitoring visibility (no 2-5 min gaps)
    - you have strict SLA requirements for sub-30-second failover
    - you need multiple active PMM instances for load distribution
    - you have expert Kubernetes skills
    - you are evaluating zero-downtime architecture for future production, or testing environments where you need to validate continuous monitoring capabilities

    [View Kubernetes HA Cluster installation guide →](../install-pmm/HA-clustered.md)