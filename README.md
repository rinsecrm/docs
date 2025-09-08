# PR-Scoped Canary Deployment Example

This repository demonstrates a complete implementation of PR-scoped canary deployments using Kustomize, Traefik, Linkerd, and Argo CD.

## Repository Structure

```
.
├── api-service/                    # HTTP API service (Go + Gorilla Mux)
│   ├── .github/workflows/         # CI/CD workflows
│   ├── internal/                  # Application code
│   └── proto/                     # Protocol buffer definitions
├── store-service/                  # gRPC backend service (Go + DynamoDB) 
│   ├── .github/workflows/         # CI/CD workflows
│   ├── internal/                  # Application code
│   └── proto/                     # Protocol buffer definitions
├── manifests-microservices/        # Centralized manifests repository
│   ├── applications/              # ArgoCD Application manifests
│   │   ├── production/            # Production environment
│   │   ├── staging/               # Staging environment
│   │   └── integration-001/       # Integration environment + PR canaries
│   ├── kustomize/                 # Kustomize configurations
│   │   ├── api-service/           # API service manifests
│   │   └── store-service/         # Store service manifests
│   └── .github/workflows/         # Cleanup workflows
├── extension-canary/              # Browser extension for X-Canary header injection
└── docs/                          # Documentation
    ├── README.md                  # This file
    └── plan.md                    # Detailed implementation plan
```

## Services

### API Service
- **Technology**: Go, Gorilla Mux, gRPC client
- **Purpose**: HTTP REST API that forwards requests to store-service
- **Canary Routing**: Traefik routes based on `X-Canary` header
- **Port**: 8080

### Store Service  
- **Technology**: Go, gRPC server, DynamoDB, following [store-automations](https://github.com/nicklanng/store-automations) patterns
- **Purpose**: Multi-tenant store management with inventory tracking
- **Features**: Enhanced data model, structured logging, comprehensive API
- **Canary Routing**: Linkerd GRPCRoute based on `X-Canary` metadata
- **Port**: 8080

## Quick Start

### Prerequisites

- Kubernetes cluster with Argo CD, Traefik, and Linkerd installed
- Docker registry access
- GitHub repository setup

### 1. Deploy Base Services

```bash
# Deploy stable versions
kubectl apply -k manifests-microservices/kustomize/api-service/overlays/production/
kubectl apply -k manifests-microservices/kustomize/store-service/overlays/production/
```

### 2. Setup Argo CD Applications

```bash
# Apply ArgoCD applications for all environments
kubectl apply -f manifests-microservices/applications/production/
kubectl apply -f manifests-microservices/applications/staging/
kubectl apply -f manifests-microservices/applications/integration-001/
```

### 3. Configure GitHub Secrets

Add these secrets to your GitHub repositories:

**For service repositories (`api-service`, `store-service`):**
- `MANIFESTS_TOKEN` - GitHub Personal Access Token with `repo` permissions for updating manifests

**For manifests repository (`manifests-microservices`):**
- `GITHUB_TOKEN` - Automatically provided by GitHub Actions

### 4. Test Canary Deployment

1. Create a PR in either service repository
2. Wait for GitHub Actions to build and push the image
3. Argo CD will automatically create the canary deployment (within 2 minutes)
4. Test with: `curl -H "X-Canary: <PR#>" https://api.dev.example.com/health`

### 5. Update Canary During Development

1. Make changes and push new commits to your PR branch
2. GitHub Actions automatically builds new image with new SHA
3. Argo CD detects the change and updates the same canary namespace
4. Test the updated canary (same URL, same header)

**Timeline**: Push → Build (2-3min) → Deploy (1-2min) → Ready to test

## Canary Routing

### HTTP (Edge) - Traefik
- **Stable**: All requests → `api` service in `apps` namespace  
- **Canary**: Requests with `X-Canary: <PR#>` → `api` service in `apps` namespace

### gRPC (East-West) - Linkerd
- **Stable**: All requests → `store` service in `apps` namespace
- **Canary**: Requests with `X-Canary: <PR#>` metadata → `store` service in `apps` namespace via cross-namespace GRPCRoute with ReferenceGrant

## Chrome Extension

Install the browser extension from `chrome-extension/` to automatically inject X-Canary headers:

1. Load unpacked extension in Chrome
2. Click extension icon and configure target domain and PR number
3. All requests to your configured domain will include the `X-Canary` header
4. Works with any domain: `api.dev.example.com`, `localhost:8080`, etc.

## Configuration

Update these values in the manifests:
- `ghcr.io/rinsecrm` → your container registry
- `api.dev.example.com` → your API domain
- `rinsecrm` → your GitHub organization
- Namespace names as needed

## Observability

Both services log canary context for monitoring:
- Look for "called with canary PR: X" in service logs
- Use `X-Canary-Echo` response header for debugging
- Set up metrics/dashboards comparing v1 vs v2 performance

## Cleanup

PR deployments automatically clean up when PRs are closed, thanks to Argo CD's prune policy.

For manual cleanup:
```bash
kubectl delete application api-service-canary-pr-<PR#>
kubectl delete application store-service-canary-pr-<PR#>
kubectl delete application api-service-canary-router-pr-<PR#>
kubectl delete application store-service-routing-pr-<PR#>
kubectl delete application store-service-reference-grant-pr-<PR#>
```

## Troubleshooting PR Updates

### Canary Not Updating After Push

1. **Check if image was built:**
   ```bash
   # Check GitHub Actions for your PR
   # Look for new image tag: ghcr.io/rinsecrm/api-service:pr-<PR#>
   ```

2. **Check ApplicationSet status:**
   ```bash
   kubectl get applicationset api-pr-canaries -o yaml
   # Look for recent generation updates
   ```

3. **Force ApplicationSet refresh:**
   ```bash
   kubectl annotate applicationset api-pr-canaries argocd.argoproj.io/refresh=now
   ```

4. **Manual sync specific app:**
   ```bash
   argocd app sync api-canary-pr-<PR#>
   ```

### Speeding Up Updates

- **Default**: 2-minute polling interval
- **With webhooks**: Immediate updates
- **With CI sync**: ~30 seconds after build completes

## Enhanced Store Service Features

The store service has been significantly enhanced following production patterns from [store-automations](https://github.com/nicklanng/store-automations):

### Multi-Tenancy
- All operations are tenant-scoped with `tenant_id`
- Complete data isolation between tenants
- Tenant context extracted from HTTP headers (`X-Tenant-ID`)

### Enhanced Data Model
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "tenant_id": 1,
  "name": "Wireless Headphones",
  "description": "Premium noise-canceling headphones",
  "price": 299.99,
  "category": "electronics",
  "status": "active",
  "sku": "WH-001",
  "inventory_count": 150,
  "tags": ["wireless", "noise-canceling", "premium"],
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z",
  "created_by": "user123",
  "updated_by": "user123"
}
```

### API Enhancements

#### New HTTP Endpoints
- `POST /api/v1/items` - Create items with full metadata
- `GET /api/v1/items?category=electronics&status=active&search=wireless` - Advanced filtering
- `PATCH /api/v1/items/{id}/inventory` - Inventory management

#### Query Parameters
- `category`: Filter by electronics, clothing, books, home, sports
- `status`: Filter by active, inactive, out_of_stock, discontinued  
- `search`: Search in name/description
- `page_size`: Pagination size
- `page_token`: Pagination token

#### Example Inventory Update
```bash
curl -X PATCH http://localhost:8080/api/v1/items/123/inventory \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: 1" \
  -H "X-User-ID: manager" \
  -d '{
    "quantity_change": -5,
    "reason": "Sale completed",
    "updated_by": "cashier123"
  }'
```

### Architecture Improvements

#### Core Structure
```
store-service/
├── core/                    # Shared utilities
│   ├── logging/            # Structured logging
│   ├── metrics/            # Metrics collection
│   └── tracing/            # Distributed tracing
├── internal/               # Application-specific packages
│   ├── data/              # Data access layer
│   ├── server/            # gRPC server implementation
│   └── canaryctx/         # Canary context handling
└── proto/                 # Protocol buffer definitions
```

#### Structured Logging
- JSON formatted logs with consistent fields
- Request tracing with correlation IDs
- Performance metrics and error tracking
- Configurable log levels (debug for development)

#### Configuration Management
- Environment-based configuration with validation
- Support for local DynamoDB endpoint
- Graceful shutdown handling

#### Production-Ready Docker Images
- **Multi-stage builds** with scratch-based final images
- **Static linking** with `CGO_ENABLED=0` for maximum compatibility
- **Security optimizations** with `-ldflags="-w -s"` to strip debug info
- **Minimal attack surface** using `FROM scratch` instead of Alpine
- **CA certificates** properly included for HTTPS/TLS connections
- **CI-optimized Dockerfiles** for faster builds in CI/CD pipelines

#### Comprehensive CI/CD Pipeline
- **Multi-job workflows** with parallel execution for faster feedback
- **Security scanning** with Gosec and SARIF reporting
- **Code quality** with golangci-lint and formatting checks
- **Artifact management** with binary uploads and Docker image builds
- **Registry integration** with automatic pushes on main branch
- **Makefile-driven** builds for consistent local and CI environments
- **Test isolation** with separate DynamoDB instances for testing

## Development

### Building Services

#### Using Makefile (Recommended)
```bash
# API Service
cd api-service
make help          # Show all available commands
make build         # Build the binary
make test          # Run tests with DynamoDB setup
make dev-db        # Start development DynamoDB
make dev-run       # Run server locally with hot reload

# Store Service  
cd store-service
make help          # Show all available commands
make build         # Build the binary
make test          # Run tests with DynamoDB setup
make dev-db        # Start development DynamoDB
make dev-run       # Run server locally with hot reload
```

#### Manual Build
```bash
# API Service
cd api-service
go mod tidy
make proto
go build -o api-service ./cmd/server

# Store Service  
cd store-service
go mod tidy
make proto
go build -o store-service ./cmd/server
```

### Running Locally

```bash
# Start store service (requires AWS credentials for DynamoDB)
cd store-service
PORT=8081 DYNAMODB_TABLE=local-items go run ./cmd/server

# Start API service
cd api-service  
PORT=8080 STORE_SERVICE_ADDR=localhost:8081 go run ./cmd/server
```

## See Also

- [plan.md](plan.md) - Detailed implementation plan and architecture
- [Argo CD ApplicationSet Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/)
- [Traefik IngressRoute](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/)
- [Linkerd Gateway API](https://linkerd.io/2.14/features/gateway-api/)
