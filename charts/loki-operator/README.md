# Loki Operator Chart

## Overview

The `loki-operator` chart deploys the Red Hat Loki Operator for OpenShift, which provides log aggregation and storage capabilities using Grafana Loki. This operator enables centralized logging with high-performance log ingestion, storage, and querying capabilities, integrating seamlessly with OpenShift's logging infrastructure and providing an alternative to Elasticsearch-based logging solutions.

## Prerequisites

- OpenShift Container Platform 4.12 or later
- Cluster administrator privileges
- Sufficient cluster resources for log storage and processing
- Storage class available for persistent volumes
- Optional: S3-compatible object storage for log retention
- Optional: OpenShift Logging Operator for log collection integration

## Architecture

```mermaid
graph TD
    A[ArgoCD] -->|Deploys| B[loki-operator Chart]
    B -->|Uses| C[helper-operator]
    B -->|Uses| D[helper-status-checker]
    
    C -->|Installs| E[Loki Operator]
    D -->|Validates| E
    
    E -->|Creates| F[LokiStack CR]
    F -->|Deploys| G[Loki Components]
    
    G --> H[Distributor]
    G --> I[Ingester]
    G --> J[Querier]
    G --> K[Query Frontend]
    G --> L[Ruler]
    
    M[Vector Collector] -->|Sends Logs| H
    H -->|Routes| I
    I -->|Stores| N[Storage Backend]
    J -->|Queries| N
    K -->|Caches| J
    L -->|Alerts| O[AlertManager]
    
    N --> P[Local Storage]
    N --> Q[S3 Object Storage]
    
    R[OpenShift Console] -->|Queries| K
    S[Grafana] -->|Queries| K
    
    style A fill:#e1f5fe
    style B fill:#fff3e0
    style C fill:#f3e5f5
    style D fill:#f3e5f5
    style E fill:#e8f5e8
    style F fill:#fff9c4
```

## Installation

### Via ArgoCD (Recommended)
This chart is designed to be deployed through ArgoCD as part of the GitOps infrastructure pattern:

```yaml
# Add to cluster-config/*/infrastructure.yaml
infrastructure:
  - chart: loki-operator
    targetRevision: 0.1.0
    namespace: openshift-logging
    values:
      lokiStack:
        size: "1x.small"
        storage:
          storageClassName: "gp3-csi"
          size:
            ingester: "50Gi"
            querier: "20Gi"
        limits:
          global:
            retention: "2160h"  # 90 days
      objectStorage:
        s3:
          enabled: true
          bucketName: "cluster-logging-loki"
          region: "us-east-1"
      helper-operator:
        operators:
          loki-operator:
            subscription:
              channel: stable-5.9
              startingCSV: loki-operator.v5.9.0
```

### Manual Installation
```bash
# Add Helm repository
helm repo add rosa-hcp https://rosa-hcp-dedicated-vpc.github.io/helm-repository/
helm repo update

# Install the chart
helm install loki-operator rosa-hcp/loki-operator \
  --namespace openshift-logging \
  --create-namespace \
  --set lokiStack.size=1x.small
```

## Configuration

### Chart Dependencies
This chart uses the following helper charts:
- **helper-operator** (~1.1.0): Manages operator installation and lifecycle
- **helper-status-checker** (~4.1.2): Validates operator deployment status

### Key Configuration Parameters

#### LokiStack Configuration
```yaml
lokiStack:
  enabled: true
  name: "logging-loki"
  size: "1x.small"                    # Deployment size (1x.demo, 1x.extra-small, 1x.small, 1x.medium)
  storage:
    storageClassName: "gp3-csi"       # Storage class for Loki components
    size:
      ingester: "10Gi"                # Storage for log ingestion
      querier: "10Gi"                 # Storage for query cache
      ruler: "10Gi"                   # Storage for alerting rules
  limits:
    global:
      retention: "744h"               # Log retention period (31 days)
      ingestionRate: "4"              # MB/s per tenant
      ingestionBurstSize: "6"         # MB burst size
      maxStreamsPerUser: "10000"      # Max streams per tenant
      maxLineSize: "256000"           # Max log line size in bytes
```

#### Tenant Configuration
```yaml
lokiStack:
  tenants:
    mode: "openshift-logging"         # Tenant mode (openshift-logging, openshift-network, static)
    authentication:
      - tenantName: "application"
        tenantId: "application"
        oidc:
          secret:
            name: "application-oidc-secret"
      - tenantName: "infrastructure"
        tenantId: "infrastructure"
        oidc:
          secret:
            name: "infrastructure-oidc-secret"
```

#### Object Storage Configuration
```yaml
objectStorage:
  s3:
    enabled: true                     # Enable S3 object storage
    bucketName: "cluster-logging-loki"
    endpoint: "s3.amazonaws.com"
    region: "us-east-1"
    accessKeyId: "AKIAIOSFODNN7EXAMPLE"
    secretAccessKey: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
    secret:
      name: "loki-s3-secret"
```

#### Rules and Alerting
```yaml
lokiStack:
  rules:
    enabled: true                     # Enable Loki alerting rules
    selector:
      matchLabels:
        loki.grafana.com/name: "logging-loki"
    namespaceSelector:
      matchLabels:
        openshift.io/cluster-monitoring: "true"
```

#### Operator Configuration
```yaml
helper-operator:
  operators:
    loki-operator:
      enabled: true
      namespace:
        name: openshift-operators-redhat
        create: false
      subscription:
        channel: stable-5.9           # Operator channel
        approval: Manual             # InstallPlan approval mode
        operatorName: loki-operator
        source: redhat-operators
```

## Generated Resources

This chart creates the following Kubernetes resources:

### Operator Resources (via helper-operator)
- **Subscription**: Loki Operator subscription in `openshift-operators-redhat`
- **InstallPlan**: Operator installation plan (auto-approved if configured)

### Loki Resources
- **LokiStack**: Main custom resource for Loki deployment configuration
- **Secret**: S3 credentials (if object storage enabled)
- **Secret**: LokiStack gateway bearer token for authentication
- **ConsolePlugin**: OpenShift Console UI plugin for log viewing
- **ClusterLogging**: Integration with OpenShift logging (if enabled)

### Loki Components (created by operator)
- **Distributor**: Receives and routes log streams
- **Ingester**: Writes log data to storage
- **Querier**: Handles log queries
- **Query Frontend**: Caches and optimizes queries
- **Ruler**: Processes alerting and recording rules
- **Gateway**: Provides authentication and multi-tenancy

### RBAC Resources (via helper-status-checker)
- **ServiceAccount**: For status checking operations
- **ClusterRole**: Permissions for operator validation
- **ClusterRoleBinding**: Binding service account to cluster role

## Features

### Log Storage & Retention
- **High-Performance Ingestion**: Optimized for high-volume log ingestion
- **Configurable Retention**: Flexible retention policies per tenant
- **Compression**: Built-in log compression for storage efficiency
- **Object Storage**: S3-compatible storage for long-term retention

### Multi-Tenancy
- **Tenant Isolation**: Secure isolation between different log sources
- **RBAC Integration**: OpenShift RBAC-based access control
- **Resource Limits**: Per-tenant ingestion and query limits
- **Authentication**: OIDC-based tenant authentication

### Query Performance
- **Parallel Processing**: Distributed query execution
- **Caching**: Query result caching for improved performance
- **Index Optimization**: Efficient log indexing and searching
- **LogQL**: Powerful query language for log analysis

### Integration
- **OpenShift Logging**: Seamless integration with cluster logging
- **Grafana**: Native Grafana data source support
- **Prometheus**: Metrics and alerting integration
- **Vector**: High-performance log collection

## Deployment Sizes

### Available Sizes
- **1x.demo**: Minimal resources for demonstration (not for production)
- **1x.extra-small**: Very small deployments (< 100GB/day)
- **1x.small**: Small deployments (< 500GB/day)
- **1x.medium**: Medium deployments (< 2TB/day)

### Resource Requirements by Size

#### 1x.small (Recommended for most environments)
- **CPU**: ~2 cores total
- **Memory**: ~6 GiB total
- **Storage**: 50-100 GiB (depending on retention)

#### 1x.medium (High-volume environments)
- **CPU**: ~6 cores total
- **Memory**: ~18 GiB total
- **Storage**: 200-500 GiB (depending on retention)

## Monitoring & Verification

### Check Operator Status
```bash
# Verify operator installation
oc get csv -n openshift-operators-redhat | grep loki

# Check operator pods
oc get pods -n openshift-operators-redhat | grep loki

# Verify subscription status
oc get subscription loki-operator -n openshift-operators-redhat
```

### Verify LokiStack Deployment
```bash
# Check LokiStack resource
oc get lokistack -n openshift-logging

# Verify Loki components
oc get pods -n openshift-logging | grep loki

# Check Loki services
oc get services -n openshift-logging | grep loki

# Verify storage
oc get pvc -n openshift-logging | grep loki
```

### Query Loki
```bash
# Port forward to Loki query frontend
oc port-forward -n openshift-logging service/logging-loki-query-frontend-http 3100:3100

# Test query (in another terminal)
curl -G -s "http://localhost:3100/loki/api/v1/query" \
  --data-urlencode 'query={namespace="default"}' \
  --data-urlencode 'limit=10'
```

### Integration with OpenShift Console
1. Navigate to OpenShift Console
2. Go to "Observe" → "Logs"
3. Verify logs are being collected and stored in Loki
4. Test log queries and filtering

## Troubleshooting

### Common Issues

#### Operator Installation Failures
```bash
# Check InstallPlan status
oc get installplan -n openshift-operators-redhat

# View operator logs
oc logs -n openshift-operators-redhat deployment/loki-operator-controller-manager

# Check subscription events
oc describe subscription loki-operator -n openshift-operators-redhat
```

#### LokiStack Deployment Issues
```bash
# Check LokiStack status
oc describe lokistack logging-loki -n openshift-logging

# View Loki component logs
oc logs -n openshift-logging deployment/logging-loki-distributor
oc logs -n openshift-logging deployment/logging-loki-ingester
oc logs -n openshift-logging deployment/logging-loki-querier

# Check storage issues
oc get events -n openshift-logging | grep -i pvc
```

#### Storage Issues
```bash
# Verify storage class exists
oc get storageclass

# Check PVC status
oc describe pvc -n openshift-logging | grep -A 10 loki

# Verify S3 connectivity (if using object storage)
oc exec -n openshift-logging deployment/logging-loki-ingester -- \
  curl -s https://s3.amazonaws.com
```

#### Performance Issues
```bash
# Check resource usage
oc top pods -n openshift-logging | grep loki

# Verify ingestion rates
oc get --raw /api/v1/namespaces/openshift-logging/services/logging-loki-distributor-http:3100/proxy/metrics | grep loki_distributor

# Check query performance
oc logs -n openshift-logging deployment/logging-loki-query-frontend | grep -i slow
```

### Performance Tuning

#### Ingestion Optimization
```yaml
lokiStack:
  limits:
    global:
      ingestionRate: "8"              # Increase for higher volume
      ingestionBurstSize: "12"        # Increase burst capacity
      maxStreamsPerUser: "20000"      # Increase stream limit
```

#### Storage Optimization
```yaml
lokiStack:
  storage:
    size:
      ingester: "100Gi"               # Increase for longer local retention
      querier: "50Gi"                 # Increase for query cache
```

#### Query Optimization
```yaml
lokiStack:
  limits:
    global:
      maxQuerySeries: "500"           # Limit query complexity
      maxQueryParallelism: "32"       # Increase parallelism
```

## Security Considerations

### RBAC
- Operator requires cluster-admin privileges for installation
- Loki components run with minimal required permissions
- Tenant isolation enforced through RBAC and network policies
- S3 credentials stored securely in Kubernetes secrets

### Network Security
- Configure network policies for Loki component communication
- Secure S3 connectivity with proper IAM roles and policies
- Enable TLS for all Loki component communications
- Restrict access to Loki query endpoints

### Data Security
- Log data encryption at rest (when using encrypted storage)
- Secure tenant authentication with OIDC
- Audit logging for administrative operations
- Regular security updates through operator upgrades

## Integration Examples

### With OpenShift Logging
```yaml
# ClusterLogging configuration for Loki integration
apiVersion: logging.openshift.io/v1
kind: ClusterLogging
metadata:
  name: instance
spec:
  logStore:
    type: "lokistack"
    lokistack:
      name: "logging-loki"
  collection:
    type: "vector"
```

### With Grafana
```yaml
# Grafana data source configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
data:
  loki.yaml: |
    apiVersion: 1
    datasources:
    - name: Loki
      type: loki
      url: http://logging-loki-query-frontend-http.openshift-logging.svc:3100
      access: proxy
```

### With AlertManager
```yaml
# LogQL alerting rule example
apiVersion: loki.grafana.com/v1
kind: AlertingRule
metadata:
  name: high-error-rate
spec:
  groups:
  - name: loki-alerts
    rules:
    - alert: HighErrorRate
      expr: |
        sum(rate({namespace="production"} |= "ERROR" [5m])) > 0.1
      for: 5m
      annotations:
        summary: "High error rate detected"
```

## Related Charts

- **[helper-operator](../helper-operator/README.md)**: Operator installation automation
- **[helper-status-checker](../helper-status-checker/README.md)**: Operator status validation
- **[cluster-logging](../cluster-logging/README.md)**: OpenShift logging integration
- **[netobserv-operator](../netobserv-operator/README.md)**: Network observability with Loki
- **[cluster-bootstrap](../cluster-bootstrap/README.md)**: Bootstrap orchestration

## Version History

- **v0.1.0**: Initial release with Loki Operator deployment

## Support

For issues related to:
- **Chart functionality**: Create issues in the helm-repository
- **Operator functionality**: Consult Red Hat OpenShift documentation
- **Configuration**: Review this README and chart values documentation

## References

- [OpenShift Logging with Loki Documentation](https://docs.openshift.com/container-platform/latest/logging/cluster-logging-loki.html)
- [Loki Operator GitHub](https://github.com/abavage/openshift-logging/tree/main/loki-operator)
- [Grafana Loki Documentation](https://grafana.com/docs/loki/latest/)
- [Red Hat Loki Operator Documentation](https://docs.openshift.com/container-platform/latest/logging/cluster-logging-deploying.html)
