# Fischer DevOps & Kubernetes Guide — Real Setup + Interview Questions
# For: Vaidant Vyas | April 2026

---

## PART 1 — HOW VARIABLES ARE USED IN KUBERNETES PODS

In Kubernetes, there are 5 ways to inject configuration into a pod. All 5 are used in Fischer.

---

### Method 1 — Hardcoded env var (plain value)

Used when the value is the same for all pods in that environment.

```yaml
# From xp1/cm.yaml
env:
  - name: SITECORE_ROLE
    value: "cm"
  - name: quarkus_profile
    value: prod
```

The value is written directly. No secret, no ConfigMap.

---

### Method 2 — From a Secret (single key)

Used when the value is sensitive — passwords, tokens, keys.

```yaml
# From xp1/cm.yaml
env:
  - name: SITECORE_DECRYPTION_KEY
    valueFrom:
      secretKeyRef:
        name: sitecore-encryptionkeys    <- K8s Secret name
        key: sitecore_decryption_key     <- key inside the Secret

  - name: ES_USERNAME
    valueFrom:
      secretKeyRef:
        name: elastic-search-secret
        key: elastic_username
```

Kubernetes reads that specific key from the Secret and sets it as an env var inside the container. The application sees it as a normal environment variable.

---

### Method 3 — From a ConfigMap (single key)

Used when the value is not sensitive but comes from a shared config.

```yaml
# From xp1/cm.yaml
env:
  - name: EXTERNAL_RENDERING_ENGINE_URL
    valueFrom:
      configMapKeyRef:
        name: ext-ssr-configmap          <- ConfigMap name
        key: external-renderer-url       <- key inside the ConfigMap
```

---

### Method 4 — envFrom secretRef (ALL keys from a Secret as env vars)

Used when a Secret has many keys and you want all of them as env vars without listing each one.

```yaml
# From catalogrestservice deployment
envFrom:
  - secretRef:
      name: crs-secrets    <- every key in this Secret becomes an env var
```

If crs-secrets has keys ldap_access_pass and ldap_access_user, both become env vars automatically.

---

### Method 5 — envFrom configMapRef (ALL keys from a ConfigMap as env vars)

Same as above but from a ConfigMap.

```yaml
# From catalogrestservice deployment
envFrom:
  - configMapRef:
      name: config    <- every key in this ConfigMap becomes an env var
```

The configmap.yaml has keys like lucene_path, ldap_host, redis_host — all become env vars inside the container.

---

### Summary — When to use which method

| Method | Use When |
|---|---|
| Plain value | Same value always, not sensitive |
| secretKeyRef | Sensitive, need one specific key |
| configMapKeyRef | Not sensitive, need one specific key |
| envFrom secretRef | Need ALL keys from a secret as env vars |
| envFrom configMapRef | Need ALL keys from a ConfigMap as env vars |

---

## PART 2 — HOW CONFIGMAPS ARE CREATED

There are two ways to create ConfigMaps in Fischer.

---

### Way 1 — Plain ConfigMap YAML file

Written manually. Values are hardcoded in the file.

```yaml
# From kustomize/catalogrestservice/base/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config
data:
  lucene_path: /data/catalogrestservices
  ldap_host: openldap
  ldap_port: "1389"
  redis_host: redis
  redis_port: "6379"
  quarkus_http_proxy_proxy_address_forwarding: "true"
  retailer_backoffice_cookie_key: "x#i3GmQn1I304Hgz"
```

This is referenced in kustomization.yaml under resources and applied directly.
Inside the pod it is consumed via envFrom configMapRef.

---

### Way 2 — ConfigMapGenerator in kustomization.yaml

Used when the ConfigMap content comes from files on disk. Kustomize reads the files and bakes their content into a ConfigMap automatically.

```yaml
# From xp1/configmaps/kustomization.yaml
configMapGenerator:
- name: sitecore-hostnames
  files:
  - cd-hostname          <- reads content of this file
  - cm-hostname          <- reads content of this file
  - id-hostname          <- reads content of this file

- name: filebeat-config
  files:
  - filebeat.yml         <- reads the entire filebeat config file
```

Kustomize creates a ConfigMap where each file becomes one key.
The key name = the filename. The value = the file contents.

---

### Why ConfigMapGenerator is used for Sitecore XML config patches

Sitecore reads configuration from XML files on disk (not env vars).
The XML file needs to be mounted inside the Windows container at a specific path.

```yaml
# From overlays/uat/cd-blue/kustomization.yaml
configMapGenerator:
  - name: config-patches-configmap-blue
    files:
      - config-patches/k8s.SitecoreJSS.KubernetesRenderer.config
      - config-patches/k8s.Redis.ConnectionString.config
```

The file config-patches/k8s.Redis.ConnectionString.config contains:
```xml
<configuration>
  <sitecore>
    <settings>
      <setting name="Redis.ConnectionString">
        <patch:attribute name="value">redis-blue.redis-product-catalog.svc.cluster.local:6379</patch:attribute>
      </setting>
    </settings>
  </sitecore>
</configuration>
```

This XML file becomes a ConfigMap. The ConfigMap is then mounted as a volume into the CD pod at:
C:\inetpub\wwwroot\App_Config\Include\z_kubernetes_cm.patches

Sitecore reads all XML files from that folder on startup — this is how color-specific Redis connection strings work.

---

### disableNameSuffixHash — important flag

By default Kustomize adds a hash to ConfigMap names (e.g., sitecore-hostnames-abc123).
This causes problems when other resources reference the ConfigMap by name.

To disable:
```yaml
generatorOptions:
  disableNameSuffixHash: true
```

Now the ConfigMap name stays exactly as defined — no hash suffix.

---

## PART 3 — HOW SECRETS ARE INJECTED INTO PODS

Fischer uses ExternalSecret operator — secrets are NOT stored in git. They live in Azure Key Vault.

---

### The Flow

```
Azure Key Vault
  (stores actual secret values)
        |
        v
ExternalSecret (K8s resource defined in git)
  (tells operator: read this key from Key Vault, create a K8s Secret)
        |
        v
K8s Secret
  (created automatically by External Secrets Operator)
        |
        v
Pod
  (mounts Secret as volume OR injects as env var)
```

---

### ExternalSecret — single key example

```yaml
# From kustomize/catalogrestservice/base/secrets-catalogrestservice.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: crs-secrets
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: external-secret-store       <- points to the Key Vault connection
  target:
    name: crs-secrets                 <- name of K8s Secret to create
    creationPolicy: "Owner"
  data:
  - secretKey: ldap_access_pass       <- key name in K8s Secret
    remoteRef:
      key: kxs--crs-prod--openldap--secrets   <- Key Vault secret name
      property: LDAP_ADMIN_PASSWORD           <- property inside that KV secret
```

The External Secrets Operator reads LDAP_ADMIN_PASSWORD from Key Vault secret kxs--crs-prod--openldap--secrets and creates a K8s Secret named crs-secrets with key ldap_access_pass.

---

### ExternalSecret — extract all keys (dataFrom)

Used when a Key Vault secret contains multiple keys (JSON format).
Extracts all keys and creates one K8s Secret key per JSON property.

```yaml
# From overlays/uat/secrets/sitecore-config-patches.yaml
spec:
  target:
    name: sitecore-config-patches
  dataFrom:
  - extract:
      key: kxs--sitecore-uat--config-patches   <- Key Vault secret (JSON)
```

If the KV secret contains JSON like:
{ "k8s.Redis.ConnectionString.config": "<xml>...</xml>", "other.config": "..." }

Then the K8s Secret gets one key per JSON property.

---

### How the Secret reaches the pod — two ways

**Way 1: Mounted as a volume (files on disk)**

```yaml
# From xp1/cm.yaml
volumes:
  - name: config-patches
    secret:
      secretName: sitecore-config-patches   <- K8s Secret

volumeMounts:
  - mountPath: C:\inetpub\wwwroot\App_Config\Include\z_kubernetes.patches
    name: config-patches
    readOnly: true
```

Each key in the Secret becomes a file in that folder.
Filename = key name. File content = secret value.
Sitecore reads the XML files from that folder — this is how connection strings reach Sitecore.

**Way 2: Injected as env var**

```yaml
env:
  - name: ES_PASSWORD
    valueFrom:
      secretKeyRef:
        name: elastic-search-secret
        key: elastic_password
```

The container sees ES_PASSWORD as a normal environment variable.

---

## PART 4 — HOW KUSTOMIZE WORKS

Kustomize is a tool that customizes Kubernetes YAML without templating. It uses a base + overlay pattern.

---

### Structure in Fischer

```
base/                         <- shared manifests (same for all environments)
  kustomization.yaml
  deployment.yaml             <- image: myapp:dummy (placeholder)
  service.yaml
  configmap.yaml
  secrets-*.yaml (ExternalSecret)

test/                         <- test environment overlay
  kustomization.yaml          <- says: take base, change these things
  deployment-patch.yaml       <- test-specific changes

uat/                          <- uat environment overlay
  kustomization.yaml

prod/                         <- prod environment overlay
  kustomization.yaml
```

---

### kustomization.yaml — the control file

Everything Kustomize does is controlled by kustomization.yaml.

```yaml
# prod/kustomization.yaml for catalogrestservice
namespace: catalogrestservice-prod      <- set namespace for all resources

resources:
  - ../base                             <- import everything from base
  - ingress.yaml                        <- add prod-specific resources
  - azurefile-catalog.yaml

patchesStrategicMerge:                  <- override parts of base manifests
  - deployment-catalogrestservice.yaml  <- prod has more memory, different PVC

images:                                 <- replace image tags
  - name: fischergroup.azurecr.io/fischer-searchbackend/fischer-catalogrestservices
    newTag: "prod-0f034443d9e5a30777cdd372b6b5018aa7c8024f"
```

---

### images — how image tags are updated

This is how deployments are triggered. When a new image is built:
1. Update newTag in kustomization.yaml
2. Push to git
3. ArgoCD detects the change
4. ArgoCD applies the updated kustomization
5. Kubernetes rolls out new pods with new image

---

### patches — two types used in Fischer

**Type 1: patchesStrategicMerge**
Merges a partial YAML on top of the base manifest. Good for overriding resource limits, replicas, PVC names.

```yaml
# prod deployment patch — only changes what is different
spec:
  template:
    spec:
      containers:
      - name: catalogrestservice
        resources:
          limits:
            memory: "8Gi"
```

**Type 2: JSON Patch (patches with op)**
Used for precise operations — add, replace, remove, test specific fields.

```yaml
# From overlays/uat/cd-blue/kustomization.yaml
patches:
- patch: |-
    - op: replace
      path: /metadata/name
      value: cd-blue
    - op: add
      path: /spec/template/spec/containers/0/env/-
      value:
        name: SITECORE_CD_DEPLOYMENT_ENV
        value: blue
  target:
    kind: Deployment
    name: cd
```

op: replace — changes a value
op: add — adds a new item
op: test — validates a value before proceeding (fails if mismatch)
op: remove — removes a field

---

### replicas — set replica count per environment

```yaml
# From overlays/uat/kustomization.yaml
replicas:
- name: cd-blue
  count: 2       <- UAT has 2 CD blue pods
- name: cd-green
  count: 0       <- Green is inactive in UAT
- name: cm
  count: 1
```

---

### vars — use ConfigMap values inside other resources

```yaml
# From xp1/configmaps/kustomization.yaml
vars:
- name: cm-hostname
  objref:
    kind: ConfigMap
    name: sitecore-hostnames
  fieldref:
    fieldpath: data.cm-hostname
```

This lets you use $(cm-hostname) as a variable reference inside other YAML files.

---

## PART 5 — HOW DEPENDENCIES ARE HANDLED IN PODS

---

### initContainers — wait before main container starts

Used when the main container depends on other services being ready first.

```yaml
# From xp1/cm.yaml
initContainers:
  - name: init-machinekey-and-wait
    image: sitecore-xp1-cm
    command: ["powershell", "-Command", "
      C:\\tools\\replace-machinekey.ps1;
      foreach ($service in $services) {
        do { Start-Sleep -Seconds 3 }
        until (Invoke-WebRequest $service returns 200)
      }
    "]
```

The CM pod does not start until all XDB services (xdbcollection, xdbsearch, cortexreporting, etc.) return HTTP 200 on /healthz/ready.

Order:
1. initContainer runs — waits for all dependencies
2. initContainer exits with code 0
3. Only then does the main Sitecore CM container start

---

### postStart lifecycle hook — run commands after container starts

```yaml
# From xp1/cm.yaml
lifecycle:
  postStart:
    exec:
      command: ["powershell", "-Command", "
        Start-Process filebeat.exe;
        C:/unicorn-sync.ps1;
        Wait for /healthz/ready;
        Run cache warming;
      "]
```

Runs immediately after container starts. If it fails (non-zero exit), Kubernetes kills the container.

---

### readinessProbe — when is the pod ready to receive traffic?

```yaml
readinessProbe:
  httpGet:
    path: /catalog/health/live
    port: 8080
  initialDelaySeconds: 65
  periodSeconds: 30
```

Kubernetes calls this endpoint every 30 seconds starting 65 seconds after start.
Pod only receives traffic when this returns HTTP 200.

---

### livenessProbe — is the pod still alive?

```yaml
livenessProbe:
  httpGet:
    path: /healthz/live
    port: 80
  timeoutSeconds: 10
  periodSeconds: 30
  failureThreshold: 3
```

If this fails 3 times in a row, Kubernetes kills and restarts the container.

---

### Service DNS — pods communicate using service names

Pods in the same namespace can reach each other using the service name directly:
- redis_host: redis (ConfigMap value)
- ldap_host: openldap (ConfigMap value)

Pods in different namespaces use full DNS:
- redis-blue.redis-product-catalog.svc.cluster.local:6379

Format: service-name.namespace.svc.cluster.local:port

---

## PART 6 — HOW ARGOCD AND K8S WORK TOGETHER

---

### ArgoCD — GitOps deployment

ArgoCD watches git repositories. When something changes it applies the change to Kubernetes automatically.

```
Git repo (fischer-sitecore-k8s)
  overlays/uat/kustomization.yaml
        |
        | ArgoCD watches this path
        v
ArgoCD detects change (image tag updated)
        |
        v
ArgoCD runs: kubectl apply -k overlays/uat/
        |
        v
Kubernetes rolls out new pods
```

---

### ApplicationSet — one ArgoCD app per environment

```yaml
# From fischer-argocd repo
spec:
  source:
    repoURL: git@bitbucket.org:smartcommerce_se/fischer-sitecore-k8s.git
    path: overlays/{{.values.env}}       <- test, uat, or prod
  destination:
    namespace: sitecore-{{.values.env}}
  syncPolicy:
    automated:
      prune: true       <- delete resources removed from git
      selfHeal: true    <- auto-fix manual changes (drift)
```

One ApplicationSet creates 3 ArgoCD apps — one per environment.
prune: true means if you remove a resource from git, ArgoCD deletes it from the cluster.
selfHeal: true means if someone manually changes something in the cluster, ArgoCD reverts it.

---

### Full application deployment flow

```
1. Developer pushes code to app repo
        |
        v
2. Jenkins/TeamCity detects push
   Builds Docker image
   Tags it (e.g., uat-build734)
        |
        v
3. Image pushed to ACR (fischergroup.azurecr.io)
        |
        v
4. DevOps updates newTag in kustomization.yaml
   Pushes to K8s git repo
        |
        v
5. ArgoCD detects change in K8s repo
   Runs kustomize build + kubectl apply
        |
        v
6. Kubernetes creates new pods with new image
   Old pods terminated (rolling update)
        |
        v
7. Application is live
```

---

## PART 7 — HOW ENVIRONMENTS ARE STRUCTURED

---

### Three environments — completely separate namespaces

| Environment | Namespace | Topology | Cluster |
|---|---|---|---|
| Test | sitecore-test | xp0 (single server) | nonprod-2 |
| UAT | sitecore-poc | xp1 (scaled) | nonprod-2 |
| Prod | sitecore-prod | xp1 (scaled) | prod-2 |

---

### Test environment (xp0)

- Single CM server (no CD separation)
- 1 replica of everything
- emptyDir volumes (no persistent data)
- Smaller resource limits
- No blue-green

---

### UAT environment (xp1 with blue-green)

```yaml
# overlays/uat/kustomization.yaml
replicas:
- name: cd-blue
  count: 2      <- active
- name: cd-green
  count: 0      <- inactive
- name: cm
  count: 1
```

- CD split into blue and green deployments
- Traffic goes to blue by default
- Green is scaled to 0 (inactive)
- Switch happens during deployment via blue-green script

---

### Prod environment (xp1 with blue-green)

Same structure as UAT but:
- More replicas
- Larger resource limits (memory, CPU)
- PVCs for persistent storage (logs, data)
- Dedicated Windows node pools for blue and green
- Node selectors: fischer.de/nodepool=blue or =green

---

### How environments share base and override specifics

```
xp1/                           <- shared base for UAT and Prod
  cm.yaml                      <- CM deployment (Windows)
  cd/                          <- CD deployment templates
  configmaps/                  <- hostname ConfigMaps
  external/                    <- ExternalSecret definitions
  secrets/                     <- Secret files

overlays/uat/                  <- UAT-specific
  kustomization.yaml           <- references ../../xp1, sets UAT image tags
  cd-blue/                     <- blue CD overlay (UAT Redis endpoint)
  cd-green/                    <- green CD overlay
  secrets/                     <- UAT-specific ExternalSecrets (UAT Key Vault)

overlays/prod/                 <- Prod-specific
  kustomization.yaml           <- references ../../xp1, sets Prod image tags
  cd-blue/                     <- blue CD overlay (Prod Redis endpoint)
  cd-green/                    <- green CD overlay
  secrets/                     <- Prod-specific ExternalSecrets (Prod Key Vault)
```

---

## PART 8 — GIT BRANCHING STRATEGY

---

### App repos (fischer-website-2.0, fischer-sitecore-jss, etc.)

```
local branch (FW-XXXX-feature-name)
        |
        v (PR)
  development       <- test and UAT builds triggered from here
        |
        v (PR, when deploying to prod)
      main          <- prod deployment, app team updates image tags
```

- Jenkins builds on push to development or main
- Tags like uat-build734 or Release-8.34.2-prod-build12 are created here

---

### K8s repos (fischer-sitecore-k8s, fischer-cluster-applications, etc.)

```
local branch (FW-XXXX-description)
        |
        v (PR, reviewed and approved)
     master         <- ArgoCD watches this branch
```

- You never commit directly to master
- All changes go through PR
- ArgoCD reads from master branch automatically

---

### Infrastructure repos (fischer-operations)

```
local branch (FW-XXXX-description)
        |
        v (PR)
      main          <- Atlantis applies Terraform on merge
```

- PR triggers Atlantis plan (automatic)
- Merge triggers Atlantis apply
- Daily Jenkins job checks for drift

---

### Why separate branches for K8s vs App?

App repos use development as integration branch because multiple teams push frequently.
K8s repos use master directly because changes are infrequent, reviewable, and must be exact.

---

## PART 9 — 120 INTERVIEW QUESTIONS WITH ANSWERS

---

### KUBERNETES BASICS

**1. What is Kubernetes and what problem does it solve?**
Kubernetes is a container orchestration platform. It solves the problem of running, scaling, and managing containers in production. It handles scheduling containers on nodes, restarting failed containers, scaling based on load, rolling updates, and service discovery.

**2. What is a Pod?**
The smallest deployable unit in Kubernetes. A pod is one or more containers that share the same network namespace and storage. Containers in a pod communicate via localhost and share volumes.

**3. What is the difference between a Pod and a Deployment?**
A Pod is a single instance of containers. A Deployment manages a set of identical Pods, handles rolling updates, rollbacks, and ensures the desired number of replicas is always running.

**4. What is a ReplicaSet?**
Ensures a specified number of Pod replicas are running. The Deployment creates and manages ReplicaSets automatically. You rarely create ReplicaSets directly.

**5. What is a Service in Kubernetes?**
A stable network endpoint to reach pods. Pods have dynamic IPs that change on restart. A Service provides a fixed DNS name and load balances traffic to pods matching its selector.

**6. What are the types of Services?**
ClusterIP — internal only, default type.
NodePort — exposes on each node's IP at a static port.
LoadBalancer — creates an Azure/AWS load balancer.
ExternalName — maps to an external DNS name.

**7. What is a Namespace?**
A logical partition within a Kubernetes cluster. Resources in different namespaces are isolated. In Fischer each application has its own namespace (catalogrestservice-prod, sitecore-prod, etc.).

**8. What is an Ingress?**
A rule for routing external HTTP/HTTPS traffic to internal services. One Ingress controller (like nginx) handles all rules. Defines path-based or host-based routing.

**9. What is a ConfigMap?**
A Kubernetes object that stores non-sensitive configuration as key-value pairs. Can be injected into pods as env vars or mounted as files.

**10. What is a Secret?**
Similar to ConfigMap but for sensitive data. Values are base64 encoded (not encrypted by default). Can be injected as env vars or mounted as files. In Fischer, Secrets are created by the External Secrets Operator from Azure Key Vault — never stored in git.

---

### ENVIRONMENT VARIABLES IN PODS

**11. What are the ways to inject configuration into a Kubernetes pod?**
1. Plain env var (value hardcoded in manifest)
2. From Secret single key (secretKeyRef)
3. From ConfigMap single key (configMapKeyRef)
4. envFrom secretRef (all Secret keys as env vars)
5. envFrom configMapRef (all ConfigMap keys as env vars)
6. Volume mount (Secret or ConfigMap mounted as files)

**12. What is envFrom and when do you use it?**
envFrom injects all keys from a Secret or ConfigMap as env vars without listing each one. Used when there are many keys and you want them all. Example: envFrom: - secretRef: name: crs-secrets injects ldap_access_pass and ldap_access_user simultaneously.

**13. What is the difference between env and envFrom?**
env injects specific named keys. envFrom injects all keys from a resource. Use env when you need specific keys with potentially different names in the container. Use envFrom when you want all keys from a config resource.

**14. How does a Secret value become available inside a container?**
Define it in the pod spec with secretKeyRef. Kubernetes mounts the secret value as an environment variable. The container process reads it like any OS environment variable. The actual value is never written to disk inside the container.

**15. What happens if a Secret referenced by a pod does not exist?**
The pod fails to start with an error: secret "name" not found. Pod stays in Pending state. This is why External Secrets Operator must create the Secret before the pod can start.

---

### CONFIGMAPS AND CONFIGMAPGENERATOR

**16. What is ConfigMapGenerator in Kustomize?**
A feature in kustomization.yaml that automatically creates ConfigMaps from files on disk. Instead of manually writing a ConfigMap YAML, you list files and Kustomize reads their content and creates the ConfigMap.

**17. Why use ConfigMapGenerator instead of writing ConfigMap YAML manually?**
When config files are complex (like YAML, XML, or JSON files), ConfigMapGenerator is cleaner. The file on disk is the source of truth. No need to escape or format content into YAML key-value pairs.

**18. What is disableNameSuffixHash in ConfigMapGenerator?**
By default Kustomize appends a hash to generated ConfigMap names (e.g., filebeat-config-7d4b9f2a). This causes problems when other resources reference the ConfigMap by exact name. disableNameSuffixHash: true keeps the name as defined.

**19. How is a ConfigMap mounted as files in a pod?**
Define it as a volume using configMap type, then mount the volume. Each key in the ConfigMap becomes a file in the mount path. The key name is the filename, the value is the file content.

**20. Why does Sitecore use ConfigMap file mounts instead of env vars?**
Sitecore's Settings.GetSetting() API reads configuration from XML files only. It does not read environment variables for classic Sitecore settings. So config must be delivered as XML files mounted at the App_Config/Include path.

**21. What is the difference between a ConfigMap and a Secret?**
ConfigMap: for non-sensitive config. Values are plain text in the manifest and in etcd. Secret: for sensitive data. Values are base64 encoded. Can be encrypted at rest. Access can be controlled with RBAC. In Fischer, Secrets are never committed to git — they come from Key Vault via ExternalSecret.

**22. How do you update a running pod when a ConfigMap changes?**
Pods do not automatically reload when a ConfigMap changes. You need to restart the pods. Some applications support hot-reload by watching file changes. For most Fischer apps, a pod restart (rolling update) is needed.

---

### SECRETS AND EXTERNAL SECRETS

**23. What is the External Secrets Operator?**
A Kubernetes operator that syncs secrets from external secret stores (like Azure Key Vault, AWS Secrets Manager) into Kubernetes Secrets. You define an ExternalSecret resource in git — the operator reads the actual value from Key Vault and creates the K8s Secret.

**24. What is an ExternalSecret resource?**
A Kubernetes custom resource that tells the External Secrets Operator: which Key Vault to read from, which secret key to read, and what to name the resulting K8s Secret. It is safe to commit to git because it contains only references — not actual secret values.

**25. What is ClusterSecretStore?**
A cluster-wide resource that defines the connection to the external secret store (Azure Key Vault). ExternalSecrets reference the ClusterSecretStore by name. One ClusterSecretStore can serve all namespaces.

**26. What is the difference between data and dataFrom in ExternalSecret?**
data: maps specific keys from the KV secret to specific K8s Secret keys. One entry per key.
dataFrom with extract: extracts ALL properties from a JSON KV secret as separate K8s Secret keys. Used in Fischer for sitecore-config-patches where KV secret contains multiple XML files.

**27. What is creationPolicy Owner in ExternalSecret?**
Means the ExternalSecret owns the created K8s Secret. If the ExternalSecret is deleted, the K8s Secret is also deleted.

**28. What is deletionPolicy Retain?**
Even if the ExternalSecret is deleted, the K8s Secret is kept. Used for critical secrets that should survive accidental ExternalSecret deletion.

**29. How does a Secret reach the Sitecore CM pod as XML files?**
Key Vault has a secret with JSON containing XML file contents. ExternalSecret reads it and creates a K8s Secret with one key per XML file. The pod mounts this Secret as a volume. Each Secret key becomes a file at the mount path. Sitecore reads the XML files from that path on startup.

**30. Why are Secrets never stored in git?**
Secrets in git are a major security risk — anyone with repo access can read them, they appear in git history permanently, and rotation requires a git commit. In Fischer, git only contains ExternalSecret definitions (references), never actual values. Values live only in Azure Key Vault.

---

### KUSTOMIZE

**31. What is Kustomize?**
A tool for customizing Kubernetes manifests without templates. Uses a base + overlay pattern. Base has the common manifests. Overlays apply environment-specific changes. No templating language — just YAML patching.

**32. What is the difference between Kustomize and Helm?**
Helm uses templates with Go templating syntax. Kustomize uses plain YAML with patches. Helm has a release management concept (install, upgrade, rollback). Kustomize is simpler — just file transformation. Fischer uses both: Kustomize for most apps, Helm for Jenkins and TeamCity.

**33. What is a base in Kustomize?**
A directory with the shared Kubernetes manifests that are the same across environments. Contains deployment.yaml, service.yaml, configmap.yaml, etc. Never applied directly — always through an overlay.

**34. What is an overlay in Kustomize?**
An environment-specific directory that references a base and applies changes. Has its own kustomization.yaml that says: take base, change image tags, set namespace, apply patches.

**35. What is resources in kustomization.yaml?**
Lists what to include — either files in the same directory or references to other directories (like ../base). Kustomize merges all listed resources.

**36. What is images in kustomization.yaml?**
Replaces image tags in all manifests without editing the base files. Matches by image name and replaces newName and newTag. Used to update deployment image versions per environment.

**37. What is patchesStrategicMerge?**
A Kustomize patching method that merges a partial YAML on top of matching resources. Good for overriding specific fields like resource limits, PVC names, or replicas while keeping everything else from base.

**38. What is a JSON patch in Kustomize?**
Uses RFC 6902 JSON Patch operations (add, replace, remove, test) on specific YAML paths. More precise than strategic merge — can add items to lists, replace specific array elements, or test values before changing. Used in Fischer for blue-green to rename deployments and add env vars.

**39. What is the test operation in JSON patch?**
Validates that a field has an expected value before applying changes. If the value does not match, the entire patch fails. Used in Fischer to verify the correct container image before replacing it (safety check).

**40. What is replicas in kustomization.yaml?**
Sets replica counts for named deployments. Cleaner than patching the deployment YAML. Used in Fischer to set cd-blue=2, cd-green=0 in UAT.

**41. How does Kustomize handle namespaces?**
Set namespace: once in kustomization.yaml and it applies to all resources in that overlay. No need to set namespace in every individual file.

**42. What is vars in Kustomize?**
Allows referencing values from one resource inside another resource using $(var-name) syntax. In Fischer, cm-hostname is read from the sitecore-hostnames ConfigMap and can be injected into other manifests.

---

### ARGOCD AND GITOPS

**43. What is GitOps?**
A deployment approach where git is the single source of truth for infrastructure state. Changes are made by committing to git. An operator (ArgoCD) continuously syncs the cluster to match git. Manual cluster changes are detected and reverted.

**44. What is ArgoCD?**
A GitOps continuous delivery tool for Kubernetes. Watches git repositories, detects changes, and applies them to Kubernetes clusters automatically. Provides a UI to see deployment status and sync state.

**45. What is an Application in ArgoCD?**
A resource that defines: which git repo and path to watch, which cluster and namespace to deploy to, and sync policies. ArgoCD continuously reconciles the cluster state with the git state.

**46. What is an ApplicationSet in ArgoCD?**
A template that generates multiple ArgoCD Applications automatically. In Fischer one ApplicationSet creates test, uat, and prod applications from the same template — each pointing to a different overlay path and namespace.

**47. What is selfHeal in ArgoCD?**
When enabled, ArgoCD automatically reverts any manual changes made to the cluster that differ from git. Ensures the cluster always matches git state. Fischer uses selfHeal: true.

**48. What is prune in ArgoCD?**
When enabled, ArgoCD deletes resources from the cluster that no longer exist in git. Without prune, deleted manifests leave orphaned resources in the cluster.

**49. What is the difference between sync and refresh in ArgoCD?**
Refresh: ArgoCD checks git for new commits and updates its view.
Sync: ArgoCD applies the current git state to the cluster.
With auto-sync enabled, both happen automatically.

**50. How do you force ArgoCD to sync immediately?**
From UI: click Sync button.
From CLI: argocd app sync app-name.
From kubectl: annotate the Application with a force-sync timestamp.
In Fischer for ExternalSecrets: kubectl annotate externalsecret name force-sync=$(date +%s).

---

### PODS — LIFECYCLE AND PROBES

**51. What is an initContainer?**
A container that runs to completion before the main container starts. Used for setup tasks — waiting for dependencies, populating shared volumes, running migrations. Must exit with code 0 for the main container to start.

**52. When would you use an initContainer vs a postStart hook?**
initContainer: runs before main container. Good for waiting on dependencies, setup tasks that must complete first.
postStart: runs after main container starts (simultaneously). Cannot delay the container start. Good for background tasks that do not block readiness.

**53. What is a postStart lifecycle hook?**
A command that runs immediately after the container starts. Runs in parallel with the container's main process. If it fails, Kubernetes kills the container. Used in Fischer CM pod to start filebeat, run unicorn sync, and warm cache.

**54. What is a readinessProbe?**
Kubernetes checks this endpoint periodically. Pod only receives traffic when readiness probe succeeds. If it fails, the pod is removed from Service endpoints but not restarted. Used to prevent traffic from reaching a pod that is still starting up.

**55. What is a livenessProbe?**
Kubernetes checks this endpoint periodically. If it fails consecutively (failureThreshold times), Kubernetes kills and restarts the container. Used to detect deadlocks or hung processes that will not recover on their own.

**56. What is a startupProbe?**
Used for applications that take a long time to start. Disables liveness and readiness probes until the startup probe succeeds. Prevents Kubernetes from killing a slow-starting container before it is ready.

**57. What is initialDelaySeconds in a probe?**
How long Kubernetes waits after container start before running the first probe. Important for applications that take time to initialize. Fischer catalogrestservice uses initialDelaySeconds: 65 because the Java app takes time to warm up.

**58. What happens to a pod during a rolling update?**
Kubernetes creates new pods with the new image. Waits for new pods to pass readiness probe. Then terminates old pods one by one. At no point are all pods down simultaneously — traffic continues to be served.

**59. What is minReadySeconds in a Deployment?**
How long a newly created pod must be ready before Kubernetes considers it available. Prevents a fast-failing pod from cycling through too quickly. Used in Fischer CM deployment: minReadySeconds: 60.

**60. What is imagePullPolicy IfNotPresent?**
Kubernetes only pulls the image from ACR if it is not already cached on the node. Faster if the image is already there. Always is the alternative — always pulls. Never skips pulling entirely.

---

### NETWORKING IN KUBERNETES

**61. How do pods in the same namespace communicate?**
Using the Service name directly as the hostname. DNS resolution: service-name -> ClusterIP of that service. Example: catalogrestservice reaches Redis using hostname redis (from ConfigMap ldap_host: openldap).

**62. How do pods in different namespaces communicate?**
Using the full DNS name: service-name.namespace.svc.cluster.local. Example: Sitecore CD connects to redis-blue.redis-product-catalog.svc.cluster.local:6379.

**63. What is a ServiceMonitor?**
A Prometheus Operator custom resource that tells Prometheus how to scrape metrics from a Service. Defines the port, path, and interval for metric collection. Fischer uses ServiceMonitor for catalogrestservice, FTP service, Redis, and others.

**64. What is a ClusterIP Service?**
The default Service type. Provides a stable internal IP accessible only within the cluster. Not reachable from outside. Most internal Services in Fischer use ClusterIP.

**65. What is an ExternalName Service?**
Maps a Kubernetes Service name to an external DNS name. Kubernetes returns a CNAME pointing to the external host. Used to give external services a cluster-internal DNS name so pods can use a consistent name regardless of environment.

---

### VOLUMES AND STORAGE

**66. What is a PersistentVolume (PV)?**
A piece of storage provisioned in the cluster. Can be Azure Disk, Azure File, NFS, etc. Exists independently of pods — data persists when pod is deleted.

**67. What is a PersistentVolumeClaim (PVC)?**
A request for storage by a pod. Kubernetes finds or provisions a PV that matches the claim. Pod mounts the PVC. In Fischer prod: catalogrestservice uses a PVC on Azure File for catalog data.

**68. What is emptyDir?**
A temporary volume that exists for the lifetime of the pod. Deleted when pod is deleted. Used in Fischer for shared data between initContainer and main container (frontend files copied by init-frontend initContainer).

**69. What is the difference between Azure Disk and Azure File in Kubernetes?**
Azure Disk: can be mounted by only ONE pod at a time (ReadWriteOnce). Higher performance. Good for databases.
Azure File: can be mounted by MULTIPLE pods simultaneously (ReadWriteMany). Good for shared log storage or shared data.

**70. Why does Fischer use Azure File for logs?**
Multiple Sitecore pods (blue and green) all write logs. Azure File allows multiple pods to mount the same share simultaneously with ReadWriteMany access mode. Logs are centralized on one share.

---

### BLUE-GREEN DEPLOYMENT

**71. What is blue-green deployment?**
A deployment strategy where two identical environments (blue and green) exist. Only one is live at a time. New version is deployed to the inactive one. Traffic is switched after validation. The old one becomes the standby. Zero downtime deployment.

**72. How is blue-green implemented in Fischer for Sitecore CD?**
Two separate Kubernetes Deployments: cd-blue and cd-green. A single Kubernetes Service (cd-default) switches its selector between app: cd-blue and app: cd-green. When green is active, Service selector points to cd-green pods.

**73. What is the role of node selectors in Fischer blue-green?**
Blue pods run on nodes with label fischer.de/nodepool=blue. Green pods run on nodes with label fischer.de/nodepool=green. This ensures blue and green pods never share the same node — complete isolation.

**74. How do you switch traffic in blue-green?**
Change the Service selector from app: cd-blue to app: cd-green. In Fischer, the blue-green deployment script updates the cd-default-service selector and pushes to git. ArgoCD applies the change. Traffic switches to green instantly.

**75. What are the advantages of blue-green over rolling update?**
Instant rollback — just switch selector back. No mixed versions in production simultaneously. Full testing on inactive environment before switching. Fischer uses it for Sitecore because Windows containers take long to start.

---

### GIT AND DEVOPS FLOW

**76. What is the git branching strategy for app repos in Fischer?**
Local feature branch -> development (PR). Development -> main (PR on prod deployment). Jenkins builds from development and main. Image tags like uat-build734 come from development builds.

**77. What is the git branching strategy for K8s repos?**
Local feature branch -> master (PR). Always create branch from origin/master to get latest changes. ArgoCD watches master branch directly.

**78. What is Atlantis and how does it work?**
A tool for automated Terraform plan and apply via pull requests. When you raise a PR with Terraform changes, Atlantis runs terraform plan and posts output as a PR comment. Comment atlantis apply to apply changes. Terraform never runs manually.

**79. What triggers an ArgoCD deployment?**
A commit to the git repository that ArgoCD is watching. Usually updating an image tag in kustomization.yaml. ArgoCD polls the repo (or uses webhooks) and detects the change. Then applies the updated manifests to the cluster.

**80. How do you roll back a deployment in Fischer?**
Revert the kustomization.yaml change (previous image tag) and push to git. ArgoCD detects the revert and applies the old manifests. Kubernetes rolls back to the previous image version.

---

### SCENARIO BASED

**81. A pod is in CrashLoopBackOff. How do you investigate?**
kubectl describe pod pod-name — see events, last restart reason.
kubectl logs pod-name --previous — see logs from the crashed container.
Check liveness probe settings — maybe failing too early.
Check initContainers — maybe one is failing.
Check if a referenced Secret or ConfigMap exists.

**82. A pod is in Pending state. What could be wrong?**
No nodes available with enough resources (CPU/memory).
Node selector or toleration mismatch — pod cannot be scheduled.
PVC not bound — waiting for storage.
Image pull secret missing — cannot pull from private registry.
kubectl describe pod shows the exact reason in Events section.

**83. An ExternalSecret is not creating the K8s Secret. How do you investigate?**
kubectl describe externalsecret name -n namespace — shows sync status and error.
Check if ClusterSecretStore is healthy.
Verify the Key Vault secret name and property name are correct.
Check if the managed identity has access to the Key Vault secret.

**84. How do you check if a ConfigMap was mounted correctly inside a pod?**
kubectl exec -it pod-name -- ls /mount/path — list files in the mount.
kubectl exec -it pod-name -- cat /mount/path/filename — read the file content.
Compare with the ConfigMap content: kubectl get configmap name -o yaml.

**85. An env var from a Secret is not available inside the pod. How do you debug?**
kubectl exec -it pod-name -- env | grep VAR_NAME — check if var exists.
kubectl get secret secret-name -o yaml — verify the key exists in the Secret.
Check pod spec — verify secretKeyRef name and key match.
kubectl describe pod — check for SecretKeyNotFound errors in events.

**86. You need to add a new environment variable to a production pod. What is the process?**
Add the env var to the appropriate YAML file (deployment or kustomization patch).
If it is sensitive: add to Key Vault first, update ExternalSecret to include the new key.
If it is not sensitive: add to ConfigMap.
Raise a PR, get it reviewed, merge to master.
ArgoCD deploys the change — pods restart with the new env var.

**87. Sitecore CM pod is in CrashLoopBackOff with FailedPostStartHook. What do you check?**
The postStart command failed with non-zero exit code.
Check if all scripts referenced in postStart exist in the container image.
Check if initContainers all passed (services it waits for must be healthy).
Check pod events: kubectl describe pod cm-pod-name.
Note: postStart output does not go to kubectl logs — it goes to container's own log system.

**88. ArgoCD shows an application is OutOfSync but not syncing. Why?**
Auto-sync might be disabled.
There might be a sync error — check ArgoCD UI for error message.
There might be a resource conflict (another tool managing the same resource).
Check if the application is suspended (sync paused).

**89. How would you add a new secret to a Fischer application?**
Add the secret value to Azure Key Vault.
Add a new entry to the ExternalSecret YAML referencing the new KV key.
Reference the new secret key in the deployment YAML via secretKeyRef or envFrom.
Raise a PR, merge, ArgoCD deploys.
Verify with: kubectl exec pod -- env | grep NEW_VAR_NAME.

**90. A deployment is stuck at 50% during a rolling update. What happened?**
New pods might be failing readiness probes.
Check if new pods have errors: kubectl describe pod new-pod-name.
Resource limits might be too low for the new version.
A required Secret or ConfigMap for the new version might be missing.
The rolling update will not proceed until new pods become Ready.

**91. How do you force restart all pods in a Deployment without changing anything?**
kubectl rollout restart deployment/deployment-name -n namespace.
This triggers a rolling restart — all pods restart with the same image and config.

**92. What is the difference between kubectl apply and kubectl create?**
kubectl create fails if the resource already exists.
kubectl apply creates if not exists, updates if exists. Idempotent — can run multiple times safely. ArgoCD uses apply internally.

**93. How do you check which image version is currently running in a pod?**
kubectl get pod pod-name -o jsonpath="{.spec.containers[0].image}"
Or: kubectl describe pod pod-name | grep Image.

**94. A new Kustomize overlay for a new environment needs to be created. What files do you create?**
Create new directory (e.g., staging/).
Create kustomization.yaml with: resources pointing to ../base, correct namespace, environment-specific image tags.
Add any environment-specific patches.
Add the new ArgoCD Application pointing to the staging overlay.
Test with: kustomize build staging/ to preview output before applying.

**95. How do you check if ArgoCD has detected your latest commit?**
ArgoCD UI: check Last Sync Revision matches your commit hash.
CLI: argocd app get app-name shows current revision.
If not detected: force refresh with argocd app get app-name --refresh.

**96. What would cause an ExternalSecret to sync successfully but the pod still has an old secret value?**
The pod was not restarted after the Secret was updated. Kubernetes does NOT automatically restart pods when a Secret changes. You need to restart the deployment: kubectl rollout restart deployment/name.

**97. How does Fischer handle the case where two CD pods (blue and green) need different Redis connection strings?**
Each has its own configMapGenerator in kustomization.yaml with a color-specific XML config file. Blue mounts k8s.Redis.ConnectionString.config pointing to redis-blue. Green mounts same filename but with redis-green endpoint. Both mount at the same path inside the container. Sitecore reads whichever file is there — color-specific connection string automatically applied.

**98. What is a nodeSelector and how is it used in Fischer?**
nodeSelector is a field in the pod spec that restricts which nodes a pod can run on. kubernetes.io/os: windows means only run on Windows nodes. fischer.de/nodepool=blue means only run on nodes in the blue node pool. Used to ensure Windows Sitecore pods only run on Windows nodes, and blue/green pods are isolated on their respective node pools.

**99. What is a toleration in Kubernetes?**
Allows a pod to be scheduled on a node that has a matching taint. Nodes can have taints that repel pods by default. Only pods with matching tolerations can be scheduled there. In Fischer, the redis-product-catalog nodepool has a taint redis-product-catalog:NoSchedule — only the Redis pod has a matching toleration.

**100. What is the purpose of imagePullSecrets in a pod spec?**
Provides credentials to pull Docker images from private registries. Without it, pods cannot pull from fischergroup.azurecr.io. The acr-secret Secret contains the ACR credentials. Every pod spec in Fischer includes imagePullSecrets: - name: acr-secret.

---

### ADVANCED QUESTIONS

**101. What is the External Secrets Operator and why use it instead of putting secrets in git?**
ESO is a Kubernetes operator that syncs secrets from external vaults (Azure Key Vault) to K8s Secrets automatically. It is used because: secret values never touch git, Key Vault provides audit logs of who accessed what, rotation in Key Vault is reflected automatically in K8s, and access can be controlled with Azure RBAC.

**102. What is a ClusterSecretStore vs SecretStore?**
SecretStore: namespace-scoped — only ExternalSecrets in the same namespace can use it.
ClusterSecretStore: cluster-wide — ExternalSecrets in any namespace can use it.
Fischer uses ClusterSecretStore so all namespaces share the same Key Vault connection.

**103. How does Kustomize handle the same file being referenced in base and overlay?**
Overlay patches are merged on top of base. Strategic merge: fields in patch override same fields in base. JSON patch: precise path-based operations on specific locations. The base is never modified — the output is a new merged manifest.

**104. What is the difference between Deployment and StatefulSet?**
Deployment: pods are interchangeable, no stable identity. StatefulSet: pods have stable names (pod-0, pod-1), stable storage (PVC per pod), ordered start/stop. Used for databases, Kafka, Zookeeper. Fischer uses StatefulSet for SolrCloud (Zookeeper + Solr nodes need stable identity).

**105. What is a DaemonSet?**
Ensures one pod runs on every node (or every matching node). Used for node-level monitoring or logging. Fischer uses DaemonSet for azure-ip-masq-agent and konnectivity-agent (AKS system components) and filebeat for log shipping.

**106. What is a ServiceAccount in Kubernetes?**
An identity for pods to authenticate to the Kubernetes API server or external services. Pods use the ServiceAccount's token to make API calls. In Fischer, catalogrestservice has its own ServiceAccount for Workload Identity (Azure managed identity).

**107. What is Workload Identity?**
Allows pods to authenticate to Azure services (Key Vault, Storage) using a Kubernetes ServiceAccount linked to an Azure Managed Identity. No credentials stored in the cluster. The pod gets an Azure identity token automatically.

**108. What happens when you delete a pod managed by a Deployment?**
Kubernetes detects the pod count is below the desired replicas. ReplicaSet creates a replacement pod immediately. The deleted pod is gone but service continuity is maintained because other replicas are still running.

**109. How does rolling update work with maxSurge and maxUnavailable?**
maxSurge: how many extra pods can exist above the desired count during update.
maxUnavailable: how many pods can be unavailable during update.
Example: replicas=3, maxSurge=1, maxUnavailable=0 means: create 1 new pod first (total=4), wait for it to be ready, then delete 1 old pod (total=3 again). Repeat until all are updated.

**110. What is a HorizontalPodAutoscaler (HPA)?**
Automatically scales pod replicas based on CPU, memory, or custom metrics. Defines min and max replicas and target metric threshold. Fischer service-1 cluster uses HPA for the kubernetes-service-1 node pool.

**111. What is Kyverno?**
A Kubernetes policy engine. Defines rules for validating, mutating, or generating Kubernetes resources. In Fischer, kyverno namespace is deployed in the cluster to enforce policies (e.g., all pods must have resource limits).

**112. What is a ServiceMonitor and how does Prometheus use it?**
ServiceMonitor is a Prometheus Operator CRD. Defines which Services Prometheus should scrape for metrics. Prometheus controller watches ServiceMonitor resources and automatically configures scraping. Fischer uses ServiceMonitors for catalogrestservice, Redis, openldap, and FTP service.

**113. What is the difference between requests and limits in resource management?**
requests: minimum resources guaranteed to the pod. Kubernetes uses this for scheduling decisions.
limits: maximum resources the pod can use. If exceeded, CPU is throttled. If memory limit is exceeded, the container is OOMKilled.
Best practice: always set both. Requests < limits.

**114. What is OOMKilled?**
Out Of Memory Killed. Container exceeded its memory limit. Kubernetes terminates it. Pod restarts. Fix: increase memory limit or fix memory leak in the application. Fischer Sitecore pods have generous limits (10Gi for CM) because Sitecore is memory-intensive.

**115. How does Fischer handle Windows container scheduling on AKS?**
Windows nodes have taint kubernetes.io/os=windows:NoSchedule. Only pods with toleration kubernetes.io/os: windows can be scheduled there. Pod specs for Sitecore (CM, CD) have nodeSelector: kubernetes.io/os: windows and matching toleration. Linux pods cannot accidentally end up on Windows nodes.

**116. What is a PodDisruptionBudget (PDB)?**
Defines minimum available pods during voluntary disruptions (node drain, upgrades). Prevents too many pods from being taken down at once. Example: minAvailable: 1 ensures at least 1 pod is always running during maintenance.

**117. What is kubectl rollout status used for?**
Shows the progress of a rolling update. kubectl rollout status deployment/name waits and shows when each replica is updated. Returns 0 when fully rolled out. Used in scripts to wait for deployments to complete.

**118. What is the difference between hard and soft pod affinity/anti-affinity?**
Hard (required): pod MUST be scheduled on nodes matching the rule. Fails if no matching node.
Soft (preferred): pod PREFERS matching nodes but can be scheduled elsewhere. Used to spread pods across availability zones while not failing if only one zone is available.

**119. How do you check resource usage of pods in Kubernetes?**
kubectl top pods -n namespace — shows current CPU and memory usage.
kubectl top nodes — shows node-level resource usage.
Requires metrics-server to be installed (Fischer has it deployed).

**120. What is the Kubernetes control plane and what components does it have?**
The control plane manages the cluster. Components:
- API Server: the front door — all kubectl commands go here
- etcd: the database — stores all cluster state including secrets
- Scheduler: decides which node a pod runs on
- Controller Manager: runs controllers (ReplicaSet, Deployment, etc.)
- konnectivity-agent: maintains tunnel between API server and nodes (AKS-specific)
In AKS, the control plane is managed by Azure — you only manage the worker nodes.
