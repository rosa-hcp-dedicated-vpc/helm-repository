# OpenShift Lightspeed Operator

This Helm chart deploys the Red Hat OpenShift Lightspeed Operator, which provides AI-powered assistance for OpenShift operations.

## Overview

OpenShift Lightspeed is an AI assistant that helps with OpenShift cluster management and troubleshooting tasks.

## Installation

This chart is designed to be deployed via ArgoCD as part of the GitOps workflow.

### Prerequisites

- OpenShift 4.14+
- Cluster administrator privileges

### Configuration

The chart uses manual installation approval by default. Key configuration options:

- `operatorNamespace`: Target namespace (default: openshift-lightspeed)
- `operatorChannel`: Operator channel (default: stable)
- `operatorInstallPlanAproval`: Installation approval mode (default: Manual)

## Dependencies

- helper-status-checker: Monitors operator installation status

## Resources Created

- Namespace: openshift-lightspeed
- OperatorGroup: Scoped to the lightspeed namespace
- Subscription: Manual approval for controlled deployment
- Status checker jobs: Monitor installation progress
