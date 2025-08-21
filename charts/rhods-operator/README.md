# Red Hat OpenShift AI Operator

![Version: 1.0.0](https://img.shields.io/badge/Version-1.0.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square)

Master chart to deploy and configure the Red Hat OpenShift AI Operator

## Description

This Helm chart deploys the Red Hat OpenShift AI Operator using the helper-operator and helper-status-checker subcharts. It provides a consistent way to install and configure OpenShift AI across different environments.

## Features

- Deploys Red Hat OpenShift AI Operator via Subscription
- Creates necessary namespaces and operator groups
- Provides status checking to ensure operator is ready
- Optional DataScienceCluster configuration
- Configurable component management states

## Prerequisites

- OpenShift 4.12+ cluster
- Cluster admin access
- ArgoCD/OpenShift GitOps (if using with GitOps)

## Usage

### Basic Installation

```yaml
helper-operator:
  operators:
    rhods-operator:
      enabled: true
      namespace:
        name: redhat-ods-operator
        create: true
      subscription:
        channel: stable
        approval: Manual
        operatorName: rhods-operator
        source: redhat-operators

helper-status-checker:
  enabled: true
  checks:
    - operatorName: rhods-operator
      namespace:
        name: redhat-ods-operator
```

### With DataScienceCluster

```yaml
openshift-ai:
  datasciencecluster:
    enabled: true
    name: default-dsc
    spec:
      components:
        dashboard:
          managementState: Managed
        workbenches:
          managementState: Managed
        # ... other components
```

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| helper-operator.operators.rhods-operator.enabled | bool | `true` | Enable the OpenShift AI operator installation |
| helper-operator.operators.rhods-operator.namespace.name | string | `"redhat-ods-operator"` | Namespace for the operator |
| helper-operator.operators.rhods-operator.namespace.create | bool | `true` | Create the namespace |
| helper-operator.operators.rhods-operator.subscription.channel | string | `"stable"` | Subscription channel |
| helper-operator.operators.rhods-operator.subscription.approval | string | `"Manual"` | Install plan approval mode |
| helper-status-checker.enabled | bool | `true` | Enable status checking |
| openshift-ai.datasciencecluster.enabled | bool | `false` | Enable DataScienceCluster creation |
| openshift-ai.datasciencecluster.name | string | `"default-dsc"` | Name of the DataScienceCluster |

## Dependencies

| Repository | Name | Version |
|------------|------|---------|
| https://rosa-hcp-dedicated-vpc.github.io/helm-repository/ | helper-operator | 1.0.29 |
| https://rosa-hcp-dedicated-vpc.github.io/helm-repository/ | helper-status-checker | 4.1.2 |

## Components

The OpenShift AI platform includes the following components that can be managed:

- **Dashboard**: Web-based user interface
- **Workbenches**: Jupyter notebook environments
- **Data Science Pipelines**: ML workflow orchestration
- **Model Serving**: KServe and ModelMesh for model deployment
- **CodeFlare**: Distributed compute for ML workloads
- **Ray**: Distributed computing framework

## Notes

- The operator CSV version should be specified in your infrastructure configuration
- Manual approval is recommended for production environments
- The DataScienceCluster is optional and can be created separately
- All components default to "Managed" state when DataScienceCluster is enabled

## Maintainers

| Name | URL |
| ---- | --- |
| rosa-hcp-dedicated-vpc | <https://github.com/rosa-hcp-dedicated-vpc> |

## Source Code

* <https://github.com/rosa-hcp-dedicated-vpc/helm-repository>
