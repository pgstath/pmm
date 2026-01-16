=== "Kubernetes (production)"

    **Best for**: Production environments, cloud-native architectures, teams with existing Kubernetes infrastructure.

    Kubernetes provides enterprise-grade high availability through automated container orchestration, self-healing capabilities, and intelligent workload distribution across multiple nodes. This leverages Kubernetes pod management and PMM's data persistence:

     - Kubernetes automatically restarts failed pods and reschedules them to healthy nodes.
     - persistent volumes preserve all PMM data, configurations, and dashboards across pod restarts.
     - health probes ensure only healthy instances receive traffic.
     - PMM Clients cache metrics locally during server unavailability.

    When infrastructure issues occur, Kubernetes automatically:

     - detects pod or node failures through health checks (within 30 seconds)
     - marks failed resources as unavailable
     - reschedules the PMM pod to a healthy node
     - mounts the existing persistent volume to restore state
     - routes traffic once readiness checks pass

    During failover (typically 2-5 minutes), data integrity is maintained through:

     - PMM Client-side caching of up to 24 hours of metrics.
     - persistentVolumeClaims that retain all historical data.
     - automatic metric synchronization once connection restores.
     - preservation of all configurations and custom dashboards.

    This solution works well for production environments that can tolerate brief monitoring interruptions during automatic failover. 
    
    The trade-off between operational simplicity and high availability makes it ideal for most production workloads. For zero-downtime requirements or multi-region deployments, consider the advanced clustering options in the following sections.

    For deployment instructions, see [Install PMM Server with Helm on Kubernetes clusters](../install-pmm/install-pmm-server/deployment-options/helm/index.md).



