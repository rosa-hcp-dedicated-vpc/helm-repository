# Application GitOps Helm Chart

This Helm chart deploys a dedicated ArgoCD instance for application-level GitOps operations, providing teams with their own isolated GitOps environment for managing applications across multiple namespaces. This chart creates a separate ArgoCD instance optimized for application deployment and management, distinct from cluster-level GitOps operations.

## Overview

The Application GitOps chart creates a dedicated ArgoCD instance that enables development teams to manage their applications using GitOps principles. This instance is configured with appropriate RBAC permissions, resource limits, and OpenShift integration to provide a secure and efficient application deployment platform.

## Prerequisites

- OpenShift Container Platform 4.10 or later
- OpenShift GitOps Operator installed
- Cluster administrator privileges for initial setup
- Sufficient cluster resources (see [Resource Requirements](#resource-requirements))

## Chart Components

This chart deploys the following components:

### ArgoCD Instance
- **Custom ArgoCD CR**: Dedicated ArgoCD instance with application-specific configuration
- **Multi-namespace support**: Can manage applications across multiple namespaces
- **OpenShift OAuth integration**: Seamless authentication with OpenShift
- **Resource optimization**: Configured with appropriate resource limits for application workloads

### RBAC Configuration
- **ClusterRole**: Comprehensive permissions for application management
- **ClusterRoleBinding**: Binds the ArgoCD service account to the cluster role
- **Namespace isolation**: Supports team-based namespace patterns

### OpenShift Integration
- **Console Link**: Direct access to ArgoCD UI from OpenShift console
- **Route configuration**: Secure external access with TLS termination
- **OAuth integration**: Uses OpenShift identity provider

## Installation

This chart is designed to be deployed using ArgoCD/OpenShift GitOps and includes ArgoCD-specific annotations for proper deployment orchestration.

### ArgoCD Deployment

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: application-gitops
  namespace: openshift-gitops
  annotations:
    argocd.argoproj.io/compare-options: IgnoreExtraneous
spec:
  project: default
  source:
    repoURL: https://rosa-hcp-dedicated-vpc.github.io/helm-repository/
    chart: application-gitops
    targetRevision: "1.3.8"
    helm:
      values: |
        name: application-gitops
        gitopsNamespace: application-gitops
        domain: your-cluster-domain.com
        argocd:
          initialRepositories:
            - url: https://github.com/your-org/app-configs
              type: git
            - url: https://your-org.github.io/helm-charts/
              type: helm
  destination:
    server: https://kubernetes.default.svc
    namespace: application-gitops
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - SkipDryRunOnMissingResource=true
```

### Alternative: Helm CLI (Not Recommended)

While the chart can be installed via Helm CLI, it's designed for GitOps workflows:

```bash
# Not recommended - use ArgoCD instead
helm repo add rosa-hcp-dedicated-vpc https://rosa-hcp-dedicated-vpc.github.io/helm-repository/
helm install application-gitops rosa-hcp-dedicated-vpc/application-gitops \
  --namespace application-gitops \
  --create-namespace \
  --set domain=your-cluster-domain.com
```

## Configuration

### Values

| Parameter | Description | Default |
|-----------|-------------|---------|
| `name` | Name of the ArgoCD instance | `application-gitops` |
| `gitopsNamespace` | Namespace for the ArgoCD instance | `application-gitops` |
| `domain` | Cluster domain for route configuration | `<placeholder_domain>` |
| `argocd.initialRepositories` | List of initial Git/Helm repositories | `[]` |

### ArgoCD Configuration

The chart creates an ArgoCD instance with the following default configuration:

#### Resource Limits

| Component | CPU Limit | Memory Limit | CPU Request | Memory Request |
|-----------|-----------|--------------|-------------|----------------|
| Controller | 4 cores | 4Gi | 500m | 2Gi |
| Server | 500m | 256Mi | 125m | 128Mi |
| Repo Server | 1 core | 1Gi | 250m | 256Mi |
| Redis | 500m | 256Mi | 250m | 128Mi |
| Dex | 500m | 256Mi | 250m | 128Mi |

#### Features

- **Multi-namespace support**: Can deploy to `*-team` namespaces and `application-gitops`
- **OpenShift OAuth**: Integrated authentication with OpenShift
- **TLS termination**: Secure routes with re-encryption
- **Resource health checks**: Custom health checks for PVCs
- **RBAC integration**: Uses OpenShift groups for authorization

### Example Custom Values

```yaml
# Custom ArgoCD instance name
name: dev-team-gitops
gitopsNamespace: dev-team-gitops

# Cluster domain (required)
domain: apps.rosa.example.com

# Initial repositories
argocd:
  initialRepositories:
    # Git repositories
    - url: https://github.com/dev-team/app-configs
      type: git
      name: app-configs
    - url: https://github.com/dev-team/microservices
      type: git
      name: microservices
    
    # Helm repositories
    - url: https://charts.bitnami.com/bitnami
      type: helm
      name: bitnami
    - url: https://dev-team.github.io/helm-charts/
      type: helm
      name: dev-team-charts
```

## Resource Requirements

### Minimum Requirements

- **CPU**: 2 cores total
- **Memory**: 3 GB RAM total
- **Storage**: 5 GB for repositories and cache

### Recommended for Production

- **CPU**: 4+ cores total
- **Memory**: 6+ GB RAM total
- **Storage**: 20+ GB for repositories and cache
- **High Availability**: Consider multiple replicas for production

## Features

### Application Management
- **GitOps Workflow**: Declarative application deployment and management
- **Multi-Repository Support**: Git and Helm repository integration
- **Automated Sync**: Configurable automatic synchronization
- **Manual Approval**: Support for manual approval workflows
- **Rollback Capabilities**: Easy rollback to previous application versions

### Security & Access Control
- **OpenShift OAuth**: Seamless integration with OpenShift authentication
- **RBAC Integration**: Uses OpenShift groups and roles
- **Namespace Isolation**: Team-based namespace access patterns
- **TLS Security**: Encrypted communication and secure routes

### Monitoring & Observability
- **Application Health**: Real-time application health monitoring
- **Sync Status**: Visual sync status and drift detection
- **Event Logging**: Comprehensive audit trail
- **Resource Visualization**: Application topology and dependencies

### Developer Experience
- **Web UI**: Rich web interface for application management
- **CLI Integration**: ArgoCD CLI for automation and scripting
- **Webhook Support**: Git webhook integration for automated deployments
- **Notification Integration**: Slack, email, and other notification channels

## Post-Installation

### Verify Installation

```bash
# Check ArgoCD instance status
oc get argocd -n application-gitops

# Check all pods are running
oc get pods -n application-gitops

# Get the ArgoCD route URL
oc get route -n application-gitops

# Check RBAC configuration
oc get clusterrole | grep application-gitops
oc get clusterrolebinding | grep application-gitops
```

### Access the ArgoCD UI

1. **Via OpenShift Console**: Look for "Application Argo CD" in the Application Menu
2. **Direct Route**: Access the route URL directly
3. **Port Forward**: For development/testing purposes

```bash
# Port forward for local access
oc port-forward svc/application-gitops-server -n application-gitops 8080:80
```

### Initial Configuration

1. **Add Repositories**: Configure your Git and Helm repositories
2. **Create Applications**: Deploy your first applications
3. **Configure RBAC**: Set up team-based access controls
4. **Set Up Notifications**: Configure notification channels
5. **Enable Webhooks**: Set up Git webhooks for automated sync

## Application Patterns

### Basic Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: application-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/my-team/my-app
    targetRevision: HEAD
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: my-team
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Helm Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-helm-app
  namespace: application-gitops
spec:
  project: default
  source:
    repoURL: https://my-team.github.io/helm-charts/
    chart: my-app
    targetRevision: "1.0.0"
    helm:
      values: |
        image:
          tag: v1.2.3
        replicas: 3
  destination:
    server: https://kubernetes.default.svc
    namespace: my-team
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Troubleshooting

### Common Issues

#### ArgoCD Instance Not Starting
```bash
# Check ArgoCD custom resource
oc describe argocd application-gitops -n application-gitops

# Check operator logs
oc logs -n openshift-gitops-operator deployment/gitops-operator-controller-manager
```

#### RBAC Permission Issues
```bash
# Verify ClusterRole exists
oc get clusterrole openshift-gitops-argocd-application-controller-application-gitops-custom

# Check ClusterRoleBinding
oc describe clusterrolebinding openshift-gitops-argocd-application-controller-application-gitops-custom
```

#### Route Access Issues
```bash
# Check route configuration
oc get route -n application-gitops -o yaml

# Verify TLS configuration
oc describe route application-gitops-server -n application-gitops
```

#### Application Sync Issues
```bash
# Check application status
oc get applications -n application-gitops

# Describe specific application
oc describe application my-app -n application-gitops

# Check ArgoCD server logs
oc logs -n application-gitops deployment/application-gitops-server
```

### Logs and Diagnostics

```bash
# ArgoCD controller logs
oc logs -n application-gitops deployment/application-gitops-application-controller

# ArgoCD server logs
oc logs -n application-gitops deployment/application-gitops-server

# Repository server logs
oc logs -n application-gitops deployment/application-gitops-repo-server

# Check events
oc get events -n application-gitops --sort-by='.lastTimestamp'
```

## Upgrading

### Helm Upgrade

```bash
# Update repository
helm repo update

# Upgrade the release
helm upgrade application-gitops rosa-hcp-dedicated-vpc/application-gitops \
  --namespace application-gitops
```

### GitOps Upgrade

Update the `targetRevision` in your ArgoCD Application manifest and commit the changes.

## Uninstallation

### Using ArgoCD

Delete the ArgoCD Application that manages this chart.

### Using Helm

```bash
# Uninstall the Helm release
helm uninstall application-gitops -n application-gitops

# Clean up remaining resources
oc delete namespace application-gitops

# Remove cluster-level RBAC (if needed)
oc delete clusterrole openshift-gitops-argocd-application-controller-application-gitops-custom
oc delete clusterrolebinding openshift-gitops-argocd-application-controller-application-gitops-custom
```

## Security Considerations

- **Namespace Isolation**: The instance is configured to manage specific namespace patterns
- **RBAC**: Comprehensive role-based access control
- **TLS Encryption**: All communication is encrypted
- **OAuth Integration**: Leverages OpenShift's identity management
- **Secret Management**: Supports sealed secrets and external secret operators

## Best Practices

### Repository Structure
- Use separate repositories for application code and Kubernetes manifests
- Implement proper branching strategies (GitFlow, GitHub Flow)
- Use environment-specific overlays with Kustomize

### Application Configuration
- Enable automated sync for non-production environments
- Use manual sync for production deployments
- Implement proper resource quotas and limits
- Use health checks and readiness probes

### Security
- Regularly rotate credentials and tokens
- Use least-privilege access principles
- Implement network policies where appropriate
- Monitor and audit ArgoCD activities

## Support

- **Documentation**: [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- **Community**: [ArgoCD Community](https://github.com/argoproj/argo-cd)
- **OpenShift GitOps**: [Red Hat OpenShift GitOps Documentation](https://docs.openshift.com/container-platform/latest/cicd/gitops/understanding-openshift-gitops.html)

## Contributing

This chart is part of the rosa-hcp-dedicated-vpc project. Please refer to the main repository for contribution guidelines.

## License

This chart is licensed under the Apache License 2.0. See the LICENSE file for details.

## Changelog

### Version 1.3.8
- Current stable release
- OpenShift OAuth integration
- Multi-namespace support
- Comprehensive RBAC configuration
- Console link integration
- Resource optimization for application workloads
