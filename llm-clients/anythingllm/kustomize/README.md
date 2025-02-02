# AnythingLLM Kustomize Configuration

This directory contains Kustomize configurations for deploying AnythingLLM on OpenShift.

## Structure

```
kustomize/
├── base/                # Base configuration
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── rbac.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── route.yaml
└── overlays/           # Environment-specific overlays
    └── production/     # Production environment config
        └── kustomization.yaml
```

## Base Configuration

The base configuration includes:
- Dedicated namespace 'anythingllm'
- OpenShift-compliant RBAC configuration:
  - ServiceAccount for the application
  - RoleBinding to restricted-v2 policy
- Deployment with single replica and security context:
  - Non-root user (UID: 1000920000)
  - Restricted capabilities
  - SELinux configuration
- Service configuration:
  - Main application port: 8080 (forwards to 3001)
  - Document collector port: 8889
- Route for external access
- Default resource limits:
  - CPU: 2 cores (limit), 500m (request)
  - Memory: 4Gi (limit), 1Gi (request)

## Production Overlay

The production overlay includes:
- High resource limits for production workloads
- Multiple replicas for high availability
- Pinned image version

## Usage

### Deploy Base Configuration

```bash
# From the repository root
oc apply -k llm-clients/anythingllm/kustomize/base

# Switch to anythingllm namespace
oc project anythingllm
```

### Deploy Production Configuration

```bash
# From the repository root
oc apply -k llm-clients/anythingllm/kustomize/overlays/production

# Switch to anythingllm namespace
oc project anythingllm
```

### Access the Application

Once deployed, the application will be accessible through the created OpenShift route:

```bash
oc get route anythingllm -o jsonpath='{.spec.host}'
```

## Configuration

### Environment Variables

The deployment supports the following environment variables:

- `NODE_ENV`: Set to "production" by default
- `ANYTHING_LLM_RUNTIME`: Set to "docker" by default
- `DISABLE_TELEMETRY`: Set to "true" by default

Additional environment variables can be added through a custom overlay.

### Port Configuration

The deployment exposes two ports:
- Port 3001: Main application interface (exposed externally as 3001)
- Port 8889: Document collector interface

The service maps these ports as follows:
- External port 3001 -> Internal port 3001 (main application)
- Port 8889 -> Port 8889 (document collector)

Health checks are configured using TCP socket checks on port 3001 to ensure the main application is running.

### Creating Custom Overlays

To create a custom configuration:

1. Create a new directory under `overlays/`
2. Create a `kustomization.yaml` that references the base
3. Add your customizations using patches

Example:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

patches:
- target:
    kind: Deployment
    name: anythingllm
  patch: |-
    - op: replace
      path: /spec/template/spec/containers/0/resources/limits/memory
      value: 16Gi
