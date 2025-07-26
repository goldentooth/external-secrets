# External Secrets

GitOps configuration for External Secrets Operator in the Goldentooth Kubernetes cluster, enabling secure integration between Kubernetes secrets and external secret management systems.

## Overview

External Secrets Operator (ESO) provides a Kubernetes-native way to sync secrets from external systems like HashiCorp Vault, AWS Secrets Manager, and Azure Key Vault into Kubernetes Secret resources.

## Purpose

### Secret Management Integration
- **HashiCorp Vault**: Primary integration with cluster Vault instance
- **AWS Secrets Manager**: Cloud-based secret storage for external resources
- **Centralized Control**: Single point of secret lifecycle management
- **Automatic Rotation**: Automated secret refresh and rotation

### Security Benefits
- **Least Privilege**: Fine-grained access control via service accounts
- **Audit Trail**: Complete audit logging of secret access and rotation
- **Encryption**: Secrets encrypted in transit and at rest
- **Separation of Concerns**: Secret management separate from application deployment

## Architecture

### Components
- **External Secrets Operator**: Core controller managing secret synchronization
- **SecretStore**: Defines connection to external secret backends
- **ExternalSecret**: Defines which secrets to sync and how to transform them
- **ClusterSecretStore**: Cluster-wide secret store configuration

### Integration Points
```
Vault/AWS → External Secrets Operator → Kubernetes Secrets → Pod Volumes
```

## Configuration

### Vault Integration
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-store
spec:
  provider:
    vault:
      server: "https://vault.services.goldentooth.net"
      path: "secret"
      auth:
        kubernetes:
          mountPath: "k8s"
          role: "external-secrets"
          serviceAccountRef:
            name: "external-secrets-sa"
```

### AWS Secrets Manager
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-store
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        serviceAccount:
          name: "external-secrets-aws-sa"
```

## Secret Synchronization

### Basic External Secret
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-database-secret
spec:
  refreshInterval: 15s
  secretStoreRef:
    name: vault-store
    kind: SecretStore
  target:
    name: app-database-secret
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: database/postgres
      property: username
  - secretKey: password
    remoteRef:
      key: database/postgres
      property: password
```

### Template-based Transformation
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-config
spec:
  secretStoreRef:
    name: vault-store
    kind: SecretStore
  target:
    name: app-config
    template:
      type: Opaque
      data:
        config.yaml: |
          database:
            host: {{ .host }}
            username: {{ .username }}
            password: {{ .password }}
            ssl: true
  data:
  - secretKey: host
    remoteRef:
      key: database/postgres
      property: host
```

## Deployment

### Argo CD Application
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets
spec:
  project: default
  source:
    repoURL: https://github.com/goldentooth/external-secrets
    path: .
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: external-secrets
```

### Installation via Helm
```bash
# Add External Secrets Operator chart repository
helm repo add external-secrets https://charts.external-secrets.io

# Install via Argo CD application
kubectl apply -f templates/Application.yaml
```

## Secret Store Configuration

### Vault Authentication
The operator authenticates to Vault using Kubernetes service accounts:

1. **Service Account**: Created with appropriate annotations
2. **Vault Role**: Configured to trust the service account
3. **Policies**: Vault policies granting access to specific secret paths
4. **Token Review**: Vault validates Kubernetes tokens

```bash
# Configure Vault Kubernetes auth
vault auth enable kubernetes
vault write auth/kubernetes/config \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://kubernetes.default.svc:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

### AWS IAM Integration
For AWS Secrets Manager integration:

1. **IAM Role**: Created with least-privilege access
2. **Service Account**: Annotated with IAM role ARN
3. **IRSA**: IAM Roles for Service Accounts configuration
4. **Resource Policies**: Fine-grained secret access control

## Common Use Cases

### Database Credentials
```yaml
# Sync database credentials from Vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: postgres-credentials
spec:
  secretStoreRef:
    name: vault-store
  target:
    name: postgres-secret
  data:
  - secretKey: POSTGRES_USER
    remoteRef:
      key: database/postgres
      property: username
  - secretKey: POSTGRES_PASSWORD
    remoteRef:
      key: database/postgres
      property: password
```

### API Keys and Tokens
```yaml
# Sync API keys from AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: api-keys
spec:
  secretStoreRef:
    name: aws-store
  target:
    name: api-keys-secret
  data:
  - secretKey: GITHUB_TOKEN
    remoteRef:
      key: "api-keys/github"
      property: token
  - secretKey: SLACK_WEBHOOK
    remoteRef:
      key: "api-keys/slack"
      property: webhook_url
```

### TLS Certificates
```yaml
# Sync TLS certificates from Vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: tls-cert
spec:
  secretStoreRef:
    name: vault-store
  target:
    name: app-tls-secret
    type: kubernetes.io/tls
  data:
  - secretKey: tls.crt
    remoteRef:
      key: certificates/app
      property: certificate
  - secretKey: tls.key
    remoteRef:
      key: certificates/app
      property: private_key
```

## Monitoring and Observability

### Metrics
External Secrets Operator provides Prometheus metrics for:
- **Secret Sync Status**: Success/failure rates
- **Refresh Intervals**: Time between secret updates
- **Error Rates**: Authentication and connection failures
- **Secret Store Health**: Backend connectivity status

### Logging
Structured logging includes:
- **Secret Access**: Which secrets were accessed when
- **Authentication Events**: Vault/AWS authentication status
- **Sync Operations**: Success/failure of secret synchronization
- **Error Details**: Detailed error messages for troubleshooting

## Security Considerations

### Access Control
- **RBAC**: Kubernetes role-based access control for operator
- **Service Accounts**: Dedicated service accounts per application
- **Network Policies**: Restricted network access to secret stores
- **Pod Security**: Security contexts and admission controllers

### Secret Handling
- **Encryption**: Secrets encrypted in etcd and in transit
- **Rotation**: Automatic secret rotation based on TTL
- **Audit**: Complete audit trail of secret access
- **Cleanup**: Automatic cleanup of orphaned secrets

### Best Practices
- **Least Privilege**: Minimal required permissions
- **Regular Rotation**: Frequent secret rotation schedules
- **Monitoring**: Continuous monitoring of secret access patterns
- **Backup**: Secure backup and recovery procedures
