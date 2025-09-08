# PR-Scoped Canary Deployments with Kustomize, Traefik, Linkerd & Argo CD

> Test new versions **per Pull Request** in a real cluster without disrupting others.
> Requests tagged with `X-Canary: <PR#>` go to canary version, everything else goes to stable.
> Auto-created on PR open, auto-pruned on PR close.

---

## Table of Contents

* [Why this approach](#why-this-approach)
* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Repository Structure](#repository-structure)
* [Service Architecture](#service-architecture)
* [Kustomize Configuration](#kustomize-configuration)
* [Traefik Routing](#traefik-routing)
* [Linkerd Routing](#linkerd-routing)
* [Argo CD ApplicationSets](#argo-cd-applicationsets)
* [GitHub Actions](#github-actions)
* [Observability Stack](#observability-stack)
* [Local Development](#local-development)
* [Promotion & Cleanup Lifecycle](#promotion--cleanup-lifecycle)
* [Troubleshooting](#troubleshooting)

---

## Why this approach

* **Fast feedback** — Every PR deploys a live, isolated canary you can hit immediately.
* **Zero disruption** — Only requests with `X-Canary: <PR#>` hit the PR namespace; everyone else stays on stable.
* **Production parity** — Test in the *real* cluster with real dependencies.
* **Automatic cleanup** — When the PR closes, Argo CD prunes the entire PR namespace automatically.
* **GitOps alignment** — Everything is declarative, auditable, reproducible.
* **Easy demos** — Share the header value and a URL; no bespoke staging envs.
* **Seamless promotion** — Release is just updating the stable image tag.
* **Single source of truth** — Same base manifests deployed everywhere, no v1/v2 duplication.

---

## Architecture

* **Web → API (HTTP)** routed by **Traefik**:
  * `X-Canary: <PR#>` → **`api-service`** in **`api-canary-pr-<PR#>`** namespace
  * otherwise → **`api-service`** in **`apps`** namespace (stable)

* **API → Store (gRPC)** routed by **Linkerd (Gateway API)**:
  * Same `X-Canary` value propagated as **gRPC metadata**.
  * `X-Canary: <PR#>` → **`store-service`** in **`store-canary-pr-<PR#>`** namespace
  * otherwise → **`store-service`** in **`apps`** namespace (stable)

> You'll run **the same base manifests** in different namespaces per PR.
> Cross-namespace routing via GRPCRoute + ReferenceGrant enables east-west traffic.

---

## Prerequisites

* Kubernetes cluster with:
  * **Argo CD** + **ApplicationSet** controller
  * **Traefik** (CRDs installed for `IngressRoute`)
  * **Linkerd** with **Gateway API dynamic request routing** enabled
* Container registry (GHCR) and CI secrets to push images
* DNS for your dev domain (e.g., `api.localhost`)
* GitHub token (PAT) for ApplicationSet PR generator

---

## Repository Structure

### Current Implementation

```
.
├── api-service/                    # HTTP REST API service
│   ├── main.go                     # Application entry point
│   ├── internal/                   # Internal packages
│   │   ├── client/                 # gRPC client for store-service
│   │   ├── server/                 # HTTP handlers and mappers
│   │   ├── canaryctx/              # Canary context handling
│   │   ├── metrics/                # Prometheus metrics
│   │   └── tracing/                # OpenTelemetry tracing
│   ├── .github/workflows/          # CI/CD workflows
│   ├── .gitignore                  # Git ignore rules
│   ├── go.mod                      # Go dependencies
│   ├── Makefile                    # Build and development tasks
│   ├── Dockerfile                  # Container image
│   └── README.md                   # Service documentation
│
├── store-service/                  # gRPC backend service
│   ├── main.go                     # Application entry point
│   ├── internal/                   # Internal packages
│   │   ├── data/                   # DynamoDB data layer
│   │   ├── server/                 # gRPC server implementation
│   │   ├── canaryctx/              # Canary context handling
│   │   ├── metrics/                # Prometheus metrics
│   │   └── tracing/                # OpenTelemetry tracing
│   ├── proto/                      # Protocol buffer definitions
│   │   ├── store.proto             # Service definition
│   │   ├── generate.go             # Code generation
│   │   ├── go/                     # Generated Go code
│   │   └── ruby/                   # Ruby gem structure
│   ├── .github/workflows/          # CI/CD workflows
│   ├── .gitignore                  # Git ignore rules
│   ├── go.mod                      # Go dependencies
│   ├── Makefile                    # Build and development tasks
│   ├── Dockerfile                  # Container image
│   └── README.md                   # Service documentation
│
├── manifests-microservices/        # Centralized manifests repository
│   ├── applications/               # Argo CD applications
│   │   ├── production/             # Production environment
│   │   ├── staging/                # Staging environment
│   │   └── integration-001/        # Integration environment
│   ├── kustomize/                  # Kustomize configurations
│   │   ├── api-service/            # API service manifests
│   │   │   ├── base/               # Base manifests
│   │   │   └── overlays/           # Environment overlays
│   │   └── store-service/          # Store service manifests
│   │       ├── base/               # Base manifests
│   │       └── overlays/           # Environment overlays
│   └── .github/workflows/          # Manifest update workflows
│
├── localdev/                       # Local development setup
│   ├── infrastructure/             # Helm charts and values
│   ├── scripts/                    # Helper scripts
│   ├── docs/                       # Development documentation
│   ├── Makefile                    # Local development tasks
│   └── setup.sh                    # Environment setup
│
└── extension-canary/               # Chrome extension for testing
    ├── manifest.json               # Extension manifest
    ├── src/                        # Extension source code
    └── README.md                   # Extension documentation
```

---

## Service Architecture

### API Service (HTTP Edge)
- **Language**: Go
- **Framework**: Standard library with custom HTTP handlers
- **Protocol**: HTTP REST API
- **Database**: None (proxies to store-service)
- **Observability**: Prometheus metrics, OpenTelemetry tracing
- **Port**: 8080 (HTTP), 9090 (metrics)

### Store Service (gRPC Backend)
- **Language**: Go
- **Framework**: gRPC
- **Protocol**: gRPC
- **Database**: DynamoDB
- **Observability**: Prometheus metrics, OpenTelemetry tracing
- **Port**: 8080 (gRPC), 9090 (metrics)

---

## Kustomize Configuration

### Centralized Manifests
All Kubernetes manifests are centralized in the `manifests-microservices` repository:

```
manifests-microservices/
├── kustomize/
│   ├── api-service/
│   │   ├── base/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── hpa.yaml
│   │   │   ├── servicemonitor.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── production/
│   │       ├── staging/
│   │       └── integration-001/
│   └── store-service/
│       ├── base/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   ├── hpa.yaml
│       │   ├── servicemonitor.yaml
│       │   ├── grpc-route-template.yaml
│       │   ├── reference-grant-template.yaml
│       │   └── kustomization.yaml
│       └── overlays/
│           ├── production/
│           ├── staging/
│           └── integration-001/
```

### Single Base Manifest Approach
- **One deployment.yaml per service** - no v1/v2 duplication
- **Environment-specific overlays** - patch image tags and configurations
- **PR canaries** - ApplicationSets patch image tags dynamically

---

## Traefik Routing

### Stable Routing
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: api-service-stable
  namespace: apps
spec:
  entryPoints: [web]
  routes:
    - match: Host(`api.localhost`) && PathPrefix(`/`)
      services:
        - name: api-service
          namespace: apps
          port: 80
```

### PR Canary Routing
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: api-service-canary-pr-{{ number }}
  namespace: apps
spec:
  entryPoints: [web]
  routes:
    - match: Host(`api.localhost`) && Headers(`X-Canary`, `{{ number }}`)
      services:
        - name: api-service
          namespace: api-canary-pr-{{ number }}
          port: 80
```

---

## Linkerd Routing

### Cross-Namespace gRPC Routing
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: store-service-canary-pr-{{ number }}
  namespace: apps
spec:
  parentRefs:
    - kind: Service
      name: store-service
  rules:
    - matches:
        - headers:
            - name: X-Canary
              value: "{{ number }}"
      backendRefs:
        - name: store-service
          namespace: store-canary-pr-{{ number }}
          port: 80
    - backendRefs:
        - name: store-service
          namespace: apps
          port: 80
```

### Reference Grant
```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-apps-grpcroute
  namespace: store-canary-pr-{{ number }}
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: GRPCRoute
      namespace: apps
  to:
    - group: ""
      kind: Service
      name: store-service
```

---

## Argo CD ApplicationSets

### Environment-Centric Structure
```
manifests-microservices/
├── applications/
│   ├── production/
│   │   ├── api-service-production.yaml
│   │   └── store-service-production.yaml
│   ├── staging/
│   │   ├── api-service-staging.yaml
│   │   └── store-service-staging.yaml
│   └── integration-001/
│       ├── api-service-integration.yaml
│       ├── store-service-integration.yaml
│       ├── applicationset-api-service-pr.yaml
│       ├── applicationset-store-service-pr.yaml
│       └── applicationset-store-service-routing-pr.yaml
```

### PR Canary ApplicationSets
- **API Service**: Deploys base manifests to PR namespace
- **Store Service**: Deploys base manifests to PR namespace  
- **Store Routing**: Creates GRPCRoute + ReferenceGrant for cross-namespace access

---

## GitHub Actions

### Generic Workflows
All workflows use `GITHUB_REPOSITORY` environment variable for dynamic naming:

```yaml
env:
  GO_VERSION: '1.25.1'
  DOCKER_IMAGE: ghcr.io/${{ github.repository }}
  REGISTRY: ghcr.io
```

### Workflow Types
1. **CI Pipeline** - Build and test on PR, build and push on main
2. **PR Canary** - Build canary image for PR testing
3. **Release** - Create staging/production PRs and GitHub releases

### Release Process
1. **Tag creation** triggers release workflow
2. **Integration** automatically updated to latest
3. **Staging PR** created for review
4. **Production PR** created for review
5. **GitHub Release** created with artifacts

---

## Observability Stack

### Local Development
- **Prometheus** - Metrics collection
- **Grafana** - Visualization and dashboards
- **Tempo** - Distributed tracing backend
- **Loki** - Log aggregation
- **Correlation** - Traces, logs, and metrics linked

### Application Metrics
- **HTTP metrics** - Request rate, latency, error rate
- **gRPC metrics** - Call rate, latency, error rate
- **Business metrics** - Custom application metrics
- **ServiceMonitor** - Prometheus scraping configuration

### Distributed Tracing
- **OpenTelemetry** - Instrumentation and export
- **Automatic HTTP tracing** - Request/response spans
- **Automatic gRPC tracing** - Client/server spans
- **Custom business spans** - Application-specific tracing
- **Trace propagation** - Headers and metadata

---

## Local Development

### Prerequisites
- **Docker Desktop** with Kubernetes enabled
- **minikube** (8 CPUs, 32GB RAM)
- **kubectl**, **helm**, **argocd** CLI
- **telepresence** for local development

### Setup
```bash
# Start minikube with sufficient resources
minikube start --memory=32768 --cpus=8

# Deploy infrastructure
cd localdev
make infrastructure

# Setup ArgoCD applications
make argocd-setup

# Start port forwards
make port-forwards
```

### Development Workflow
```bash
# Intercept service for local development
telepresence intercept api-service --namespace apps --port 8080

# Run local service
make dev-run

# Test canary deployment
curl -H "X-Canary: 123" http://api.localhost/health
```

---

## Promotion & Cleanup Lifecycle

### PR Lifecycle
1. **PR Created** → GitHub Actions builds `pr-{{ number }}` image
2. **ApplicationSet Triggers** → Deploys to `<service>-canary-pr-<PR#>` namespace
3. **Routing Updates** → Traefik/Linkerd route `X-Canary: <PR#>` to canary
4. **Testing** → Use Chrome extension or curl with header
5. **PR Updated** → New image built, same namespace updated
6. **PR Closed** → Argo CD prunes namespace automatically

### Release Lifecycle
1. **Tag Created** → Release workflow triggered
2. **Integration Updated** → Automatically updated to latest
3. **Staging PR** → Created for review and testing
4. **Production PR** → Created for review and approval
5. **GitHub Release** → Created with release notes and artifacts

---

## Troubleshooting

### Common Issues
- **Header present but API still stable** - Check Traefik IngressRoute match
- **API routes to canary but backend still stable** - Verify gRPC interceptors
- **CORS errors** - Add `Access-Control-Allow-Headers: X-Canary`
- **ApplicationSet didn't create PR app** - Check PAT and repo permissions
- **No auto-cleanup** - Ensure `automated.prune: true` in ApplicationSet

### Debugging Commands
```bash
# Check ApplicationSet status
kubectl get applicationset -n argocd

# Check PR applications
kubectl get applications -n argocd | grep canary

# Check routing
kubectl get ingressroute -n apps
kubectl get grpcroute -n apps

# Check canary pods
kubectl get pods -n api-canary-pr-123
kubectl get pods -n store-canary-pr-123
```

---

## Current Implementation Status

### ✅ Completed
- **Service Architecture** - API and Store services with proper separation
- **Centralized Manifests** - Single source of truth in manifests-microservices
- **PR Canary Deployments** - Full ApplicationSet automation
- **Observability Stack** - Prometheus, Grafana, Tempo, Loki
- **Local Development** - Complete minikube setup with Telepresence
- **CI/CD Pipelines** - Generic GitHub Actions workflows
- **Chrome Extension** - Browser tooling for testing
- **Documentation** - Comprehensive READMEs and guides

### 🎯 Key Features
- **Single Base Manifests** - No v1/v2 duplication
- **Environment-Centric Structure** - Production, staging, integration
- **Cross-Namespace Routing** - GRPCRoute + ReferenceGrant
- **Automatic Cleanup** - PR namespaces pruned on close
- **Generic Workflows** - Reusable across services
- **Full Observability** - Metrics, logs, and traces

This implementation is **production-ready** and follows modern GitOps practices with comprehensive observability and developer experience.
