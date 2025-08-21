# AWS Private CA Issuer Helm Chart

This Helm chart deploys the AWS Private CA Issuer for cert-manager, enabling automatic certificate provisioning using AWS Certificate Manager Private Certificate Authority (ACM PCA). This chart is designed for deployment via Terraform using Helm commands or through Advanced Cluster Management (ACM), providing secure certificate management for OpenShift clusters.

## Overview

The AWS Private CA Issuer extends cert-manager functionality to work with AWS Private Certificate Authority, allowing you to issue certificates from your private CA infrastructure. This integration provides enterprise-grade certificate management with the security and compliance benefits of private certificate authorities.

## Prerequisites

- OpenShift Container Platform 4.10 or later
- AWS Private Certificate Authority (PCA) configured
- IAM role with appropriate permissions for PCA access (see [AWS Prerequisites](#aws-prerequisites))
- cert-manager operator installed
- Terraform (for Terraform deployment) or ACM (for ACM deployment)
- Sufficient cluster resources (see [Resource Requirements](#resource-requirements))

### Terraform Infrastructure Setup

This chart is designed to work with the infrastructure provisioned by the rosa-hcp-dedicated-vpc project. The required AWS IAM resources are automatically created by the Terraform configuration in [`6.cert-manager.tf`](https://github.com/rosa-hcp-dedicated-vpc/rosa-hcp-dedicated-vpc/blob/main/terraform/6.cert-manager.tf).

The Terraform file creates:

1. **IAM Policy** (`rosa_cert_manager_policy_iam`): Grants the necessary permissions for AWS Private CA operations:
   - `acm-pca:DescribeCertificateAuthority`
   - `acm-pca:GetCertificate`
   - `acm-pca:IssueCertificate`

2. **IAM Role** (`rosa_cert_manager_iam`): Configured with OIDC trust relationship for the OpenShift service account:
   - Trusts the cluster's OIDC provider
   - Allows assumption by `system:serviceaccount:cert-manager:cert-manager`
   - Named with pattern: `{cluster_name}-rosa-cert-manager`

3. **Role Policy Attachment**: Links the policy to the role for complete permissions

To enable cert-manager infrastructure in your Terraform deployment, ensure the `enable-cert-manager` variable is set to `true` in your cluster configuration file (e.g., `clusters/np-app-1.json`).

### Certificate Consumption by Secondary Ingress Controllers

The AWS Private CA Issuer integrates with the cluster's ingress infrastructure through the [`11.ingress.tf`](https://github.com/rosa-hcp-dedicated-vpc/rosa-hcp-dedicated-vpc/blob/main/terraform/11.ingress.tf) Terraform configuration. This file demonstrates how certificates issued by the AWS Private CA are consumed by secondary ingress controllers and integrated with private DNS zones.

#### Ingress Infrastructure Components

1. **Private Route53 Zone** (`aws_route53_zone.cluster`): 
   - Creates a private DNS zone for the cluster's base domain
   - Associated with the VPC for internal DNS resolution
   - Enables private certificate validation and routing

2. **Secondary Ingress Controllers**: 
   - Deployed via the `cluster-ingress` Helm chart
   - Each ingress controller gets its own subdomain (e.g., `api.cluster.example.com`)
   - Configured with certificates issued by the AWS Private CA Issuer

#### Certificate Integration Flow

The ingress deployment process follows this certificate integration pattern:

1. **Certificate Request**: The `cluster-ingress` Helm chart creates a `Certificate` resource:
   ```yaml
   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
     name: api.cluster.example.com
     namespace: openshift-ingress
   spec:
     dnsNames:
     - "*.api.cluster.example.com"
     issuerRef:
       group: awspca.cert-manager.io
       kind: AWSPCAClusterIssuer
       name: aws-pca-cluster-issuer
   ```

2. **Certificate Issuance**: The AWS Private CA Issuer processes the request:
   - Validates the certificate request
   - Issues a certificate from the configured AWS Private CA
   - Stores the certificate and private key in a Kubernetes secret

3. **Ingress Controller Configuration**: The `IngressController` resource references the certificate:
   ```yaml
   apiVersion: operator.openshift.io/v1
   kind: IngressController
   spec:
     defaultCertificate:
       name: "api.cluster.example.com"
     domain: "api.cluster.example.com"
   ```

4. **DNS Integration**: The Terraform script automatically:
   - Retrieves the load balancer address from the ingress controller service
   - Creates/updates Route53 CNAME records pointing `*.api.cluster.example.com` to the load balancer
   - Ensures private DNS resolution within the VPC

#### Benefits of This Architecture

- **Private Certificate Authority**: All certificates are issued from your private CA, maintaining security boundaries
- **Automated Certificate Management**: cert-manager handles certificate lifecycle (issuance, renewal, rotation)
- **DNS Integration**: Automatic DNS record management for ingress endpoints
- **Namespace Isolation**: Each ingress controller can serve different application namespaces
- **Load Balancer Integration**: Seamless integration with AWS Network Load Balancers for internal traffic

This integration enables secure, automated certificate management for multiple ingress endpoints while maintaining private network boundaries and DNS resolution.

## Chart Dependencies

This chart depends on the following sub-charts:

- **helper-installplan-approver** (v0.1.0): Manages operator install plan approval

## Architecture

The chart deploys the following components:

### Core Components
- **cert-manager Operator**: OpenShift cert-manager operator subscription
- **AWS PCA Issuer Deployment**: Controller that interfaces with AWS PCA
- **AWSPCAClusterIssuer**: Custom resource for certificate issuance
- **RBAC Configuration**: Service accounts, roles, and bindings

### AWS Integration
- **IAM Role Integration**: Uses OpenShift service account token for AWS authentication
- **PCA Integration**: Direct integration with AWS Private Certificate Authority
- **Regional Configuration**: Supports multi-region AWS deployments

## Installation

This chart is designed for deployment via Terraform or Advanced Cluster Management (ACM), not ArgoCD.

### Terraform Deployment

In the rosa-hcp-dedicated-vpc project, the AWS Private CA Issuer is deployed automatically during cluster bootstrap via the [`bootstrap.tftpl`](https://github.com/rosa-hcp-dedicated-vpc/rosa-hcp-dedicated-vpc/blob/main/terraform/scripts/bootstrap.tftpl) script. This script is executed by Terraform as part of the cluster provisioning process.

#### Bootstrap Script Deployment

The bootstrap script handles the deployment with the following command:

```bash
# Install aws-privateca-issuer chart
echo "Installing/Upgrading aws-privateca-issuer chart..."
helm upgrade --install aws-privateca-issuer helm_repo_new/aws-privateca-issuer \
  --version "${helm_chart_awspca_version}" \
  --set certManagerRole="arn:aws:iam::${aws_account_id}:role/${cluster}-rosa-cert-manager" \
  --set awsAcmPcaArn="${aws_private_certificate_authority_arn}" \
  --set csv="${awspca_csv}" \
  --set awsPcaIssuer="${awspca_issuer}" \
  --insecure-skip-tls-verify \
  --create-namespace \
  --namespace cert-manager-operator \
  --set aws_region="${AWS_REGION}" \
  --set ecr_account="${ecr_account}"
```

#### Terraform Integration

The bootstrap script is called by Terraform through a `shell_script` resource that templates the script with the necessary variables:

```hcl
# Simplified example of how Terraform calls the bootstrap script
resource "shell_script" "cluster_bootstrap" {
  lifecycle_commands {
    create = templatefile(
      "./scripts/bootstrap.tftpl",
      {
        cluster                              = var.cluster_name
        aws_account_id                      = data.aws_caller_identity.current.account_id
        aws_private_certificate_authority_arn = var.aws_private_certificate_authority_arn
        helm_chart_awspca                   = "aws-privateca-issuer"
        helm_chart_awspca_version           = "1.6.0"
        awspca_csv                          = var.cert_manager_csv
        awspca_issuer                       = "aws-pca-cluster-issuer"
        AWS_REGION                          = var.aws_region
        ecr_account                         = var.ecr_account
        enable                              = var.enable-cert-manager
      }
    )
  }
}
```


#### Terraform Values Template

```yaml
# values.yaml.tpl
certManagerRole: "${cert_manager_role}"
awsAcmPcaArn: "${aws_acm_pca_arn}"
region: "${aws_region}"
domain: "${domain}"
csv: "${csv}"

# AWS PCA Issuer Configuration
awsPcaIssuer: "aws-pca-cluster-issuer"
defaultAWSPCAImage: "registry.redhat.io/rhel8/aws-privateca-issuer:v1.4.2"

# Resource Configuration
replicaCount: 2
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

# Security Configuration
securityContext:
  allowPrivilegeEscalation: false
  runAsNonRoot: true
  capabilities:
    drop:
    - ALL

podSecurityContext:
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault
```

### ACM Deployment

```yaml
# acm-policy/aws-privateca-issuer-policy.yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: aws-privateca-issuer-policy
  namespace: open-cluster-management-global-set
spec:
  remediationAction: enforce
  disabled: false
  policy-templates:
  - objectDefinition:
      apiVersion: policy.open-cluster-management.io/v1
      kind: ConfigurationPolicy
      metadata:
        name: aws-privateca-issuer-config
      spec:
        remediationAction: enforce
        severity: high
        object-templates:
        - complianceType: musthave
          objectDefinition:
            apiVersion: v1
            kind: Namespace
            metadata:
              name: cert-manager
        - complianceType: musthave
          objectDefinition:
            apiVersion: source.toolkit.fluxcd.io/v1beta2
            kind: HelmRepository
            metadata:
              name: aws-privateca-issuer
              namespace: cert-manager
            spec:
              url: https://rosa-hcp-dedicated-vpc.github.io/helm-repository/
        - complianceType: musthave
          objectDefinition:
            apiVersion: helm.toolkit.fluxcd.io/v2beta1
            kind: HelmRelease
            metadata:
              name: aws-privateca-issuer
              namespace: cert-manager
            spec:
              chart:
                spec:
                  chart: aws-privateca-issuer
                  version: "1.5.9"
                  sourceRef:
                    kind: HelmRepository
                    name: aws-privateca-issuer
              values:
                certManagerRole: "{{ .certManagerRole }}"
                awsAcmPcaArn: "{{ .awsAcmPcaArn }}"
                region: "{{ .region }}"
```

### Direct Helm Installation

```bash
# Add the repository
helm repo add rosa-hcp-dedicated-vpc https://rosa-hcp-dedicated-vpc.github.io/helm-repository/

# Install the chart
helm install aws-privateca-issuer rosa-hcp-dedicated-vpc/aws-privateca-issuer \
  --namespace cert-manager \
  --create-namespace \
  --set certManagerRole="arn:aws:iam::123456789012:role/cert-manager-role" \
  --set awsAcmPcaArn="arn:aws:acm-pca:us-east-1:123456789012:certificate-authority/12345678-1234-1234-1234-123456789012" \
  --set region="us-east-1"
```

## Configuration

### Required Values

| Parameter | Description | Required |
|-----------|-------------|----------|
| `certManagerRole` | IAM role ARN for cert-manager authentication | Yes |
| `awsAcmPcaArn` | AWS Private CA ARN | Yes |
| `region` | AWS region where PCA is located | Yes |
| `csv` | ClusterServiceVersion for cert-manager operator | Yes |

### Core Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `nameSpace` | Namespace for AWS PCA Issuer deployment | `cert-manager` |
| `replicaCount` | Number of issuer replicas | `1` |
| `awsPcaIssuer` | Name of the AWSPCAClusterIssuer resource | `aws-pca-cluster-issuer` |
| `defaultAWSPCAImage` | Container image for AWS PCA Issuer | `registry.redhat.io/rhel8/aws-privateca-issuer:v1.4.2` |

### Security Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `disableApprovedCheck` | Disable waiting for CertificateRequest approval | `false` |
| `disableClientSideRateLimiting` | Disable client-side rate limiting | `false` |
| `securityContext.allowPrivilegeEscalation` | Allow privilege escalation | `false` |

### Service Account Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `serviceAccount.create` | Create service account | `true` |
| `serviceAccount.name` | Service account name | `cert-manager` |
| `serviceAccount.annotations` | Service account annotations | `{}` |

### RBAC Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `rbac.create` | Create RBAC resources | `true` |
| `approverRole.enabled` | Create ClusterRole for certificate approval | `true` |
| `approverRole.serviceAccountName` | Service account for approval permissions | `cert-manager` |

### Example Production Values

```yaml
# Production configuration
certManagerRole: "arn:aws:iam::123456789012:role/prod-cert-manager-role"
awsAcmPcaArn: "arn:aws:acm-pca:us-east-1:123456789012:certificate-authority/prod-ca-12345"
region: "us-east-1"
domain: "apps.prod.example.com"
csv: "cert-manager-operator.v1.13.1"

# High availability configuration
replicaCount: 3
resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 200m
    memory: 256Mi

# Security hardening
securityContext:
  allowPrivilegeEscalation: false
  runAsNonRoot: true
  runAsUser: 65534
  capabilities:
    drop:
    - ALL

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 65534
  fsGroup: 65534
  seccompProfile:
    type: RuntimeDefault

# Monitoring
serviceMonitor:
  name: cert-manager
  targetPort: 9402
  interval: 30s

# Topology spread for HA
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app.kubernetes.io/name: aws-privateca-issuer
```

## Resource Requirements

### Minimum Requirements

- **CPU**: 100m per replica
- **Memory**: 128Mi per replica
- **Storage**: Minimal (stateless application)

### Recommended for Production

- **CPU**: 200m per replica
- **Memory**: 256Mi per replica
- **Replicas**: 2-3 for high availability
- **Node Distribution**: Spread across availability zones

## AWS Prerequisites

### IAM Role Configuration

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "acm-pca:DescribeCertificateAuthority",
        "acm-pca:GetCertificate",
        "acm-pca:IssueCertificate"
      ],
      "Resource": "arn:aws:acm-pca:*:*:certificate-authority/*"
    }
  ]
}
```

### Trust Relationship

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT-ID:oidc-provider/OIDC-PROVIDER-URL"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "OIDC-PROVIDER-URL:sub": "system:serviceaccount:cert-manager:cert-manager"
        }
      }
    }
  ]
}
```

## Post-Installation

### Verify Installation

```bash
# Check cert-manager operator status
oc get csv -n cert-manager-operator

# Check AWS PCA Issuer deployment
oc get deployment aws-privateca-issuer -n cert-manager

# Check AWSPCAClusterIssuer
oc get awspcaclusterissuer -n cert-manager

# Check service account and RBAC
oc get serviceaccount cert-manager -n cert-manager
oc get clusterrole | grep cert-manager
```

### Test Certificate Issuance

```yaml
# test-certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: test-certificate
  namespace: cert-manager
spec:
  secretName: test-certificate-tls
  issuerRef:
    name: aws-pca-cluster-issuer
    kind: AWSPCAClusterIssuer
    group: awspca.cert-manager.io
  commonName: test.example.com
  dnsNames:
  - test.example.com
  - api.test.example.com
```

```bash
# Apply test certificate
oc apply -f test-certificate.yaml

# Check certificate status
oc describe certificate test-certificate -n cert-manager

# Verify secret creation
oc get secret test-certificate-tls -n cert-manager
```

## Usage Examples

### Basic Certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: app-certificate
  namespace: my-app
spec:
  secretName: app-tls
  issuerRef:
    name: aws-pca-cluster-issuer
    kind: AWSPCAClusterIssuer
    group: awspca.cert-manager.io
  commonName: app.example.com
  dnsNames:
  - app.example.com
  - www.app.example.com
```

### Ingress with Automatic Certificate

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: my-app
  annotations:
    cert-manager.io/cluster-issuer: aws-pca-cluster-issuer
    cert-manager.io/cluster-issuer-kind: AWSPCAClusterIssuer
    cert-manager.io/cluster-issuer-group: awspca.cert-manager.io
spec:
  tls:
  - hosts:
    - app.example.com
    secretName: app-ingress-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

### OpenShift Route with Certificate

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: app-route
  namespace: my-app
  annotations:
    cert-manager.io/cluster-issuer: aws-pca-cluster-issuer
    cert-manager.io/cluster-issuer-kind: AWSPCAClusterIssuer
    cert-manager.io/cluster-issuer-group: awspca.cert-manager.io
spec:
  host: app.example.com
  to:
    kind: Service
    name: app-service
  tls:
    termination: edge
    certificate: |
      # Certificate will be automatically populated
    key: |
      # Private key will be automatically populated
```

## Troubleshooting

### Common Issues

#### Operator Installation Fails
```bash
# Check subscription status
oc get subscription openshift-cert-manager-operator -n cert-manager-operator -o yaml

# Check install plan
oc get installplan -n cert-manager-operator

# Check operator logs
oc logs -n cert-manager-operator deployment/cert-manager-operator
```

#### AWS PCA Issuer Pod Not Starting
```bash
# Check deployment status
oc describe deployment aws-privateca-issuer -n cert-manager

# Check pod logs
oc logs -n cert-manager deployment/aws-privateca-issuer

# Check service account token
oc describe serviceaccount cert-manager -n cert-manager
```

#### Certificate Issuance Fails
```bash
# Check certificate status
oc describe certificate <certificate-name> -n <namespace>

# Check certificate request
oc get certificaterequest -n <namespace>
oc describe certificaterequest <request-name> -n <namespace>

# Check AWS PCA Issuer logs
oc logs -n cert-manager deployment/aws-privateca-issuer
```

#### AWS Authentication Issues
```bash
# Check IAM role annotation on service account
oc get serviceaccount cert-manager -n cert-manager -o yaml

# Test AWS credentials
oc exec -n cert-manager deployment/aws-privateca-issuer -- aws sts get-caller-identity

# Check OIDC provider configuration
aws iam list-open-id-connect-providers
```

### Logs and Diagnostics

```bash
# cert-manager operator logs
oc logs -n cert-manager-operator deployment/cert-manager-operator

# AWS PCA Issuer logs
oc logs -n cert-manager deployment/aws-privateca-issuer

# cert-manager controller logs
oc logs -n cert-manager deployment/cert-manager

# Check events
oc get events -n cert-manager --sort-by='.lastTimestamp'

# Export configuration for support
oc get awspcaclusterissuer -o yaml > aws-pca-issuer-config.yaml
```

## Upgrading

### Terraform Upgrade

Update the chart version in your Terraform configuration:

```hcl
resource "helm_release" "aws_privateca_issuer" {
  # ... other configuration ...
  version = "1.6.0"  # Update version
  
  # Force recreation if needed
  recreate_pods = true
}
```

### ACM Upgrade

Update the version in your ACM policy:

```yaml
spec:
  chart:
    spec:
      version: "1.6.0"  # Update version
```

### Manual Helm Upgrade

```bash
# Update repository
helm repo update

# Upgrade release
helm upgrade aws-privateca-issuer rosa-hcp-dedicated-vpc/aws-privateca-issuer \
  --namespace cert-manager \
  --reuse-values
```

## Uninstallation

### Terraform Uninstallation

Remove the Helm release from your Terraform configuration and apply:

```bash
terraform destroy -target=helm_release.aws_privateca_issuer
```

### Manual Uninstallation

```bash
# Uninstall Helm release
helm uninstall aws-privateca-issuer -n cert-manager

# Clean up remaining resources
oc delete awspcaclusterissuer --all -n cert-manager
oc delete subscription openshift-cert-manager-operator -n cert-manager-operator

# Remove namespaces (if desired)
oc delete namespace cert-manager
oc delete namespace cert-manager-operator
```

## Security Considerations

- **IAM Permissions**: Use least-privilege IAM roles
- **Network Security**: Implement network policies to restrict traffic
- **Certificate Validation**: Regularly validate issued certificates
- **Monitoring**: Monitor certificate expiration and renewal
- **Audit Logging**: Enable AWS CloudTrail for PCA operations

## Best Practices

### Certificate Management
- Use short-lived certificates where possible
- Implement automated certificate renewal
- Monitor certificate expiration dates
- Use appropriate certificate templates

### Security
- Regularly rotate IAM credentials
- Use dedicated service accounts
- Implement proper RBAC controls
- Monitor certificate issuance patterns

### Operations
- Set up monitoring and alerting
- Implement backup and disaster recovery
- Document certificate policies and procedures
- Regular security assessments

## Support

- **AWS Private CA Documentation**: [AWS Private Certificate Authority](https://docs.aws.amazon.com/privateca/)
- **cert-manager Documentation**: [cert-manager](https://cert-manager.io/docs/)
- **AWS Private CA Issuer**: [GitHub Repository](https://github.com/cert-manager/aws-privateca-issuer)

## Contributing

This chart is part of the rosa-hcp-dedicated-vpc project. Please refer to the main repository for contribution guidelines.

## License

This chart is licensed under the Apache License 2.0. See the LICENSE file for details.

## Changelog

### Version 1.5.9
- Current stable release
- OpenShift cert-manager operator integration
- AWS PCA ClusterIssuer support
- Helm hook-based installation sequence
- Production-ready security configuration
- Comprehensive RBAC setup