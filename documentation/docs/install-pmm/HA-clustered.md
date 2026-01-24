# Understand PMM HA Clustered

!!! warning "Technical Preview: Not production-ready"
    This feature is in **Technical Preview** for testing and feedback only. Expect [known issues](#known-issues), breaking changes, and incomplete features.
    
    **Test in non-production environments only** and [provide feedback](#provide-feedback) to shape the GA release.

!!! danger "VictoriaMetrics limitations"
    This Tech Preview does not support:
    
    - **Prometheus data imports**: Cannot import existing Prometheus files
    - **Metrics downsampling**: No automatic historical data optimization
    
    If your strategy requires these features, evaluate carefully before testing.
 
Standard PMM monitoring goes offline for minutes during server failures. PMM HA Clustered keeps monitoring running with automatic failover in under 30 seconds.

PMM HA Clustered keeps your database monitoring running continuously, even when servers fail or during maintenance windows.

Unlike [single-instance deployments](../install-pmm/HA-kubernetes-single-instance.md) where a server failure means minutes of monitoring downtime, PMM HA Clustered automatically switches to backup servers in under 30 seconds. 

Whether a server crashes, you're upgrading software, or scaling your infrastructure, your monitoring stays active with no blind spots or missed incidents.

### Key benefits

- **Continuous monitoring**: Three PMM server replicas ensure monitoring never stops, even during failures
- **Fast automatic failover**: Traffic switches to healthy servers in under 30 seconds with no data loss
- **Automatic traffic routing**: HAProxy routes traffic to the active leader and handles failover transparently
- **Resilient data storage**: Distributed databases (ClickHouse, VictoriaMetrics, PostgreSQL) eliminate single points of failure
- **Scales with your needs**: Add more capacity as your database infrastructure grows

### Choose your Kubernetes deployment type

| Consideration | Single-instance (GA) | HA Clustered (Tech Preview) |
|--------------|----------------------|-----------------------------|
| **Production status** | ✅ Production-ready | ⚠️ Testing only |
| **Failover time** | 2-5 minutes | <30 seconds |
| **Setup complexity** | ● Low | ●●●●● High |
| **Resource overhead** | 1x baseline | 3-5x baseline |
| **Setup time** | ~5 minutes | ~20 minutes |
| **PMM instances** | 1 pod | 3 pods with leader election |
| **Automatic failover routing** | No | Yes (HAProxy routes to active leader) |
| **Databases** | Built-in | External clusters |
| **Anti-affinity** | No | Yes (pods distributed across nodes) |

## Before you begin

### Check prerequisites

- **Kubernetes**: 1.22 or higher
- **Helm**: 3.2.0 or higher
- **kubectl**: configured to access your cluster
- **Persistent Volume Provisioner**: available in your cluster

### Verify Kubernetes operators

PMM HA requires three Kubernetes operators to manage distributed database resources. These operators must be installed **before** deploying PMM HA, as they manage the lifecycle of database resources through Custom Resource Definitions (CRDs).

- **VictoriaMetrics Operator** (v0.56.4+): Manages VictoriaMetrics cluster for metrics storage
- **Altinity ClickHouse Operator** (v0.25.4+): Manages ClickHouse cluster for QAN data
- **Percona PostgreSQL Operator** (v2.8.0+): Manages PostgreSQL cluster for Grafana metadata

You can install these operators via the `pmm-ha-dependencies` chart (recommended) or manually. See [Installation](#install-pmm-ha) for details.

### Check if your platform is supported

!!! info "Tested Platform: Amazon EKS only"
    This Tech Preview is validated exclusively on **Amazon EKS (Kubernetes 1.24+)**. Other platforms (GKE, AKS, on-premise, OpenShift) may work but are untested. VMware Tanzu is not supported.

## Plan your resources

Before installing PMM HA, ensure your Kubernetes cluster has sufficient capacity to run the distributed architecture. This section helps you calculate the resources you'll need based on your monitoring requirements.

### Minimum cluster requirements

At minimum, your cluster needs:

- **CPU**: 12+ cores
- **Memory**: 20-40 GB RAM
- **Storage**: 100+ GB with persistent volume provisioner

This baseline supports monitoring **1-10 database services** with standard retention periods. 

If you're planning a larger deployment, use the sizing guidelines below to calculate your resource needs.

### Sizing guidelines

Use this table to estimate resources based on your monitoring scale:

| Monitored services | PMM replicas | ClickHouse replicas | VictoriaMetrics storage | Total CPU | Total memory | Total storage |
|-------------------|--------------|---------------------|------------------------|-----------|--------------|---------------|
| 1-10 | 3 | 3 | 3 | 12 cores | 20 GB | 100 GB |
| 11-50 | 3 | 3 | 3 | 15 cores | 30 GB | 200 GB |
| 51-100 | 3 | 3 | 5 | 20 cores | 40 GB | 500 GB |
| 100+ | 5 | 5 | 5 | 30+ cores | 60+ GB | 1+ TB |

### Factors affecting resource usage

Your actual resource needs may vary based on:

- number of monitored database instances
- metrics resolution and retention period
- Query Analytics (QAN) volume
- number of concurrent users
- custom dashboards and queries

### Understand how resources are distributed

PMM HA spreads resource consumption across multiple components to ensure high availability. Understanding this breakdown helps you identify which components to scale as your monitoring needs grow, and where bottlenecks might occur.

| Component | CPU | Memory | Storage | Notes |
|-----------|-----|--------|---------|-------|
| **Per PMM Server pod** | 2 cores | 4 GB | 50 GB | Storage varies with retention period |
| **HAProxy (3 replicas)** | 1 core | 2 GB | - | Routing and failover overhead |
| **ClickHouse cluster** | 3-6 cores | 8-12 GB | 20+ GB | Scales with QAN volume |
| **VictoriaMetrics cluster** | 2-4 cores | 4-8 GB | 20+ GB | Scales with metrics volume |
| **PostgreSQL cluster** | 1-2 cores | 2-4 GB | 10 GB | Grafana metadata storage |
| **Kubernetes operators** | 1-2 cores | 2-4 GB | - | Operator management overhead |

## Learn the architecture

The PMM HA architecture diagram below shows how components interact and communicate. 

The architecture consists of:

- three PMM server replicas with automatic leader election
- HAProxy for routing traffic to the active leader and handling failover
- operator-managed database clusters (ClickHouse, VictoriaMetrics, and PostgreSQL) for resilient data storage

![HA Clustered diagram](../images/HA-diagram.jpg)

### Learn high availability mechanisms
PMM HA uses several mechanisms to ensure continuous operation:

- **Leader election**: PMM servers use Raft consensus protocol for leader election (ports 9096, 9097)
- **Automatic failover**: HAProxy detects when the active leader becomes unhealthy and routes traffic to the new leader
- **Pod anti-affinity**: Kubernetes scheduler distributes components across different nodes
- **Health checks**: Comprehensive readiness and liveness probes on all components
- **Rolling updates**: Zero-downtime upgrades with sequential pod updates

## Ready to deploy?

Now that you understand how PMM HA Clustered works, you can deploy it on your Kubernetes cluster. 

The installation process uses Helm to set up all three replicas, configure HAProxy load balancing, and deploy the distributed databases automatically.

[Install PMM HA Clustered →](../install-pmm/install-HA-clustered.md){.md-button} 