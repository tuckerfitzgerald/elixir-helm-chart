# Elixir Helm Chart

A mostly production-ready Helm chart for deploying Elixir/Phoenix applications to Kubernetes.

## Features

- 🚀 **Database Migrations**: Automated database migration job before deployment
- 📈 **Autoscaling**: Horizontal Pod Autoscaler (HPA) support with CPU and memory metrics
- 💾 **Resource Management**: Configurable CPU and memory requests/limits
- 🏥 **Health Checks**: Readiness, liveness, and startup probes
- 🛡️ **High Availability**: Pod Disruption Budget support
- 🔧 **Flexible Configuration**: Environment variables, volumes, sidecars, and more

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- kubectl configured to access your cluster
- A container registry with your Elixir application image

## Installation

### Quick Start

```bash
# Install with default values
helm install my-app ./elixir-helm-chart

# Install with custom values file
helm install my-app ./elixir-helm-chart -f my-values.yaml

# Install with custom values
helm install my-app ./elixir-helm-chart \
  --set image.repository=my-registry/my-app \
  --set image.tag=v1.0.0 \
  --set env.APP_ENV=production
```

### Upgrade

```bash
helm upgrade my-app ./elixir-helm-chart -f my-values.yaml
```

### Uninstall

```bash
helm uninstall my-app
```

## Configuration

The following table lists the configurable parameters and their default values:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | Docker image repository | `ghcr.io/stordco/sre-technical-challenge` |
| `image.tag` | Docker image tag | `latest` |
| `env.DATABASE_URL` | PostgreSQL connection string | `postgresql://postgres:password@sre-db-postgresql/sre-technical-challenge` |
| `env.POOL_SIZE` | Database connection pool size | `10` |
| `env.PORT` | Application port | `4000` |
| `env.APP_ENV` | Application environment | `development` |
| `env.SECRET_KEY_BASE` | 64 char random string | `kexHrtev472WDA8M4CbnMitJwUTV7LRtRZsUAMomkkb6qEh4jGWDG52abuVXKMGn` |
| `enableDbMigration` | Enable database migration job | `true` |
| `replicaCount` | Number of replicas (when autoscaling disabled) | `{}` |
| `autoscale.enabled` | Enable HPA | `false` |
| `autoscale.minReplicas` | Minimum replicas for HPA | `1` |
| `autoscale.maxReplicas` | Maximum replicas for HPA | `10` |
| `autoscale.targetCPUUtilizationPercentage` | CPU threshold for scaling | `70` |
| `autoscale.targetMemoryUtilizationPercentage` | Memory threshold for scaling | `70` |
| `podDisruptionBudget.enabled` | Enable Pod Disruption Budget | `false` |
| `podDisruptionBudget.minAvailable` | Minimum available pods | `50%` |
| `startupProbe.enabled` | Enable startup probe | `false` |

## Detailed Configuration Examples

### Basic Production Setup

```yaml
image:
  repository: my-registry/my-elixir-app
  tag: v1.2.3

env:
  APP_ENV: production
  PORT: 4000
  DATABASE_URL: postgresql://user:pass@db-host/myapp_prod
  POOL_SIZE: 20
  SECRET_KEY_BASE: kexHrtev472WDA8M4CbnMitJwUTV7LRtRZsUAMomkkb6qEh4jGWDG52abuVXKMGn

resources:
  cpu:
    requests: "200m"
    limits: "1000m"
  memory:
    requests: "256Mi"
    limits: "1Gi"

autoscale:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

podDisruptionBudget:
  enabled: true
  minAvailable: 50%
```

### With Custom Environment Variables

```yaml
env:
  APP_ENV: production
  PORT: 4000
  DATABASE_URL: postgresql://user:pass@db/myapp

extraEnvVars:
  - name: API_KEY
    value: "your-api-key"
  - name: FEATURE_FLAG_X
    value: "enabled"
```

### With Volumes and Volume Mounts

```yaml
volumes:
  - name: config
    configMap:
      name: app-config
  - name: secrets
    secret:
      secretName: app-secrets

volumeMounts:
  - name: config
    mountPath: /etc/app/config
  - name: secrets
    mountPath: /etc/app/secrets
    readOnly: true
```

### With Sidecar Container

```yaml
sidecars:
  - name: log-shipper
    image: fluent-bit:latest
    volumeMounts:
      - name: logs
        mountPath: /var/log
```

### Database Migration Configuration

The chart includes an optional database migration job that runs before deployment:

```yaml
enableDbMigration: true
```

The migration job:
- Runs as a Kubernetes Job with `pre-install` and `pre-upgrade` hooks
- Automatically deletes after successful completion
- Uses a timestamped name to ensure uniqueness

## Application Requirements

Your Elixir/Phoenix application should:

1. **Health Endpoint**: Expose a `/_health` endpoint for health checks
2. **Migration Command**: Support a `migrate` command for database migrations
3. **Server Command**: Support a `server` command to start the application
4. **Port Configuration**: Accept port configuration via `PORT` environment variable

### Example Dockerfile

```dockerfile
FROM elixir:1.15-alpine

WORKDIR /app

# Install dependencies
COPY mix.exs mix.lock ./
RUN mix deps.get --only prod

# Copy application
COPY . .

# Compile
RUN mix compile

# Default command
CMD ["server"]
```

## Architecture

The chart creates the following Kubernetes resources:

- **Deployment**: Main application deployment
- **Service**: ClusterIP service for internal access
- **Job**: Database migration job (if enabled)
- **HorizontalPodAutoscaler**: Auto-scaling (if enabled)
- **PodDisruptionBudget**: HA protection (if enabled)

## Health Checks

The chart configures three types of health checks:

1. **Readiness Probe**: Checks `/_health` endpoint to determine if pod can receive traffic
2. **Liveness Probe**: Checks `/_health` endpoint to determine if pod is healthy
3. **Startup Probe**: Optional probe for applications with long startup times

## Autoscaling

When `autoscale.enabled: true`, the chart creates an HPA that:

- Monitors CPU and memory utilization
- Scales pods based on configured thresholds
- Supports both scale-up and scale-down policies
- Can monitor additional containers (sidecars)

## Troubleshooting

### Check Pod Status

```bash
kubectl get pods -l app=<release-name>
kubectl describe pod <pod-name>
```

### View Logs

```bash
kubectl logs -l app=<release-name>
kubectl logs -l app=<release-name> -c <container-name>  # For sidecars
```

### Check Migration Job

```bash
kubectl get jobs -l app=<release-name>
kubectl logs job/<release-name>-db-migration-<timestamp>
```

### Debug Deployment

```bash
# Dry run to see rendered templates
helm template my-app ./elixir-helm-chart

# Check values
helm get values my-app

# Debug with verbose output
helm install my-app ./elixir-helm-chart --debug --dry-run
```

### Common Issues

**Pods not starting:**
- Check resource limits and requests
- Verify image pull secrets if using private registry
- Check ServiceAccount permissions

**Database migrations failing:**
- Verify DATABASE_URL is correct
- Check database connectivity from cluster
- Review migration job logs

**Health checks failing:**
- Ensure `/_health` endpoint exists and responds with 200
- Check PORT environment variable matches application configuration
- Verify network policies allow health check traffic

## Development

### Linting

```bash
helm lint .
```

### Testing

```bash
# Validate templates
helm template my-app . --debug

# Test with a specific values file
helm template my-app . -f values.yaml
```
