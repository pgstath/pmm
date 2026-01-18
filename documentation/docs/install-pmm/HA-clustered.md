# Install PMM with Kubernetes HA (clustered)



# Install PMM with Kubernetes HA (clustered)

Deploy PMM with zero-downtime high availability featuring multiple active instances, distributed databases, and automatic load balancing for enterprise monitoring environments.

!!! warning "Technical Preview - Do NOT use in production"
    This deployment option is in **Technical Preview** and is **NOT intended for production use**. 
    
    **What this means:**
    
    - This is an early-access release for community testing and feedback
    - [Known Issues](#known-issues-and-limitations) and limitations exist
    - Breaking changes may occur between releases
    - Features are incomplete and subject to change
    
    **We encourage early adopters to test in non-production environments and [provide feedback](#providing-feedback)** to help us improve this feature for General Availability (GA).

!!! danger "VictoriaMetrics Enterprise feature limitations"
    In this Tech Preview, PMM does not support the following VictoriaMetrics Enterprise features. 
    
    If your monitoring strategy relies on importing Prometheus data or requires downsampling for storage optimization, evaluate these limitations before testing:
    
    - **Prometheus data file reading**: Cannot import existing Prometheus data files into PMM HA
    - **Metrics downsampling**: No automatic downsampling of historical metrics for long-term storage efficiency
    

## What is PMM HA clustered?

PMM HA Clustered provides enterprise-grade high availability through a fully distributed architecture:

- **3 PMM server replicas** with automatic leader election and instant failover
- **HAProxy load balancing** with 3 replicas for traffic distribution and redundancy  
- **Distributed databases** (ClickHouse, VictoriaMetrics, PostgreSQL) managed by Kubernetes operators
- **Sub-30-second failover** with zero monitoring data loss
- **Horizontal scalability** to handle growing monitoring demands

This deployment eliminates single points of failure and maintains continuous monitoring visibility even during infrastructure failures, planned maintenance, or scaling operations.

## Quick Start

Get PMM HA Cluster running in 10 minutes:

**Prerequisites:**
- Kubernetes 1.22+
- Helm 3.2.0+
- kubectl configured
- PV provisioner support

**Installation:**
{.power-number}

1. Add Percona Helm repository:
   ```sh
   helm repo add percona https://percona.github.io/percona-helm-charts/
   helm repo update
   ```

2. Create namespace:
   ```sh
   kubectl create namespace pmm
   ```

3. Install required Kubernetes operators:
   ```sh
   helm install pmm-operators percona/pmm-ha-dependencies --namespace pmm
   
   # Wait for all operators to be ready (this may take 2-3 minutes)
   kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=victoria-metrics-operator -n pmm --timeout=300s
   kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=altinity-clickhouse-operator -n pmm --timeout=300s
   kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=pg-operator -n pmm --timeout=300s
   ```

4. Create PMM secret with your passwords:
   ```sh
   kubectl create secret generic pmm-secret \
     --from-literal=PMM_ADMIN_PASSWORD="your-secure-password" \
     --from-literal=PMM_CLICKHOUSE_USER="clickhouse_pmm" \
     --from-literal=PMM_CLICKHOUSE_PASSWORD="clickhouse-password" \
     --from-literal=VMAGENT_remoteWrite_basicAuth_username="victoriametrics_pmm" \
     --from-literal=VMAGENT_remoteWrite_basicAuth_password="vm-password" \
     --from-literal=PG_PASSWORD="postgres-password" \
     --from-literal=GF_PASSWORD="grafana-password" \
     --namespace pmm
   ```

5. Install PMM HA:
   ```sh
   helm install pmm-ha percona/pmm-ha --namespace pmm
   ```

6. Wait for deployment to complete:
   ```sh
   kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=pmm -n pmm --timeout=600s
   ```

7. Access PMM UI:
   ```sh
   # Port-forward for local access
   kubectl port-forward -n pmm svc/pmm-ha-haproxy 8443:443
   
   # Open https://localhost:8443 in your browser
   # Login: admin / your-secure-password
   ```

For configuration options, see the [Full installation](#installation).

## Architecture

### Two-Chart Architecture

PMM HA uses a **two-chart architecture** to separate operator management from PMM deployment. Separating operators from PMM allows independent lifecycle management, easier upgrades, and cleaner uninstalls.

1. **pmm-ha-dependencies** chart:
   - installs three Kubernetes operators
   - must be installed **before** the PMM HA chart
   - manages Custom Resource Definitions (CRDs)
   - provides: VictoriaMetrics Operator, Altinity ClickHouse Operator, Percona PostgreSQL Operator

2. **pmm-ha** chart (this deployment):
   - deploys PMM server replicas
   - creates operator-managed database resources
   - deploys HAProxy load balancer
   - requires operators from step 1

### Architecture components

#### PMM Server Cluster
- 3 PMM server replicas (configurable)
- Automatic leader election via Raft consensus
- Gossip protocol for failure detection
- Active leader handles all write operations
- Followers ready for immediate takeover

#### HAProxy Load Balancer
- 3 HAProxy replicas for redundancy
- Pod anti-affinity ensures distribution across nodes
- Health checks via `/v1/leaderHealthCheck` endpoint
- Automatic routing to current leader
- Traffic failover in < 30 seconds

#### Operator-Managed Databases

**ClickHouse Cluster** (Altinity ClickHouse Operator):
- 3 replicas with ClickHouse Keeper for coordination
- Stores Query Analytics (QAN) metrics
- Distributed queries across replicas
- Automatic failover and replication

**VictoriaMetrics Cluster** (VictoriaMetrics Operator):
- Distributed metrics storage
- vminsert, vmselect, vmstorage components
- Horizontal scalability
- Configurable replication factor

**PostgreSQL Cluster** (Percona PostgreSQL Operator):
- HA cluster for Grafana metadata
- Automatic failover with Patroni
- Streaming replication
- Point-in-time recovery support

### Architecture diagram

For a visual representation of the PMM HA architecture, see the [Miro architecture diagram](https://miro.com/app/board/uXjVGfZtx_c=/).

## What's different from single-instance Kubernetes

| Feature | Single-Instance K8s (GA) | HA Cluster K8s (Tech Preview) |
|---------|--------------------------|-------------------------------|
| **PMM Server instances** | 1 pod | 3 pods with leader election |
| **Failover time** | 2-5 minutes | < 30 seconds |
| **Data loss during failover** | None (client-side caching) | None (distributed storage) |
| **Load balancing** | Not applicable | HAProxy with 3 replicas |
| **Database architecture** | Built-in (single container) | External clustered (operators) |
| **Production status** | ✅ GA | ⚠️ Tech Preview |
| **Complexity** | Low | High |
| **When server fails** | Pod reschedules to new node | Traffic redirects to follower |
| **Resource overhead** | 1x + K8s | 3-5x + operators + K8s |
| **Kubernetes operators required** | None | 3 operators |
| **Setup time** | 5 minutes | 15-20 minutes |

## Why use HA Cluster over single-instance?

**Advantages**:
- **Zero-downtime failover**: Traffic automatically redirects to followers in < 30 seconds
- **True high availability**: Multiple active PMM instances eliminate single point of failure
- **Distributed databases**: ClickHouse, VictoriaMetrics, and PostgreSQL clusters for resilience
- **Horizontal scalability**: Add more replicas as monitoring load increases
- **Load distribution**: HAProxy distributes traffic across multiple instances

**Disadvantages**:
- **Complexity**: Requires managing three Kubernetes operators and multiple database clusters
- **Resource overhead**: Minimum 3x the resources of single-instance deployment
- **Technical Preview status**: Known bugs and limitations (see below)
- **Not production-ready**: Subject to breaking changes
- **Higher operational cost**: More components to monitor and maintain

## Prerequisites

### Required Software

- **Kubernetes**: 1.22 or higher
- **Helm**: 3.2.0 or higher
- **kubectl**: Configured to access your cluster
- **Persistent Volume Provisioner**: Available in your cluster

### Required Kubernetes Operators

PMM HA requires three Kubernetes operators to manage distributed database resources:

- **VictoriaMetrics Operator** (v0.56.4+): Manages VictoriaMetrics cluster for metrics storage
- **Altinity ClickHouse Operator** (v0.25.4+): Manages ClickHouse cluster for QAN data
- **Percona PostgreSQL Operator** (v2.8.0+): Manages PostgreSQL cluster for Grafana metadata

These operators can be installed via the `pmm-ha-dependencies` chart (recommended) or manually.

### Platform compatibility

!!! note "Platform Testing Status"
    Only **Amazon EKS** has been extensively tested. Other platforms (Google GKE, Azure AKS, on-premise Kubernetes, OpenShift) may work but have not been validated in this Tech Preview release.

| Platform | Kubernetes Version | Status | Notes |
|----------|-------------------|--------|-------|
| **Amazon EKS** | 1.24+ | ✅ Tested | Primary testing platform |
| **Google GKE** | 1.24+ | ⚠️ Not tested | Should work but not verified |
| **Azure AKS** | 1.24+ | ⚠️ Not tested | Should work but not verified |
| **On-Premise** | 1.22+ | ⚠️ Not tested | Requires LoadBalancer support |
| **OpenShift** | 4.10+ | ⚠️ Not tested | May require security context adjustments |
| **VMware Tanzu** | - | ❌ Not supported | Not tested, may have compatibility issues |

### Resource Requirements

Minimum recommended resources for a 3-replica PMM HA deployment:

| Component | CPU | Memory | Storage | Notes |
|-----------|-----|--------|---------|-------|
| **Per PMM Server pod** | 2 cores | 4 GB | 50 GB | Storage varies with retention period |
| **HAProxy (3 replicas)** | 1 core | 2 GB | - | Load balancer overhead |
| **ClickHouse cluster** | 3-6 cores | 8-12 GB | 20+ GB | Scales with QAN volume |
| **VictoriaMetrics cluster** | 2-4 cores | 4-8 GB | 20+ GB | Scales with metrics volume |
| **PostgreSQL cluster** | 1-2 cores | 2-4 GB | 10 GB | Grafana metadata storage |
| **Kubernetes operators** | 1-2 cores | 2-4 GB | - | VictoriaMetrics, ClickHouse, PostgreSQL operators |
| **Total cluster minimum** | **10-20 cores** | **20-40 GB** | **100+ GB** | Varies with scale and retention |
See [Resource Planning](#resource-planning) for detailed sizing guidance.

#### Step 5: Install PMM HA

=== "Default Installation"

    Install PMM HA with default settings:
```sh
    helm install pmm-ha percona/pmm-ha --namespace pmm
```

=== "Custom configuration"

    Create a `values.yaml` file with your custom settings:
  ```yaml
    # Example custom values
    replicas: 3  # Number of PMM server replicas

    haproxy:
      service:
        type: LoadBalancer  # Change to LoadBalancer for external access

    storage:
      size: 100Gi  # Adjust storage size as needed
  ```

    Install with custom values:
  ```sh
    helm install pmm-ha percona/pmm-ha --namespace pmm -f values.yaml
  ```

#### Step 6: Verify installation
```sh
# Check PMM server pods
kubectl get pods -l app.kubernetes.io/name=pmm -n pmm

# Check HAProxy pods
kubectl get pods -l app.kubernetes.io/name=haproxy -n pmm

# Check operator-managed resources
kubectl get vmcluster,postgrescluster,clickhouseinstallation -n pmm

# Wait for all PMM pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=pmm -n pmm --timeout=600s
```

### Option B: Installing operators manually (Advanced)

If you need more control over operator configurations or already have these operators installed cluster-wide:

=== "VictoriaMetrics operator"

    **Step 1: Install VictoriaMetrics Operator**
    {.power-number}

    1. Add the VictoriaMetrics Helm repository:
    ```sh
       helm repo add vm https://victoriametrics.github.io/helm-charts/
       helm repo update
    ```

    2. Install the operator:
    ```sh
       helm install victoria-metrics-operator vm/victoria-metrics-operator \
         --namespace pmm \
         --create-namespace \
         --set admissionWebhooks.enabled=true
    ```

    3. Verify installation:
    ```sh
       kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=victoria-metrics-operator -n pmm --timeout=300s
    ```

=== "ClickHouse operator"

    **Step 2: Install Altinity ClickHouse Operator**
    {.power-number}

    1. Add the Altinity Helm repository:
    ```sh
       helm repo add altinity https://helm.altinity.com
       helm repo update
    ```

    2. Install the operator:
    ```sh
       helm install clickhouse-operator altinity/altinity-clickhouse-operator \
         --namespace pmm
    ```

    3. Verify installation:
    ```sh
       kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=altinity-clickhouse-operator -n pmm --timeout=300s
    ```

=== "PostgreSQL Operator"

    **Step 3: Install Percona PostgreSQL Operator**

    1. Install the operator:
```sh
       helm install postgres-operator percona/pg-operator \
         --namespace pmm
```

    2. Verify installation:
    ```sh
       kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=pg-operator -n pmm --timeout=300s
    ```

=== "Install PMM HA"

    **Step 4: Create PMM Secret and Install PMM HA**

    After all three operators are installed and ready, follow the instructions from Option A:
    
    - [Step 4: Create PMM Secret](#step-4-create-pmm-secret)
    - [Step 5: Install PMM HA](#step-5-install-pmm-ha)

## Configuration

### Service endpoints

PMM HA provides the following service endpoints:

| Service | Description | Port | Use For |
|---------|-------------|------|---------|
| `pmm-ha-haproxy` | **Recommended** - HAProxy load balancer that routes to the active PMM leader | 443 (HTTPS) | All external clients, PMM Clients, Percona Operators |
| `monitoring-service` | Headless service for direct PMM pod access (used internally) | 8443 (HTTPS) | Internal cluster communication only |

!!! note "Service endpoint recommendation"
    Always use `pmm-ha-haproxy` as your PMM server endpoint for all external connections. The `monitoring-service` is for internal cluster communication only.

### External access configuration

By default, HAProxy is only accessible within the Kubernetes cluster. To enable external access, configure the HAProxy service type.

=== "LoadBalancer (Recommended)"

    **Best for**: Cloud environments (AWS, GCP, Azure)

    The LoadBalancer service type automatically provisions a cloud load balancer with a public IP/DNS.

    **Configuration:**
```yaml
    # values.yaml
    haproxy:
      service:
        type: LoadBalancer
```

    **Apply the configuration:**
```sh
    helm upgrade pmm-ha percona/pmm-ha --namespace pmm --set haproxy.service.type=LoadBalancer
```

    **Get the external endpoint:**
```sh
    kubectl get svc pmm-ha-haproxy -n pmm

    # Look for EXTERNAL-IP column
    # Connect via: https://<EXTERNAL-IP>:443
```

    !!! tip "Cloud Provider Integration"
        LoadBalancer automatically provisions and configures a load balancer in your cloud provider. See [Cloud-Specific Configurations](#cloud-specific-configurations) for platform-specific annotations.

=== "NodePort"

    **Best for**: Bare-metal, on-premise, or testing environments

    NodePort exposes PMM on a static port on each cluster node.

    **Configuration:**
```yaml
    # values.yaml
    haproxy:
      service:
        type: NodePort
```

    **Apply the configuration:**
```sh
    helm upgrade pmm-ha percona/pmm-ha --namespace pmm --set haproxy.service.type=NodePort
```

    **Get the assigned port:**
```sh
    # Get the NodePort number
    kubectl get svc pmm-ha-haproxy -n pmm -o jsonpath='{.spec.ports[0].nodePort}'

    # Access via: https://<any-node-ip>:<nodeport>
```

    !!! warning "NodePort limitations"
        - Ports are typically in the range 30000-32767
        - You must manage firewall rules to allow access to this port
        - Any node IP can be used, but if that node fails, you need to use a different node IP
        - Consider using a LoadBalancer or Ingress for production deployments

=== "ClusterIP (Default)"

    **Best for**: Internal cluster access only

    ClusterIP makes PMM accessible only within the Kubernetes cluster. This is the default setting.

    **Configuration:**
```yaml
    # values.yaml
    haproxy:
      service:
        type: ClusterIP
```

    **Access from within the cluster:**
```sh
    # From any pod in the cluster
    curl https://pmm-ha-haproxy.pmm.svc.cluster.local:443
```

    **Port-forward for local testing:**
```sh
    kubectl port-forward -n pmm svc/pmm-ha-haproxy 8443:443

    # Access via: https://localhost:8443
```

    !!! info "When to Use ClusterIP"
        Use ClusterIP when:
        
        - PMM only needs to be accessed by applications within the cluster
        - You're using an Ingress controller for external access
        - You want maximum security by not exposing PMM externally

#### Cloud-specific configurations

Configure LoadBalancer settings optimized for your cloud provider:

=== "Amazon EKS"

    **Recommended for**: AWS deployments
```yaml
    haproxy:
      service:
        type: LoadBalancer
        annotations:
          # Use Network Load Balancer (recommended for performance)
          service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
          
          # Internal (VPC only) or internet-facing
          service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"  # or "internet-facing"
          
          # Optional: Use specific Elastic IPs for stable addresses (internet-facing only)
          # service.beta.kubernetes.io/aws-load-balancer-eip-allocations: "eipalloc-xxx,eipalloc-yyy"
```

    **Apply the configuration:**
```sh
    helm upgrade pmm-ha percona/pmm-ha --namespace pmm -f values.yaml
```

    !!! tip "AWS Best Practices"
        - Use Network Load Balancer (NLB) for better performance and lower latency
        - Use `internal` scheme for VPC-only access to keep PMM private
        - Allocate Elastic IPs in advance for stable public addresses that survive load balancer recreation

=== "Google Cloud GKE"

    **Recommended for**: GCP deployments
```yaml
    haproxy:
      service:
        type: LoadBalancer
        # Optional: Use a reserved static IP (create with: gcloud compute addresses create pmm-ip --region=REGION)
        loadBalancerIP: "35.x.x.x"
        annotations:
          # Internal LB (VPC only) - remove for external/public access
          networking.gke.io/load-balancer-type: "Internal"
```

    **Create a static IP (optional):**
```sh
    # Reserve a static IP address
    gcloud compute addresses create pmm-ip --region=us-central1
    
    # Get the IP address
    gcloud compute addresses describe pmm-ip --region=us-central1 --format="value(address)"
```

    **Apply the configuration:**
```sh
    helm upgrade pmm-ha percona/pmm-ha --namespace pmm -f values.yaml
```

    !!! tip "GCP Best Practices"
        - Reserve static IPs in advance to maintain consistent access endpoints
        - Use internal load balancer for VPC-only access
        - Regional load balancers provide better availability than zonal ones

=== "Azure AKS"

    **Recommended for**: Azure deployments
```yaml
    haproxy:
      service:
        type: LoadBalancer
        # Optional: Use a pre-created static public IP
        loadBalancerIP: "20.x.x.x"
        annotations:
          # Internal LB (VNet only) - remove for external/public access
          service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```

    **Create a static IP (optional):**
```sh
    # Create a static public IP in the same resource group as your AKS cluster
    az network public-ip create \
      --resource-group MC_myResourceGroup_myAKSCluster_eastus \
      --name pmmPublicIP \
      --sku Standard \
      --allocation-method static
    
    # Get the IP address
    az network public-ip show \
      --resource-group MC_myResourceGroup_myAKSCluster_eastus \
      --name pmmPublicIP \
      --query ipAddress \
      --output tsv
```

    **Apply the configuration:**
```sh
    helm upgrade pmm-ha percona/pmm-ha --namespace pmm -f values.yaml
```

    !!! tip "Azure best practices"
        - Create public IPs in the AKS cluster's infrastructure resource group (MC_*)
        - Use Standard SKU for production workloads
        - Internal load balancers provide better security for VNet-only access

=== "On-Premise (MetalLB)"

    **Recommended for**: Bare-metal and on-premise Kubernetes clusters
```yaml
    haproxy:
      service:
        type: LoadBalancer
        loadBalancerIP: "192.168.1.100"
        annotations:
          metallb.universe.tf/address-pool: "production-pool"
```

    **Prerequisites:**

    1. Install MetalLB in your cluster:
```sh
       kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
```

    2. Configure an IP address pool:
```yaml
       apiVersion: metallb.io/v1beta1
       kind: IPAddressPool
       metadata:
         name: production-pool
         namespace: metallb-system
       spec:
         addresses:
         - 192.168.1.100-192.168.1.110
```

    **Apply the configuration:**
```sh
    helm upgrade pmm-ha percona/pmm-ha --namespace pmm -f values.yaml
```

    !!! tip "MetalLB Best Practices"
        - Pre-allocate IP ranges that don't conflict with DHCP
        - Use Layer 2 mode for simplicity or BGP mode for advanced routing
        - Ensure your network infrastructure routes traffic to these IPs correctly

### Custom SSL certificates

PMM ships with self-signed SSL certificates. For production, provide your own certificates:

```yaml
certs:
  name: pmm-certs
  files:
    certificate.crt: |
      -----BEGIN CERTIFICATE-----
      ... your certificate ...
      -----END CERTIFICATE-----
    certificate.key: |
      -----BEGIN PRIVATE KEY-----
      ... your private key ...
      -----END PRIVATE KEY-----
    ca-certs.pem: |
      -----BEGIN CERTIFICATE-----
      ... your CA certificate ...
      -----END CERTIFICATE-----
    dhparam.pem: |
      -----BEGIN DH PARAMETERS-----
      ... your DH parameters ...
      -----END DH PARAMETERS-----
```

### Storage configuration

Configure storage size and class for PMM data:

```yaml
storage:
  name: pmm-storage
  size: 100Gi  # Adjust based on retention and monitored services
  storageClassName: "fast-ssd"  # Use your preferred storage class
```

For existing persistent volumes:

```yaml
storage:
  selector:
    matchLabels:
      app: pmm-data
```

To start from a volume snapshot:

```yaml
storage:
  dataSource:
    name: pmm-snapshot-20250117
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

### Resource limits

Configure resource requests and limits for PMM server pods:

```yaml
pmmResources:
  requests:
    cpu: "2"
    memory: "4Gi"
  limits:
    cpu: "4"
    memory: "8Gi"
```

### Helm parameters reference

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicas` | Number of PMM server replicas | `3` |
| `image.repository` | PMM server image repository | `percona/pmm-server` |
| `image.tag` | PMM server image tag | `3.5.0` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `secret.create` | Create secret automatically | `false` |
| `secret.name` | Name of the PMM secret | `pmm-secret` |
| `storage.size` | PVC size | `10Gi` |
| `storage.storageClassName` | Storage class name | `""` |
| `haproxy.replicaCount` | Number of HAProxy replicas | `3` |
| `haproxy.service.type` | HAProxy service type | `ClusterIP` |
| `clickhouse.cluster.replicas` | ClickHouse replicas | `3` |
| `victoriaMetrics.vmstorage.replicaCount` | VictoriaMetrics storage replicas | `3` |
| `pg-db.enabled` | Enable PostgreSQL cluster | `true` |
| `pg-db.pmm.enabled` | Enable automatic PMM monitoring of PostgreSQL | `true` |

For a complete list of parameters, see the [values.yaml file](https://github.com/percona/percona-helm-charts/blob/main/charts/pmm-ha/values.yaml).

## Connecting clients

### Connecting PMM Clients

To connect a PMM client to the HA cluster, use the HAProxy service endpoint:

```sh
# From within the Kubernetes cluster
pmm-admin config --server-url=https://admin:your-password@pmm-ha-haproxy:443 --server-insecure-tls

# Using service token (recommended for automation)
pmm-admin config --server-url=https://service_token:your-token@pmm-ha-haproxy:443 --server-insecure-tls
```

### PostgreSQL automatic monitoring

When `pg-db.pmm.enabled: true` (default), PostgreSQL metrics are automatically pushed to PMM:

-  A **service account token** is automatically created in PMM by the `pmm-token-init` Job
- The token is stored in the `pg-pmm-secret` Kubernetes secret
- PostgreSQL pods use this token to authenticate and push metrics to PMM via the `pmm-ha-haproxy` endpoint

**No manual configuration required** - the Percona PostgreSQL Operator handles the integration automatically.

### Retrieving Service Tokens

To retrieve the auto-generated PostgreSQL monitoring token:

```sh
kubectl get secret pg-pmm-secret -n pmm -o jsonpath='{.data.PMM_SERVER_TOKEN}' | base64 -d
```

To create additional service tokens manually, see the [PMM documentation on service accounts](https://docs.percona.com/percona-monitoring-and-management/api/authentication.html).

## Key Features

### Leader Node Identification

The PMM UI displays a badge showing the current leader PMM node and cluster health status:

- **Leader node name**: Displays which PMM instance (e.g., `pmm-ha-0`, `pmm-ha-1`, `pmm-ha-2`) is currently handling monitoring operations
- **Health status indicators**:
  - **Healthy**: All nodes in "alive" status
  - **Degraded**: ⅓ of nodes not in "alive" status  
  - **Critical**: ⅔ of nodes not in "alive" status
  - **Down**: All nodes not in "alive" status

Access via: **PMM Home Dashboard → HA Badge** (top right corner)

**Note**: Due to a known issue (see [Known Issues](#known-issues-and-limitations)), the health status may not always display correctly in this Tech Preview release.

### HA Role Information in Inventory

View detailed HA role information for all PMM nodes in the Inventory:
{.power-number}

1. Go to **PMM Configuration → Inventory**
2. In the **Services** tab, apply these filters:
   - **Service Type**: `pmm-server`
   - **Service Name**: Contains `pmm-ha`
3. The **Labels** column shows:
   - **Leader** status (which node is currently active)
   - **Follower** status (which nodes are standby)
   - **Health** status of each node

This provides a centralized view of your entire PMM HA cluster state.

## Scaling

### Scaling PMM Server Replicas

To scale PMM server replicas:

```sh
helm upgrade pmm-ha percona/pmm-ha --namespace pmm --set replicas=5
```

**Important Behavior**: When you scale PMM HA up or down, **all PMM pods will be recreated**. This happens because the `PMM_HA_PEERS` environment variable is dynamically generated based on replica count and must be updated on all pods.

**Impact:**
- Brief service interruption during pod recreation (typically < 1 minute per pod)
- HAProxy continues routing to available pods during rollout
- No data loss (distributed storage)
- Rolling update strategy minimizes downtime

### Scaling HAProxy Replicas

```sh
helm upgrade pmm-ha percona/pmm-ha --namespace pmm --set haproxy.replicaCount=5
```

### Scaling Database Components

**ClickHouse:**
```sh
helm upgrade pmm-ha percona/pmm-ha --namespace pmm --set clickhouse.cluster.replicas=5
```

**VictoriaMetrics:**
```sh
helm upgrade pmm-ha percona/pmm-ha --namespace pmm \
  --set victoriaMetrics.vmselect.replicaCount=3 \
  --set victoriaMetrics.vminsert.replicaCount=3 \
  --set victoriaMetrics.vmstorage.replicaCount=5
```

**PostgreSQL:**
PostgreSQL scaling is managed through the Percona PostgreSQL Operator. See the [operator documentation](https://docs.percona.com/percona-operator-for-postgresql/) for details.

### Pre-Pulling Images Before Upgrades

PMM images can be large (several GB). Before performing upgrades or scaling operations, pre-pull images on all nodes to avoid timeout issues:

```sh
# Get list of nodes
kubectl get nodes

# For each node, pre-pull the image (example for node1)
kubectl debug node/node1 -it --image=percona/pmm-server:3.5.0
```

## Monitoring and Troubleshooting

### Health Check Commands

Check the health of your PMM HA deployment:

```sh
# Check all PMM HA resources
kubectl get all -l app.kubernetes.io/instance=pmm-ha -n pmm

# Check PMM server pods
kubectl get pods -l app.kubernetes.io/name=pmm -n pmm

# Check HAProxy pods
kubectl get pods -l app.kubernetes.io/name=haproxy -n pmm

# Check ClickHouse cluster
kubectl get clickhouseinstallation -n pmm
kubectl get pods -l clickhouse.altinity.com/app=chop -n pmm

# Check VictoriaMetrics cluster
kubectl get vmcluster,vmagent,vmauth -n pmm

# Check PostgreSQL cluster
kubectl get postgrescluster -n pmm
kubectl get pods -l postgres-operator.crunchydata.com/cluster -n pmm

# View PMM server logs
kubectl logs -l app.kubernetes.io/name=pmm -n pmm --tail=100

# View HAProxy logs
kubectl logs -l app.kubernetes.io/name=haproxy -n pmm --tail=100
```

### Access PMM UI via Port-Forward

For local testing or troubleshooting:

```sh
kubectl port-forward -n pmm svc/pmm-ha-haproxy 8443:443

# Visit https://localhost:8443 in your browser
```

### Common Issues

**Issue**: Pods stuck in `Pending` state
**Solution**: Check PV provisioner and ensure sufficient cluster resources

**Issue**: PMM not accessible after install
**Solution**: Verify HAProxy service is running and has endpoints:
```sh
kubectl get svc pmm-ha-haproxy -n pmm
kubectl get endpoints pmm-ha-haproxy -n pmm
```

**Issue**: High memory usage
**Solution**: Adjust resource limits and check retention settings:
```sh
kubectl top pods -n pmm
```

### Upgrading

PMM HA uses rolling updates for zero-downtime upgrades. Each pod updates sequentially while HAProxy maintains traffic flow.

=== "Standard upgrade"

    Upgrade to the latest PMM HA version with default settings.
    {.power-number}

    1. Update Helm repository:  
    ```sh
       helm repo update percona
    ```

    2. (Optional but recommended) Pre-pull new images on all nodes to avoid timeouts

    3. Upgrade PMM HA:
    ```sh
       helm upgrade pmm-ha percona/pmm-ha --namespace pmm
    ```

    4. Monitor the rollout:
    ```sh
       kubectl rollout status statefulset pmm-ha -n pmm
    ```

=== "Upgrade with custom values"

    To upgrade while preserving or updating your custom configuration:
    {.power-number}

    1. Update Helm repository:
    ```sh
       helm repo update percona
    ```

    2. (Optional but recommended) Pre-pull new images on all nodes to avoid timeouts

    3. Upgrade with your values file:
    ```sh
       helm upgrade pmm-ha percona/pmm-ha --namespace pmm -f values.yaml
    ```

    4. Monitor the rollout:
    ```sh
       kubectl rollout status statefulset pmm-ha -n pmm
    ```

    !!! tip "Preserving Custom Configuration"
        Always use the same `values.yaml` file (or flags) that you used during installation to avoid losing custom settings during upgrades.

### Rollback

If an upgrade fails, rollback to the previous version:

```sh
helm rollback pmm-ha --namespace pmm
```

## Resource planning

### Sizing guidelines

| Monitored Services | PMM Replicas | ClickHouse Replicas | VictoriaMetrics Storage | Total CPU | Total Memory | Total Storage |
|-------------------|--------------|---------------------|------------------------|-----------|--------------|---------------|
| 1-10 | 3 | 3 | 3 | 10 cores | 20 GB | 100 GB |
| 11-50 | 3 | 3 | 3 | 15 cores | 30 GB | 200 GB |
| 51-100 | 3 | 3 | 5 | 20 cores | 40 GB | 500 GB |
| 100+ | 5 | 5 | 5 | 30+ cores | 60+ GB | 1+ TB |

**Factors affecting resource usage:**
- Number of monitored database instances
- Metrics resolution and retention period
- Query Analytics (QAN) volume
- Number of concurrent users
- Custom dashboards and queries

## Known issues and limitations

### Scaling limitations

!!! danger "Scaling Down to Single Replica"
    When scaling down to a single PMM replica (from 3 to 1), ensure the **Raft leader is on pmm-0** before scaling. Kubernetes StatefulSets remove pods in reverse ordinal order (highest first).
    
    - Scaling 3→1 removes pmm-2 and pmm-1, keeping only pmm-0
    - **If the Raft leader is on pmm-1 or pmm-2 when you scale down, PMM will become unreachable**
    
    **Workaround**: Check leader status before scaling:
    ```sh
    kubectl exec -it pmm-ha-0 -n pmm -- pmm-admin status
    ```
    
    Only scale down after confirming pmm-0 is the leader.


### VictoriaMetrics enterprise limitations

PMM HA Tech Preview does not support these VictoriaMetrics Enterprise features:

- **Prometheus data file reading**: Cannot import existing Prometheus data files into PMM HA
- **Metrics downsampling**: No automatic downsampling of historical metrics for long-term storage efficiency

**Impact**: If you currently rely on these features, plan accordingly for your monitoring strategy. 


### Known issues

The following bugs are tracked and will be fixed before General Availability:

#### [PMM-14704](https://perconadev.atlassian.net/browse/PMM-14704): Node dropdown shows PostgreSQL instances incorrectly

Node selector dropdown includes PostgreSQL database instances in addition to PMM server nodes, making the UI confusing.

**Workaround**: Select only nodes named `pmm-ha-0`, `pmm-ha-1`, or `pmm-ha-2`.
#### [PMM-14705](https://perconadev.atlassian.net/browse/PMM-14705): Services added via pmm-admin don't display dashboard data

Services registered through `pmm-admin` command-line tool appear with UNSPECIFIED status. QAN works correctly, but metrics dashboards show no data.

**Workaround**: Add all services through the PMM UI instead of command-line.
#### [PMM-14706](https://perconadev.atlassian.net/browse/PMM-14706): PostgreSQL nodes have extra 'pmm-' prefix in names

PostgreSQL cluster nodes appear with names like `pmm-pmm-ha-pg-ha1-xxxx` instead of `pmm-ha-pg-ha1-xxxx`. This is cosmetic only and doesn't affect monitoring functionality.

**Workaround**: None needed - monitoring works correctly.
#### [PMM-14707](https://perconadev.atlassian.net/browse/PMM-14707): PostgreSQL monitoring shows incorrect status

PostgreSQL monitoring status appears as FAILED or UNSPECIFIED in the inventory, even though metrics are being collected successfully.

**Workaround**: Ignore the status display and verify metrics are flowing by checking PostgreSQL dashboards.


#### HA health badge shows incorrect status

The HA badge on the PMM Home Dashboard may not accurately reflect the true cluster health.

**Workaround**: Use Inventory view or kubectl commands to check actual cluster status.

For a complete list of tracked issues, see the [PMM HA bugs in Jira](https://perconadev.atlassian.net/issues/?jql=parent%3DPMM-14338%20and%20issuetype%3DBug%20and%20status%20not%20in%20(done%2C%20%22Pending%20Release%22)%20ORDER%20BY%20rank).

### Platform support

- **Fully tested**: Amazon EKS only
- **Not tested**: Google GKE, Azure AKS, on-premise Kubernetes, OpenShift
- **Not supported**: VMware Tanzu

**Recommendation**: Only deploy on Amazon EKS for this Tech Preview release. Other platforms may work but have not been validated.

## Uninstall PMM HA

Remove PMM HA and all associated resources from your Kubernetes cluster.

!!! danger "Critical: Follow the Correct Uninstall Order"
    Uninstalling in the **wrong order** will leave orphaned Custom Resources that cannot be cleaned up automatically.
    
    **Required sequence:**
    {.power-number}

    1. Uninstall PMM HA chart
    2. Wait for operator-managed resources to be removed
    3. Uninstall operators
    4. (Optional) Delete CRDs
    5. (Optional) Remove persistent volumes

### Uninstall Procedure

**1. Uninstall PMM HA**

Remove the PMM HA deployment:
```sh
helm uninstall pmm-ha --namespace pmm
```

This initiates the cleanup process. Operators will automatically begin removing their managed resources.

**2. Wait for Resource Cleanup**

Wait for operator-managed resources to be fully removed before proceeding:
```sh
# Wait for VictoriaMetrics resources
kubectl wait --for=delete vmcluster -l app.kubernetes.io/instance=pmm-ha -n pmm --timeout=300s

# Wait for PostgreSQL resources
kubectl wait --for=delete postgrescluster -l app.kubernetes.io/instance=pmm-ha -n pmm --timeout=300s

# Wait for ClickHouse resources
kubectl wait --for=delete clickhouseinstallation -l app.kubernetes.io/instance=pmm-ha -n pmm --timeout=300s
```

!!! tip "Verify Cleanup"
    If the wait commands timeout, check for remaining resources:
```sh
    kubectl get vmcluster,postgrescluster,clickhouseinstallation -n pmm
```

**3. Uninstall Operators**

Choose the method based on how you installed the operators:

**If you used pmm-ha-dependencies chart:**
```sh
helm uninstall pmm-operators --namespace pmm
```

**If you installed operators manually:**
```sh
helm uninstall victoria-metrics-operator --namespace pmm
helm uninstall clickhouse-operator --namespace pmm
helm uninstall postgres-operator --namespace pmm
```

**4. Delete CRDs (Optional)**

!!! danger "WARNING: Cluster-Wide Deletion and Permanent Data Loss"
    This step removes CRDs **cluster-wide**, deleting **ALL** custom resources of these types across **ALL** namespaces.
    
    **Only do this if:**
    
    - You're completely removing these operators from the entire cluster
    - You have **no other deployments** using these operators anywhere
    - You understand this causes **permanent, irreversible data loss**

Delete CRDs (if you're certain):
```sh
# Remove VictoriaMetrics CRDs
kubectl delete crds $(kubectl get crds -o name | grep victoriametrics)

# Remove ClickHouse CRDs
kubectl delete crds $(kubectl get crds -o name | grep clickhouse)

# Remove PostgreSQL Operator CRDs
kubectl delete crds $(kubectl get crds -o name | grep -E "(postgres-operator|perconapg)")
```

!!! warning "Check Before Deleting"
    List all resources using these CRDs across your cluster first:
```sh
    kubectl get vmcluster,postgrescluster,clickhouseinstallation --all-namespaces
```

**5. Remove Storage (Optional)**

!!! warning "Permanent Data Loss"
    This permanently deletes all PMM monitoring data, including historical metrics, Query Analytics data, custom dashboards, configuration settings, and user accounts.

**Option A: Delete only PMM PVCs**
```sh
# List PVCs to review
kubectl get pvc -n pmm

# Delete PMM-related PVCs
kubectl delete pvc -l app.kubernetes.io/instance=pmm-ha -n pmm
```

**Option B: Delete entire namespace**
```sh
# This removes everything in the namespace
kubectl delete namespace pmm
```

!!! danger "Namespace Deletion"
    Deleting the namespace removes all resources within it, including any non-PMM resources you may have added.

### Verification

After uninstalling, verify all resources are removed:
```sh
# Check for remaining PMM resources
kubectl get all -n pmm

# Check for remaining CRDs (if you deleted them)
kubectl get crds | grep -E "(victoriametrics|clickhouse|postgres-operator|perconapg)"

# Check for remaining PVCs
kubectl get pvc -n pmm
```

### Troubleshooting Uninstall Issues

**Resources stuck in "Terminating" state**

Some resources may have finalizers preventing deletion. Remove finalizers manually:
```sh
# Remove finalizer from a stuck resource
kubectl patch <resource-type> <resource-name> -n pmm -p '{"metadata":{"finalizers":[]}}' --type=merge
```

**Namespace stuck in "Terminating" state**

Check for resources with finalizers:
```sh
# List resources in the namespace
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -n 1 kubectl get --show-kind --ignore-not-found -n pmm
```

## Providing feedback

This Tech Preview release is designed to gather community feedback before General Availability. We encourage you to:

### Report issues
- [PMM Jira Issue Tracker](https://perconadev.atlassian.net/jira/software/c/projects/PMM/issues/) - Report bugs and technical issues
- [Percona Community Forum](https://forums.percona.com/c/percona-monitoring-and-management-pmm/) - Discuss features and ask questions

**Share feedback:**
- What works well in your environment?
- What's challenging or confusing?
- What features are you missing?
- How does performance compare to single-instance deployments?

**Get help:**
- [Community Slack](https://percona.community/join-percona-slack) - Real-time chat with engineers and community
- [Percona Support](https://www.percona.com/services/support) - Enterprise support for production issues

Your feedback directly influences the feature set and improvements for the GA release!

## Additional resources

- [PMM HA Helm Chart on GitHub](https://github.com/percona/percona-helm-charts/tree/main/charts/pmm-ha)
- [PMM Documentation](https://docs.percona.com/percona-monitoring-and-management/)
- [Percona Kubernetes Operators](https://www.percona.com/software/percona-kubernetes-operators)
- [VictoriaMetrics Operator Documentation](https://docs.victoriametrics.com/operator/)
- [Altinity ClickHouse Operator Documentation](https://docs.altinity.com/operatorforkubernetes/)
- [Percona PostgreSQL Operator Documentation](https://docs.percona.com/percona-operator-for-postgresql/)