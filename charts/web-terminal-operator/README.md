# Web Terminal Operator

## Overview

The **Web Terminal Operator** Helm chart deploys the Red Hat OpenShift Web Terminal Operator, which provides browser-based terminal access to OpenShift clusters. This operator enables developers and administrators to access command-line tools directly from the OpenShift web console.

## Introduction

Red Hat OpenShift Web Terminal provides a cloud-based terminal experience integrated with the OpenShift web console. The Web Terminal Operator simplifies the deployment and management of web-based terminal sessions, providing:

- **Browser-Based Access**: No local terminal or SSH client required
- **Pre-installed Tools**: Common CLI tools (oc, kubectl, helm, etc.) pre-configured
- **Secure Access**: Integration with OpenShift authentication and RBAC
- **Persistent Sessions**: Session persistence across browser refreshes
- **Custom Tooling**: Ability to customize terminal environments

## Prerequisites

- OpenShift cluster with cluster-admin privileges
- ArgoCD or OpenShift GitOps installed
- Web browser with JavaScript enabled

## Deployment

This chart is deployed via **ArgoCD** as part of the GitOps infrastructure pattern.

### ArgoCD Application Example

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-terminal-operator
  namespace: openshift-gitops
  annotations:
    argocd.argoproj.io/sync-wave: '1'
spec:
  destination:
    namespace: openshift-operators
    server: https://kubernetes.default.svc
  project: default
  sources:
    - repoURL: https://rosa-hcp-dedicated-vpc.github.io/helm-repository/
      chart: web-terminal-operator
      targetRevision: 0.1.0
      helm:
        valueFiles:
        - $values/cluster-config/nonprod/np-app-1/infrastructure.yaml
        values: |
          appTeam: web-terminal-operator
    - repoURL: https://github.com/rosa-hcp-dedicated-vpc/cluster-config.git
      targetRevision: HEAD
      ref: values
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - SkipDryRunOnMissingResource=true
```

## Configuration

### Key Configuration Options

| Parameter | Description | Default |
|-----------|-------------|---------|
| `operatorChannel` | Operator subscription channel | `fast` |
| `operatorNamespace` | Target namespace for operator | `openshift-operators` |
| `operatorInstallPlanApproval` | InstallPlan approval mode | `Automatic` |
| `helper-status-checker.enabled` | Enable status checking | `true` |
| `syncwave` | ArgoCD sync wave | `1` |

## Dependencies

This chart includes the following dependencies:

- **helper-operator** (~1.1.0): Manages operator subscription and installation
- **helper-status-checker** (~4.1.2): Validates operator deployment status

## Usage

After deployment, users can access the web terminal through the OpenShift web console. A terminal icon will appear in the console header, allowing users to launch terminal sessions.

### Custom DevWorkspaceTemplate

```yaml
apiVersion: workspace.devfile.io/v1alpha2
kind: DevWorkspaceTemplate
metadata:
  name: custom-web-terminal
  namespace: openshift-operators
spec:
  components:
  - name: web-terminal
    container:
      image: registry.redhat.io/ubi8/ubi:latest
      command: ['/bin/bash']
      args: ['-c', 'sleep infinity']
      env:
      - name: PS1
        value: '\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
      memoryLimit: 512Mi
      cpuLimit: 500m
      mountSources: true
```

## Support

For issues and support:

- Check operator logs and web terminal pod status
- Review Red Hat OpenShift Web Terminal documentation
- Contact Red Hat support for enterprise customers

## Version History

| Version | Changes |
|---------|---------|
| 0.1.0 | Initial release with basic operator deployment |
