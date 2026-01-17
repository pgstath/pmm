# Install PMM with Kubernetes HA (clustered)

!!! warning "Technical Preview - Do NOT use in production"
    This deployment option is in **Technical Preview** and is **NOT intended for production use**. This is an early release to gather feedback from the community. Expect known bugs, breaking changes, and incomplete features.
    
    **We encourage early adopters to test this feature and provide feedback** to help us improve it for General Availability (GA).
    
    **Known limitations**: See the [Known Issues and Limitations](#known-issues-and-limitations) section for details on current bugs and limitations.

!!! danger "VictoriaMetrics Enterprise Limitations"
    PMM HA Tech Preview does not support these VictoriaMetrics Enterprise features:
    
    - **Prometheus data file reading**: Cannot import existing Prometheus data files
    - **Metrics downsampling**: No automatic downsampling of historical metrics
    
    These limitations may impact your monitoring strategy if you rely on these features.

Zero-downtime high availability with multiple active PMM instances, distributed databases, and automatic load balancing.

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

Continue to the [Full Installation Guide](#installation) for production configuration options.

## Overview

PMM HA Cluster on Kubernetes provides enterprise-grade high availability through a clustered deployment with:

- 3 PMM server replicas with automatic leader election via Raft consensus
- HAProxy load balancing with 3 replicas for traffic distribution
- External clustered databases managed by Kubernetes operators
- Sub-30-second failover time
- Zero downtime during planned and unplanned outages

## Architecture

### Two-Chart Architecture

PMM HA uses a **two-chart architecture** to separate operator management from PMM deployment:

1. **pmm-ha-dependencies** chart:
   - Installs three Kubernetes operators
   - Must be installed **before** the PMM HA chart
   - Manages Custom Resource Definitions (CRDs)
   - Provides: VictoriaMetrics Operator, Altinity ClickHouse Operator, Percona PostgreSQL Operator

2. **pmm-ha** chart (this deployment):
   - Deploys PMM server replicas
   - Creates operator-managed database resources
   - Deploys HAProxy load balancer
   - Requires operators from step 1

**Why this split?** Separating operators from PMM allows independent lifecycle management, easier upgrades, and cleaner uninstalls.

### Architecture Components

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

### Architecture Diagram

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

### Platform Compatibility

| Platform | Kubernetes Version | Status | Notes |
|----------|-------------------|--------|-------|
| **Amazon EKS** | 1.24+ | ✅ Tested | Primary testing platform |
| **Google GKE** | 1.24+ | ⚠️ Not tested | Should work but not verified |
| **Azure AKS** | 1.24+ | ⚠️ Not tested | Should work but not verified |
| **On-Premise** | 1.22+ | ⚠️ Not tested | Requires LoadBalancer support |
| **OpenShift** | 4.10+ | ⚠️ Not tested | May require security context adjustments |
| **VMware Tanzu** | - | ❌ Not supported | Not tested, may have compatibility issues |

**Important**: Only Amazon EKS has been extensively tested. Other platforms may work but have not been validated in this Tech Preview release.

### Resource Requirements

Minimum recommended resources for a 3-replica PMM HA deployment:

**Per PMM Server Pod:**
- CPU: 2 cores
- Memory: 4 GB
- Storage: 50 GB (varies with retention)

**Total Cluster Resources:**
- CPU: 10-20 cores (PMM + operators + databases)
- Memory: 20-40 GB
- Storage: 100+ GB across all components

See [Resource Planning](#resource-planning) for detailed sizing guidance.

## Installation

### Option A: Using pmm-ha-dependencies Chart (Recommended)

This is the simplest installation method. The `pmm-ha-dependencies` chart installs all required operators in one step.

#### Step 1: Add Helm Repository
{.power-number}

1. Add the Percona Helm repository:
   ```sh
   helm repo add percona https://percona.github.io/percona-helm-charts/
   helm repo update
   ```

2. Verify the repository was added:
   ```sh
   helm search repo percona/pmm-ha
   ```

#### Step 2: Create Namespace

```sh
kubectl create namespace pmm
```

#### Step 3: Install Operators

```sh
helm install pmm-operators percona/pmm-ha-dependencies --namespace pmm
```

This installs:
- VictoriaMetrics Operator
- Altinity ClickHouse Operator  
- Percona PostgreSQL Operator

Wait for all operators to be ready:

```sh
# Wait for VictoriaMetrics Operator
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=victoria-metrics-operator -n pmm --timeout=300s

# Wait for ClickHouse Operator
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=altinity-clickhouse-operator -n pmm --timeout=300s

# Wait for PostgreSQL Operator
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=pg-operator -n pmm --timeout=300s
```

#### Step 4: Create PMM Secret

**Important**: The `secret.create` parameter is set to `false` by default in the Helm chart. You **must** create the `pmm-secret` manually before installing PMM HA.

**Why?** This prevents Helm from overwriting your secrets during upgrades and keeps sensitive credentials out of your `values.yaml` file.

Create the secret using kubectl:

```sh
kubectl create secret generic pmm-secret \
  --from-literal=PMM_ADMIN_PASSWORD="your-secure-password" \
  --from-literal=PMM_CLICKHOUSE_USER="clickhouse_pmm" \
  --from-literal=PMM_CLICKHOUSE_PASSWORD="your-clickhouse-password" \
  --from-literal=VMAGENT_remoteWrite_basicAuth_username="victoriametrics_pmm" \
  --from-literal=VMAGENT_remoteWrite_basicAuth_password="your-vm-password" \
  --from-literal=PG_PASSWORD="your-postgres-password" \
  --from-literal=GF_PASSWORD="your-grafana-password" \
  --namespace pmm
```

Or create a YAML file (`pmm-secret.yaml`):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pmm-secret
  namespace: pmm
type: Opaque
stringData:
  PMM_ADMIN_PASSWORD: "your-secure-password"
  PMM_CLICKHOUSE_USER: "clickhouse_pmm"
  PMM_CLICKHOUSE_PASSWORD: "your-clickhouse-password"
  VMAGENT_remoteWrite_basicAuth_username: "victoriametrics_pmm"
  VMAGENT_remoteWrite_basicAuth_password: "your-vm-password"
  PG_PASSWORD: "your-postgres-password"
  GF_PASSWORD: "your-grafana-password"
```

Apply it:

```sh
kubectl apply -f pmm-secret.yaml
```

#### Step 5: Install PMM HA

```sh
helm install pmm-ha percona/pmm-ha --namespace pmm
```

For custom configuration, create a `values.yaml` file:

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

#### Step 6: Verify Installation

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

### Option B: Installing Operators Manually (Advanced)

For users who need more control over operator configurations or already have these operators installed cluster-wide.

#### Step 1: Install VictoriaMetrics Operator
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

#### Step 2: Install Altinity ClickHouse Operator

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

#### Step 3: Install Percona PostgreSQL Operator

1. Install the operator:
   ```sh
   helm install postgres-operator percona/pg-operator \
     --namespace pmm
   ```

2. Verify installation:
   ```sh
   kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=pg-operator -n pmm --timeout=300s
   ```

#### Step 4: Create PMM Secret and Install PMM HA

Follow [Step 4](#step-4-create-pmm-secret) and [Step 5](#step-5-install-pmm-ha) from Option A above.

## Configuration

### Service Endpoints

PMM HA provides the following service endpoints:

| Service | Description | Port | Use For |
|---------|-------------|------|---------|
| `pmm-ha-haproxy` | **Recommended** - HAProxy load balancer that routes to the active PMM leader | 443 (HTTPS) | All external clients, PMM Clients, Percona Operators |
| `monitoring-service` | Headless service for direct PMM pod access (used internally) | 8443 (HTTPS) | Internal cluster communication only |

**Important**: Always use `pmm-ha-haproxy` as your PMM server endpoint for all external connections.

### External Access Configuration

By default, HAProxy is only accessible within the Kubernetes cluster. To enable external access, configure the HAProxy service type.

#### Using LoadBalancer (Recommended for Cloud)

The LoadBalancer service type automatically provisions a cloud load balancer with a public IP/DNS:

```yaml
# values.yaml
haproxy:
  service:
    type: LoadBalancer
```

Apply the configuration:

```sh
helm upgrade pmm-ha percona/pmm-ha --namespace pmm --set haproxy.service.type=LoadBalancer
```

Get the external endpoint:

```sh
kubectl get svc pmm-ha-haproxy -n pmm

# Look for EXTERNAL-IP column
# Connect via: https://<EXTERNAL-IP>:443
```

#### Using NodePort (For Bare-Metal or Testing)

```yaml
# values.yaml
haproxy:
  service:
    type: NodePort
```

Access PMM via any node's IP address:

```sh
# Get the assigned NodePort
kubectl get svc pmm-ha-haproxy -n pmm -o jsonpath='{.spec.ports[0].nodePort}'

# Access via: https://<any-node-ip>:<nodeport>
```

#### Cloud-Specific Configurations

##### Amazon EKS

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

##### Google Cloud GKE

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

##### Azure AKS

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

##### On-Premise with MetalLB

```yaml
haproxy:
  service:
    type: LoadBalancer
    loadBalancerIP: "192.168.1.100"
    annotations:
      metallb.universe.tf/address-pool: "production-pool"
```

### Custom SSL Certificates

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

### Storage Configuration

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

### Resource Limits

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

### Helm Parameters Reference

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

## Connecting Clients

### Connecting PMM Clients

To connect a PMM client to the HA cluster, use the HAProxy service endpoint:

```sh
# From within the Kubernetes cluster
pmm-admin config --server-url=https://admin:your-password@pmm-ha-haproxy:443 --server-insecure-tls

# Using service token (recommended for automation)
pmm-admin config --server-url=https://service_token:your-token@pmm-ha-haproxy:443 --server-insecure-tls
```

### PostgreSQL Automatic Monitoring

When `pg-db.pmm.enabled: true` (default), PostgreSQL metrics are automatically pushed to PMM:

1. A **service account token** is automatically created in PMM by the `pmm-token-init` Job
2. The token is stored in the `pg-pmm-secret` Kubernetes secret
3. PostgreSQL pods use this token to authenticate and push metrics to PMM via the `pmm-ha-haproxy` endpoint

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

## Upgrading

### Standard Upgrade Procedure

1. Update Helm repository:
   ```sh
   helm repo update percona
   ```

2. (Optional) Pre-pull new images on all nodes to avoid timeouts

3. Upgrade PMM HA:
   ```sh
   helm upgrade pmm-ha percona/pmm-ha --namespace pmm
   ```

4. Monitor the rollout:
   ```sh
   kubectl rollout status statefulset pmm-ha -n pmm
   ```

**Note**: The rolling update strategy ensures zero-downtime upgrades. Pods are updated one at a time, with HAProxy automatically routing traffic to available instances.

### Upgrading with Custom Values

```sh
helm upgrade pmm-ha percona/pmm-ha --namespace pmm -f values.yaml
```

### Rollback

If an upgrade fails, rollback to the previous version:

```sh
helm rollback pmm-ha --namespace pmm
```

## Resource Planning

### Sizing Guidelines

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

## Known Issues and Limitations

### Critical Limitations

!!! danger "Scaling Down to Single Replica"
    When scaling down to a single PMM replica (from 3 to 1), ensure the **Raft leader is on pmm-0** before scaling. Kubernetes StatefulSets remove pods in reverse ordinal order (highest first).
    
    - Scaling 3→1 removes pmm-2 and pmm-1, keeping only pmm-0
    - **If the Raft leader is on pmm-1 or pmm-2 when you scale down, PMM will become unreachable**
    
    **Workaround**: Check leader status before scaling:
    ```sh
    kubectl exec -it pmm-ha-0 -n pmm -- pmm-admin status
    ```
    
    Only scale down after confirming pmm-0 is the leader.

### Known Bugs

The following bugs are tracked and will be fixed before General Availability:

**PMM-14704**: Node dropdown in PMM UI shows PostgreSQL instances incorrectly
- **Symptom**: Node selector dropdown includes PostgreSQL database instances in addition to PMM server nodes
- **Impact**: Confusing UI, but doesn't affect functionality
- **Workaround**: Select only nodes named `pmm-ha-0`, `pmm-ha-1`, or `pmm-ha-2`

**PMM-14705**: Services added via `pmm-admin` show status as UNSPECIFIED and dashboards don't display data
- **Symptom**: Services registered through `pmm-admin` command-line tool appear with UNSPECIFIED status. QAN works correctly, but metrics dashboards show no data
- **Impact**: Cannot use `pmm-admin` for service registration
- **Workaround**: Add all services through the PMM UI instead of command-line

**PMM-14706**: PostgreSQL nodes in inventory have extra 'pmm-' prefix in names
- **Symptom**: PostgreSQL cluster nodes appear with names like `pmm-pmm-ha-pg-ha1-xxxx` instead of `pmm-ha-pg-ha1-xxxx`
- **Impact**: Cosmetic issue, doesn't affect monitoring functionality
- **Workaround**: None needed, monitoring works correctly

**PMM-14707**: PostgreSQL monitoring shows FAILED or UNSPECIFIED status despite functioning correctly
- **Symptom**: PostgreSQL monitoring status appears as FAILED or UNSPECIFIED in the inventory, even though metrics are being collected successfully
- **Impact**: Status display is incorrect, but monitoring data is accurate
- **Workaround**: Ignore the status display - verify metrics are flowing by checking PostgreSQL dashboards

**Health Status Badge**: HA health badge may show incorrect status
- **Symptom**: The HA badge on the PMM Home Dashboard may not accurately reflect the true cluster health
- **Impact**: Cannot rely on the badge for health monitoring
- **Workaround**: Use Inventory view or kubectl commands to check actual cluster status

### VictoriaMetrics Enterprise Limitations

PMM HA Tech Preview does not support these VictoriaMetrics Enterprise features:

- **Prometheus data file reading**: Cannot import existing Prometheus data files into PMM HA
- **Metrics downsampling**: No automatic downsampling of historical metrics for long-term storage efficiency

**Impact**: If you currently rely on these features, plan accordingly for your monitoring strategy.

### Platform Support

- **Fully tested**: Amazon EKS only
- **Not tested**: Google GKE, Azure AKS, on-premise Kubernetes, OpenShift
- **Not supported**: VMware Tanzu

**Recommendation**: Only deploy on Amazon EKS for this Tech Preview release. Other platforms may work but have not been validated.

### All Known Issues

For a complete list of tracked issues, see the [PMM HA bugs in Jira](https://perconadev.atlassian.net/issues/?jql=parent%3DPMM-14338%20and%20issuetype%3DBug%20and%20status%20not%20in%20(done%2C%20%22Pending%20Release%22)%20ORDER%20BY%20rank).

## Uninstalling

!!! danger "Critical: Uninstall Order"
    You **must** uninstall PMM HA **before** uninstalling the operators. Uninstalling in the wrong order will leave orphaned Custom Resources that cannot be cleaned up automatically.
    
    **Correct order**:
    1. Uninstall PMM HA chart
    2. Wait for operator-managed resources to be removed
    3. Uninstall operators (pmm-ha-dependencies or individual operators)
    4. (Optional, dangerous) Delete CRDs

### Step 1: Uninstall PMM HA

```sh
helm uninstall pmm-ha --namespace pmm
```

### Step 2: Wait for Resources to be Cleaned Up

Wait for operator-managed resources to be fully removed (recommended):

```sh
# Wait for VictoriaMetrics resources
kubectl wait --for=delete vmcluster -l app.kubernetes.io/instance=pmm-ha -n pmm --timeout=300s

# Wait for PostgreSQL resources
kubectl wait --for=delete postgrescluster -l app.kubernetes.io/instance=pmm-ha -n pmm --timeout=300s

# Wait for ClickHouse resources
kubectl wait --for=delete clickhouseinstallation -l app.kubernetes.io/instance=pmm-ha -n pmm --timeout=300s
```

### Step 3: Uninstall Operators

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

### Step 4: Clean Up CRDs (Optional and Dangerous)

!!! danger "WARNING: Data Loss"
    This will remove CRDs **cluster-wide** and delete **ALL** custom resources of these types in **ALL** namespaces! This action causes **permanent data loss**.
    
    Only do this if you're completely removing the operators and have no other deployments using them anywhere in your cluster.

```sh
# Remove VictoriaMetrics CRDs
kubectl delete crds $(kubectl get crds -o name | grep victoriametrics)

# Remove ClickHouse CRDs
kubectl delete crds $(kubectl get crds -o name | grep clickhouse)

# Remove PostgreSQL Operator CRDs
kubectl delete crds $(kubectl get crds -o name | grep -E "(postgres-operator|perconapg)")
```

### Step 5: Remove Persistent Volumes (Optional)

!!! warning "Data Loss"
    This will permanently delete all PMM monitoring data, including metrics, dashboards, and configuration.

```sh
# List PVCs
kubectl get pvc -n pmm

# Delete PMM PVCs
kubectl delete pvc -l app.kubernetes.io/instance=pmm-ha -n pmm

# Delete namespace (removes everything)
kubectl delete namespace pmm
```

## Providing Feedback

This Tech Preview release is designed to gather community feedback before General Availability. We encourage you to:

**Report Issues:**
- [PMM Jira Issue Tracker](https://perconadev.atlassian.net/jira/software/c/projects/PMM/issues/) - Report bugs and technical issues
- [Percona Community Forum](https://forums.percona.com/c/percona-monitoring-and-management-pmm/) - Discuss features and ask questions

**Share Feedback:**
- What works well in your environment?
- What's challenging or confusing?
- What features are you missing?
- How does performance compare to single-instance deployments?

**Get Help:**
- [Community Slack](https://percona.community/join-percona-slack) - Real-time chat with engineers and community
- [Percona Support](https://www.percona.com/services/support) - Enterprise support for production issues

Your feedback directly influences the feature set and improvements for the GA release!

## Additional Resources

- [PMM HA Helm Chart on GitHub](https://github.com/percona/percona-helm-charts/tree/main/charts/pmm-ha)
- [PMM Documentation](https://docs.percona.com/percona-monitoring-and-management/)
- [Percona Kubernetes Operators](https://www.percona.com/software/percona-kubernetes-operators)
- [VictoriaMetrics Operator Documentation](https://docs.victoriametrics.com/operator/)
- [Altinity ClickHouse Operator Documentation](https://docs.altinity.com/operatorforkubernetes/)
- [Percona PostgreSQL Operator Documentation](https://docs.percona.com/percona-operator-for-postgresql/)