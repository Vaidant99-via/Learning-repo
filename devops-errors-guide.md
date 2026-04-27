# DevOps Errors Guide — Real Errors, Causes, and Fixes
# For: Vaidant Vyas | April 2026

---

## PART 1 — KUBERNETES POD ERRORS

---

### CrashLoopBackOff

**What it means:** Container is starting, crashing, and Kubernetes keeps restarting it.
Kubernetes backs off the restart time exponentially (10s, 20s, 40s, 80s...) — that is the "BackOff" part.

**How to check:**
```
kubectl describe pod pod-name -n namespace    <- check Events section
kubectl logs pod-name -n namespace --previous <- logs from crashed container
```

**Common causes and fixes:**

| Cause | Fix |
|---|---|
| Application throws exception on startup | Fix the application code or config |
| Missing environment variable | Add the env var or fix the Secret/ConfigMap reference |
| Referenced Secret does not exist | Create the Secret or fix ExternalSecret |
| Referenced ConfigMap does not exist | Create the ConfigMap |
| Wrong command in pod spec | Fix the command/args in deployment YAML |
| postStart hook fails (FailedPostStartHook) | Fix the script in postStart or add error handling |
| Out of memory on startup | Increase memory limits |
| Application port mismatch | Align containerPort with what app listens on |
| initContainer fails | Fix the initContainer command/dependencies |

**Real Fischer example:** CM pod CrashLoopBackOff with FailedPostStartHook because unicorn-sync.ps1 was referenced in postStart but not present in the container image.

---

### OOMKilled

**What it means:** Container exceeded its memory limit. Kubernetes killed it.

**How to check:**
```
kubectl describe pod pod-name    <- shows: Last State: OOMKilled
```

**Causes and fixes:**
- Memory limit is too low — increase limits in deployment YAML
- Memory leak in application — fix the application
- Too much data loaded into memory — tune application settings (e.g., Redis maxmemory)

---

### ImagePullBackOff / ErrImagePull

**What it means:** Kubernetes cannot pull the container image from the registry.

**How to check:**
```
kubectl describe pod pod-name    <- shows: Failed to pull image
```

**Causes and fixes:**

| Cause | Fix |
|---|---|
| Wrong image name or tag | Check kustomization.yaml — correct image name and newTag |
| Image tag does not exist in ACR | Verify the build pushed the image with that exact tag |
| imagePullSecret missing or wrong | Check acr-secret exists in namespace, verify credentials |
| ACR login expired | Refresh ACR credentials in the secret |
| Registry is private, no pull secret | Add imagePullSecrets to pod spec |
| Network issue reaching ACR | Check cluster networking, DNS resolution |

**Real Fischer example:** After ACR migration from smartcommerce.azurecr.io to fischergroup.azurecr.io — some kustomization.yaml files still had the old registry name. Pods got ImagePullBackOff because fischergroup ACR did not have images under the old name.

---

### Pending — pod not scheduling

**What it means:** Pod is waiting to be assigned to a node.

**How to check:**
```
kubectl describe pod pod-name    <- Events section shows why
```

**Causes and fixes:**

| Cause | Fix |
|---|---|
| Insufficient CPU on all nodes | Scale up node pool or reduce CPU requests |
| Insufficient memory on all nodes | Scale up node pool or reduce memory requests |
| nodeSelector does not match any node | Fix nodeSelector or add label to node |
| Toleration missing for tainted node | Add correct toleration to pod spec |
| PVC not bound | Check PVC status: kubectl get pvc -n namespace |
| imagePullSecret missing (different symptom) | Add the secret |

**Real Fischer example:** konnectivity-agent pods stuck in Pending because 6 nodes were unreachable (node.kubernetes.io/unreachable taint) and only 1 node was available — not enough to schedule. Error: "0/12 nodes available: 6 nodes had untolerated taint node.kubernetes.io/unreachable."

---

### Error: failed to create containerd task

**What it means:** Container runtime cannot start the container.

**Causes and fixes:**
- Windows container on Linux node — add nodeSelector: kubernetes.io/os: windows
- Privileged container not allowed by policy — check security context
- Volume mount path does not exist — verify mountPath is correct

---

### CreateContainerConfigError

**What it means:** Kubernetes cannot create the container configuration.

**Causes and fixes:**
- Secret key referenced does not exist in the Secret
- ConfigMap key referenced does not exist in the ConfigMap
- Environment variable name conflict

**How to check:**
```
kubectl describe pod pod-name    <- shows exact missing key
```

---

### RunContainerError

**What it means:** Container started but immediately failed.

**Causes:**
- Container entrypoint/command does not exist in image
- Permission denied on startup script
- Wrong working directory

---

### FailedScheduling

**What it means:** Kubernetes scheduler cannot find a suitable node.

**Common messages:**
- "0/5 nodes are available: 5 Insufficient memory" — no node has enough memory
- "0/5 nodes are available: 5 node(s) had untolerated taint" — taint mismatch
- "0/5 nodes are available: 5 node(s) didn't match node selector" — nodeSelector mismatch

---

### Evicted

**What it means:** Kubernetes removed the pod from a node. Usually resource pressure.

**Causes:**
- Node is under memory pressure — DiskPressure or MemoryPressure condition
- Pod exceeded ephemeral storage limit
- Node is being drained (maintenance)

**How to check:**
```
kubectl get pods -n namespace    <- shows Evicted status
kubectl describe node node-name  <- shows conditions (MemoryPressure, DiskPressure)
```

---

### Terminating (stuck)

**What it means:** Pod is stuck in Terminating state and will not die.

**Causes:**
- Finalizers on the pod preventing deletion
- Volume unmount stuck
- Network plugin issue

**Fix:**
```
kubectl delete pod pod-name -n namespace --force --grace-period=0
```

---

## PART 2 — KUBERNETES SERVICE AND NETWORKING ERRORS

---

### Service not reachable (connection refused)

**Causes and fixes:**

| Cause | Fix |
|---|---|
| Pod not running | Check pod status — fix CrashLoopBackOff first |
| Service selector does not match pod labels | Compare service selector with pod labels |
| Wrong port in Service definition | Verify containerPort in pod matches port in Service |
| Pod failing readiness probe | Check readiness probe endpoint |
| NetworkPolicy blocking traffic | Check NetworkPolicy resources |

---

### DNS resolution failure (could not resolve host)

**Causes and fixes:**
- Wrong service name in config — verify exact service name: kubectl get svc -n namespace
- Service in different namespace — use full DNS: service.namespace.svc.cluster.local
- CoreDNS pod not running — kubectl get pods -n kube-system | grep coredns
- Pod DNS config wrong — check dnsPolicy in pod spec

---

### 502 Bad Gateway (from Ingress)

**What it means:** Ingress controller reached the backend but got a bad response.

**Causes and fixes:**

| Cause | Fix |
|---|---|
| Backend pod not running | Fix pod |
| Backend uses HTTPS but Ingress sends HTTP | Add backend protocol annotation |
| Wrong backend port | Verify service port in Ingress matches Service |
| Pod readiness probe failing | Fix the probe or the application |
| TLS backend without BackendTLSPolicy (NGF) | Add BackendTLSPolicy with CA cert |

**Real Fischer example:** Filebeat 502 error because nginx-gateway-fabric was sending plain HTTP to ECK Elasticsearch which only accepts HTTPS on port 9200. Fix: add BackendTLSPolicy with ECK CA cert.

---

### 503 Service Unavailable

**Causes:**
- No healthy pods available for the Service
- All pods failing readiness probes
- Service has no endpoints: kubectl get endpoints service-name -n namespace

---

### Connection timeout

**Causes:**
- NetworkPolicy blocking the connection
- Firewall rule (NSG) blocking the port
- Service ClusterIP routing issue
- Pod not listening on expected port

---

## PART 3 — ARGOCD ERRORS

---

### OutOfSync

**What it means:** Cluster state does not match git state.

**Not always an error** — can be intentional (manual change). Click Sync to apply git state.

**Causes:**
- Someone made a manual change in the cluster
- New commit in git not yet applied
- Resource drift (Azure changed something)

---

### Sync Failed

**Common causes and fixes:**

| Error | Cause | Fix |
|---|---|---|
| "error validating data" | Invalid YAML in manifest | Fix YAML syntax |
| "unknown field" | Wrong Kubernetes API field | Check correct API version |
| "namespace not found" | Namespace does not exist | Create namespace or add it to resources |
| "unable to replace" | Immutable field changed | Delete resource and recreate |
| "resource already exists" | Two apps managing same resource | Remove from one app |

---

### ComparisonError

**What it means:** ArgoCD cannot compare git state with cluster state.

**Causes:**
- Git repo unreachable (SSH key expired)
- Kustomize build failing — run kustomize build path/ locally to see error
- Invalid YAML

---

### Degraded

**What it means:** Application is synced but health checks are failing.

**Causes:**
- Pods are CrashLoopBackOff or Pending
- Deployment not reaching desired replica count
- Check pod status in the ArgoCD UI tree view

---

### Kustomize build error in ArgoCD

**Common messages:**

"accumulating resources: accumulation err='merging resources from..."
- Duplicate resource names in kustomization.yaml

"no matches for kind X in version Y"
- Wrong API version for the Kubernetes version

"json: cannot unmarshal"
- Malformed patch in kustomization.yaml

**How to debug locally:**
```
kustomize build overlays/uat/    <- shows exact error
kubectl apply -k overlays/uat/ --dry-run=client    <- validates against cluster
```

---

## PART 4 — EXTERNAL SECRETS ERRORS

---

### SecretSyncedError

**How to check:**
```
kubectl describe externalsecret name -n namespace
```

**Common causes:**

| Error | Cause | Fix |
|---|---|---|
| "could not find property" | Property name wrong in ExternalSecret | Check exact property name in Key Vault |
| "secret not found" | Key Vault secret name wrong | Check exact secret name in KV |
| "unauthorized" | Managed identity no access to KV | Grant Key Vault Secrets User role |
| "context deadline exceeded" | Network issue reaching Key Vault | Check private endpoint, DNS resolution |

---

### Secret not updating after Key Vault change

**Cause:** ExternalSecret has a refresh interval. By default it syncs every hour.

**Force resync:**
```
kubectl annotate externalsecret name force-sync=$(date +%s) -n namespace --overwrite
```

---

### K8s Secret exists but pod still has old value

**Cause:** Pod was not restarted after Secret updated.

**Fix:**
```
kubectl rollout restart deployment/name -n namespace
```

---

## PART 5 — KUSTOMIZE ERRORS

---

### "no matches for kind ... in version ..."

**Cause:** Resource uses an API version that does not exist in this Kubernetes version.
Example: networking.k8s.io/v1beta1 Ingress removed in K8s 1.22.

**Fix:** Update API version to the current one (e.g., networking.k8s.io/v1).

---

### "accumulating resources: read error"

**Cause:** A file listed in resources does not exist.

**Fix:** Check all files listed in kustomization.yaml actually exist. Check spelling.

---

### "field not found" in JSON patch

**Cause:** The path in your JSON patch does not exist in the base manifest.

**Example:** op: add to /spec/template/spec/volumes/- but base has no volumes array.

**Fix:** Check the exact structure of the base manifest. Add an empty volumes: [] first if needed.

---

### Patch not applying (no visible error but change not there)

**Cause:** Target selector in patch does not match any resource.

**Fix:** Verify the target kind, name, group, and version exactly match the resource.

---

### Hash suffix on ConfigMap name breaking references

**Cause:** ConfigMapGenerator adds a hash suffix by default. Other resources reference the old name.

**Fix:** Add to kustomization.yaml:
```yaml
generatorOptions:
  disableNameSuffixHash: true
```

---

### "json: cannot unmarshal string into Go value of type"

**Cause:** YAML value is a string but Kubernetes expects a different type (number, bool).

**Fix:** Check field type in Kubernetes API docs. Use quotes for string values: port: "6379"

---

## PART 6 — TERRAFORM ERRORS

---

### Error: "Error acquiring the state lock"

**Cause:** Another Terraform process is running and holds the state lock. Or a previous run crashed without releasing the lock.

**Fix:**
```
terraform force-unlock LOCK_ID
```
Find LOCK_ID in the error message. Only do this if you are sure no other process is running.

---

### Error: "RequestDisallowedByPolicy"

**Cause:** Azure Policy is blocking the resource operation.

**Real Fischer example:** NSG update blocked by policy "Management port access from Internet must be blocked" during AKS node reimage. Caused 1h40min outage.

**Fix:** Add a policy exemption for the resource group, or modify the resource to comply with the policy.

---

### Error: "PrivateIPAddressIsAllocated"

**Cause:** Azure Load Balancer frontend IP config is trying to use an IP already allocated elsewhere.

**Real Fischer example:** kubernetes-internal load balancer failing because two frontend IP configurations both claimed the same private IP 10.209.169.186.

**Fix:** Check which Service is causing the conflict. kubectl get svc -A — look for duplicate loadBalancerIP annotations.

---

### Error: "Provider produced inconsistent result after apply"

**Cause:** Terraform applied a change but the provider returned a different value than expected.

**Fix:** Usually a provider bug. Try terraform refresh and apply again. May need to taint and recreate the resource.

---

### Error: "Error: Invalid value for variable"

**Cause:** Variable validation rule failed.

**Fix:** Check the validation block in the variable definition. Provide a value that passes the condition.

---

### Error: "Backend configuration changed"

**Cause:** Backend config in code changed but terraform init not re-run.

**Fix:**
```
terraform init -reconfigure
```

---

### Error: "State file is locked"

**Cause:** State lock was not released (crashed Terraform process).

**Fix:** Check Azure Blob for the lock lease. Break the lease manually or use:
```
terraform force-unlock LOCK_ID
```

---

### Error: "Resource already exists"

**Cause:** A resource exists in Azure that Terraform is trying to create but is not in state.

**Fix:** Import the existing resource:
```
terraform import resource_type.name /azure/resource/id
```

---

### Error: "Cycle detected" 

**Cause:** Resource A depends on B and B depends on A — circular dependency.

**Fix:** Remove one of the dependencies. Use data sources to break the cycle.

---

### Plan shows resource will be destroyed and recreated

**Cause:** An immutable field was changed (like resource name, location, or certain AKS properties).

**Fix:** Either accept the destroy/recreate, or use terraform state mv to rename without recreating. Check the plan output — it shows which field is forcing the replacement.

---

### Error: "No valid credential sources found"

**Cause:** Terraform cannot authenticate to Azure. Client ID, secret, or tenant ID wrong.

**Fix:** Verify the service principal credentials. In Fischer: Atlantis injects these as env vars — check Atlantis variable configuration.

---

## PART 7 — DOCKER AND ACR ERRORS

---

### Error: "unauthorized: authentication required"

**Cause:** Not logged in to the registry.

**Fix:**
```
az acr login --name fischergroup
docker login fischergroup.azurecr.io
```

---

### Error: "denied: requested access to the resource is denied"

**Cause:** Logged in but the user/service principal does not have permission to push or pull.

**Fix:** Check ACR role assignment. Need AcrPull to pull, AcrPush to push.

---

### Error: "no space left on device" during docker build

**Cause:** Build node is out of disk space.

**Fix:** Prune unused images and containers:
```
docker system prune -af
```
Or increase disk size on the build node.

---

### Error: "image not found" when running docker push

**Cause:** Image was not built with the correct tag before pushing.

**Fix:** Run docker build -t registry/image:tag first, then push.

---

### Error: "manifest unknown" when pulling

**Cause:** The image tag does not exist in the registry.

**Fix:** Verify the exact tag exists: az acr repository show-tags --name fischergroup --repository image-name

---

### Error: "failed to solve: failed to read dockerfile"

**Cause:** Dockerfile not found in the build context.

**Fix:** Check -f flag or ensure Dockerfile exists in the directory where docker build is run.

---

## PART 8 — GIT ERRORS

---

### Error: "Updates were rejected because the tip of your current branch is behind"

**Cause:** Remote has commits your local branch does not have.

**Fix:**
```
git pull --rebase origin master
git push origin branch-name
```

Never push --force to shared branches.

---

### Error: "Your local changes would be overwritten by merge"

**Cause:** You have uncommitted changes and trying to pull or checkout.

**Fix:**
```
git stash          <- save your changes
git pull
git stash pop      <- restore your changes
```

---

### Error: "Merge conflict"

**Cause:** Two people changed the same lines in the same file.

**Fix:** Open the conflicting file, resolve conflicts (remove <<<, ===, >>> markers), then:
```
git add conflicting-file
git commit
```

---

### Error: "refusing to merge unrelated histories"

**Cause:** Two repos with no common commit trying to merge.

**Fix:**
```
git pull origin master --allow-unrelated-histories
```

---

### Error: "remote: Repository not found"

**Cause:** Wrong remote URL or no access to the repository.

**Fix:** Check remote URL: git remote -v. Verify SSH key is added to Bitbucket.

---

### Error: "fatal: not a git repository"

**Cause:** Running git command outside a git repo.

**Fix:** Navigate to the correct directory. Or git init if starting a new repo.

---

## PART 9 — AZURE / AKS ERRORS

---

### Error: "node.kubernetes.io/unreachable" taint

**What it means:** Node is not sending heartbeats to the API server.

**Causes:**
- Node VM is down or rebooting
- Network disruption between node and API server
- konnectivity-agent crashed
- Azure maintenance or reimage

**Real Fischer example:** AKS auto-reimage during node maintenance. NSG policy blocked network reconfiguration. 6 nodes simultaneously went unreachable causing full outage.

---

### Error: AKS upgrade fails halfway

**Cause:** Node pool cannot upgrade due to policy, quota, or resource constraints.

**Fix:** Check Azure Activity Log for the exact error. Common issues: NSG policy blocking, insufficient quota for surge nodes, PodDisruptionBudget blocking node drain.

---

### Error: "QuotaExceeded"

**Cause:** Azure subscription vCPU quota exceeded when trying to add nodes.

**Fix:** Request quota increase in Azure portal or reduce node VM size.

---

### Error: "InsufficientCapacity"

**Cause:** Azure does not have the requested VM SKU available in the region.

**Fix:** Try a different region or different VM SKU.

---

### Error: "NodeNotReady" in kubectl

**Cause:** Node has stopped reporting to the API server.

**Check:**
```
kubectl describe node node-name    <- see conditions and events
kubectl get events -n kube-system | grep NodeNotReady
```

---

### Error: Key Vault access denied

**Cause:** Managed identity does not have permission to read the Key Vault secret.

**Fix:** Grant the managed identity the "Key Vault Secrets User" role on the Key Vault or specific secret.

---

### Error: "PrivateLinkServiceNetworkPoliciesDisabled"

**Cause:** Private endpoint subnet has network policies enabled.

**Fix:** Disable network policies on the subnet for private link services.

---

## PART 10 — JENKINS / CI ERRORS

---

### Error: "permission denied" in Jenkins pipeline

**Cause:** The K8s pod running Jenkins agent does not have permission to execute a command.

**Fix:** Check serviceAccount permissions. Add RBAC role binding for the Jenkins service account.

---

### Error: "container is not running" in Jenkins pod

**Cause:** A container in the Jenkins build pod crashed.

**Fix:** Check Jenkins pod logs: kubectl logs pod-name -c container-name -n jenkins

---

### Error: "No space left on device" in Jenkins

**Cause:** Jenkins home directory or workspace disk is full.

**Fix:** Clean up old builds. Increase PVC size. Add disk cleanup plugin.

---

### Error: "docker: Cannot connect to the Docker daemon"

**Cause:** Jenkins agent cannot reach the Docker-in-Docker (dind) sidecar container.

**Fix:** Verify dind container is running in the pod. Check DOCKER_HOST environment variable is set to tcp://localhost:2376.

---

### Error: "mvn: command not found"

**Cause:** Wrong container being used — Maven not installed.

**Fix:** Ensure the pipeline step runs inside the correct container (java21 not jnlp).

---

### Trivy scan fails but image is fine

**Cause:** Trivy database could not be downloaded (network issue) or vuln threshold set too low.

**Fix:** Check trivyIgnoreFailures flag. Add exception for known false positives. Ensure Trivy can reach the internet for vulnerability DB updates.

---

## PART 11 — HELM ERRORS

---

### Error: "release already exists"

**Cause:** Trying to helm install a release that already exists.

**Fix:** Use helm upgrade --install instead of helm install (idempotent).

---

### Error: "rendered manifests contain a resource that already exists"

**Cause:** A resource created by this Helm chart exists in the cluster but is not in the Helm release state.

**Fix:**
```
helm upgrade release-name chart --force
```
Or import the existing resource into Helm state.

---

### Error: "chart not found"

**Cause:** Wrong chart name or repo not added.

**Fix:** helm repo add and helm repo update first. Check chart name spelling.

---

### Error: "values.yaml validation failed"

**Cause:** Value provided does not match the schema defined in values.schema.json.

**Fix:** Check the schema and provide a valid value.

---

## PART 12 — COMMON INTERVIEW QUESTIONS ON ERRORS

---

**Q: A pod keeps restarting. What is your step-by-step debugging process?**

1. kubectl get pods -n namespace — confirm it is CrashLoopBackOff
2. kubectl describe pod pod-name — check Events section for clues
3. kubectl logs pod-name --previous — check logs from crashed container
4. Check if all Secrets and ConfigMaps exist: kubectl get secret, kubectl get configmap
5. Check if initContainers passed: kubectl logs pod-name -c init-container-name
6. Check resource limits — is it OOMKilled?
7. Check readiness/liveness probe — is it failing too early?

---

**Q: A deployment rolled out but application is returning 500 errors. How do you investigate?**

1. Check if the right image is running: kubectl get pod -o jsonpath="{.spec.containers[0].image}"
2. Check application logs: kubectl logs pod-name -n namespace
3. Check if env vars are correct: kubectl exec pod -- env
4. Check if the service is reachable: kubectl exec pod -- curl localhost:8080/health
5. Check if dependencies (DB, Redis, other services) are reachable from inside the pod
6. Compare with the previous working version — what changed?

---

**Q: ArgoCD shows Degraded. What do you do?**

1. Open ArgoCD UI — check which resources are red
2. Click on the failing resource — see the error
3. kubectl describe pod — check pod events
4. Fix the root cause (image issue, secret missing, etc.)
5. ArgoCD auto-heals once the root cause is fixed

---

**Q: ExternalSecret shows SecretSyncedError. How do you fix it?**

1. kubectl describe externalsecret name -n namespace — read the error message
2. Verify the Key Vault secret name is exactly correct
3. Verify the property name inside the KV secret is correct
4. Verify the managed identity has Key Vault Secrets User role
5. Force resync: kubectl annotate externalsecret name force-sync=$(date +%s) --overwrite

---

**Q: Terraform plan shows a resource will be destroyed. How do you prevent accidental destruction?**

1. Read the plan carefully — understand WHY it will be destroyed
2. If it is wrong: check if someone changed an immutable field
3. Add lifecycle { prevent_destroy = true } to critical resources
4. Use terraform plan -target=specific_resource to check just that resource
5. Never run terraform apply without reviewing the plan

---

**Q: A node is NotReady. What do you check?**

1. kubectl describe node node-name — check Conditions section
2. Check which pods are on that node: kubectl get pods -A --field-selector spec.nodeName=node-name
3. Check node events: kubectl get events -n kube-system | grep node-name
4. Check Azure portal — is the underlying VM healthy?
5. Check azure-ip-masq-agent and konnectivity-agent status on that node
6. If VM is healthy but node is not: try SSH to node (from JumpVM) and check kubelet status

---

**Q: You pushed a new image tag but ArgoCD is not deploying it. Why?**

1. ArgoCD may not have detected the commit yet — force refresh
2. Auto-sync might be disabled — check application settings
3. There might be a sync error — check application status
4. The kustomization.yaml change might not have been pushed to the correct branch
5. ArgoCD might be watching a different branch — check repoURL and targetRevision in ArgoCD app

---

**Q: How do you handle a situation where a Kubernetes Secret needs to be rotated?**

1. Update the value in Azure Key Vault
2. Force ExternalSecret to resync: kubectl annotate externalsecret name force-sync=$(date +%s)
3. Verify the K8s Secret was updated: kubectl get secret name -o yaml
4. Restart the deployment to pick up the new value: kubectl rollout restart deployment/name
5. Verify the application is working with the new secret

---

**Q: A rolling update is stuck and not completing. Why?**

1. New pods are failing readiness probe — they never become Ready
2. Kubernetes will not remove old pods until new ones are Ready
3. Check new pods: kubectl describe pod new-pod-name
4. Fix the readiness probe issue or fix the application
5. If you need to rollback: kubectl rollout undo deployment/name
