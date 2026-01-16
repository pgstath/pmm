=== "Docker (basic)"

    **Best for**: Development environments, single-server deployments, teams wanting basic restart capabilities without true HA.

    Docker's built-in restart capabilities combined with PMM's client-side data buffering provide basic availability improvements, but this is not a true high availability solution. This approach leverages Docker's automatic container recovery:

     - Docker automatically restarts the PMM Server container after crashes or system reboots.
     - PMM Clients buffer metrics locally when the server is unavailable, preventing data loss during outages.

    To increase PMM availability and ensure the PMM Server automatically restarts after minor issues, launch the PMM Server in Docker with the `--restart=always` flag.

    When the PMM Server becomes unavailable, PMM Clients automatically:

     - detect the connection failure
     - begin caching metrics data locally
     - continue attempting to reconnect
     - transfer all cached data once the connection is restored

    This solution works well for environments where brief interruptions are acceptable and post-incident analysis is more important than real-time availability. For mission-critical deployments requiring higher availability, consider the more advanced options described in the following sections.

    For deployment instructions, see [Install PMM Server with Docker](../install-pmm/install-pmm-server/deployment-options/docker/index.md).

