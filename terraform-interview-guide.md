# Terraform Guide — Fischer Setup + 150 Interview Questions with Answers

---

## PART 1 — HOW TERRAFORM IS STRUCTURED IN FISCHER

---

### Folder Structure

```
fischer-operations/
  atlantis.yaml                  <- controls which module runs with which workspace
  terraform/
    defaults.tf                  <- shared tags and location used across all modules
    provider.tf                  <- shared provider config
    kubernetes/
      main.tf                    <- AKS cluster creation
      variables.tf               <- all variables as maps (one value per environment)
      backend.hcl                <- statefile key for Azure Blob Storage
    keyvault/
      main.tf                    <- Key Vault creation
      backend.hcl
    frontdoor/
      main.tf                    <- Azure Frontdoor, WAF, domains, routes
      backend.hcl
    dns/
      main.tf                    <- DNS records
      modules/dns/               <- reusable DNS module
      backend.hcl
    checkly/
      main.tf                    <- Checkly monitoring checks
    allquiet/
    mssql/
    network/
    storage/
```

---

### Backend — Where State is Stored

Every module has this inside main.tf:

```hcl
backend "azurerm" {
  storage_account_name = "ecomtfstatestorage"
  container_name       = "tfstate"
  # key is NOT here — comes from backend.hcl
}
```

And a separate backend.hcl file:
```
key = "prod.terraform.tfstate/kubernetes"
```

Atlantis passes it at init:
```bash
terraform init -backend-config=backend.hcl
```

This is called Partial Backend Configuration.
State lives in Azure Blob Storage — never locally, never in git.

---

### Workspaces

| Module     | Workspaces                   |
|------------|------------------------------|
| kubernetes | prod-2, nonprod-2, service-1 |
| keyvault   | prod, nonprod, service       |
| frontdoor  | default                      |
| dns        | default                      |
| checkly    | default                      |

Atlantis defines workspace in atlantis.yaml:
```yaml
- name: kubernetes-prod-2
  dir: terraform/kubernetes
  workspace: prod-2
```

Each workspace gets its own statefile:
```
env:/prod-2/prod.terraform.tfstate/kubernetes
env:/nonprod-2/prod.terraform.tfstate/kubernetes
```

---

### How Variables Are Handled

Simple variable:
```hcl
variable "location" {
  default = "westeurope"
}
```

Map variable (different value per environment):
```hcl
variable "node_count" {
  default = {
    "prod-2"    = 6
    "nonprod-2" = 6
    "service-1" = 3
  }
}
```

Picking the right value using lookup():
```hcl
locals {
  node_count = lookup(var.node_count, terraform.workspace, 0)
}
```

workspace = prod-2  -> node_count = 6
workspace = service-1 -> node_count = 3

---

### defaults.tf — Shared Values

```hcl
locals {
  default_tags = {
    "managed-by" = "terraform"
  }
  default_location = "westeurope"
}
```

---

### count — Conditional Resource

```hcl
resource "azurerm_key_vault_access_policy" "frontdoor-policy" {
  count = (startswith(local.stage, "prod")) ? 1 : 0
}
```
count = 1 -> create | count = 0 -> skip

---

### for_each — Multiple Resources from Set/Map

```hcl
resource "azurerm_key_vault_access_policy" "admin-policy" {
  for_each  = var.admin_oids
  object_id = each.key
}
```
Creates one access policy per admin OID.

---

### dynamic block — Conditional Block Inside Resource

```hcl
dynamic "windows_profile" {
  for_each = local.is_service_workspace ? [] : [1]
  content {
    admin_username = "azureuser"
  }
}
```
for_each = [] -> block skipped | for_each = [1] -> block included

---

### data — Read Existing Resource

```hcl
data "azurerm_private_dns_zone" "ecom_private_ugfischer_com" {
  name                = "ecom-private-ugfischer.com"
  resource_group_name = local.rg-network
}
```
Reads existing resource — does not create anything.

---

### lifecycle block

```hcl
lifecycle {
  precondition {
    condition     = terraform.workspace != "default"
    error_message = "Workspace default not allowed"
  }
  prevent_destroy      = true
  create_before_destroy = true
  ignore_changes       = [tags]
}
```

---

### Atlantis Flow

1. Raise PR in fischer-operations
2. Atlantis auto-runs terraform plan for all changed modules
3. Plan posted as PR comment
4. Comment: atlantis apply -p kubernetes-prod-2
5. Atlantis applies
6. PR merged to main

---

## PART 2 — 150 INTERVIEW QUESTIONS WITH ANSWERS

---

### BASICS

**1. What is Terraform and what problem does it solve?**
Terraform is an open-source Infrastructure as Code tool by HashiCorp. It lets you define infrastructure (VMs, networks, databases) in code files and provision it automatically. It solves the problem of manual, inconsistent, and non-repeatable infrastructure setup.

**2. What is Infrastructure as Code and why is it better than clicking in the portal?**
IaC means writing infrastructure configuration in code files instead of manually clicking in a portal. Benefits: version controlled, repeatable, reviewable, auditable, and can be automated.

**3. What is the difference between Terraform and Ansible?**
Terraform is for provisioning infrastructure (create VMs, networks, databases). Ansible is for configuring software on existing infrastructure (install packages, configure services). Terraform is declarative. Ansible is procedural. They complement each other.

**4. What is the difference between Terraform and ARM templates?**
Both provision Azure resources. Terraform is cloud-agnostic (works with AWS, GCP, Azure). ARM templates are Azure-only. Terraform has better readability, state management, and plan/preview features.

**5. What are the main Terraform commands?**
- terraform init — download providers, set up backend
- terraform plan — show what will change
- terraform apply — make the changes
- terraform destroy — delete all managed resources
- terraform fmt — format code
- terraform validate — check syntax
- terraform state — manage state

**6. What does terraform init do?**
Downloads the required provider plugins (e.g. azurerm), initializes the backend (connects to remote state storage), and prepares the working directory. Must run before plan or apply.

**7. What does terraform plan do?**
Reads current state, queries Azure for actual current state of resources, compares with your code, and shows exactly what will be created, modified, or destroyed — without making any changes.

**8. What does terraform apply do?**
Executes the plan — makes the actual changes in Azure. Creates, updates, or deletes resources as needed. Updates the statefile after each resource operation.

**9. What does terraform destroy do?**
Destroys all resources managed by the current Terraform configuration. Used for cleanup of environments. It is destructive — always review before running.

**10. What is terraform fmt?**
Formats Terraform code to follow the canonical style. Fixes indentation, alignment. Should be run before committing code.

**11. What is terraform validate?**
Checks the syntax and structure of your Terraform files without connecting to any provider. Catches typos, missing required fields, incorrect block structure. Does not check if resources actually exist in Azure.

**12. What is terraform refresh?**
Updates the local statefile to match the real state of infrastructure in Azure. Used when someone has made manual changes in the portal. In newer Terraform versions this is done automatically during plan.

**13. What is a provider in Terraform?**
A plugin that allows Terraform to interact with a specific cloud or service. For example azurerm is the Azure provider, aws is for AWS, checkly is for Checkly monitoring. Providers are declared in the required_providers block and downloaded during terraform init.

**14. What is the Terraform Registry?**
registry.terraform.io — the public repository of providers and modules. You can find official providers (by HashiCorp), verified partners (like Azure, AWS), and community modules here.

**15. What is HCL?**
HashiCorp Configuration Language — the language Terraform code is written in. It is declarative, human-readable, and designed to describe the desired state of infrastructure.

---

### BLOCKS

**16. What are all the block types in Terraform?**
- terraform — configuration for Terraform itself (backend, required_providers, required_version)
- provider — configure a provider (credentials, region)
- resource — create/manage a resource
- data — read an existing resource
- variable — declare an input variable
- locals — declare local computed values
- output — expose values after apply
- module — call a reusable module
- lifecycle — control resource lifecycle behavior (inside resource)
- dynamic — generate repeated nested blocks (inside resource)
- provisioner — run scripts on resources (generally avoided)
- connection — define how to connect to a resource for provisioners

**17. What is a resource block?**
Declares a piece of infrastructure to be created and managed by Terraform. Format: resource "provider_type" "local_name" {}. Example: resource "azurerm_resource_group" "my_rg" {}.

**18. What is a data block?**
Reads an existing resource from the provider — does not create anything. Used to fetch IDs, names, or properties of resources that already exist or are managed elsewhere. Example: data "azurerm_key_vault" "existing" {}.

**19. What is a variable block?**
Declares an input that can be passed to the module. Can have type, description, default, validation, and sensitive fields. If no default is set, Terraform requires the value to be provided at runtime.

**20. What is a locals block?**
Defines computed values that are reused within a module. Unlike variables, locals cannot be overridden from outside — they are internal to the module. Good for complex expressions you do not want to repeat.

**21. What is an output block?**
Exposes a value after terraform apply. Used to share resource attributes (like IPs or IDs) with other modules or display useful information after apply.

**22. What is a module block?**
Calls a reusable module (a folder of Terraform files). Passes inputs to it via arguments. Example: module "dns" { source = "./modules/dns" records = var.records }.

**23. What is the terraform block?**
The top-level configuration block for Terraform itself. Used to set: required_version, required_providers, and backend. Example in Fischer: sets azurerm provider version and azurerm backend.

**24. What is a provider block?**
Configures a specific provider with credentials and settings. Example: provider "azurerm" { subscription_id = var.subscription_id client_id = var.client_id }. Must be configured before Terraform can create resources with that provider.

**25. What is required_providers block?**
Nested inside the terraform block. Declares which providers the module needs and their versions. Terraform downloads them during init. Example: azurerm = { source = "hashicorp/azurerm" version = "~> 4.18.0" }.

**26. What is the backend block?**
Nested inside the terraform block. Tells Terraform where to store the statefile. In Fischer it uses azurerm backend pointing to Azure Blob Storage.

**27. What is a lifecycle block?**
Nested inside a resource block. Controls how Terraform handles resource creation, update, and deletion. Options: prevent_destroy, create_before_destroy, ignore_changes, precondition, postcondition.

**28. What is a dynamic block?**
Generates repeated nested blocks inside a resource dynamically, based on a collection. Used when you do not know how many nested blocks are needed at write time. Example: dynamic "windows_profile" that is only included for certain workspaces.

**29. What is a provisioner block and why is it avoided?**
Runs scripts on a resource after creation (local-exec or remote-exec). Generally avoided because: failures are hard to handle, not idempotent, breaks the declarative model. Prefer cloud-init, Ansible, or custom scripts via other tools.

**30. What is a connection block?**
Defines how Terraform connects to a resource to run provisioners. Specifies SSH or WinRM credentials. Used with remote-exec provisioner.

**31. What is a precondition block?**
Inside lifecycle. Validates a condition before creating/modifying a resource. If condition is false, Terraform throws an error and stops. Example in Fischer keyvault: prevents running in default workspace.

**32. What is a postcondition block?**
Inside lifecycle. Validates a condition after creating/modifying a resource. If condition is false, Terraform marks the apply as failed. Used to validate that a resource was created with correct properties.

**33. Can you have multiple provider blocks for the same provider?**
Yes, using aliases. Example: two azurerm providers with different subscription IDs. Referenced in resources as provider = azurerm.secondary.

**34. What is the difference between locals and variable?**
Variables are inputs — they can be passed from outside the module. Locals are internal computed values — they cannot be passed from outside. Use variables for things callers should control, use locals for internal calculations.

**35. Can a data block reference a resource block output?**
Yes. Example: create a resource group with resource block, then use data block to read it back. But normally you just reference the resource directly — data is used when the resource is outside this module's management.

---

### VARIABLES

**36. What are the primitive types in Terraform?**
string, number, bool.

**37. What are the complex types in Terraform?**
list(type), set(type), map(type), object({ key = type }), tuple([types]).

**38. What is a map type variable?**
A collection of key-value pairs where all values are the same type. Example: map(string) = { "prod" = "eastus" "nonprod" = "westus" }. Access with var.locations["prod"] or lookup(var.locations, "prod", "default").

**39. What is a list vs set?**
List: ordered, allows duplicates, accessed by index. Set: unordered, no duplicates, cannot access by index. for_each requires set or map, not list — so convert lists with toset().

**40. What is a set type?**
An unordered collection of unique values. Used with for_each to create multiple resources. Example: set(string) of admin OIDs in Fischer keyvault.

**41. What is an object type?**
A collection of named attributes with potentially different types. Example: object({ name = string, count = number, enabled = bool }). Useful for grouping related configuration.

**42. What is a tuple type?**
An ordered collection of values with potentially different types. Example: tuple([string, number, bool]). Rarely used in practice.

**43. How do you pass variable values — all ways in order of precedence (highest to lowest)?**
1. -var flag on command line
2. -var-file flag on command line
3. terraform.tfvars file (auto-loaded)
4. terraform.tfvars.json (auto-loaded)
5. *.auto.tfvars files (auto-loaded)
6. TF_VAR_ environment variables
7. Default value in variable block

**44. What is a tfvars file?**
A file containing variable values. Example: location = "westeurope" node_count = 6. Automatically loaded if named terraform.tfvars. Otherwise pass with -var-file=prod.tfvars.

**45. What is TF_VAR_ environment variable?**
Setting TF_VAR_subscription_id=abc in the shell passes "abc" as the value for variable "subscription_id". Used by Atlantis to inject secrets without putting them in files.

**46. How do you mark a variable as sensitive?**
Add sensitive = true to the variable block. Terraform will not show its value in plan or apply output. Still stored in state — state should also be protected.

**47. What happens if a required variable has no value?**
Terraform prompts interactively for the value when running locally. In Atlantis/CI it fails with an error. Always provide defaults or pass values via tfvars/env vars in automation.

**48. What is the difference between var.something and local.something?**
var.something reads an input variable — can be overridden from outside. local.something reads a locally computed value — only available within the current module, cannot be overridden.

**49. Can a variable reference another variable?**
No. Variables cannot reference other variables or resources. Use locals for computed values that depend on other values.

**50. What is any type?**
A special type that accepts any value. Terraform infers the actual type from the value provided. Use sparingly — specific types are better for validation.

---

### MAPS

**51. What is a map in Terraform?**
A collection of key-value pairs where all values are the same type. Like a lookup table or dictionary. Keys are strings.

**52. How do you define a map variable with defaults?**
```hcl
variable "node_count" {
  type = map(number)
  default = {
    "prod-2"    = 6
    "nonprod-2" = 6
    "service-1" = 3
  }
}
```

**53. How do you use lookup() to read from a map?**
lookup(var.node_count, "prod-2", 0) — looks for key "prod-2" in the map, returns its value or 0 if not found.

**54. What is the third argument of lookup() and why is it important?**
The default value returned if the key does not exist in the map. Without it, Terraform throws an error if the key is missing. Important for safety when workspace names might not all be in the map.

**55. How do you create an inline map using tomap()?**
tomap({ "prod" = "value1" "nonprod" = "value2" }) — converts a literal object to a map type. Used in Fischer dns module to pass a_records inline.

**56. How do you merge two maps?**
merge(map1, map2) — combines two maps. If both have the same key, the second map's value wins.

**57. How do you get all keys from a map?**
keys(var.my_map) — returns a sorted list of all keys.

**58. How do you get all values from a map?**
values(var.my_map) — returns a list of values in key-sorted order.

**59. How do you create a map from two lists using zipmap()?**
zipmap(["a","b"], [1,2]) returns { "a" = 1 "b" = 2 }. Keys list and values list must be the same length.

**60. How are maps used for multi-environment in Fischer?**
Variables are defined as maps with workspace names as keys. locals block uses lookup(var.something, terraform.workspace, default) to pick the right value for the current workspace. Same code, different values per environment.

---

### STATE FILE

**61. What is a Terraform statefile?**
A JSON file (terraform.tfstate) that records the current known state of all resources managed by Terraform. Maps your code's resource names to actual cloud resource IDs. Terraform uses it to calculate what needs to change on next apply.

**62. Why should you never store the statefile in git?**
It contains sensitive information (passwords, keys, IDs). It changes on every apply — causes constant merge conflicts. Not designed for version control.

**63. What is the problem with storing statefile locally in a team?**
Each team member has their own local state. Two people can apply conflicting changes. No locking — two people can apply at the same time and corrupt state. No shared source of truth.

**64. What is a remote backend and what are its benefits?**
Stores state in a shared remote location (Azure Blob, S3, Terraform Cloud). Benefits: shared state across team, state locking to prevent concurrent applies, encrypted at rest, versioned.

**65. What backends have you used?**
In Fischer: azurerm backend (Azure Blob Storage). Others common in industry: S3 (AWS), GCS (Google), Terraform Cloud.

**66. What is state locking?**
When Terraform runs plan or apply, it acquires a lock on the statefile. Other Terraform processes cannot run simultaneously. Prevents corruption from concurrent applies. Azure Blob Storage uses blob leases for locking.

**67. How does Azure Blob Storage handle state locking?**
Uses Azure Blob lease mechanism. When Terraform acquires the lock it creates a lease on the blob. Other processes trying to acquire the lease will fail with a lock error. Lock is released after apply completes.

**68. What happens if two people run terraform apply at the same time?**
The second one fails with a state lock error. The first one proceeds. After it completes and releases the lock, the second can retry.

**69. How do you view current state?**
terraform state list — lists all resources in state.
terraform state show resource_type.name — shows details of one resource.

**70. How do you remove a resource from state without destroying it?**
terraform state rm resource_type.name — removes the resource from state but leaves the actual cloud resource running. Used when you want to stop managing a resource with Terraform without deleting it.

**71. What is terraform import?**
Imports an existing cloud resource into Terraform state. You write the resource block first, then run terraform import resource_type.name /azure/resource/id. After import, Terraform manages it.

**72. What do you do if statefile is corrupted or deleted?**
If backup exists in Azure (versioning enabled), restore previous version from blob storage. If no backup, use terraform import to re-import all resources one by one. This is why remote backends with versioning are essential.

---

### BACKEND

**73. What is a backend in Terraform?**
Defines where Terraform stores its statefile and how operations are executed. Two types: local (default, stores state locally) and remote (stores state in a shared system).

**74. What is partial backend configuration?**
Splitting backend config across two files. The backend block in main.tf has storage_account_name and container_name but no key. The key comes from a separate backend.hcl file. Passed to init via: terraform init -backend-config=backend.hcl.

**75. Why would you split backend config?**
Security: different keys per module without exposing them in main.tf. Flexibility: same main.tf code, different backend.hcl per environment. Fischer uses this so each module has its own unique statefile key.

**76. What is the key in Azure backend?**
The path/name of the statefile blob inside the container. Example: prod.terraform.tfstate/kubernetes. When combined with workspace, becomes: env:/prod-2/prod.terraform.tfstate/kubernetes.

**77. What happens to statefile path when you use workspaces?**
Terraform automatically prefixes the key with env:/workspace-name/. So key = "prod.terraform.tfstate/kubernetes" with workspace prod-2 becomes env:/prod-2/prod.terraform.tfstate/kubernetes.

**78. Can you change the backend after terraform init?**
Yes but you need to run terraform init again. Terraform will ask if you want to copy existing state to the new backend. Always do this — do not lose your state.

**79. What is the difference between local and remote backend?**
Local: state stored in terraform.tfstate file on your machine. Simple but not suitable for teams. Remote: state stored in a shared system (Azure Blob, S3). Supports locking, sharing, versioning.

**80. How is backend configured in Fischer?**
Each module has backend "azurerm" block in main.tf with storage account and container. The statefile key is in backend.hcl. Atlantis passes backend.hcl at init. State is in Azure Blob Storage account ecomtfstatestorage.

---

### WORKSPACES

**81. What is a Terraform workspace?**
A named instance of state within a single backend configuration. Allows you to use the same code with different statefiles for different environments.

**82. What is the default workspace?**
Called "default". This is the workspace Terraform uses if you never create or switch to another one.

**83. How do you create and switch workspaces?**
terraform workspace new prod-2 — creates a new workspace.
terraform workspace select prod-2 — switches to it.
terraform workspace list — shows all workspaces.
terraform workspace show — shows current workspace.

**84. What is terraform.workspace?**
A built-in variable that holds the name of the currently selected workspace. Used in code to make decisions: lookup(var.node_count, terraform.workspace, 0). Atlantis sets the workspace before running, so this variable has the correct environment value.

**85. How does statefile path change per workspace?**
Terraform automatically creates: env:/workspace-name/original-key. So env:/prod-2/prod.terraform.tfstate/kubernetes and env:/nonprod-2/prod.terraform.tfstate/kubernetes are completely separate statefiles.

**86. Workspaces vs separate directories for environments?**
Workspaces: same code, shared backend, different state. Good when environments are nearly identical (different sizes, counts). Separate directories: completely isolated, different code possible. Good when environments differ significantly. Fischer uses workspaces for kubernetes and keyvault because structure is identical — only values differ.

**87. What are the limitations of Terraform workspaces?**
All workspaces share the same code — cannot have fundamentally different configurations. Easy to apply to wrong workspace accidentally. Not suitable when environments have very different infrastructure shapes. No access control per workspace in open-source Terraform.

**88. How does Atlantis set the workspace?**
Atlantis reads the workspace field from atlantis.yaml for each project. Before running terraform plan or apply, it runs terraform workspace select workspace-name. Then terraform.workspace has the correct value during the run.

---

### COUNT

**89. What is count in Terraform?**
A meta-argument on resource or module blocks. Tells Terraform how many instances to create. count = 3 creates 3 identical resources.

**90. How do you conditionally create a resource using count?**
count = condition ? 1 : 0. If condition is true, count=1 creates the resource. If false, count=0 skips it. Example: count = startswith(local.stage, "prod") ? 1 : 0.

**91. What is count.index?**
The zero-based index of the current instance when count > 1. Used to make each instance unique. Example: name = "server-${count.index}" creates server-0, server-1, server-2.

**92. What is the main problem with count for named resources?**
If you have count=3 with a list ["a","b","c"] and remove "b", Terraform renumbers the indexes. "c" becomes index 1 instead of 2 and Terraform thinks it needs to destroy and recreate "c". Use for_each instead when resources have names.

**93. What is for_each in Terraform?**
A meta-argument that creates one resource instance per item in a map or set. Each instance is identified by its key, not an index. Changes to one item do not affect others.

**94. What types can you use with for_each?**
map(any) or set(string). Cannot use a list directly — convert with toset() first.

**95. What is each.key and each.value?**
Inside a for_each resource: each.key is the map key or set value. each.value is the map value (for maps). Example: for_each = { "prod" = "large" } gives each.key = "prod", each.value = "large".

**96. Advantage of for_each over count?**
Resources are identified by key not index. Removing one item does not affect others. State entries are named resource_type.name["key"] instead of resource_type.name[0] — much clearer.

**97. What happens when you remove an item from count list vs for_each?**
count: Terraform re-indexes. Remaining resources may be destroyed and recreated to fill gaps.
for_each: Only the removed item is destroyed. All others are untouched.

**98. What is a for expression?**
A way to transform or filter collections inline. Example: [for s in var.names : upper(s)] — creates a new list with all names uppercased. Similar to list comprehension in Python.

**99. How do you filter a list with a for expression?**
[for s in var.names : s if length(s) > 3] — returns only names longer than 3 characters.

**100. Difference between for expression and for_each?**
for expression: transforms/filters a collection into a new value. Used in variable assignments.
for_each: creates multiple resource instances from a collection. Used as a resource meta-argument.

**101. What is a dynamic block?**
Generates multiple nested configuration blocks inside a resource dynamically. Used when the number of nested blocks is not known at write time or depends on a condition. Example: dynamic "windows_profile" that appears only in non-service workspaces.

**102. How does for_each inside dynamic block work?**
for_each on dynamic block defines how many times the nested block appears. for_each = [] means zero times (block skipped). for_each = [1] means once (block included). for_each = ["a","b"] means twice. Inside content block, use each.value to access the current item.

---

### CONDITIONS AND FUNCTIONS

**103. What is the ternary operator in Terraform?**
condition ? value_if_true : value_if_false. Example: count = local.is_prod ? 1 : 0. If is_prod is true, count=1. If false, count=0.

**104. What is startswith()?**
Returns true if a string starts with a given prefix. Example: startswith(terraform.workspace, "prod") returns true for "prod-2", "prod-1". Used in Fischer to detect prod workspaces without hardcoding each name.

**105. What is can() function?**
Evaluates an expression and returns true if it succeeds, false if it throws an error. Used with regex: can(regex("^service-", terraform.workspace)) returns true if workspace starts with "service-". Avoids errors from failed expressions.

**106. What is coalesce()?**
Returns the first non-null, non-empty value from a list of arguments. Example: coalesce(var.custom_name, local.default_name) — uses custom_name if set, otherwise default_name.

**107. What is concat()?**
Combines multiple lists into one. Example: concat(var.default_ips, var.extra_ips) — merges both IP lists. Used in Fischer to combine default API server IP ranges with nonprod-specific ones.

**108. What is merge() for maps?**
Combines multiple maps into one. If duplicate keys exist, last one wins. Example: merge(local.default_tags, { "env" = "prod" }) adds env tag to default tags.

**109. What is flatten()?**
Converts a list of lists into a single flat list. Example: flatten([["a","b"],["c"]]) returns ["a","b","c"]. Useful when for expressions produce nested lists.

**110. What is toset() and why use it for for_each?**
Converts a list to a set (removes duplicates, loses order). for_each requires a set or map — not a list. Use toset(var.my_list) to convert before using in for_each.

**111. What is substr() used for?**
Extracts a substring. substr(string, offset, length). Example in Fischer: substr(terraform.workspace, 0, length(terraform.workspace) - 2) removes the last 2 characters to derive the stage name from workspace name (prod-2 -> prod).

**112. What is format() used for?**
Creates a formatted string. Similar to printf. Example: format("fischer-%s-k8s", terraform.workspace) produces "fischer-prod-2-k8s". Used for consistent resource naming.

**113. What is trim() used for?**
Removes specified characters from the start and end of a string. Example in Fischer: trim(var.client_id, "\"") removes surrounding quotes. Used to clean up values injected with extra quotes by Atlantis.

**114. What is file() function?**
Reads the contents of a file at plan time and returns it as a string. Example in Fischer checkly: file("scripts/browser-check.spec.ts") loads the TypeScript script content into a local variable for use in the monitoring check resource.

**115. What is can(regex()) used for?**
Checks if a string matches a regex pattern without throwing an error. Example: can(regex("^service-", terraform.workspace)) returns true if workspace starts with "service-". Used in Fischer to set is_service_workspace boolean in locals.

---

### DEPENDENCIES

**116. What is implicit dependency?**
When one resource references another resource's attribute in its configuration, Terraform automatically knows it must create the referenced resource first. Example: azurerm_key_vault_access_policy referencing azurerm_key_vault.keyvault.id — Terraform creates the key vault first automatically.

**117. What is explicit dependency and when do you need it?**
Using depends_on = [resource_type.name] to tell Terraform one resource must be created after another, even when there is no direct attribute reference. Needed when there is a logical dependency that Terraform cannot detect from the code alone.

**118. What is the Terraform execution graph?**
Terraform builds a directed acyclic graph (DAG) of all resources and their dependencies. It uses this to determine the order of operations and which resources can be created in parallel.

**119. Can two resources be created in parallel?**
Yes. Resources with no dependency between them are created in parallel by Terraform. This speeds up large applies significantly.

**120. What happens if a dependency resource fails during apply?**
Terraform marks the failed resource as tainted and skips all resources that depend on it. The rest of the apply continues for unrelated resources. On next apply, it retries the failed resource.

**121. When would you use depends_on with a module?**
When a module creates resources that another module depends on, but that dependency is not expressed through output/input references. Example: module A creates a network, module B deploys VMs — if B does not reference A's outputs directly, you need depends_on = [module.A].

**122. What is the risk of overusing depends_on?**
Creates unnecessary sequential dependencies. Resources that could run in parallel are forced to run one after another. Slows down applies significantly on large configurations.

---

### MODULES

**123. What is a Terraform module?**
Any directory containing Terraform files is a module. Modules are used to package and reuse infrastructure code. You call them with a module block and pass inputs. They return outputs.

**124. What is root module vs child module?**
Root module: the directory where you run terraform plan/apply. Child module: a module called from within the root module via a module block. Child modules can call other child modules.

**125. How do you pass inputs to a module?**
As arguments in the module block: module "dns" { source = "./modules/dns" zone_name = "example.com" records = var.records }. These map to variable blocks defined inside the module.

**126. How do you get outputs from a module?**
The module must define output blocks. You reference them in the calling module as: module.module_name.output_name. Example: module.kubernetes.cluster_id.

**127. What is source in a module block?**
Tells Terraform where to find the module code. Can be: local path (./modules/dns), Terraform Registry (hashicorp/consul/aws), Git URL, or HTTP URL.

**128. How do you version-lock a module from the Registry?**
Add version = "~> 2.0" in the module block. This pins to 2.x versions. Always version-lock modules in production to prevent unexpected changes.

**129. When should you create a module vs write resources directly?**
Create a module when: the same pattern is repeated in multiple places, you want to enforce standards, or the configuration is complex enough to benefit from abstraction. Do not create a module for one-off use — it adds complexity without benefit.

**130. How is the DNS module used in Fischer?**
The module is in terraform/dns/modules/dns. The main.tf calls it with source = "./modules/dns" and passes zone_name, zone_rg, a_records, and cname_records. The module creates the actual DNS record resources. This avoids repeating the DNS record resource block each time a record is added.

---

### SCENARIO BASED

**131. A resource was created manually in Azure. How do you bring it under Terraform management?**
Write the resource block in your Terraform code matching the existing resource. Then run terraform import resource_type.name /full/azure/resource/id. Terraform adds it to state. Run terraform plan to verify no unintended changes. If plan shows changes, adjust your code to match the actual resource configuration.

**132. You need to rename a Terraform resource without destroying and recreating it.**
Use terraform state mv old_resource_type.old_name new_resource_type.new_name. This renames the state entry. Then update the resource block name in code. Run terraform plan — it should show no changes. The cloud resource is untouched.

**133. Terraform apply failed halfway. What is the state?**
Resources that completed before the failure are recorded in state. Resources that failed or were not reached are not in state (or partially in state). Run terraform plan again — Terraform will show what still needs to be done. Fix the root cause and re-apply. Terraform handles partial failures gracefully.

**134. Two engineers run terraform apply at the same time. What happens?**
The second one fails immediately with a state lock error. The first one proceeds. After the first finishes and releases the lock, the second can retry. This is why remote backends with state locking are essential.

**135. Your statefile was accidentally deleted. How do you recover?**
If Azure Blob versioning is enabled, restore the previous blob version from Azure portal. If no versioning, you must use terraform import to re-import every resource one by one. This is painful and time-consuming — always enable blob versioning on your state storage account.

**136. You have 3 environments — test, uat, prod. How would you structure Terraform?**
Option 1 (workspaces): One directory, three workspaces. Variables as maps keyed by workspace name. Same code, different state per workspace. Good when environments are structurally identical.
Option 2 (directories): Separate directory per environment (envs/prod, envs/uat, envs/test). More isolation, can have different code per env. More duplication.
Fischer uses workspaces for kubernetes and keyvault since all clusters have the same structure.

**137. A resource has been modified manually in Azure (drift). What happens on next plan/apply?**
Terraform detects the drift during plan by comparing state (what Terraform thinks exists) with reality (what Azure actually has). It shows the drift as a change and will revert it on next apply to match your code. This is the declarative model — Terraform always enforces the desired state from code.

**138. You want to destroy only one specific resource. How?**
terraform destroy -target=resource_type.name. Only destroys that specific resource. Use with caution — -target is not recommended for regular use as it can leave state inconsistent.

**139. A Terraform module you are using has a breaking change in a new version. How do you handle it?**
Do not upgrade yet. Pin the current version (version = "~> 1.0"). Read the changelog and migration guide. Test the new version in a non-prod environment first. Update your code to handle breaking changes. Then upgrade version pin and apply to prod.

**140. You need to move a resource from one module to another without destroying it.**
Use terraform state mv module.old_module.resource_type.name module.new_module.resource_type.name. This moves the state entry. Update your code to reflect the new module structure. Run terraform plan — should show no changes if done correctly.

**141. How do you prevent accidental destruction of a production database?**
Add lifecycle { prevent_destroy = true } to the resource block. Terraform will throw an error if anything tries to destroy it — even terraform destroy. To actually delete the resource, you must first remove prevent_destroy from the code.

**142. A variable contains a password. How do you handle it securely?**
Mark it sensitive = true in the variable block. Never put the value in .tfvars files committed to git. Pass it via environment variable (TF_VAR_password) or secret management (Atlantis injects from Key Vault). Protect the statefile — sensitive values are stored in state in plaintext.

**143. You need to add a new environment (staging) to the Fischer kubernetes setup. What files do you change?**
1. variables.tf in terraform/kubernetes — add "staging" key to all map variables with appropriate values
2. atlantis.yaml — add a new project entry with dir=terraform/kubernetes and workspace=staging
3. Raise a PR — Atlantis will run plan for the new staging project automatically.

**144. A for_each resource has 5 items. You remove one from the middle. What happens to the other 4?**
Only the removed item is destroyed. The other 4 are completely unaffected. This is the advantage of for_each over count — resources are identified by their key, not by position.

**145. You have a resource that depends on an external system not managed by Terraform.**
Use a data source to read the external resource if it is accessible via the provider. If it is completely external, use variables to pass its ID/endpoint into your Terraform code. If there is a timing dependency, use depends_on with a null_resource that has a local-exec provisioner to wait for readiness — though this is a last resort.

**146. How would you test Terraform code before applying to production?**
1. terraform validate — check syntax
2. terraform fmt — check formatting
3. terraform plan — review changes
4. Apply to test/nonprod environment first
5. Use tools like terratest for automated testing
6. In Fischer: Atlantis posts plan on PR — team reviews before approving apply

**147. terraform plan shows a resource will be destroyed and recreated. How do you find out why?**
Run terraform plan -out=tfplan then terraform show -json tfplan | jq to see detailed reasons. Or look at the plan output carefully — Terraform shows which attribute changed that forces replacement. Use terraform state show resource_type.name to see current state values vs what code expects. Common reasons: immutable attributes changed (like resource names, regions).

**148. You need to apply only to kubernetes-prod-2 without touching nonprod-2 via Atlantis.**
Comment on the PR: atlantis apply -p kubernetes-prod-2. The -p flag specifies the exact project name from atlantis.yaml. Only that project is applied. Other projects (kubernetes-nonprod-2, kubernetes-service-1) are not touched.

**149. A lifecycle precondition is failing. How do you investigate?**
Read the error_message in the plan output — it tells you what condition failed. Check the condition expression and what values it is evaluating. Use terraform console to test expressions interactively. In Fischer example: if workspace = default then precondition fails with "Workspace default not allowed" — fix by selecting the correct workspace.

**150. How do you handle a situation where the same IP is needed across different Terraform modules?**
Define the IP in one module and export it via an output block. The other module reads it as an input variable. If modules are in separate state files, use a data source with terraform_remote_state to read the output from another module's state. Or store the IP in a shared configuration system (like Key Vault) and both modules read it via data source.
