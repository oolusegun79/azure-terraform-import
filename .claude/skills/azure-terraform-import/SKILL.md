---
name: azure-terraform-import
description: 'Use this to reverse-engineer existing Azure environments into Terraform using Azure CLI discovery and Azure Verified Modules (AVM). It is the preferred approach for importing subscriptions, resource groups, or specific ARM resource IDs; generating Terraform for portal-built environments; creating import blocks; selecting the correct AVM module and version; mapping dependencies; and resolving unexpected plan changes after imports. Apply it even when users describe the problem indirectly, such as “we have no Terraform for this subscription,” “the portal is our source of truth,” “how do we stop drift,” or “can you codify what’s already deployed?” In these cases, the underlying need is an AVM-based import. The configuration it produces is always rebuildable: delete the resource group and `terraform apply` recreates everything, and pointing it at another scope is a change of variables, not a different workflow. So use it just as readily for "recreate PROD in TEST", "stand this up in another subscription", "we need the same platform on a second machine", or a rebuild after a teardown. For existing Azure resources, prefer this approach over manually writing azurerm_* resources.'
---

# Azure Terraform Import

Convert existing Azure infrastructure into maintainable Terraform code using Azure resource discovery and Azure Verified Modules (AVM).

## The Contract

**The deliverable is a configuration that can rebuild the environment from nothing.** Delete the
resource group, run `terraform apply`, and everything comes back.

Import blocks are a means, not the goal. They let Terraform adopt what is already live so the first
apply does not duplicate it, and they are deleted once state exists. What survives is a configuration
that does not depend on any resource it did not create.

Every rule below serves that contract. Three consequences are easy to get wrong, and each has its
own section:

- **The scope owns itself.** The resource group is a resource the configuration creates, never one it
  reads. See step 3 and the rebuildability rules in step 7.
- **Nothing in scope is read with a data source.** A data source is a dependency on something the
  configuration cannot create.
- **Every value needed to create a resource lives in the configuration or in `terraform.tfvars`.** A
  value that only exists in Azure is a hole in the rebuild.

> Test it the way it will be used: delete the resource group, run `terraform apply`, and confirm the
> environment returns. That is step 10, and it is not optional.

> **This skill copies infrastructure shape, not data.** It captures and reproduces the ARM control
> plane: resources, their configuration, and the relationships between them. It does not copy blob
> contents, database rows, VM disk contents, or Key Vault secret values. When the goal is a working
> TEST copy of PROD, say this before generating anything, because "clone PROD" is almost always
> heard as including the data.

## When to Use This Skill

Use this skill when the user asks to:

- Import existing Azure resources into Terraform
- Prefer AVM modules over handwritten `azurerm_*` resources for existing infrastructure
- Recreate infrastructure from a subscription, resource group, or set of resource IDs
- Produce Terraform that can rebuild an environment from nothing after a teardown
- Stand a captured environment up in another scope: clone PROD to TEST, or the same platform in a
  second subscription or tenant
- Troubleshoot Terraform imports, drift, or unexpected plan changes
- Map dependencies between discovered Azure resources
- Select and implement the appropriate AVM modules

## Prerequisites

- Azure CLI installed and authenticated (`az login`)
- Access to the target subscription or resource group
- Access to Terraform Registry and AVM resources
- Terraform CLI installed

## Inputs

At least one of `subscription-id`, `resource-group-name`, or `resource-id` is required.

| Parameter | Required | Default | Description |
|---|---|---|---|
| `subscription-id` | No | Active CLI context | Azure subscription used to discover resources at the subscription scope and set the Azure CLI context. |
| `resource-group-name` | No | None | Azure resource group used to discover resources within the resource-group scope. |
| `resource-id` | No | None | One or more Azure ARM resource IDs used to discover resources at the specific-resource scope. |
| `target-subscription-id` | No | None | **Second copy only.** The subscription to stand a copy up in while the original still exists. Never the discovery source. |
| `target-resource-group-name` | No | None | **Second copy only.** The resource group the copy deploys into. |
| `environment-name` | No | None | **Second copy only.** Short token distinguishing the copy (`test`, `uat`, `dr`). Drives the naming strategy in step R2. |

The last three are needed only to stand a **second** copy up alongside an original that still exists.
A rebuild in place needs none of them: the names are free once the resource group is gone, so
`terraform apply` returns the environment as it was. When a copy is wanted, it needs a source scope
**and** a target; supplying only one is the error that puts the copy into the source subscription.


## Workflow

### 1. Define the Discovery Scope (Required)

Do not ask whether the user wants an exact copy, a TEST variant, or a rebuild. The answer is always
the same configuration: one that recreates the environment as discovered, and can be pointed at
another scope by changing variables. Build that, every time.

Before running any discovery commands, identify one of the following scopes:

- Subscription scope: `<subscription-id>`
- Resource group scope: `<resource-group-name>`
- Specific resource scope: one or more Azure ARM `<resource-id>` values

Scope Rules:

- Treat Azure ARM resource IDs (for example, `/subscriptions/.../providers/...`) as Azure resource identifiers, not local file paths.
- Use resource IDs only with Azure CLI commands that support `--ids`.
- Never pass resource IDs to file operations such as `cat`, `ls`, `file readers`, or `path-based` searches unless the user explicitly identifies them as local files.
- If a valid scope has already been provided, do not request additional scope information unless required to resolve an error.
- Avoid follow-up questions that can be answered from the supplied scope.

If no scope is provided, request it before proceeding.

#### Derive the Scope Slug

Derive a `<scope-slug>` from the selected scope. It names the discovery directory in step 5 and the
state key in step 7, so every generated artifact is traceable to the Azure scope it came from.

| Scope | `<scope-slug>` |
|---|---|
| Resource group | The resource group name, e.g. `rg-olu-data-eng` |
| Subscription | The subscription name, lowercased and hyphenated; fall back to the subscription ID if the name is generic (`Azure subscription 1`) or absent |
| Specific resource | The resource name; for several IDs, the resource group they share, or a short name describing the set |

Lowercase the slug and replace any character outside `a-z`, `0-9`, and `-` with `-`.

> **Never name the discovery directory `docs/terraform`, `docs/infra`, or any other generic label.**
> A generic name collides the moment a second scope is imported into the same repository, and it
> tells a reader nothing about which Azure scope the artifacts came from.

### 2. Authenticate and Set Azure Context

Run only the commands required for the selected scope.

Subscription scope:

```bash
az login
az account set --subscription <subscription-id>
az account show --query "{subscriptionId:id, name:name, tenantId:tenantId}" -o json
```

Expected output:

```json
{
  "subscriptionId": "<subscription-id>",
  "name": "<subscription-name>",
  "tenantId": "<tenant-id>"
}
```

Resource group or specific-resource scope:

```bash
az login
```

- `az account set` is optional if the active subscription context is already correct.
- For resource-specific imports, prefer direct `--ids` queries and avoid unnecessary subscription-wide discovery.

### 3. Discover Existing Resources

Collect resource metadata from Azure based on the selected scope.

```bash
# Subscription scope
az resource list --subscription <subscription-id> -o json

# Resource group scope
az resource list --resource-group <resource-group-name> -o json

# Specific resource scope
az resource show --ids <resource-id-1> <resource-id-2> ... -o json
```

Capture all relevant resource information, including:

- id
- type
- name
- location
- tags
- properties

This discovery data becomes the source of truth for Terraform generation.

#### Capture the Scope Container Itself

> **`az resource list` does not return the resource group.** It lists what is *inside* the group. A
> configuration built only from that list has no resource group in it, so it can only reference one
> that already exists. Delete the group and `terraform plan` fails with
> `Resource Group ... was not found` before it creates anything. This is the single most common way a
> generated configuration turns out not to be rebuildable.

Capture the container as a resource in its own right:

```bash
az group show --name <resource-group-name>   --query "{name:name, location:location, tags:tags, managedBy:managedBy}" -o json
```

For subscription scope, do this for every resource group the discovery touched. Record each one in
the discovery artifacts alongside the resources it holds, and model it in step 7 with
`avm-res-resources-resourcegroup` (or `azurerm_resource_group`) so the configuration creates it.

The same applies to anything else the resources sit inside and the configuration should own: a
Log Analytics workspace that diagnostic settings point at, a shared VNet, a Key Vault holding
referenced secrets. If it is in scope, it is created, not read.

#### 3.1 Enumerate Child Objects Explicitly

> **Mandatory:** `az resource list` returns only top-level ARM resources. Container services hold
> child objects that it never lists, and those objects are usually the substance of the environment.

For every discovered resource that can contain children, query each child collection directly. Never
infer that a resource is empty from a summary counter, a statistics block, or the absence of children
in `az resource list`.

| Service | Collections to enumerate |
|---|---|
| Data Factory | `pipelines`, `datasets`, `linkedservices`, `integrationruntimes`, `triggers`, `dataflows`, `managedVirtualNetworks`, `credentials` |
| Synapse | `sqlPools`, `bigDataPools`, `firewallRules`, `integrationRuntimes`, `managedIdentitySqlControlSettings` |
| Storage | `blobServices/default/containers`, `fileServices/default/shares`, `queueServices/default/queues`, `tableServices/default/tables` |
| Key Vault | `keys`, `secrets`, `certificates` |
| Databricks | in-workspace objects are data-plane only and need the `databricks` provider, not `azurerm` |

Where no first-party CLI command exists, or the required CLI extension is not installed, use the ARM
REST API directly rather than skipping the check:

```bash
az rest --method get \
  --url "https://management.azure.com/subscriptions/<sub>/resourceGroups/<rg>/providers/<provider>/<type>/<name>/<collection>?api-version=<version>" \
  --query "value[].name" -o tsv
```

> **Counters lie.** A Data Factory reports `properties.factoryStatistics.totalResourceCount = 0` even
> when it holds published pipelines, datasets, linked services and integration runtimes. Treat any
> such aggregate as unverified until the collection endpoints confirm it.

Child objects also carry cross-resource references that never appear in the parent's ARM properties -
an ADF linked service pointing at a storage account or Key Vault, for example. Feed these into the
dependency analysis in step 4.

#### 3.2 Enumerate Extension Resources (AVM Interfaces)

> **Mandatory:** Step 3.1 covers *child* objects held inside a container resource. This step covers
> *extension* resources attached to any resource. Neither appears in `az resource list`, and missing
> the extension resources produces an import that looks complete and is not.

Every AVM resource module implements a set of standard cross-cutting interfaces. Discover the Azure
objects that feed them, for every resource found in step 3:

| Extension resource | Discover with | AVM module input |
|---|---|---|
| Role assignments | `az role assignment list --scope "<resource id>"` | `role_assignments` |
| Diagnostic settings | `az monitor diagnostic-settings list --resource "<resource id>" --query "[].name"` | `diagnostic_settings` |
| Management locks | `az lock list --resource-group <resource-group-name>` | `lock` |
| Private endpoints | `az network private-endpoint list -g <resource-group-name>` | `private_endpoints` |

Model whatever is found through the module input, not as a standalone `azurerm_*` resource - see
"Standard AVM Interfaces" in step 6.1. Record the results in the discovery artifacts from step 5 even
when a category is empty, so the check is visibly done rather than assumed.

Two failure modes will silently report "nothing found". Both are real, and both have produced a
falsely clean result:

- **Query per resource, not per resource group.** `az role assignment list --resource-group <rg>`
  does **not** return assignments scoped to resources *inside* that group. A resource group can
  report `0` while its storage account carries two. Iterate every discovered resource ID with
  `--scope`; do not substitute a single resource-group query.

- **Guard ARM resource IDs on Git Bash / MSYS shells.** Set `MSYS_NO_PATHCONV=1` before any `az`
  command that takes an ARM ID in `--scope`, `--resource`, or `--ids`. Otherwise the shell rewrites
  the leading `/subscriptions/...` into a local filesystem path
  (`C:/Program Files/Git/subscriptions/...`) and the call fails. This is the same hazard as the
  "Treat ARM resource IDs as Azure identifiers, never local file paths" rule, arriving through the
  shell rather than through a file tool.

```bash
export MSYS_NO_PATHCONV=1   # Git Bash / MSYS only
az role assignment list --scope "<resource id>" \
  --query "[].{role:roleDefinitionName,principalId:principalId,type:principalType}" -o json
```

> **Never suppress errors on these calls.** Piping to `2>/dev/null`, or defaulting an empty result to
> `0`, turns a failed command into an apparently empty collection. Read the error instead: an empty
> result and a broken command must never look the same.

Role assignments frequently encode the wiring between resources - a workspace identity granted data
access to a storage account, a factory identity granted secret access to a Key Vault. Feed them into
the dependency analysis in step 4 alongside the child-object references from step 3.1.

#### 3.3 Exclude Azure-Managed Resources

Some resources are created and owned by another resource's provider. They appear in discovery like
anything else, but they must **never** be imported: Terraform does not own their lifecycle, and the
owning resource will recreate or mutate them.

Recognise and exclude at minimum:

| Pattern | Owned by |
|---|---|
| `synapseworkspace-managedrg-*` | Synapse workspace |
| `databricks-rg-*` | Databricks workspace (absent on `Serverless` workspaces) |
| `MC_*` | AKS cluster node resource group |
| Any resource group named in a parent's `managedResourceGroupName` / `managedResourceGroupId` | that parent |
| Any resource whose ARM body carries a non-null `managedBy` | the resource named in `managedBy` |

Where the owning resource exposes the name as an input (`managed_resource_group_name` on both Synapse
and Databricks, for example), set it to the discovered value on the parent module rather than
importing the group. It is usually ForceNew, so leaving it to the module default plans a replacement.

Two related exclusions worth stating:

- **Provider-managed sub-objects.** A Databricks workspace carries an `authorizations` entry granting
  the Databricks resource provider Contributor. It is created by Azure, not by the user, and is not
  configuration to reproduce.
- **Data-plane objects.** Databricks clusters, notebooks and Unity Catalog objects; Synapse notebooks;
  storage blobs. These need a different provider entirely, not `azurerm` or `azapi`.

Record exclusions and the reason in the discovery documentation from step 5 so the omission is
visibly deliberate rather than an oversight.

### 4. Analyze Dependencies

Before generating Terraform, build a dependency model from the discovered resources.

Identify:
- Parent-child relationships (for example, `NIC → Subnet → VNet`)
- References between resources in properties
- Required Terraform creation and import order
- Shared infrastructure dependencies

The resulting dependency graph should ensure imports and module composition accurately reflect the deployed environment.

### 5. Generate Discovery Documentation

Create a `docs/<scope-slug>/` directory in the project root and save the discovery outputs there,
using the `<scope-slug>` derived in step 1.

#### Required Artifacts

##### docs/&lt;scope-slug&gt;/exported-resources.json
- Complete inventory of discovered Azure resources
- Resource metadata
- Child objects from step 3.1
- Extension resources from step 3.2 - role assignments, diagnostic settings, locks, private
  endpoints - recorded even when a category is empty, so the check is evidently done
- Dependency mappings
- Cross-resource references

##### docs/&lt;scope-slug&gt;/exported-architecture.md
- Human-readable architecture summary
- Resource hierarchy and relationships
- Dependency overview
- Key infrastructure components and design observations

These artifacts serve as both audit documentation and the foundation for AVM-based Terraform generation.

### 6. Select Azure Verified Modules (AVM)

> **Required:** Use Azure Verified Modules whenever a suitable AVM module exists. Only use native `azurerm_*` resources when no AVM module is available, and document the reason for the exception.

#### Identify the Correct Module

For each discovered Azure resource:

1. Find the corresponding AVM module.
2. Select the latest compatible module version.
3. Record the module source and version before generating Terraform code.

##### Module Discovery Sources

**Terraform Registry**

- Search for `avm <resource-name>`
- Filter results by the **Partner** publisher tag
- Example: `avm storage account`

**Official AVM Module Indexes**

The AVM indexes provide the authoritative list of available modules:

- Terraform Resource Modules  
  `https://raw.githubusercontent.com/Azure/Azure-Verified-Modules/refs/heads/main/docs/static/module-indexes/TerraformResourceModules.csv`

- Terraform Pattern Modules  
  `https://raw.githubusercontent.com/Azure/Azure-Verified-Modules/refs/heads/main/docs/static/module-indexes/TerraformPatternModules.csv`

- Terraform Utility Modules  
  `https://raw.githubusercontent.com/Azure/Azure-Verified-Modules/refs/heads/main/docs/static/module-indexes/TerraformUtilityModules.csv`

> These files always reference the latest content on the main branch. Use a release tag if a point-in-time version is required.

#### Gather Module Documentation

If module metadata is not available locally, retrieve it from:

- Terraform Registry  
  `https://registry.terraform.io/modules/Azure/<module>/azurerm/latest`

- GitHub Repository  
  `https://github.com/Azure/terraform-azurerm-avm-res-<service>-<resource>`

The repository `README.md` is the primary source of module documentation and implementation guidance.

---

### 6.1 Read the Module README Before Writing Code

> **Mandatory:** Never write AVM-based Terraform from memory or from raw `azurerm` provider knowledge.

For every selected module, review the module README before generating HCL.

Retrieve documentation from either:

```text
https://raw.githubusercontent.com/Azure/terraform-azurerm-avm-res-<service>-<resource>/refs/heads/main/README.md
```

Or from the downloaded module source:

```bash
cat .terraform/modules/<module_key>/README.md
```

#### Capture the Following Information

##### Required Inputs

Identify all required inputs and determine which resources are managed internally by the module.

Examples may include:

- Network interfaces
- Virtual machine extensions
- Public IPs
- Subnets

Do not create separate modules for resources that are explicitly managed by a parent module.

##### Optional Inputs

Document:

- Variable names
- Supported types
- Accepted values
- Default behavior

Do not assume AVM variables use the same names or structures as the underlying `azurerm` provider.

##### Usage Patterns

Review examples to understand:

- Resource group identifiers (`parent_id` vs `resource_group_name`)
- Nested resource definitions
- Map structures
- Provider-specific conventions

##### Standard AVM Interfaces

AVM modules implement a common set of cross-cutting inputs. These are how the extension resources
discovered in step 3.2 are expressed:

- `role_assignments`
- `diagnostic_settings`
- `private_endpoints`
- `lock`
- `managed_identities`
- `customer_managed_key`
- `tags`, `enable_telemetry`, and on some modules `retry` / `timeouts`

> **Model extension resources through these inputs, never as standalone `azurerm_*` resources.**
> The module owns the object; a parallel native resource declaring the same assignment or setting
> will fight it, and the two will overwrite each other on alternate applies.

Coverage is **not** uniform, so confirm each interface in the module's own `variables.tf` rather than
assuming it exists. Observed across five modules:

| Interface | storage 0.9.0 | keyvault 0.11.0 | databricks 0.5.0 | datafactory 0.2.0 | synapse 0.1.0 |
|---|---|---|---|---|---|
| `role_assignments` | yes | yes | yes | yes | yes |
| `lock` | yes | yes | yes | yes | yes |
| `private_endpoints` | yes | yes | yes | yes | - |
| `diagnostic_settings` | - | yes | yes | yes | - |
| `managed_identities` | yes | - | - | yes | yes |
| `customer_managed_key` | yes | - | - | - | yes |

Where a module lacks the interface for an object that exists in Azure, fall back to a native
`azurerm_*` resource under the documented-exception rule in step 6.

Some modules add related controls that change how the interface is written - the storage module's
`role_assignment_definition_lookup_enabled`, for example, governs whether role definitions are given
as names or as GUIDs. Read `variables.tf` before populating the input.

#### AVM Implementation Rules

For every selected module:

- Determine resource ownership from **Required Inputs**
- Determine accepted parameters from **Optional Inputs** and `variables.tf`
- Follow README examples for identifier formats and input structures
- Never infer arguments from native `azurerm_*` resources

---

### 7. Generate Terraform Configuration

Generate Terraform code using the selected AVM modules and dependency model.

#### Output Directory

Write the configuration to the **project root**. The repository is itself the Terraform root module,
so the `.tf` files sit alongside `docs/`, with no wrapper directory.

```text
<project-root>/
├── docs/
│   └── <scope-slug>/
│       ├── exported-resources.json
│       └── exported-architecture.md
├── providers.tf
├── main.tf
├── variables.tf
├── outputs.tf
├── imports.tf
├── terraform.tfvars.example
└── .gitignore
```

> **This layout holds exactly one Terraform root module, so it holds exactly one Azure scope.** The
> scope slug survives only in `docs/<scope-slug>/` and in the backend state key, which is what keeps
> the artifacts traceable. If a second scope is ever imported into the same repository, it has
> nowhere to go without a restructure. Raise that with the user before generating a second
> configuration rather than merging two scopes into one root module and one state file.

If the project already has an established layout for infrastructure code, follow that instead, and
name the leaf directory after `<scope-slug>` rather than a generic label.

#### Required Files

Create, in the project root:
- `providers.tf`
- `main.tf`
- `variables.tf`
- `outputs.tf`
- `imports.tf`: the import blocks from step 7.1
- `terraform.tfvars.example`
- `.gitignore`: ignore `.terraform/`, `*.tfstate*`, and `terraform.tfvars`; commit `.terraform.lock.hcl`

#### Rebuildability Rules

> **Mandatory:** These are what make the difference between a configuration that describes the
> environment and one that can recreate it. Check every one before generating.

**1. The resource group is a managed resource.** Generate it from the `az group show` output in step
3. Every other module takes its name or ID from that module's output, never from a hardcoded string
and never from a data source.

**2. No data source for anything in scope.** `data "azurerm_resource_group"`, `data
"azurerm_key_vault"`, `data "azurerm_log_analytics_workspace"` and friends are each a dependency on
something the configuration cannot create. Reference the module output instead. Data sources are
legitimate only for things genuinely outside the scope and genuinely pre-existing, such as a hub VNet
owned by another team; list every one in the report as an external prerequisite.

**3. Defer data sources that AVM modules read internally.** Some modules read their parent with
`data "azurerm_resource_group"` inside the module. Terraform resolves a data source at plan time
whenever its configuration is fully known, so the read happens *before* the resource group is
created and the plan fails. Break that by making the dependency explicit:

```hcl
module "storage" {
  source     = "Azure/avm-res-storage-storageaccount/azurerm"
  version    = "<pinned>"
  parent_id  = module.rg.resource_id
  depends_on = [module.rg]   # defers every data source inside the module to apply
}
```

`depends_on` at module level is the only thing that defers a module's internal data sources. A
resource-level reference does not.

**4. Every create-time value is in the configuration.** Anything Azure will not return on import
(passwords, keys, connection strings) is still required to *create* the resource. Declare it as a
variable, mark it sensitive, and put a placeholder in `terraform.tfvars.example`. A configuration
that imports cleanly but cannot create is a failed deliverable, not a trade-off.

**5. `imports.tf` never gates a plan.** Import blocks resolve at plan time and fail if their target
does not exist, which is exactly the state after a teardown. They belong in their own file, they are
deleted once state exists (step 9), and a rebuild plan runs without them.

**6. Names are variables, not literals.** Drive resource names from a variable with the discovered
value as its default. Rebuilding in place then needs no edit, and standing a copy up elsewhere needs
only a different `terraform.tfvars`. This is what removes the need to ask whether the user wants a
duplicate or a separate environment: the same configuration does both.

#### Derive Provider Version Constraints

Module versions are not enough - `required_providers` must satisfy every module at once. Read the
**Requirements** section of each selected module's README, then intersect the constraints and pin the
result in `providers.tf`. Four modules declaring `>= 4.81, < 5.2`, `>= 4.12, < 5.0`, `>= 4.28, < 5.0`
and `>= 3.0, < 5.0` intersect to `>= 4.81, < 5.0`.

Do the same for `azapi`, `random`, `time` and `modtm`. Record the intersection as a comment so the
next person does not have to re-derive it. Commit `.terraform.lock.hcl` so resolved versions are
reproducible.

#### Check Whether the Scope Is Already Managed

> **Mandatory, and before generating any Terraform.** A backend existing is not the same as a
> backend being empty. List the objects inside it.

Finding a state file that already manages the target scope changes the whole task: the job becomes
reconciling with that configuration, not writing a second one. Two root modules managing one set of
resources is the worst outcome an import can produce - each will fight the other, and neither owner
will know why.

```bash
az storage blob list --account-name <backend account> --container-name <container> \
  --auth-mode login --query "[].{name:name,size:properties.contentLength,modified:properties.lastModified}" -o table
```

Inspect any candidate before concluding it is unrelated - read the resource ADDRESSES only, never
the attribute values, which hold secrets in plaintext:

```bash
terraform show -json <state> | python -c "import sys,json; d=json.load(sys.stdin); [print(r.get('module','root'), r['type']+'.'+r['name']) for r in d['values']['root_module'].get('resources',[])]"
```

If a state file covers the scope, stop and raise it with the user. If its owning configuration
cannot be found on disk, say so plainly rather than assuming it does not exist: it may live on
another machine or in CI, which is common and is not a reason to write a competing config.

#### Configure State Before the First Apply

> **Decide the backend before applying, not after.** Migrating state later is avoidable work, and an
> import is worth nothing without the state file it produces.

Default Terraform writes state to a local `terraform.tfstate`. For an import that is rarely
acceptable:

- **It is the only record of the import.** Lose it and every resource must be discovered and imported
  again. It lives on one machine and is gitignored, so it is backed up by nothing.
- **Concurrent runs diverge.** Two people importing the same scope produce two conflicting states with
  no locking between them.
- **State contains secrets in plaintext.** Any value that had to be supplied because Azure would not
  return it - a SQL administrator password, a connection string, an access key - is written to state
  in clear text. This is why state must never be committed, and why the storage holding it needs the
  same protection as a secret store.

Configure an `azurerm` backend (blob storage with versioning enabled, which gives both locking and
point-in-time recovery), or the state backend the project already standardises on. Raise it with the
user if no backend exists yet rather than silently shipping local state.

State is also why the `-lock=false` convenience flag belongs in read-only planning only. Never apply
with locking disabled.

#### Pin Module Versions

Always specify a fixed module version:

```hcl
module "example" {
  source  = "Azure/<module>/azurerm"
  version = "<latest-compatible-version>"
}
```

#### AVM Telemetry

Every AVM module accepts `enable_telemetry`, and it **defaults to `true`**. Decide deliberately, and
state which way it was set - do not leave it to the default without saying so.

What it does:

- Adds the `modtm` provider as a genuine dependency. It must be declared in `required_providers`
  alongside `azurerm` and `azapi`, or `terraform init` fails.
- Contributes a `random_uuid` and a `modtm_telemetry` resource **per module and per submodule**. The
  count therefore scales with module nesting, not with the number of Azure resources. A five-module
  import can easily produce twenty telemetry resources, every one of which appears in the plan as an
  addition.

Set `enable_telemetry = false` when a plan that shows only real infrastructure matters more than the
telemetry - which is usually the case for an import, where the whole point is a readable diff against
what already exists.

> Telemetry resources are never a reason to call a plan unclean, but they are also never an excuse
> for an unexplained count. Know how many are expected before running the plan, and reconcile the
> number in the results.

---

### 7.1 Inspect Module Source Before Writing Import Blocks

> **Mandatory:** Never construct Terraform import addresses from memory.

After downloading modules with `terraform init`, inspect the module source to determine the exact resource addresses.

#### Step A: Identify Resource Types and Labels

```bash
grep "^resource" .terraform/modules/<module_key>/main*.tf
```

Determine:

- Provider type (`azurerm` vs `azapi`)
- Resource labels
- Actual Terraform addresses

#### Step B: Discover Nested Modules

```bash
grep "^module" .terraform/modules/<module_key>/main*.tf
```

If resources are deployed through child modules, imports must include the complete nested path.

Example:

```text
module.<root_module>.module.<child_module>["<key>"].<resource_type>.<label>[<index>]
```

#### Step C: Check `count` and `for_each`

```bash
grep -n "count\|for_each" .terraform/modules/<module_key>/main*.tf
```

Import addresses must match implementation details:

- `count` resources require numeric indices such as `[0]`
- `for_each` resources require string keys

#### Common Import Address Patterns

The following patterns are examples only and must be verified against the downloaded module source code.

| Resource Type | Example Import Address |
|---------------|------------------------|
| Virtual Network | `module.<vnet>.azapi_resource.vnet` |
| Subnet | `module.<vnet>.module.subnet["<name>"].azapi_resource.subnet[0]` |
| Linux VM | `module.<vm>.azurerm_linux_virtual_machine.this[0]` |
| VM NIC | `module.<vm>.azurerm_network_interface.virtualmachine_network_interfaces["<nic>"]` |
| VM Extension | `module.<vm>.module.extension["<extension>"].azurerm_virtual_machine_extension.this` |

Always derive the final address from module source code, not from examples.

#### Importing azapi Resources

AVM is migrating from `azurerm` to `azapi` - several resource modules are already azapi-backed, and
AVM publishes `avm-utl-*-azapi-replicator` utility modules to help move others. Expect azapi inside
modules and account for three behaviours it does not share with `azurerm`:

**The import ID carries the API version.** Append `?api-version=<version>`, matching the version
pinned in the module's `type` string exactly:

```hcl
import {
  to = module.<name>.azapi_resource.this
  id = "<arm resource id>?api-version=2025-08-01"
}
```

Read the version from the module source (`type = "Microsoft.X/y@<version>"`) or from its
`resource_types` variable defaults. Do not assume the newest API version - the module sends the one it
pins, and a mismatch produces a body diff.

**The first plan after import always shows an in-place update.** Importing an `azapi_resource` stores
the *entire* body Azure returned, while the configuration body carries only what the module manages.
The first apply normalises state to the configured body. The removed keys are server-computed
(`provisioningState`, `createdBy`, timestamps, resource-generated URLs) or already at their default,
and modules generally set `ignore_null_property = true` so nulls are never sent. Plans after the first
apply are clean.

This is expected, not drift - but say so explicitly when reporting, and confirm the body diff contains
no property that would actually change. If a *functional* property is being added or dropped, that is
a real change and must be reconciled, not waved through.

**`azapi_update_resource` cannot be imported.** It models a PATCH against an existing object rather
than owning an ARM resource, and the provider returns *Resource Import Not Implemented*. It will
always appear as an addition in the plan. Verify that applying it sends only values already live, then
document it as a known non-importable item rather than trying to import it.

---

### 7.2 Reconcile Live Configuration with Module Defaults

> **Mandatory:** Imported infrastructure should not drift because module defaults differ from live Azure settings.

> **Redeploying?** This step pins live values so that an *import* produces an empty plan. For a
> redeployment, pin only what is semantically part of the environment: SKUs you intend to carry over,
> network policy flags, feature toggles. Leave identity, naming, location and address values
> parameterised. Pinning a source environment's names or principal IDs into a target is a defect, not
> a reconciliation.

For each discovered resource:

1. Retrieve detailed live properties from Azure.
2. Compare live values against AVM defaults in `variables.tf`.
3. Explicitly configure any value that differs.

#### One Input Can Silently Override Another

Setting the input that *names* the property is not enough. AVM modules frequently expose both a
coarse input and a set of finer ones, and resolve them with `coalesce` - where the coarse input
**wins whenever it is non-null, and its default is often not null**.

Real example, `avm-res-storage-storageaccount` 0.9.0:

```hcl
sku_name = coalesce(var.account_sku_name, "${var.account_tier}_${var.account_replication_type}")
```

`account_sku_name` defaults to `"Standard_ZRS"`, not to `null`. So `account_replication_type = "LRS"`
is **inert** - the account deploys as ZRS, and nothing in the configuration hints at why. The
variable description says so; the variable name does not.

Before setting any input, read how the module actually consumes it:

```bash
grep -rn "<variable name>" .terraform/modules/<module_key>/*.tf
```

If it appears inside a `coalesce`, `try` or `one_of`-style expression, find what outranks it and
check that input's default. Then set the input that actually reaches Azure.

#### Common Sources of Drift

- Timeouts and idle settings
- Network policy configuration
- SKU and allocation settings
- Availability zones
- Storage redundancy and replication settings
- Database configuration options

Example property checks:

```bash
az network public-ip show \
  --ids <resource_id> \
  --query "{idleTimeout:idleTimeoutInMinutes,sku:sku.name,zones:zones}" \
  -o json
```

```bash
az network vnet subnet show \
  --ids <resource_id> \
  --query "{privateEndpointPolicies:privateEndpointNetworkPolicies,delegations:delegations}" \
  -o json
```

Do not rely solely on `az resource list`, as it may omit nested or computed properties.

---

### 7.3 Reconcile Attributes That Do Not Survive Import

> **Mandatory:** Step 7.2 handles values that differ between Azure and the module default. This step
> handles values that are simply **absent from state after import**, which is a different failure and
> the more dangerous one.

> **Redeploying?** The attributes that do not survive import are exactly the ones a redeployment must
> supply explicitly, because the target has no live resource to read them back from. Treat every
> write-only value found here as a required per-environment input and record it in
> `terraform.tfvars.example`. A redeployment missing them cannot complete its first apply.

Terraform import populates state from the provider's Read function. Any attribute the provider does
not read back lands as `null`, no matter what is configured in Azure. Configuring the correct value
then reads as a change - and where the attribute is ForceNew, as a **destroy and recreate**.

Three causes, all common:

- **Write-only in the API.** Azure never returns the value. Passwords, access keys, and secrets.
- **Not implemented in the provider's Read.** The value exists in Azure but the provider does not map
  it back, often because the resource models a newer API shape in an older way.
- **Represented differently in Azure.** The provider looks for a property that this resource does not
  use - for example a linked service that stores discrete `server` / `database` properties while the
  provider reads a single legacy `connectionString`.

#### The Decision Rule

For every attribute that imports as null:

1. **Is it ForceNew?** Check the provider schema. If yes, **match the null** - always. Asserting a
   value on a ForceNew attribute destroys and recreates live infrastructure to change nothing.
2. **Otherwise, assert the value.** The contract is a configuration that can rebuild the
   environment, and a value the configuration does not record is a value the rebuild cannot supply.
   Accept the one in-place write, where re-asserting is semantically a no-op against what is already
   live.

   *Match the null* only when asserting would change live infrastructure. That buys a clean plan at
   the cost of a hole in the rebuild, so record it as a **rebuild gap** in the report, naming the
   resource, the attribute, and what will be missing or wrong after a `terraform apply` into an empty
   scope. A gap that is written down is a decision; one that is not is a defect waiting to surface at
   the worst moment.
3. **If the value cannot be discovered at all**, expose it as a required variable, mark it
   `sensitive`, document that it cannot be read back, and state the consequence of supplying the wrong
   one. Never invent a plausible-looking value: a guessed password or connection string is applied as
   a real change.

Check the provider schema rather than guessing at ForceNew and Computed:

```bash
terraform providers schema -json |   python -c "import sys,json; d=json.load(sys.stdin); print(json.dumps(d['provider_schemas']['registry.terraform.io/hashicorp/azurerm'] ['resource_schemas']['<resource_type>']['block']['attributes'], indent=1))"
```

An attribute that is `optional` but not `computed` is the classic null-on-import case: the provider
will never populate it, so config and state can only agree if the config is null too.

#### Null Does Not Reach Through `optional()`

The "match the null" technique fails inside a module input typed as `map(object({...}))` with
`optional(<type>, <default>)` attributes. **Terraform substitutes the default whenever an optional
object attribute is null**, so passing `null` sends the default, not null.

There is no way to express "leave unset" through such an input. When that default would change live
infrastructure, the options are:

- use a different module input that does not force the value, or
- declare the resource outside the module - `azapi_resource` reproduces the live JSON exactly - and
  document it as an exception under the step 6 rule.

Verify which happened by reading the plan, not by reasoning about it: if the diff still shows the
default being applied after passing null, the substitution is why.

---

### 8. Validate the Generated Terraform

Run:

```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan
```

#### Verify the Plan Mechanically, Not by Eye

> A plan for a five-resource environment runs to hundreds of lines. Reading it with `grep` and a
> scrollback is how a real change gets missed - a `sku` block sitting just outside the window that
> was actually read is enough.

Export the plan and compare it against the discovery artifacts in code:

```bash
terraform plan -out=plan.bin
terraform show -json plan.bin > plan.json
```

Then assert property by property, sourcing the expected values from `docs/<scope-slug>/` rather
than from memory, and exit non-zero on any difference so the check can gate a pipeline. Compare
configuration posture - SKUs, replication, network rules, TLS, retention, child-object inventories.
Do not compare values that are legitimately environment-specific, such as resource names, principal
IDs and Azure-generated managed resource group names.

Never report "no functional changes" from a visual scan of a diff. Report it from a check that
enumerated the properties and told you the count it compared.

#### Success Criteria

The configuration is considered valid when:

- No syntax errors are reported
- Validation completes successfully
- Dependencies are resolved correctly
- The plan accurately reflects the discovered environment

The import should not be considered complete until the plan shows:

- **0 destroys**
- **0 unintended updates**

Telemetry-related resources created by modules are acceptable, but only when their count has been
predicted rather than discovered. Reconcile the additions in the plan against the expectation set in
"AVM Telemetry" in step 7, and report the number explicitly. An addition that cannot be attributed to
either telemetry or a stated exception is an unresolved finding, not noise.

A clean plan means the configuration is ready to apply. It does not mean the import is done - the
state file does not exist yet. Continue to step 9.

---

### 9. Complete the Import

> **Mandatory:** An import is not complete at `plan`. Until `apply` runs there is no state file, and
> the configuration manages nothing.

Confirm the backend from step 7 is configured before the first apply. Then:

**1. Apply.** Never with `-lock=false`.

```bash
terraform apply
```

Import errors surface here rather than at plan: a resource that no longer exists, an ID whose casing
does not match, an address that resolves to nothing. Resolve them and re-apply.

**2. Prove idempotency.** The real test of an import is the *second* plan, not the first:

```bash
terraform plan   # must report: No changes
```

Anything remaining is genuine drift that the pre-apply plan hid behind an import or an addition. A
plan that is clean before apply and dirty after is a failed import, not a finished one.

**3. Remove the import blocks.** They are one-shot migration aids; once state exists they do nothing.
Delete `imports.tf`, then plan again to confirm removing them changed nothing:

```bash
terraform plan   # must still report: No changes
```

Keep the discovery documentation from step 5 - that is the audit record, and it stays useful. The
import blocks are not.

**4. Report** against the requirements below, including the applied resource count and confirmation
that the post-apply plan is empty.

Do not run `apply` on the user's behalf without confirming they want it. Importing writes state and
can mutate live infrastructure through any in-place update in the plan; the decision to cross that
line is theirs. Present the plan, state what it will change, and ask.

---

### 10. Prove the Configuration Can Rebuild

> **Mandatory:** An empty plan proves the configuration matches what is live. It proves nothing about
> whether the configuration could create it. Those are different claims, and only the second one is
> the contract.

Everything up to step 9 was verified against resources that already existed. The rebuild path has
never run. Prove it before reporting the work complete.

**1. Plan against an empty scope.** Point the configuration at a scope that does not exist, with
`imports.tf` deleted and a state file that is empty:

```bash
terraform plan -state=/dev/null -var-file=rebuild-check.tfvars
```

Every resource must appear as `+ create`, and the plan must complete without error. What you are
looking for is not the diff but the failures:

- `Resource Group ... was not found`, or any other *not found*, means something is being read rather
  than created. Find the data source and remove it, or add the module-level `depends_on` from step 7.
- A missing required variable means a create-time value is not in the configuration. Add it.
- A resource that does not appear at all was never modelled. Go back to steps 3.1 and 3.2.

**2. Rebuild for real, at least once.** A plan catches missing references; only an apply catches
values that are wrong rather than absent. Into a throwaway resource group, ideally in a separate
subscription:

```bash
terraform apply -var-file=rebuild-check.tfvars   # then destroy it
```

This is the step that would have caught a deleted resource group before the user found it. If the
user declines the cost or the time, say plainly that rebuildability is untested rather than letting
the report imply otherwise.

**3. Report the rebuild gaps.** Every attribute recorded as a gap in 7.3, every external prerequisite
left as a data source, and every value the user must supply in `terraform.tfvars` for a rebuild to
work. This list is part of the deliverable, not a footnote.

---

## Standing the Environment Up Alongside the Original

A configuration that satisfies the contract already rebuilds its own scope: delete the resource
group, `terraform apply`, and it returns under the same names, because those names are free again.
Nothing extra is needed for that case, and nothing needs asking.

One case does need more: standing a **second** copy up while the first still exists, such as a TEST
alongside a live PROD. The configuration does not change, but some values in `terraform.tfvars` must,
because two environments cannot share them. Work through R1 to R7 when, and only when, the original
is still standing.

| | Rebuild in place | Second copy alongside |
|---|---|---|
| Names | Unchanged; the originals are free | Must change wherever the namespace is global |
| State | The same backend key | Its own key or workspace |
| Identities | New principal IDs, so role assignments rebuild from module outputs | Same |
| Network | Unchanged | New address space if it peers to the same hub |
| Verification | Step 10 | Step 10, then R7 against the original |

Three things break when a configuration that pinned discovered values is applied alongside the
original:

- **Principal IDs.** Role assignments bind to the source environment's managed identities, which do
  not exist in the target. Take them from module outputs instead.
- **Azure-generated names.** A Synapse `managedResourceGroupName` or similar carries one
  environment's GUID into every other. Leave it unset and let Azure name it.
- **Write-only values.** Anything supplied as a sensitive variable because Azure would not return it
  must be re-supplied per environment, and a value that cannot be known before the resource exists
  (a storage account key, for instance) has to be read back with a data source rather than passed
  in, or the first apply is impossible.

Keep the discovery documentation regardless. When the source environment is gone, it is the only
record of what the configuration is supposed to reproduce.

### R1. Set the Target Context

Discovery ran against the source. Everything from here targets a different scope, and conflating the
two is the failure with the worst blast radius in this workflow.

- Confirm the target subscription and tenant explicitly. Never infer them from the active CLI
  context, which is still pointed at the source after step 2.
- Verify the target is not the source before generating anything:

```bash
az account show --query "{subscriptionId:id, name:name}" -o json
```

- Where source and target appear in one configuration, give each an aliased provider and pass the
  alias explicitly to every module. An unaliased default provider is how resources land in the wrong
  subscription.
- Where they are separate root modules, the target gets its own `terraform.tfvars`. Prefer this
  unless the user needs both environments in one apply.

> If the environment allows it, run the first target `plan` under credentials that cannot reach the
> source. A redeployment that cannot see PROD cannot damage PROD.

### R2. Parameterise Names, and Know Which Ones Must Change

The table above says parameterise every identifier. Some are not merely better parameterised, they
are **globally unique across Azure**, and reusing the source name fails the apply outright:

| Resource | Namespace and constraint |
|---|---|
| Storage account | Global. 3-24 chars, lowercase alphanumeric, no hyphens |
| Key Vault | Global per cloud. A soft-deleted vault still holds its name |
| Container registry | Global, alphanumeric only |
| App Service / Function App | Global, `*.azurewebsites.net` |
| SQL Server, Cosmos DB, Redis, Service Bus, Event Hub namespace | Global per service |
| Public IP DNS label | Global per region |
| Private DNS zone | Unique within the resource group, and its VNet links must point at the target |

Drive one convention from `environment-name` rather than renaming resource by resource:

```hcl
locals {
  suffix   = var.environment_name                                          # "test"
  name     = "${var.workload}-${local.suffix}"                             # rg-, vnet-, kv- style
  squished = lower(replace("${var.workload}${local.suffix}", "-", ""))     # storage, ACR style
}
```

Two traps worth naming:

- **Soft delete holds the name.** A Key Vault or storage account deleted in an earlier attempt keeps
  its name for the retention period. `az keyvault list-deleted` shows them. Purge, or pick another
  name; the error message does not explain itself.
- **Length limits bite after the suffix.** A convention that fits in `prod` can overflow 24
  characters in `preproduction`. Check the longest name in the set, not the first.

### R3. Give the Target Its Own State

A redeployment that writes into the source's state file will, at the next plan, propose destroying
the source. This is the most damaging mistake available here, and it is silent until the plan is
read carefully.

- A separate backend **key** per environment, or a separate workspace. Never a shared default.
- Confirm which is active before the first apply:

```bash
terraform workspace show
```

- Where source and target share a repository, the backend key carries the environment token, exactly
  as the state key in step 7 carries the scope slug.

### R4. Reconcile the Network Before Applying

Network configuration is the part of a clone most likely to apply cleanly and still be wrong.

- **Address space.** A duplicated VNet range cannot peer to the same hub as the source. Decide
  whether the target is isolated (reuse the range) or peered (allocate a new one), and record which.
- **Peerings.** Discovered peerings carry source-side VNet resource IDs. Repoint them at the target
  or drop them; a peering left pointing at the source is a live link between environments.
- **Private endpoints and private DNS.** A private endpoint in the target must link to the *target's*
  private DNS zones. Carrying the source's zone links resolves target traffic to source resources,
  which looks like it is working.
- **Rules that name source addresses.** NSGs, firewall rules and service endpoints allow-listing a
  source subnet or public IP need the target's equivalents.

### R5. Decide Sizing Deliberately

Reproduce the source faithfully by default, including SKUs, capacity and zone redundancy. Do not ask
whether the user wants something smaller: the configuration is generated once and sizing is a
variable, so a cheaper TEST is a different `terraform.tfvars`, not a different generation run.

What that requires is that sizing *is* a variable. Expose SKU, capacity, replication and zone inputs
rather than hardcoding the discovered values into module blocks, with the discovered value as the
default. Mention in the report that a faithful copy carries the source's cost, and that these are the
inputs to change.

Check the target region offers what the source uses. SKU availability, zone counts and some service
features are regional; a configuration valid in one region can be unbuildable in another.

### R6. State What Does Not Come Across

Deliver this before the first apply, as a list of what the user must still do for the environment to
be usable:

- **Data.** Blob and file contents, database rows, VM disk contents, queue messages. None of it.
- **Secrets.** Key Vault secret, key and certificate *values*. The vault is reproduced empty.
- **Identities and their access.** New managed identities mean new principal IDs. Role assignments
  rebuild against the new ones, and anything outside the environment that trusts the source's
  principals will not trust the target's.
- **Write-only values.** Everything step 7.3 identified as absent from state must be re-supplied per
  environment.

### R7. Verify the Clone Against the Source

An empty second plan proves the target matches its own configuration. It says nothing about whether
the target matches the source, which is the actual requirement. Verify it separately:

1. Run discovery (steps 3 to 5) against the **target** once the apply completes.
2. Diff the target's `exported-resources.json` against the source's, ignoring what is legitimately
   environment-specific: names, resource IDs, principal IDs, endpoints, timestamps.
3. Report every remaining difference as either intended (an R5 sizing decision) or a gap in the
   redeployment.

This diff is what proves the clone. Without it, the claim that TEST matches PROD is untested.

## Troubleshooting

| Issue | Likely Cause | Recommended Action |
|---------|-------------|-------------------|
| Azure CLI authorization errors | Incorrect tenant, subscription, or RBAC permissions | Re-authenticate and verify access |
| Discovery returns no resources | Incorrect scope | Validate subscription, resource group, or resource IDs |
| No AVM module available | Module does not exist yet | Use `azurerm_*` and document the exception |
| `terraform validate` fails | Missing variables or dependencies | Add required inputs and dependency references |
| Unknown module argument | AVM variable differs from provider argument | Check README and `variables.tf` |
| Import address not found | Incorrect provider type, label, nesting, or index | Inspect downloaded module source |
| Unexpected updates in plan | Live values differ from module defaults | Explicitly set live values in configuration |
| Child-resource ownership issues | Resource managed by parent module | Model resource through the parent module inputs |
| Nested import failures | Missing module path, map key, or index | Build the complete address from module source |
| ARM IDs treated as file paths | Incorrect handling of resource identifiers | Use Azure CLI `--ids` arguments only |
| Plan replaces a resource on an attribute that looks correct | Attribute imports as null and is ForceNew | Match the null - see step 7.3 |
| Plan replaces a resource over an ID that differs only in casing | ARM returns some IDs non-canonically, e.g. `/resourcegroups/` for role assignment scopes; the attribute is ForceNew and the provider does not normalise | Match the casing Azure returned, deriving it from the module output with `replace()` |
| Passing `null` to a module input has no effect | `optional(<type>, <default>)` substitutes the default for null | Use another input, or declare the resource outside the module - step 7.3 |
| Role assignments are not found | `az role assignment list --resource-group` omits assignments scoped to resources inside the group | Query per resource with `--scope` - step 3.2 |
| An `az` call with an ARM ID fails or returns nothing on Git Bash | MSYS rewrote the ID into a local filesystem path | Set `MSYS_NO_PATHCONV=1`; never suppress the error |
| `Resource Import Not Implemented` | The target is an `azapi_update_resource`, which models a PATCH and cannot be imported | Document it as a known non-importable addition - step 7.1 |
| azapi resource shows an in-place update immediately after import | Import stored the full remote body; the first apply normalises it to the configured body | Expected; confirm no functional property changes - step 7.1 |
| Plan is clean before apply but dirty after | Drift hidden behind an import or addition | The post-apply plan is the real test - step 9 |
| A resource cannot be imported or reappears after apply | It is Azure-managed, e.g. a managed resource group | Exclude it - step 3.3 |
| `terraform init` fails on provider constraints | Module provider requirements were not intersected | Derive the intersection from each module's Requirements section - step 7 |
| `Missing Resource Import State` / *ImportState method returned no State* | The azapi import target does not exist. The provider reports a 404 this way, and blames itself in the message | Confirm the resource is still there (`az resource show --ids`, `az group show`) and check the activity log for a delete before debugging the configuration |
| A plan that worked earlier now fails with no config change | Live infrastructure changed underneath, not Terraform | Check `az monitor activity-log list --offset <n>h` for deletes and their caller before re-deriving anything |
| An input that names a property has no effect | Another module input outranks it via `coalesce`, and its default is not null | Read how the module consumes the variable - step 7.2 |
| `Cycle: module.a ... module.b` | Two AVM modules each need an output of the other, typically a resource ID one way and a principal ID back | Move the role assignment out to the root module as a native `azurerm_role_assignment`; the cycle is between modules, not resources |
| `Resource Group ... was not found` during PLAN, before anything is created | Either the resource group is not in the configuration at all (`az resource list` never returned it), or a module reads it with `data "azurerm_resource_group"`, which Terraform resolves at plan time because the config is fully known | Model the group as a managed resource from `az group show` (step 3), reference it by module output, and add module-level `depends_on` so any data source inside a module defers to apply - step 7 rebuildability rules |
| Plan fails after the resource group was deleted | `imports.tf` is still present, and import blocks resolve at plan time against resources that no longer exist | Delete `imports.tf`; it is a one-shot adoption aid, not part of the configuration - step 9 |
| Rebuild apply fails on a missing required variable | A create-time value was never captured because import did not need it | Every value needed to *create* must be a variable with a placeholder in `terraform.tfvars.example` - step 7.3 |
| A second configuration is discovered for resources already under management | The backend was checked for existence but not for contents | List the state blobs before generating - step 7 |

---

## Response Requirements

When presenting results, include:

1. Discovery scope used, and the `<scope-slug>` derived from it
2. Documentation and discovery files created, with their paths
3. Resource types identified
4. Selected AVM modules and versions
5. Terraform files generated or modified, with their paths in the project root
6. Validation command results, including the post-apply plan when step 9 has run
7. Attributes that could not be discovered, and the consequence of supplying the wrong value
8. Resources deliberately excluded, and why
9. Outstanding gaps or required user input
10. The rebuild check from step 10: whether a plan against an empty scope succeeded, whether an
    actual rebuild apply was run, and every rebuild gap, external prerequisite and variable the user
    must supply for `terraform apply` to recreate the environment

For a redeployment, also include:

10. Source scope and target scope, stated separately and unambiguously
11. The naming convention applied, and every name that had to change
12. The state backend key or workspace the target uses
13. The R7 diff of target against source, with each remaining difference marked intended or a gap
14. What the user must still supply or restore: secrets, data, external trust of the new principals

---

## Agent Rules

- Do not proceed without a defined scope.
- Derive a `<scope-slug>` from the scope and use it for `docs/<scope-slug>/` and the backend state
  key. Write the Terraform itself to the project root, with no wrapper directory.
- The root layout holds one scope. If a second scope is requested for the same repository, raise the
  restructure with the user rather than merging two scopes into one root module and one state file.
- Enumerate child collections explicitly for every container resource. Never conclude that a resource
  is empty from a summary counter or from its absence in `az resource list`.
- Enumerate extension resources - role assignments, diagnostic settings, locks, private endpoints -
  per resource, and model them through the AVM interface inputs rather than as standalone
  `azurerm_*` resources. Query role assignments with `--scope` per resource, never with a single
  `--resource-group` query.
- Never suppress errors on discovery commands. An empty collection and a failed call must not be
  allowed to look the same.
- Account for AVM telemetry resources explicitly when reporting a plan. Predict the count from the
  module and submodule tree; never present them as unexplained additions.
- Do not skip dependency analysis before code generation.
- Prefer AVM modules whenever available.
- Explicitly justify every non-AVM implementation.
- Read each AVM module README before writing code.
- Determine child-resource ownership from module documentation, not assumptions.
- Inspect downloaded module source before creating import blocks.
- Treat ARM resource IDs as Azure identifiers, never local file paths.
- Avoid unnecessary scope-related prompts when valid scope information is already available.
- Exclude Azure-managed resources from the import. Set the name on the owning module instead.
- Check whether an attribute is ForceNew before configuring a value that imported as null. Never
  assert a value on a ForceNew attribute to correct a null.
- Never invent a value that cannot be discovered. Expose it as a sensitive variable and say what
  supplying the wrong one will do.
- Decide the state backend before the first apply, and never apply with `-lock=false`.
- Do not run `terraform apply` without the user's explicit agreement.
- Do not declare the import complete until `apply` has run and the *post-apply* plan reports no
  changes. A clean pre-apply plan is not a completed import.
- List the contents of the state backend before generating Terraform, not just its existence. If a
  state file already manages the scope, stop and raise it.
- Deliver a configuration that can rebuild the environment from nothing. Never ask whether the user
  wants a duplicate, a TEST variant, or a rebuild; the answer is one configuration that does all
  three through its variables.
- Model the resource group, and every other in-scope container, as a resource the configuration
  creates. `az resource list` does not return it, so capture it with `az group show`.
- Never use a data source for anything in scope. Where an AVM module reads its parent internally, add
  module-level `depends_on` so the read defers to apply. Report every remaining data source as an
  external prerequisite.
- Prove the rebuild before reporting completion: plan against an empty scope with `imports.tf`
  removed, and say plainly when an actual rebuild apply was not run.
- Never let a redeployment write to the source environment's state file, workspace, or provider
  context. Confirm the target subscription explicitly rather than inheriting the CLI context used
  for discovery.
- Never carry a source environment's globally-unique names, principal IDs, or private DNS zone links
  into a target. Parameterise them and drive the names from one convention.
- State plainly, before the first apply of a redeployment, that it copies infrastructure and not
  data, secrets, or certificate values.
- Do not claim a redeployment matches its source without the R7 diff. An empty plan proves only that
  the target matches its own configuration.
- Check how a module consumes an input before setting it. An input guarded by `coalesce` behind
  another input with a non-null default does nothing.
- Verify a plan by exporting it to JSON and asserting against the discovery artifacts. Never claim
  "no functional changes" from reading a diff by eye.
- When live infrastructure stops matching what a previous plan showed, check the Azure activity log
  before re-deriving the configuration.

## References

- Azure Verified Modules Index (Terraform)  
  `https://github.com/Azure/Azure-Verified-Modules/tree/main/docs/static/module-indexes`

- Terraform AVM Registry Namespace  
  `https://registry.terraform.io/namespaces/Azure`