=== "Clustered (future)"

    **Best for**: Large enterprises, geographically distributed teams, maximum resilience requirements.

    A fully clustered PMM deployment is under development to provide true high availability with zero downtime and horizontal scalability. This enterprise-grade architecture will leverage Kubernetes orchestration and distributed database technologies:

     - multiple active PMM instances with automatic leader election via Raft consensus
     - clustered databases ensure no single point of failure across all data stores
     - geographic distribution support for multi-region deployments
     - automatic failover with zero data loss and minimal service interruption

    The clustered architecture will include:

     - **PMM instances**: multiple servers in active-passive configuration with automatic leader election.
     - **PostgreSQL cluster**: replicated metadata and configuration storage with automatic failover.
     - **ClickHouse cluster**: distributed query analytics data across multiple shards and replicas.
     - **VictoriaMetrics cluster**: horizontally scaled metrics storage with configurable replication factor.
     - **HAProxy**: intelligent load balancing and automatic routing to the current leader.

    This solution will address enterprise requirements for mission-critical monitoring infrastructure where any downtime is unacceptable. It will support complex scenarios including disaster recovery, multi-datacenter deployments, and regulatory compliance requiring data residency.

    This feature is currently in development. For immediate high availability needs, consider the Kubernetes deployment option described above, which provides robust automatic recovery suitable for most production environments.