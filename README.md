# Qalby Proxy Helm Chart

A Helm chart for deploying Qalby Proxy - a FastAPI application with audio and agent capabilities.

## Overview

This Helm chart deploys the Qalby Proxy application on a Kubernetes cluster. The application provides audio processing and AI agent capabilities through a FastAPI-based service.

## Features

- **Automatic Configuration**: Host and port are automatically configured based on the service port
- **Secret Management**: All sensitive environment variables are stored in Kubernetes secrets
- **Existing Secret Support**: Use your own pre-created secret or let the chart create one
- **Health Checks**: Built-in liveness and readiness probes
- **Scalability**: Optional horizontal pod autoscaling
- **Security**: Configurable security contexts and service accounts
- **Ingress Support**: Optional ingress configuration for external access
- **Persistence**: Optional PVC for file-based storage

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+

## Installation

### Add the repository (if hosted)

```bash
helm repo add qalby-proxy https://your-chart-repo-url
helm repo update
```

### Install the chart

```bash
helm install qalby-proxy qalby-proxy/qalby-proxy
```

### Install with custom values

```bash
helm install qalby-proxy qalby-proxy/qalby-proxy -f custom-values.yaml
```

### Install with --set flags

```bash
helm install qalby-proxy qalby-proxy/qalby-proxy \
  --set env.redis_url="redis://redis:6379/0" \
  --set env.encryption_salt="my-secret-salt" \
  --set env.logfire_write_token="my-logfire-token"
```

## Configuration

### Key Configuration Options

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of replicas | `1` |
| `image.repository` | Container image repository | `kamasalyamov/livellm-proxy` |
| `image.tag` | Container image tag | `""` (uses Chart.appVersion) |
| `service.port` | Service port | `8000` |
| `service.type` | Service type | `ClusterIP` |
| `existingSecret` | Name of existing secret to use | `""` |
| `env.logfire_write_token` | Logfire write token | `""` |
| `env.otel_exporter_otlp_endpoint` | OpenTelemetry endpoint | `""` |
| `env.encryption_salt` | Encryption salt for provider settings | `""` |
| `env.redis_url` | Redis URL for persistence | `""` |
| `env.token_count_overhead` | Token count overhead multiplier | `""` |
| `env.enable_storage_reconciliation` | Enable storage reconciliation | `""` |
| `persistence.enabled` | Enable persistent storage | `true` |
| `persistence.size` | PVC size | `1Gi` |

### Automatic Configuration

The chart automatically configures:
- **HOST**: Always set to `0.0.0.0` for container binding
- **PORT**: Automatically set to `service.port` value

## Secret Management

The chart supports two modes for managing sensitive environment variables:

### Option 1: Let the Chart Create the Secret

Simply set values under `env` and the chart will create a Kubernetes secret automatically:

```yaml
env:
  logfire_write_token: "your-logfire-token"
  otel_exporter_otlp_endpoint: "https://your-otel-endpoint"
  encryption_salt: "your-encryption-salt"
  redis_url: "redis://redis:6379/0"
  token_count_overhead: "1.20"
  enable_storage_reconciliation: "true"
```

Or via `--set`:

```bash
helm install qalby-proxy . \
  --set env.redis_url="redis://redis:6379/0" \
  --set env.encryption_salt="my-secret-salt"
```

### Option 2: Use an Existing Secret

Create your own secret first, then reference it:

**1. Create the secret manually:**

```yaml
# my-livellm-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: livellm-proxy-env
type: Opaque
stringData:
  LOGFIRE_WRITE_TOKEN: "your-logfire-token"
  OTEL_EXPORTER_OTLP_ENDPOINT: "https://your-otel-endpoint"
  ENCRYPTION_SALT: "your-encryption-salt"
  REDIS_URL: "redis://redis:6379/0"
  TOKEN_COUNT_OVERHEAD: "1.20"
  ENABLE_STORAGE_RECONCILIATION: "true"
```

```bash
kubectl apply -f my-livellm-secret.yaml
```

**2. Reference the secret in values:**

```yaml
existingSecret: "livellm-proxy-env"
```

Or via `--set`:

```bash
helm install qalby-proxy . --set existingSecret=livellm-proxy-env
```

### Available Secret Keys

| Key | Description | Required |
|-----|-------------|----------|
| `LOGFIRE_WRITE_TOKEN` | Logfire write token for observability | No |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OpenTelemetry exporter endpoint | No |
| `ENCRYPTION_SALT` | Salt for encrypting provider settings | No |
| `REDIS_URL` | Redis connection URL (enables Redis persistence) | No |
| `TOKEN_COUNT_OVERHEAD` | Safety multiplier for token counting (default: 1.20) | No |
| `ENABLE_STORAGE_RECONCILIATION` | Enable file/Redis sync (default: true) | No |

## Example Configurations

### Basic Installation (File Storage)

```yaml
replicaCount: 1

persistence:
  enabled: true
  size: 1Gi

env:
  encryption_salt: "my-secret-salt"
```

### Redis Persistence

```yaml
replicaCount: 2

persistence:
  enabled: true  # Still needed for reconciliation backup

env:
  redis_url: "redis://redis-master:6379/0"
  encryption_salt: "my-secret-salt"
  enable_storage_reconciliation: "true"
```

### Production with Existing Secret

```yaml
replicaCount: 3

existingSecret: "livellm-proxy-production-env"

persistence:
  enabled: true
  size: 5Gi

ingress:
  enabled: true
  className: "nginx"
  hosts:
    - host: livellm.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

### Full values.yaml Example

```yaml
replicaCount: 2

image:
  repository: kamasalyamov/livellm-proxy
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8000

env:
  logfire_write_token: "your-logfire-token"
  otel_exporter_otlp_endpoint: "https://your-otel-endpoint"
  encryption_salt: "your-encryption-salt"
  redis_url: "redis://redis:6379/0"
  token_count_overhead: "1.20"
  enable_storage_reconciliation: "true"

persistence:
  enabled: true
  size: 2Gi

ingress:
  enabled: true
  className: "nginx"
  hosts:
    - host: qalby-proxy.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

## Development

### Local Development

1. Clone this repository
2. Make your changes
3. Test the chart locally:

```bash
# Lint the chart
helm lint .

# Test template rendering
helm template qalby-proxy .

# Test with specific values
helm template qalby-proxy . --set env.redis_url="redis://localhost:6379"

# Dry run installation
helm install qalby-proxy . --dry-run --debug
```

### Chart Structure

```
chart/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default configuration values
├── templates/              # Kubernetes manifest templates
│   ├── deployment.yaml     # Deployment configuration
│   ├── service.yaml        # Service configuration
│   ├── secret.yaml         # Secret for env vars (if not using existingSecret)
│   ├── ingress.yaml        # Ingress configuration
│   ├── hpa.yaml           # Horizontal Pod Autoscaler
│   ├── pvc.yaml           # Persistent Volume Claim
│   ├── serviceaccount.yaml # Service Account
│   └── tests/             # Test templates
└── README.md              # This file
```

## Upgrading

When upgrading, be aware that:

1. **Secrets**: If you switch from `env` values to `existingSecret`, the chart-created secret will be removed
2. **PVC**: Persistent volume claims are not deleted on upgrade by default
3. **Breaking changes**: Check the changelog for any breaking changes between versions

```bash
helm upgrade qalby-proxy . -f values.yaml
```

## Uninstalling

```bash
helm uninstall qalby-proxy
```

**Note**: PVCs are not automatically deleted. To fully clean up:

```bash
kubectl delete pvc -l app.kubernetes.io/instance=qalby-proxy
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

## License

This chart is licensed under the same license as the Qalby Proxy application.

## Support

For issues and questions:
- Create an issue in the repository
- Contact the maintainers

## Changelog

### 0.2.0
- Added secret management (auto-created or existing secret support)
- Added Redis URL configuration
- Added token count overhead configuration
- Added storage reconciliation toggle
- Removed custom env vars in favor of explicit configuration
- All sensitive env vars now stored in Kubernetes secrets

### 0.1.0
- Initial release
- Automatic host/port configuration
- Support for Logfire and OpenTelemetry
- Health checks and autoscaling support
