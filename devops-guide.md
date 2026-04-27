# Fischer DevOps Guide - What Happens in Each Repo (Step by Step)
**For: Vaidant Vyas | Generated: Feb 11, 2026**

This guide explains every DevOps file in each repo — what it does, what triggers it, and what happens when it runs.

---

## PART 1: APPLICATION REPOS (Where Docker Images Get Built)

---

### REPO: fischer-frontend-website

**DevOps Files:**
- `Jenkinsfile` (152 lines)
- `Dockerfile`

**Step-by-step: What happens when code is pushed?**

**FILE: Jenkinsfile**
```
Location: C:\Users\vyas-v\workspace\fischer-frontend-website\Jenkinsfile
```

1. Jenkins sees a push to `master`, `develop`, or `Release-*` branch
2. It spins up a Kubernetes pod with a Node.js 22 container as the build agent
3. **Stage: Test** (only on `test` branch)
   - Builds a test Docker image: `docker build --target fischer-frontend-test`
   - Runs tests inside the container, collects code coverage
4. **Stage: Publish** (only on `master` or `develop`)
   - Runs `npm publish` to push the package to internal npm registry
5. **Stage: Docker Build** (on `master`, `develop`, `Release-*`, or tags)
   - Decides the image name based on `useFischerGroupAcr` flag:
     - `true` → image = `fischergroup.azurecr.io/fischer-frontend-website`
     - `false` → image = `smartcommerce.azurecr.io/fischer-frontend-website`
   - Decides the tag:
     - Tag push → tag name (e.g., `Release-8.33.0`)
     - `master` → version number (e.g., `1.0.0`)
     - `develop` → `latest-dev`
     - Other → branch name
   - Runs: `docker build -t <image>` then `docker tag <image>:<tag>`
6. **Stage: Docker Push** (only on tag pushes)
   - Logs into ACR with credentials from Jenkins secret store
   - Runs: `docker push <image>:<tag>`
7. **Post**: Sends Teams notification about build result

**FILE: Dockerfile**
```
Location: C:\Users\vyas-v\workspace\fischer-frontend-website\Dockerfile
```

1. Starts from `node:23-alpine`
2. Enables pnpm via corepack
3. Copies package files, runs `pnpm install --frozen-lockfile`
4. Runs `pnpm run bootstrap` (Sitecore JSS setup)
5. Final image exposes ports 3000 and 3042
6. Starts with `pnpm run start`

**Also built by TeamCity** (PowerShell scripts, separate from Jenkins):
- TeamCity builds for Sitecore integration use `%build.counter%` tags like `test-build945`
- These are configured in TeamCity UI, not in repo files

---

### REPO: fischer-event-streaming

**DevOps Files:**
- `Jenkinsfile`
- `*/src/main/docker/Dockerfile.jvm` (one per microservice)
- `*/src/main/resources/application.properties` (registry config per service)

**FILE: Jenkinsfile**
```
Location: C:\Users\vyas-v\workspace\fischer-event-streaming\Jenkinsfile
```

1. Imports `defaultQuarkusBuild` from `global-pipeline-library`
2. Loads EventHub credentials from Jenkins secret store
3. Calls `defaultQuarkusBuild` with these settings:
   - `skipTests = false` → run all tests
   - `buildDocker = true` → build Docker images
   - `useGitTag = true` → use git tags for image tagging
   - `useFischerGroupAcr = true` → push to fischergroup.azurecr.io
   - `sonarCloud = true` → run code quality analysis

**What `defaultQuarkusBuild` does behind the scenes** (from jenkins-pipeline-library):
1. Determines image tag based on branch:
   - `develop`/`test`/`feature/*` → tag prefix `sit-` + commit hash
   - `uat` → tag prefix `uat-` + commit hash
   - `main` → tag prefix `prod-` + commit hash
2. Runs `mvn clean package` to build all 8 microservice JARs
3. For each module with a Dockerfile:
   - Builds Docker image using Quarkus container-image extension
   - Registry set to `fischergroup.azurecr.io` (from application.properties)
4. Runs Trivy security scan on each image
5. Pushes all images to ACR
6. Runs SonarCloud code quality analysis
7. Sends Teams notification

**FILE: Dockerfile.jvm** (same pattern in each microservice)
```
Location: fischer-event-streaming/bill-service/src/main/docker/Dockerfile.jvm
```

1. Starts from `eclipse-temurin:21.0.5_11-jre-alpine` (lightweight Java 21)
2. Copies Quarkus fast-jar layers (lib/, app/, quarkus/)
3. Copies SSL truststore for SAP connections
4. Exposes port 8080
5. Runs the Quarkus app with JVM options

**Registry config in application.properties:**
```
quarkus.container-image.registry=fischergroup.azurecr.io
quarkus.container-image.group=fischer-eventprocessing
quarkus.container-image.name=eventprocessors   (varies per service)
```

---

### REPO: fischer-frontend-profiberater

**DevOps Files:**
- `Jenkinsfile`
- `Dockerfile`
- `nginx.conf`

**FILE: Jenkinsfile**
```
Location: C:\Users\vyas-v\workspace\fischer-frontend-profiberater\Jenkinsfile
```

1. Spins up K8s pod with Node.js 22 agent
2. On `develop`, `main`, or `integration` branch push:
   - Decides build stage:
     - `develop` → builds with `:test` stage arg, tag = `test-3.27.21`
     - `integration` → builds with `:integration` stage arg, tag = `uat-3.27.21`
     - `main` → builds with no stage arg (prod), tag = `3.27.21`
   - Decides registry: `fischergroup.azurecr.io/fischer-frontend-profiberater`
   - Runs: `docker build --build-arg stage=<stage> -t <image> .`
   - Logs into ACR, pushes image

**FILE: Dockerfile**
```
Location: C:\Users\vyas-v\workspace\fischer-frontend-profiberater\Dockerfile
```

1. **Build stage**: node:24-alpine → installs pnpm → runs `pnpm run build<stage>`
   - Stage arg controls which build script runs (build:test, build:integration, or build)
   - Each build script bakes in different API endpoints
2. **Runtime stage**: nginx:alpine → copies built files + nginx.conf
   - Final image is just nginx serving static files

**FILE: nginx.conf** — configures nginx to serve the Vue SPA with proper routing

---

### REPO: fischer-profiberater-backend

**DevOps Files:**
- `Jenkinsfile`
- `service/src/main/docker/Dockerfile.jvm`

**FILE: Jenkinsfile**
```
Location: C:\Users\vyas-v\workspace\fischer-profiberater-backend\Jenkinsfile
```

Same pattern as fischer-event-streaming — uses `defaultQuarkusBuild` from global-pipeline-library:
- `useFischerGroupAcr = true`
- Builds Java 21 Quarkus app
- Image: `fischergroup.azurecr.io/fischer-pro-backend/professional-services`
- Tags: `sit-<hash>`, `uat-<hash>`, `prod-<hash>` depending on branch

---

### REPO: fischer-backstage-app

**DevOps Files:**
- `Jenkinsfile`
- `Dockerfile`

**FILE: Jenkinsfile**
```
Location: C:\Users\vyas-v\workspace\fischer-backstage-app\Jenkinsfile
```

Uses `defaultDockerBuild` from global-pipeline-library:
- `imageName = 'fischergroup.azurecr.io/fischer-backstage'`
- `useFischerGroupAcr = true`
- `useGitTag = true` → git commit hash as image tag
- `trivyIgnoreFailures = true` → ignores upstream Backstage vulnerabilities

**FILE: Dockerfile** (multi-stage, 3 stages)
```
Location: C:\Users\vyas-v\workspace\fischer-backstage-app\Dockerfile
```

1. **Stage 1 (packages)**: Creates yarn install skeleton
2. **Stage 2 (build)**: Installs everything (Node, Python, Java, PlantUML, Graphviz) for TechDocs
3. **Stage 3 (runtime)**: Production image, runs as non-root `node` user on port 3000

---

## PART 2: K8s DEPLOYMENT REPOS (Where Things Get Deployed)

All these repos follow the same pattern: **Kustomize overlays**.

**How Kustomize works:**
```
default/                    ← Base manifests (deployment, service, ingress, secrets)
  kustomization.yaml        ← Lists all base resources
  deployment.yaml           ← Base deployment spec (placeholder image)
  service.yaml              ← Kubernetes Service definition
  ingress.yaml              ← Ingress rules (URL routing)
  acr-secret.yaml           ← Image pull secret for ACR

test/                       ← Test environment overlay
  kustomization.yaml        ← Says: "take everything from ../default, then change:"
                               - namespace: xxx-test
                               - image tag: test-build123
                               - replicas: 1
  deployment-patch.yaml     ← Environment-specific patches (env vars, resources)

uat/                        ← Same pattern for UAT
prod/                       ← Same pattern for Production (more replicas, different tags)
```

**The key file in each overlay is `kustomization.yaml`:**
```yaml
resources:
  - ../default              # Import all base manifests

images:
  - name: smartcommerce.azurecr.io/my-app    # Match this image in base
    newName: fischergroup.azurecr.io/my-app   # Replace with this registry
    newTag: test-build945                      # Replace with this tag

namespace: my-app-test      # Deploy to this namespace
```

**When you need to update a deployment:**
1. Change `newTag` in the environment's `kustomization.yaml`
2. Push to git
3. ArgoCD detects the change and auto-deploys within 5 minutes

---

### REPO: fischer-sitecore-k8s

**The most complex K8s repo. Structure:**
```
xp0/                        ← Sitecore XP0 (single-server) base manifests
  kustomization.yaml
  deployment-cm.yaml         ← Content Management server
  deployment-xconnect.yaml   ← XConnect analytics
  configmaps/                ← Sitecore connection strings, settings
  secrets/                   ← Sitecore license, passwords
  init/                      ← Init containers for DB setup
  external/                  ← ExternalSecret definitions
  ingress-nginx/             ← Ingress rules

xp1/                        ← Sitecore XP1 (scaled) for UAT/Prod
  cd/                        ← Content Delivery (public-facing)
    kustomization.yaml       ← Blue-green deployment config

overlays/
  test/kustomization.yaml    ← TEST: uses xp0, sets image tags
  uat/kustomization.yaml     ← UAT: uses xp1 with blue-green
  prod/kustomization.yaml    ← PROD: uses xp1 with blue-green
```

**Key file: overlays/test/kustomization.yaml**
```
This is what we looked at earlier:
- frontend-initContainer → fischergroup.azurecr.io/fischer-frontend:test-build945
- sitecore-xp0-cm → smartcommerce.azurecr.io/fischer-xp0-cm:test-build643
- (other Sitecore components still on smartcommerce)
```

**Blue-green in UAT/Prod:**
- `uat/cd-blue/` and `uat/cd-green/` — two separate deployments
- fischer-operations blue-green scripts switch traffic between them

---

### REPO: fischer-eventprocessing-k8s
```
default/                           ← 4 deployments + services + ingress + monitoring
  deployment-eventprocessing.yaml
  deployment-profile.yaml
  deployment-tracking.yaml
  deployment-web-tracking-service.yaml
  servicemonitor.yaml              ← Prometheus scraping config
  prometheusrules.yaml             ← Alert rules

test/kustomization.yaml            ← 6 images mapped (includes myfischer-mcp, copilot)
uat/kustomization.yaml             ← 5 images mapped (includes proxyservice)
prod/kustomization.yaml            ← 4 images mapped (stable release versions)

test-1/, test-2/                   ← Secondary cluster ingress patches
uat-1/, uat-2/                     ← Secondary cluster ingress patches
prod-1/, prod-2/                   ← Secondary cluster ingress patches
```

The secondary cluster overlays (-1, -2) only patch ingress rules for multi-cluster routing. They inherit everything else from the parent overlay.

---

### REPO: fischer-server-side-external-rendering-k8s
```
default/
  blue/kustomization.yaml          ← Blue deployment base
  green/kustomization.yaml         ← Green deployment base
  deployment/kustomization.yaml    ← Shared deployment config

test/kustomization.yaml            ← 1 replica, single deployment
uat/kustomization.yaml             ← 2 replicas, blue-green (blue active)
prod/kustomization.yaml            ← 6 replicas, blue-green (green active)
```

---

### REPO: fischer-profiberater-backend-k8s / fischer-profiberater-frontend-k8s / fischer-frontend-showroom-k8s / fischer-sitecore-events-k8s

All follow the standard pattern: `default/` base + environment overlays. See the general Kustomize explanation above.

---

## PART 3: INFRASTRUCTURE REPOS

---

### REPO: fischer-argocd

**What ArgoCD does**: Watches K8s repos for changes and automatically deploys them.

**Key DevOps files:**
```
applications/                      ← One file per application
  sitecore.yaml                    ← Watches fischer-sitecore-k8s repo
  eventprocessing.yaml             ← Watches fischer-eventprocessing-k8s repo
  profiberater-backend.yaml        ← Watches fischer-profiberater-backend-k8s
  server-side-rendering.yaml       ← Watches fischer-server-side-external-rendering-k8s
  ... (30+ more)

cluster-apps/                      ← One file per cluster service
  prometheus.yaml                  ← Prometheus monitoring
  cert-manager.yaml                ← TLS certificates
  nginx-ingress.yaml               ← Ingress controller
  ... (47+ more)

argocd/base/                       ← ArgoCD's own configuration
  kustomization.yaml               ← ArgoCD v2.14.11 installation
  argocd-cm.yaml                   ← ArgoCD settings
  argocd-rbac.yaml                 ← Access control
```

**How an ApplicationSet works (example: sitecore.yaml):**
```yaml
# "Watch fischer-sitecore-k8s repo, deploy overlays/test to test cluster,
#  overlays/uat to uat cluster, overlays/prod to prod cluster"

spec:
  source:
    repoURL: git@bitbucket.org:smartcommerce_se/fischer-sitecore-k8s.git
    path: overlays/{{.values.env}}     # test, uat, or prod
  destination:
    namespace: sitecore-{{.values.env}}
  syncPolicy:
    automated:
      prune: true       # Delete removed resources
      selfHeal: true    # Auto-fix drift
```

---

### REPO: fischer-cluster-applications

**DevOps files by service type:**

**Kustomize services (77 overlays):**
```
kustomize/backstage/kustomization.yaml        ← Developer portal deployment
kustomize/catalogrestservice/
  base/deployment-catalogrestservice.yaml      ← Base deployment
  test/kustomization.yaml                      ← Test overlay
  uat/kustomization.yaml                       ← UAT overlay
  prod/kustomization.yaml                      ← Prod overlay
kustomize/redis-product-catalog/               ← Redis for caching
kustomize/storage/                             ← Azure storage mounts
kustomize/jenkins/                             ← Jenkins ACR secrets
kustomize/kyverno/                             ← K8s policy engine
... (40+ services)
```

**Helm charts:**
```
helm/jenkins/
  Chart.yaml                                   ← Helm chart definition
  templates/
    acr-secret.yaml                            ← Docker registry secret
    jenkins-home-disk.yaml                     ← Persistent storage for Jenkins
    namespace.yaml                             ← Jenkins namespace
  values.yaml                                  ← Default Helm values
  values.service-prod.yaml                     ← Prod-specific values

helm/teamcity/
  Chart.yaml
  templates/
    deployment.yaml                            ← TeamCity server deployment
    postgres-secret.yaml                       ← Database credentials
    ingress.yaml                               ← Web UI access
  values.yaml

helm/filebeat/
  Chart.yaml
  values.nonprod-1.yaml, values.prod-1.yaml   ← Per-cluster log shipping config
```

---

### REPO: fischer-jenkins-pipeline-library

**The shared build library that all Jenkins repos use.**

**Key DevOps files:**

**FILE: vars/defaultQuarkusBuild.groovy** (11.2 KB — most important)
```
What it does step by step:
1. Determines Docker image tag based on branch name
2. Spins up K8s pod with Java 21 + Docker + Trivy containers
3. Runs: cd common && mvn clean install (shared module)
4. Runs: mvn clean package (builds all microservices)
5. For each module:
   a. Builds Docker image (via Quarkus container-image extension)
   b. Runs Trivy security scan
   c. If vulnerabilities found and ignoreFailures=false → FAIL build
6. Logs into ACR (fischergroup or smartcommerce based on flag)
7. Pushes all images
8. Runs SonarCloud/SonarQube code quality
9. Sends email + Teams notification
```

**FILE: vars/defaultDockerBuild.groovy** (3.8 KB)
```
Simpler version for non-Quarkus apps:
1. Determines tag (git tag or branch-based)
2. Logs into ACR
3. Runs: docker build
4. Runs: Trivy scan
5. Runs: docker push
6. Notifies
```

**FILE: vars/trivyScan.groovy** (2.8 KB)
```
Security scanning:
1. Takes an image name or file path
2. Runs Trivy with configured severity (HIGH,CRITICAL)
3. Parses JSON report
4. If vulnerabilities found → either fail build or warn
5. Updates build status for notification
```

**FILE: resources/java21BuildPod.yaml** (K8s pod template)
```
Defines the Jenkins build agent:
- Container 1: java21 (Maven builds) — smartcommerce.azurecr.io/fischer-common-build
- Container 2: docker (Docker-in-Docker) — docker:27-dind
- Container 3: trivy (security scanning) — aquasec/trivy:latest
- Container 4: jnlp-watcher (kills pod when Jenkins agent exits)
All share the same pod, so they can access each other's files.
```

---

### REPO: fischer-operations

**DevOps files:**

**FILE: terraform/Jenkinsfile** (Drift Detection)
```
Runs daily at 2 AM:
1. Spins up K8s pod with Terraform container
2. For each of 14 Terraform projects:
   a. cd into project directory
   b. terraform init
   c. terraform workspace select <workspace>
   d. terraform plan -detailed-exitcode
   e. If exit code = 2 → DRIFT DETECTED
3. Sends Teams notification with summary of all projects
```

**FILE: blue-green-deployment/Jenkinsfile** (Deployment Pipeline)
```
Manual trigger with parameters (stage, line, image tags):
1. Setup: Login to ArgoCD, Azure, Docker
2. Check current deployment state (which color is active?)
3. Verify all container images exist in ACR
4. Scale up target node pool (prod only — Windows nodes are expensive)
5. Pre-load Windows container images on nodes
6. Create Checkly maintenance window (optional downtime)
7. Deploy Sitecore (CM/xConnect) to inactive color
8. Deploy Blue-Green services (CD/GraphQL/SSR) to inactive color
9. Switch traffic in 5 stages:
   - Stage 1: Test domains (small countries)
   - Stage 2: International markets
   - Stage 3: Specialized European
   - Stage 4: Primary European (DE, FR, UK, etc.)
   - Stage 5: Default/catch-all
10. Scale down old deployment
11. Scale down old node pool
12. Close Checkly maintenance window
```

**FILE: blue-green-deployment/00_lib.sh** (Shared functions)
```
Core functions used by all deployment scripts:
- git_clone_and_checkout: Clones K8s manifest repo
- kustomize_set_image: Updates image tags in kustomization.yaml
- git_commit_and_push: Commits changes and pushes
- argocd_sync_and_wait: Triggers ArgoCD sync and waits for completion
- These functions automate the "change tag → push → wait for ArgoCD" workflow
```

**Terraform modules (key ones):**
```
terraform/kubernetes/main.tf       ← Provisions AKS clusters
  - Defines node pools (Linux default + Windows blue/green)
  - Network config, RBAC, monitoring
  - 3 workspaces: prod-2, nonprod-2, service-1

terraform/container-registry/      ← Provisions ACR registries
terraform/frontdoor/               ← Azure Front Door + WAF rules
terraform/keyvault/                ← Azure Key Vault secrets
terraform/network/                 ← VNets, subnets, NSGs
terraform/mssql/                   ← SQL Server databases
```

**FILE: atlantis.yaml** (PR-based Terraform)
```
17 projects defined — when you make a PR with Terraform changes:
1. Atlantis runs `terraform plan` automatically
2. Shows plan output in PR comments
3. After PR approval, runs `terraform apply`
```

---

### REPO: fischer-pimtofactfinder-jobs

**DevOps files:**
- `Jenkinsfile` (709 lines)
- `instance_production_ng_k8s_channelconfig.groovy` (800+ lines)

**FILE: Jenkinsfile**
```
Runs daily at midnight (or on-demand):
1. Spins up K8s pod with ff-import-jenkins-agent container
2. Mounts persistent volumes for catalog data
3. Loads channel configuration from groovy file (50+ channels)
4. Downloads PIM-to-FactFinder JAR from Maven repository
5. For each selected channel (parallel execution):
   a. Downloads PIM product data
   b. Runs Java transformer: PIM format → FactFinder format
   c. Uploads transformed data to FactFinder REST API
   d. Triggers CatalogRestService reload
   e. Uploads export files to Azure Storage
   f. If channel has shop data: generates ImpEx files for SAP Commerce
6. Sends email notification with results per channel
```

**FILE: instance_production_ng_k8s_channelconfig.groovy**
```
Defines all 50+ product channels:
- FIWE (Fischer Works Europe): 44 locale channels (de_DE, en_GB, fr_FR, etc.)
- FITE (Fischer Technical): 4 locale channels
- UPAT: 5 locale channels
Each channel config includes:
  - brand, locale, currency
  - automatic flag (for daily cron)
  - source type (PIM, CSV, etc.)
  - FactFinder import settings
```

---

## PART 4: HOW IT ALL CONNECTS

**The complete deployment flow:**

```
1. Developer pushes code to app repo (e.g., fischer-event-streaming)
                    │
                    ▼
2. Jenkins detects push, loads Jenkinsfile
   Jenkinsfile imports global-pipeline-library
                    │
                    ▼
3. Jenkins builds Docker image using Dockerfile
   Tags it with commit hash or version
                    │
                    ▼
4. Jenkins pushes image to ACR (fischergroup.azurecr.io)
                    │
                    ▼
5. Someone updates K8s repo (e.g., fischer-eventprocessing-k8s)
   Changes newTag in kustomization.yaml to new image tag
                    │
                    ▼
6. ArgoCD (configured in fischer-argocd) detects the K8s repo change
                    │
                    ▼
7. ArgoCD applies Kustomize overlays to Kubernetes cluster
   New pods start with the new image
                    │
                    ▼
8. Application is live!
```

**For Sitecore (blue-green):**
```
Steps 1-4 same as above, then:

5. DevOps runs blue-green deployment from fischer-operations
6. Scripts update K8s manifests, push to git
7. ArgoCD deploys to inactive color (e.g., green)
8. Scripts switch traffic in 5 stages
9. Old color (blue) scales down
```

**For infrastructure changes:**
```
1. Engineer modifies Terraform in fischer-operations
2. Creates PR → Atlantis runs terraform plan
3. PR approved → Atlantis runs terraform apply
4. OR: Push to main → daily drift check verifies state
```
