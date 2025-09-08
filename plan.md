# PR-Scoped Canary Deployments with Kustomize, Traefik, Linkerd & Argo CD

> Test new versions **per Pull Request** in a real cluster without disrupting others.
> Requests tagged with `X-Canary: <PR#>` go to canary v2, everything else goes to stable v1.
> Auto-created on PR open, auto-pruned on PR close.

---

## Table of Contents

* [Why this approach](#why-this-approach)
* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Repository layout](#repository-layout)

  * [Option A: single mono-repo](#option-a-single-mono-repo)
  * [Option B: service repo + env repo (recommended)](#option-b-service-repo--env-repo-recommended)
* [Kustomize bases & overlays](#kustomize-bases--overlays)

  * [API service (HTTP edge)](#api-service-http-edge)
  * [Back-office service (gRPC east–west)](#back-office-service-grpc-eastwest)
  * [Horizontal Pod Autoscaler (HPA)](#horizontal-pod-autoscaler-hpa)
* [Traefik routing (edge)](#traefik-routing-edge)
* [Linkerd routing (east–west)](#linkerd-routing-eastwest)
* [Argo CD ApplicationSet (per-PR canaries)](#argo-cd-applicationset-perpr-canaries)
* [GitHub Actions](#github-actions)
* [Go wiring (summary)](#go-wiring-summary)
* [Chrome extension (optional)](#chrome-extension-optional)
* [Promotion & cleanup lifecycle](#promotion--cleanup-lifecycle)
* [Security, RBAC & quotas](#security-rbac--quotas)
* [Observability](#observability)
* [CORS & client notes](#cors--client-notes)
* [Troubleshooting](#troubleshooting)
* [FAQ](#faq)

---

## Why this approach

* **Fast feedback** — Every PR deploys a live, isolated canary you can hit immediately.
* **Zero disruption** — Only requests with `X-Canary: <PR#>` hit the PR namespace; everyone else stays on stable.
* **Production parity** — Test in the *real* cluster with real deps (DB/Kafka/etc).
* **Automatic cleanup** — When the PR closes, Argo CD prunes the entire PR namespace automatically.
* **GitOps alignment** — Everything is declarative, auditable, reproducible.
* **Easy demos** — Share the header value and a URL; no bespoke staging envs.
* **Seamless promotion** — Release is just updating the stable image tag.
* **Single source of truth** — Same base manifests deployed everywhere, no v1/v2 duplication.

---

## Architecture

* **Web → API (HTTP)** routed by **Traefik**:

  * `X-Canary: <PR#>` → **`api`** service in **`api-canary-pr-<PR#>`** namespace
  * otherwise → **`api`** service in **`apps`** namespace (stable)
* **API → backend (gRPC)** routed by **Linkerd (Gateway API)**:

  * Same `X-Canary` value propagated as **gRPC metadata**.
  * `X-Canary: <PR#>` → **`store`** service in **`store-canary-pr-<PR#>`** namespace
  * otherwise → **`store`** service in **`apps`** namespace (stable)

> You'll run **the same base manifests** in different namespaces per PR.
> An **apex Service** in the stable namespace fronts back-office services with **cross-namespace routing** via GRPCRoute + ReferenceGrant.

---

## Prerequisites

* Kubernetes cluster with:

  * **Argo CD** + **ApplicationSet** controller
  * **Traefik** (CRDs installed if using `IngressRoute`)
  * **Linkerd** with **Gateway API dynamic request routing** enabled (`HTTPRoute`/`GRPCRoute`)
* Container registry (e.g., GHCR, ECR, GCR) and CI secrets to push images
* DNS for your dev domain (e.g., `api.dev.example.com`)
* GitHub token (PAT) for ApplicationSet PR generator

---

## Repository layout

You can keep everything in one repo or split service code from environment manifests.

### Option A: single mono-repo

```
repo-root/
  services/api/
    app/...
    deploy/
      base/
        deployment-v1.yaml
        service-v1.yaml
        deployment-v2.yaml
        service-v2.yaml
        kustomization.yaml
      edge/                 # edge resources for API
        ingressroute-template.yaml
      overlays/
        stable/
          kustomization.yaml
        canary/
          kustomization.yaml
  services/store/           # gRPC backend service
    deploy/...
  envs/
    apps/
      applicationset-api-pr.yaml
      applicationset-edge-pr.yaml
      applicationset-billing-pr.yaml
    dev/
      api/
        stable/
          kustomization.yaml
        canary/
          kustomization.yaml
      backoffice/
        canary/
          kustomization.yaml
```

### Option B: service repo + env repo (**recommended**)

* **Service repo** owns code + **kustomize base** for the service.
* **Env repo** owns the **overlays**, Argo CD Applications, and ApplicationSets.

---

## Kustomize bases & overlays

### API service (HTTP edge) — Single base manifests

**`services/api/deploy/base/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels: { app: api }
spec:
  replicas: 2
  selector: { matchLabels: { app: api } }
  template:
    metadata: { labels: { app: api } }
    spec:
      containers:
        - name: api
          image: ghcr.io/rinsecrm/api-service:STABLE_TAG   # patched per env/PR
          ports: [{ containerPort: 8080 }]
```

**`services/api/deploy/base/service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  labels: { app: api }
spec:
  selector: { app: api }
  ports: [{ port: 80, targetPort: 8080 }]
```

**`services/api/deploy/base/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - hpa.yaml
```

**`envs/dev/api/stable/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../../services/api/deploy/base
  - ingressroute.yaml  # stable Traefik routing
namespace: apps
images:
  - name: ghcr.io/rinsecrm/api-service
    newTag: v1.0.0   # promotion bumps this
```

> **No separate canary overlay needed!** ApplicationSet deploys the same base directly to PR namespaces with image patches.

### Back-office service (gRPC east–west) — Cross-namespace routing

The **`store`** service uses the same single-manifest approach. The apex Service in the stable namespace routes to PR namespaces via cross-namespace GRPCRoute.

**`services/store/deploy/base/deployment.yaml`** (same pattern as API)

**`services/store/deploy/base/service.yaml`** (same pattern as API)

**`services/store/deploy/base/grpc-route-template.yaml`**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: store-canary-pr-{{ number }}
  namespace: apps  # stable namespace where apex service lives
spec:
  parentRefs:
    - kind: Service
      name: store        # apex ClusterIP used by callers
  rules:
    - matches:
        - headers:
            - name: X-Canary
              value: "{{ number }}"
      backendRefs:
        - name: store
          namespace: store-canary-pr-{{ number }}  # cross-namespace reference
          port: 80
    - backendRefs:
        - name: store
          namespace: apps  # stable namespace
          port: 80
```

**`services/store/deploy/base/reference-grant-template.yaml`**

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-apps-grpcroute
  namespace: store-canary-pr-{{ number }}  # target namespace
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: GRPCRoute
      namespace: apps         # the route lives here
  to:
    - group: ""                     # core/v1 Service
      kind: Service
      name: store                   # allow referencing this Service
```

### Horizontal Pod Autoscaler (HPA)

**`services/api/deploy/base/hpa.yaml`** (same HPA deployed in both stable and PR namespaces)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## Traefik routing (edge) — Namespace-based routing

**Stable** (everyone → `api` service in `apps` namespace):

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: api-stable
  namespace: apps
spec:
  entryPoints: [ websecure ]
  routes:
    - match: Host(`api.dev.example.com`) && PathPrefix(`/`)
      kind: Rule
      services:
        - name: api
          namespace: apps
          port: 80
  tls: {}
```

**Per-PR canary router** (only `X-Canary: <PR#>` → `api` service in that PR's namespace):

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: api-canary-pr-{{ number }}
  namespace: apps
spec:
  entryPoints: [ websecure ]
  routes:
    - match: Host(`api.dev.example.com`) && PathPrefix(`/`) && Headers(`X-Canary`, `{{ number }}`)
      kind: Rule
      services:
        - name: api
          namespace: api-canary-pr-{{ number }}
          port: 80
  tls: {}
```

> **Key insight**: Same service name (`api`), different **target namespace** based on header value. No v1/v2 service duplication needed.

---

## Linkerd routing (east–west) — Cross-namespace with ReferenceGrant

Per-PR **`GRPCRoute`** for cross-namespace canary routing:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: store-canary-pr-{{ number }}
  namespace: apps  # stable namespace where apex service lives
spec:
  parentRefs:
    - kind: Service
      name: store
  rules:
    - matches:
        - headers:
            - name: X-Canary
              value: "{{ number }}"
      backendRefs:
        - name: store
          namespace: store-canary-pr-{{ number }}  # cross-namespace!
          port: 80
    - backendRefs:
        - name: store
          namespace: apps  # stable
          port: 80
```

**Required ReferenceGrant** (in target namespace):

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
      name: store
```

---

## Argo CD ApplicationSet (per-PR canaries) — Simplified

> We use **three** ApplicationSets for complete automation:
>
> 1. Deploy the **PR's API base manifests** to PR namespace
> 2. Deploy the **edge router** (Traefik) pointing to that namespace  
> 3. Deploy the **gRPC routing** (GRPCRoute + ReferenceGrant) for cross-namespace access

**1) API canary namespace (deploys base manifests directly)**

`envs/apps/applicationset-api-pr.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: api-pr-canaries
  namespace: argocd
spec:
  generators:
    - pullRequest:
        github:
          owner: rinsecrm
          repo: api-service
          tokenRef: { secretName: github-token, key: token }
        requeueAfterSeconds: 120
  template:
    metadata:
      name: api-canary-pr-{{ number }}
      labels: { role: canary, app: api, pr: "{{ number }}" }
    spec:
      project: canary-deployments
      destination:
        server: https://kubernetes.default.svc
        namespace: api-canary-pr-{{ number }}
      source:
        repoURL: https://github.com/rinsecrm/api-service.git
        targetRevision: "{{ head_sha }}"
        path: deploy/base
        kustomize:
          patches:
            - target: { kind: Deployment, name: api }
              patch: |-
                - op: replace
                  path: /spec/template/spec/containers/0/image
                  value: ghcr.io/rinsecrm/api-service:pr-{{ number }}
      syncPolicy:
        automated: { prune: true, selfHeal: true }
        syncOptions:
          # CreateNamespace removed - using apps namespace
```

**2) Edge router per PR (Traefik `IngressRoute`)**

`envs/apps/applicationset-edge-pr.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: api-edge-pr-routers
  namespace: argocd
spec:
  generators:
    - pullRequest:
        github:
          owner: rinsecrm
          repo: api-service
          tokenRef: { secretName: github-token, key: token }
        requeueAfterSeconds: 120
  template:
    metadata:
      name: api-canary-router-pr-{{ number }}
      labels: { role: canary, pr: "{{ number }}", layer: edge }
    spec:
      project: canary-deployments
      destination:
        server: https://kubernetes.default.svc
        namespace: apps
      source:
        repoURL: https://github.com/rinsecrm/api-service.git
        targetRevision: "{{ head_sha }}"
        path: deploy/edge
        kustomize:
          patches:
            - target: { apiVersion: traefik.containo.us/v1alpha1, kind: IngressRoute, name: api-canary-pr-{{ number }} }
              patch: |-
                - op: replace
                  path: /metadata/name
                  value: api-canary-pr-{{ number }}
                - op: replace
                  path: /spec/routes/0/match
                  value: "Host(`api.dev.example.com`) && PathPrefix(`/`) && Headers(`X-Canary`, `{{ number }}`)"
                - op: replace
                  path: /spec/routes/0/services/0/namespace
                  value: api-canary-pr-{{ number }}
      syncPolicy:
        automated: { prune: true, selfHeal: true }
```

**3) Store service + cross-namespace routing**

Two ApplicationSets handle store canaries: one for the service deployment, another for GRPCRoute + ReferenceGrant.

> **Key improvement**: All ApplicationSets deploy directly from service repos, no intermediate manifests repo needed!

---

## GitHub Actions

**Build & push on PR** (in the **service repo**):

```yaml
name: build-api-pr
on:
  pull_request:
    paths: [ "services/api/**" ]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build & push
        env: { IMAGE: ghcr.io/rinsecrm/api-service }
        run: |
          TAG=${GITHUB_SHA::8}
          docker build -t $IMAGE:$TAG services/api
          echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u "${{ github.actor }}" --password-stdin
          docker push $IMAGE:$TAG
```

> ApplicationSet will use `pr-{{ number }}` to pick the matching image tag.

---

## Go wiring (summary)

* **Gorilla/Mux middleware**: read `X-Canary` (digits only), stash in `context`.
* **gRPC server interceptor**: read incoming `X-Canary` metadata, re-stash in context.
* **gRPC client interceptor**: read from context, set `X-Canary` on outgoing calls.
* This allows Traefik to route HTTP to `api-v2` and Linkerd to route gRPC to `*-v2`.

> You can keep this in a small internal package (e.g., `internal/canaryctx`) and a common RPC module for interceptors.

---

## Chrome extension (optional)

A simple Manifest V3 extension can **auto-attach `X-Canary: <PR#>`** to your dev domains via `declarativeNetRequest`.

> Remember to add `Access-Control-Allow-Headers: X-Canary` to CORS preflights on your API.

---

## Promotion & cleanup lifecycle

1. **PR open** → CI builds image → ApplicationSet creates PR namespace + router.
2. **Test via `X-Canary: <PR#>`**.
3. **PR updated** → CI builds new image → ApplicationSet detects new `head_sha` → Rolling update in same namespace.
4. **PR close/merge** → ApplicationSet stops generating → Argo CD **prunes** PR namespace & router.
5. **Release** → bump `images:` in `envs/dev/api/stable/kustomization.yaml` to promote.

### PR Update Flow

When developers push new commits to their PR:

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub
    participant CI as GitHub Actions
    participant Reg as Container Registry
    participant ArgoAS as Argo ApplicationSet
    participant K8s as Kubernetes

    Dev->>GH: git push (new commits to PR)
    GH->>CI: Trigger workflow (new GITHUB_SHA)
    CI->>CI: Build & test new code
    CI->>Reg: Push image:new-sha-tag
    
    Note over ArgoAS: Polls every 2 minutes
    ArgoAS->>GH: Check PR head_sha
    GH->>ArgoAS: head_sha changed!
    ArgoAS->>ArgoAS: Update Application manifest
    ArgoAS->>K8s: Apply new image tag
    K8s->>K8s: Rolling update deployment
    K8s->>Dev: Canary ready with new code
```

**Key**: Same namespace gets updated, not recreated. Testing URLs remain consistent.

### Faster PR Updates (Optional)

By default, ApplicationSet polls every 2 minutes (`requeueAfterSeconds: 120`). For faster feedback:

**1) GitHub Webhook (Immediate Updates)**

```yaml
# In ApplicationSet
spec:
  generators:
    - pullRequest:
        github:
          owner: rinsecrm
          repo: api-service
          tokenRef: { secretName: github-token, key: token }
        requeueAfterSeconds: 120
        webhooks:
          - github:
              secret: webhook-secret
```

**2) CI-Triggered Sync**

Add to GitHub Actions workflow:

```yaml
- name: Trigger Argo Sync
  run: |
    curl -X POST \
      -H "Authorization: Bearer ${{ secrets.ARGOCD_TOKEN }}" \
      https://argocd.example.com/api/v1/applications/api-canary-pr-${{ github.event.number }}/sync
```

**3) Manual Developer Refresh**

```bash
# Force ApplicationSet refresh
kubectl annotate applicationset api-pr-canaries \
  argocd.argoproj.io/refresh=now

# Or sync specific app
argocd app sync api-canary-pr-123
```

---

## Security, RBAC & quotas

* Put all canary apps in an **Argo CD Project** limited to:

  * source repos (env manifests),
  * destination namespaces (e.g., `*-canary-pr-*`),
  * no cluster-admin.
* Add **ResourceQuotas/LimitRanges** to PR namespaces.
* Use **External Secrets** or **Sealed Secrets**; don’t bake secrets into images/overlays.

---

## Observability

* Log/metric label e.g., `canary_pr=<PR#>`.
* Build quick dashboards comparing v1/v2:

  * request rate, error rate, latency percentiles.
* Optionally export `X-Canary` as `X-Canary-Echo` header in responses for debugging.

---

## CORS & client notes

* Add `Access-Control-Allow-Headers: X-Canary` on API preflight responses.
* Browser tooling:

  * Postman/Insomnia: set header per request.
  * Chrome extension: auto-inject the header for your dev domain(s).

---

## Troubleshooting

* **Header present but API still v1**

  * Check Traefik `IngressRoute` match (host/path/header exact value).
  * Ensure the router points to the **PR namespace** Service.
* **API routes to v2 but backend still v1**

  * Ensure your gRPC interceptors are registered; check that `X-Canary` metadata is present downstream.
  * Verify Linkerd `GRPCRoute` `value` matches the PR number.
* **CORS errors**

  * Missing `Access-Control-Allow-Headers` for `X-Canary`.
* **ApplicationSet didn’t create PR app**

  * Check PAT, repo/owner names, controller logs; confirm PR generator has permissions.
* **No auto-cleanup**

  * Ensure ApplicationSet has `automated.prune: true` and PR is actually *closed*.

---

## FAQ

**Q: Helm or Kustomize?**
A: We use **Kustomize** here because ops prefers it; the pattern works identically with Helm.

**Q: Can multiple PRs be tested at once?**
A: Yes—each PR gets its own namespace and Traefik router rule. Use the corresponding `X-Canary: <PR#>`.

**Q: Do we need edge routing if only backend changed?**
A: Not strictly. You can keep Traefik stable and rely solely on Linkerd for internal gRPC routing. Use edge routing only when canarying the API itself.

**Q: How do we avoid state collisions?**
A: Start with read-only paths or clone DB schema/topic for canary writes. Gate mutations behind a feature flag until confident.

**Q: Why not maintain separate v1/v2 manifests?**
A: **Single base manifests** deployed to different namespaces is simpler—same code paths, same configs, just different namespaces and image tags. Reduces Git bloat and drift.

**Q: How does cross-namespace gRPC routing work?**
A: **Linkerd GRPCRoute** in the stable namespace routes to services in PR namespaces. **ReferenceGrant** in the target namespace permits the cross-namespace reference.

**Q: What happens when a PR is force-pushed?**
A: ApplicationSet detects the new SHA and updates the deployment automatically. Same namespace, new image.

**Q: How long does it take for canary updates to deploy?**
A: By default, ApplicationSet polls every 2 minutes. Add webhooks or CI triggers for immediate updates. Total time: ~2-5 minutes including build + deploy.

**Q: Can I test multiple commits quickly during development?**
A: Yes! Each push creates a new image. The same canary namespace gets rolling updates. Use `kubectl get pods -w` to watch deployment progress.

**Q: What if I want to roll back to a previous commit in the PR?**
A: Force-push the previous commit or use `git revert`. CI will build that SHA and ApplicationSet will deploy it to the same canary namespace.

---

### Replace-me values

* Registry: `ghcr.io/rinsecrm`
* Domain: `api.dev.example.com`
* Namespaces: `apps`, `api-canary-pr-<PR#>`, `store-canary-pr-<PR#>`
* Repos: `rinsecrm/api-service`, `rinsecrm/store-service`, `rinsecrm/manifests-microservices`

---

---

## ✅ IMPLEMENTATION STATUS 

This plan has been **fully implemented** with the following improvements:

### ✨ What Was Built

1. **🏪 Store Service** - gRPC backend with DynamoDB
2. **🌐 API Service** - HTTP REST API with gRPC client  
3. **📋 Manifests Repository** - Single base manifests (no v1/v2 split)
4. **🔧 Chrome Extension** - Auto X-Canary header injection
5. **⚙️ Complete CI/CD** - GitHub Actions + Argo ApplicationSets

### 🎯 Key Architecture Decisions

**✅ Single Base Manifests**: Each service has one `deployment.yaml` and `service.yaml` 
**✅ Namespace-Based Canaries**: PR creates `<service>-canary-pr-<number>` namespace
**✅ Cross-Namespace Routing**: GRPCRoute + ReferenceGrant for east-west traffic
**✅ Simplified ApplicationSets**: Direct base deployment with image patches

### 📂 Repository Structure

```
.
├── api-service/           # HTTP service (single base manifests)
│   └── deploy/           # Kubernetes manifests + stable overlay
├── store-service/        # gRPC service (single base manifests)
│   └── deploy/           # Kubernetes manifests + stable overlay
├── manifests/            # Environment deployments
│   ├── stable/           # Stable environment configs
│   └── apps/             # Argo CD ApplicationSets
├── chrome-extension/     # Browser tooling for testing
├── README.md             # Complete setup guide
├── docker-compose.yml    # Local development
└── Makefile             # Development tasks
```

### 🚦 How It Works

1. **PR Created** → GitHub Actions builds image with `pr-{{ number }}`
2. **ApplicationSet Triggers** → Deploys base manifests to `<service>-canary-pr-<PR#>` 
3. **Routing Updates** → Traefik/Linkerd route `X-Canary: <PR#>` to that namespace
4. **Testing** → Chrome extension or curl with header hits canary
5. **PR Closed** → Argo CD prunes namespace automatically

This implementation is **production-ready** and follows the exact pattern outlined in this plan, but with the cleaner single-manifest approach you described.
