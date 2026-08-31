# cloud-infra-to-iaac

A Claude Code skill that reverse-engineers **live Azure infrastructure into maintainable Terraform**, built on Azure CLI discovery and [Azure Verified Modules (AVM)](https://github.com/Azure/Azure-Verified-Modules).

The skill lives at [`.claude/skills/azure-terraform-import/SKILL.md`](.claude/skills/azure-terraform-import/SKILL.md) and is invoked as `azure-terraform-import`.

---

## What it does

You point it at something that already exists in Azure: a subscription, a resource group, or a handful of ARM resource IDs. It discovers what's there (including the child objects and extension resources that `az resource list` never returns), builds a dependency model, writes audit documentation, selects the right AVM module for each resource type, generates the Terraform, derives the exact `import {}` addresses from the downloaded module source, reconciles live values against module defaults, and does not stop at a clean plan: it applies, then proves the second plan is empty.

### The contract

**The deliverable is a configuration that can rebuild the environment from nothing.** Delete the resource group, run `terraform apply`, and everything comes back. Pointing it at a different subscription is a change of variables, not a different workflow, so the skill never asks whether you want a duplicate, a TEST variant, or a rebuild. One configuration does all three.

Import blocks are a means, not the goal. They let Terraform adopt what is already live so the first apply does not duplicate it, then they are deleted. What survives is a configuration that depends on nothing it did not create:

| Rule | Why |
|---|---|
| The resource group is a resource the configuration **creates** | `az resource list` never returns it. A configuration built only from that list can only reference a group that already exists, so it fails the moment the group is gone |
| Nothing in scope is read with a **data source** | A data source is a dependency on something the configuration cannot create |
| Every create-time value lives in the config or `terraform.tfvars` | A value that only exists in Azure is a hole in the rebuild |
| Names and sizing are **variables**, defaulted to the discovered values | Rebuilding in place needs no edit; a copy elsewhere needs only different tfvars |

> **It copies infrastructure shape, not data.** The ARM control plane is reproduced: resources, their configuration, and the relationships between them. Blob contents, database rows, VM disk contents and Key Vault secret values are not. Worth saying out loud, because "clone PROD" is usually heard as including the data.

## When to use it

- You inherited an environment built by hand in the portal and need it under version control
- You want to codify a subscription, resource group, or set of resource IDs as-is, without a rebuild
- You need to stop configuration drift on resources nobody has a source of truth for
- You are troubleshooting a Terraform import, or a plan that shows unexpected changes after one
- You need dependencies mapped between discovered Azure resources
- You need the appropriate AVM modules selected and implemented
- You want Terraform that can rebuild an environment from nothing after a teardown
- You want to clone an environment: PROD into TEST, or the same platform in a second subscription or tenant

## Prerequisites

| Requirement | Why |
|---|---|
| Azure CLI, authenticated (`az login`) | All discovery runs through `az` |
| Access to the target subscription or resource group | Discovery fails without the right RBAC |
| Terraform CLI | `init` / `validate` / `plan` / `apply`, and to download module source for address derivation |
| Access to Terraform Registry and AVM resources | Module selection and README lookups |

## Inputs

At least **one** scope is required. Everything else is optional.

| Parameter | Required | Default | Description |
|---|---|---|---|
| `subscription-id` | No | Active CLI context | Discovery at the subscription scope, and setting the Azure CLI context |
| `resource-group-name` | No | None | Discovery within the resource-group scope |
| `resource-id` | No | None | One or more ARM resource IDs, for discovery at the specific-resource scope |
| `target-subscription-id` | No | None | **Second copy only.** The subscription to stand a copy up in while the original still exists |
| `target-resource-group-name` | No | None | **Second copy only.** The resource group the copy deploys into |
| `environment-name` | No | None | **Second copy only.** Short token (`test`, `uat`, `dr`) driving the naming convention where the namespace is global |

A rebuild in place needs none of these. The originals are free once the resource group is gone, so `terraform apply` returns the environment under the same names.

ARM resource IDs are treated strictly as **Azure identifiers**, never as local file paths; they are only ever passed to Azure CLI commands that support `--ids`.

---

## Step-by-step functionality

### 1. Define the discovery scope

One of subscription, resource group, or resource IDs. Once a valid scope is present the skill stops asking; there is no re-prompting for a subscription when resource IDs were already given, and it never asks whether the output should be a copy, a TEST variant or a rebuild.

A `<scope-slug>` is then derived from the scope (resource group name, hyphenated subscription name, or resource name) and used to name `docs/<scope-slug>/` and the backend state key, so every artefact is traceable to the Azure scope it came from. Generic names like `docs/terraform` are explicitly rejected, since they collide the moment a second scope enters the repository.

### 2. Authenticate and set Azure context

```bash
az login
az account set --subscription <subscription-id>
az account show --query "{subscriptionId:id, name:name, tenantId:tenantId}" -o json
```

For resource-group or specific-resource scope, `az account set` is optional when the active context is already correct, and `--ids` queries are preferred over subscription-wide discovery.

### 3. Discover existing resources

```bash
az resource list --subscription <subscription-id> -o json           # subscription scope
az resource list --resource-group <resource-group-name> -o json     # resource group scope
az resource show --ids <resource-id-1> <resource-id-2> ... -o json  # specific resources
```

Captures `id`, `type`, `name`, `location`, `tags` and `properties`. This becomes the source of truth for everything downstream. But it is only the top layer, which is why the next two steps exist.

**The resource group itself is captured separately**, with `az group show`, because `az resource list` returns only what is *inside* it. Skipping this is what produces a configuration that can only reference a group it did not create, and it is the direct cause of `Resource Group ... was not found` at plan time after a teardown. The same applies to any other in-scope container the configuration should own: a Log Analytics workspace diagnostic settings point at, a shared VNet, a Key Vault holding referenced secrets.

### 3.1 Enumerate child objects explicitly

> `az resource list` returns only top-level ARM resources. Container services hold child objects it never lists, and those objects are usually the substance of the environment.

Every container resource has its collections queried directly: Data Factory pipelines, datasets, linked services, integration runtimes and triggers; Synapse pools and firewall rules; storage containers, shares, queues and tables; Key Vault keys, secrets and certificates. Where no first-party CLI command exists, `az rest` against the ARM API rather than skipping the check.

**Counters lie.** A Data Factory reports `factoryStatistics.totalResourceCount = 0` while holding published pipelines. Any aggregate is unverified until the collection endpoints confirm it.

Child objects also carry cross-resource references that never appear in the parent's ARM properties, such as a linked service pointing at a storage account, and those feed the dependency analysis.

### 3.2 Enumerate extension resources (AVM interfaces)

Separate from child objects: the cross-cutting resources attached to anything. Missing them produces an import that looks complete and is not.

| Extension resource | Discover with | AVM module input |
|---|---|---|
| Role assignments | `az role assignment list --scope "<resource id>"` | `role_assignments` |
| Diagnostic settings | `az monitor diagnostic-settings list --resource "<resource id>"` | `diagnostic_settings` |
| Management locks | `az lock list --resource-group <rg>` | `lock` |
| Private endpoints | `az network private-endpoint list -g <rg>` | `private_endpoints` |

Two failure modes silently report "nothing found", and both have produced falsely clean results:

- **Query per resource, not per resource group.** `az role assignment list --resource-group <rg>` does not return assignments scoped to resources inside that group. The group reports `0` while its storage account carries two.
- **Guard ARM IDs on Git Bash / MSYS.** Without `MSYS_NO_PATHCONV=1`, the shell rewrites `/subscriptions/...` into `C:/Program Files/Git/subscriptions/...` and the call fails.

Errors are never suppressed on these calls. An empty collection and a broken command must not look the same.

### 4. Analyze dependencies

A dependency model is built before any Terraform is written: parent-child relationships (`NIC → Subnet → VNet`), references buried in `properties`, the required creation and import order, and shared infrastructure. This is what makes module composition reflect the deployed environment rather than a guess at it.

### 5. Generate discovery documentation

Written to `docs/<scope-slug>/`:

| Artefact | Contents |
|---|---|
| `exported-resources.json` | Complete inventory: resources, metadata, dependency mappings, cross-resource references |
| `exported-architecture.md` | Human-readable architecture summary, hierarchy, dependency overview, design observations |

These are the audit record, and they remain useful long after the import. For a redeployment they are the specification of the source.

### 6. Select Azure Verified Modules

> **Required:** use an AVM module wherever a suitable one exists. Native `azurerm_*` resources only where none is available, with the reason documented.

Found via the Terraform Registry (search `avm <resource>`, filter by the **Partner** tag) or the official AVM indexes for [resource](https://raw.githubusercontent.com/Azure/Azure-Verified-Modules/refs/heads/main/docs/static/module-indexes/TerraformResourceModules.csv), [pattern](https://raw.githubusercontent.com/Azure/Azure-Verified-Modules/refs/heads/main/docs/static/module-indexes/TerraformPatternModules.csv) and [utility](https://raw.githubusercontent.com/Azure/Azure-Verified-Modules/refs/heads/main/docs/static/module-indexes/TerraformUtilityModules.csv) modules. Source and version are recorded before any code is generated.

### 6.1 Read the module README before writing code

> **Mandatory:** never write AVM-based Terraform from memory, or from raw `azurerm` provider knowledge.

| From | What to extract | Why it matters |
|---|---|---|
| **Required Inputs** | Which resources the module manages internally (NICs, extensions, public IPs, subnets) | Creating a separate module for a parent-owned resource is the classic AVM failure |
| **Optional Inputs** | Variable names, types, accepted values, defaults | AVM names and structures frequently differ from the underlying provider's |
| **Usage patterns** | `parent_id` vs `resource_group_name`, nested definitions, map structures | Identifier style varies per module, including between AzAPI- and azurerm-backed ones |

**Standard AVM interfaces.** Modules implement a common set of cross-cutting inputs, and these are how the extension resources found in step 3.2 are expressed: `role_assignments`, `diagnostic_settings`, `private_endpoints`, `lock`, `managed_identities`, `customer_managed_key`, plus `tags` and `enable_telemetry`.

> Model extension resources through these inputs, **never as standalone `azurerm_*` resources**. The module owns the object, so a parallel native resource declaring the same assignment or setting will fight it, and the two overwrite each other on alternate applies.

Coverage is not uniform across modules, so each interface is confirmed in the module's own `variables.tf` rather than assumed. The skill carries an observed coverage matrix across five modules for exactly this reason.

### 7. Generate Terraform configuration

The repository is itself the Terraform root module, so `.tf` files sit in the project root alongside `docs/`, with no wrapper directory:

```text
<project-root>/
├── docs/<scope-slug>/{exported-resources.json, exported-architecture.md}
├── providers.tf, main.tf, variables.tf, outputs.tf
├── imports.tf
├── terraform.tfvars.example
└── .gitignore
```

> One root module holds one Azure scope. A second scope has nowhere to go without a restructure, which is raised with the user rather than merging two scopes into one state file.

**Check whether the scope is already managed, before generating anything.** A backend existing is not the same as a backend being empty: the objects inside it get listed. Finding a state file that already manages the scope changes the task into reconciling with that configuration rather than writing a second one, because two root modules managing one set of resources will fight, and neither owner is correct. If the owning configuration cannot be found on disk, that is said plainly rather than assumed absent; it may live on another machine or in CI.

**Provider constraints are derived, not guessed.** Module versions alone are not enough, since `required_providers` must satisfy every module at once. The **Requirements** section of each module README is read and the constraints intersected: four modules declaring `>= 4.81, < 5.2`, `>= 4.12, < 5.0`, `>= 4.28, < 5.0` and `>= 3.0, < 5.0` intersect to `>= 4.81, < 5.0`. Same for `azapi`, `random`, `time` and `modtm`. The intersection is recorded as a comment, and `.terraform.lock.hcl` is committed.

**Rebuildability rules.** These are what separate a configuration that describes the environment from one that can recreate it:

1. **The resource group is a managed resource**, generated from `az group show`. Every module takes its name or ID from that module's output, never a hardcoded string and never a data source.
2. **No data source for anything in scope.** Data sources are legitimate only for things genuinely outside the scope and genuinely pre-existing, such as a hub VNet owned by another team, and every one is reported as an external prerequisite.
3. **Defer data sources AVM modules read internally.** Some modules read their parent with `data "azurerm_resource_group"`. Terraform resolves a data source at plan time whenever its configuration is fully known, so the read happens before the group is created and the plan fails. Module-level `depends_on` is the only thing that defers it; a resource-level reference does not.
4. **Every create-time value is in the configuration.** A password or key that Azure will not return on import is still required to *create* the resource. Sensitive variable, placeholder in `terraform.tfvars.example`.
5. **`imports.tf` never gates a plan.** Import blocks resolve at plan time and fail if the target is gone, which is exactly the state after a teardown.
6. **Names and sizing are variables** defaulted to the discovered values.

**State is decided before the first apply, not after.** Local state is rarely acceptable for an import: it is the only record of the work, it is gitignored and backed up by nothing, two concurrent runs diverge with no locking, and it holds in plaintext every secret that had to be supplied because Azure would not return it. An `azurerm` backend on versioned blob storage gives both locking and point-in-time recovery.

**AVM telemetry defaults to on.** It adds the `modtm` provider as a real dependency, and contributes a `random_uuid` plus a `modtm_telemetry` resource *per module and submodule*, so the count scales with module nesting. Five modules can easily mean twenty additions in the plan. Usually set `enable_telemetry = false` for an import, where a readable diff is the whole point. Either way, the count is predicted before the plan, never explained after it.

Module versions are always pinned.

### 7.1 Inspect module source before writing import blocks

> **Mandatory:** never construct import addresses from memory.

After `terraform init`, the real addresses come from the downloaded source:

```bash
grep "^resource" .terraform/modules/<module_key>/main*.tf            # azurerm vs azapi, labels
grep "^module"   .terraform/modules/<module_key>/main*.tf            # nested modules, full path
grep -n "count\|for_each" .terraform/modules/<module_key>/main*.tf   # [0] index vs string key
```

```text
module.<root_module>.module.<child_module>["<key>"].<resource_type>.<label>[<index>]
```

`azapi_update_resource` is the exception: it models a PATCH rather than owning a resource, cannot be imported at all, and is documented as a known non-importable item instead.

### 7.2 Reconcile live configuration with module defaults

Live properties are retrieved per resource, compared against the AVM defaults in `variables.tf`, and anything that differs is set explicitly. Targeted `az ... show` calls, because `az resource list` omits nested and computed properties.

**One input can silently override another.** AVM modules often expose a coarse input and finer ones, resolved with `coalesce`, where the coarse input wins whenever it is non-null and its default is frequently not null. In `avm-res-storage-storageaccount` 0.9.0, `sku_name = coalesce(var.account_sku_name, "${var.account_tier}_${var.account_replication_type}")` with `account_sku_name` defaulting to `"Standard_ZRS"` makes `account_replication_type = "LRS"` inert. Grep how a module consumes an input before setting it.

### 7.3 Reconcile attributes that do not survive import

7.2 handles values that differ from a default. This handles values **absent from state after import**, which is the more dangerous failure: anything the provider's Read does not return lands as `null`, so configuring the correct value reads as a change, and on a ForceNew attribute as a destroy and recreate.

The decision rule: if the attribute is ForceNew, **match the null**, always. Otherwise choose deliberately between matching the null (clean plan, configuration does not record the value) and asserting it (complete configuration, one in-place write), and say which.

`null` also does not reach through `optional()`: Terraform substitutes the default whenever an optional object attribute is null, so passing `null` sends the default. Where that default would change live infrastructure, use a different input or drop to `azapi_resource`.

### 8. Validate the generated Terraform

```bash
terraform init && terraform fmt -recursive && terraform validate
terraform plan -out=plan.bin && terraform show -json plan.bin > plan.json
```

**Verify the plan mechanically, not by eye.** A five-resource plan runs to hundreds of lines, and a `sku` block just outside the scrollback that was actually read is enough to miss a real change. Assert property by property against `docs/<scope-slug>/`, exit non-zero on any difference, and ignore only what is legitimately environment-specific (names, principal IDs, Azure-generated resource group names). "No functional changes" is reported from a check that counted what it compared, never from a visual scan.

Success is **0 destroys and 0 unintended updates**, with every addition attributable to predicted telemetry or a stated exception. A clean plan means ready to apply. It does not mean done.

### 9. Complete the import

> **Mandatory:** an import is not complete at `plan`. Until `apply` runs there is no state file, and the configuration manages nothing.

1. **Apply**, never with `-lock=false`. Import errors surface here rather than at plan: a resource that no longer exists, an ID whose casing does not match, an address resolving to nothing.
2. **Prove idempotency.** The real test is the *second* plan. A plan clean before apply and dirty after is a failed import, not a finished one.
3. **Remove the import blocks.** One-shot migration aids; delete `imports.tf` and confirm the plan is still empty. The discovery documentation stays.
4. **Report**, including the applied resource count and the empty post-apply plan.

`apply` is never run without the user's explicit agreement. It writes state and can mutate live infrastructure through any in-place update in the plan.

### 10. Prove the configuration can rebuild

> An empty plan proves the configuration matches what is live. It proves nothing about whether the configuration could create it. Only the second claim is the contract.

Everything up to step 9 was verified against resources that already existed, so the rebuild path has never run.

1. **Plan against an empty scope**, with `imports.tf` deleted and no state. Every resource must appear as `+ create` and the plan must complete without error. A *not found* means something is being read rather than created; a missing required variable means a create-time value is not in the configuration; a resource that does not appear was never modelled.
2. **Rebuild for real at least once**, into a throwaway resource group. A plan catches missing references; only an apply catches values that are wrong rather than absent. If this is skipped, the report says rebuildability is untested rather than implying otherwise.
3. **Report the rebuild gaps**: every attribute recorded as a gap in 7.3, every external prerequisite left as a data source, and every value the user must supply for a rebuild to work.

---

## Standing a second copy up alongside the original

A configuration that satisfies the contract already rebuilds its own scope, and needs nothing extra to do it. One case does need more: standing a **second** copy up while the first still exists, such as TEST alongside a live PROD. The configuration does not change; some `terraform.tfvars` values must, because two environments cannot share them.

| | Rebuild in place | Second copy alongside |
|---|---|---|
| Names | Unchanged, the originals are free | Must change wherever the namespace is global |
| State | The same backend key | Its own key or workspace |
| Identities | New principal IDs, so role assignments rebuild from module outputs | Same |
| Network | Unchanged | New address space if it peers to the same hub |
| Verification | Step 10 | Step 10, then R7 against the original |

R1 to R7 apply only when the original is still standing:

| Step | What it covers |
|---|---|
| **R1. Set the target context** | Confirm the target subscription and tenant explicitly rather than inheriting the CLI context still pointed at the source. Aliased providers where both appear in one configuration, separate tfvars where they do not. |
| **R2. Parameterise names** | Storage accounts, Key Vaults, ACR, App Service, SQL, Cosmos, public DNS labels are globally unique, so reusing a source name fails the apply. One convention driven from `environment-name`, plus the soft-delete and name-length traps. |
| **R3. Give the target its own state** | A separate backend key or workspace. Writing into the source's state means the next plan proposes destroying the source. |
| **R4. Reconcile the network** | Address space reuse versus peering, repointing peerings, binding private endpoints to the target's private DNS zones, and rules that name source addresses. |
| **R5. Decide sizing deliberately** | A faithful clone reproduces the source's SKUs, zone redundancy and bill. Agree per resource class what the target should be, and check the target region offers it. |
| **R6. State what does not come across** | Data, secret and certificate values, principal IDs and the access that trusts them, and every write-only value. Delivered as a list of what the user must still do. |
| **R7. Verify the clone against the source** | Re-run discovery against the target and diff it against the source, ignoring what is legitimately environment-specific. Each remaining difference is either an intended R5 decision or a gap. |

R5 is worth reading carefully: the skill reproduces the source faithfully, cost included, and does not ask whether you want something smaller. Sizing is exposed as variables, so a cheaper TEST is a different tfvars file rather than a different generation run.

---

## What you get back

1. Discovery scope used, and the `<scope-slug>` derived from it
2. Documentation and discovery files created, with their paths
3. Resource types identified
4. Selected AVM modules and versions
5. Terraform files generated or modified
6. Validation results, including the post-apply plan when step 9 has run
7. Attributes that could not be discovered, and the consequence of supplying the wrong value
8. Resources deliberately excluded, and why
9. Outstanding gaps or required user input
10. The step 10 rebuild check: whether a plan against an empty scope succeeded, whether a real rebuild apply was run, and every rebuild gap, external prerequisite and variable needed for `terraform apply` to recreate the environment

A redeployment also reports source and target scopes separately, the naming convention and every name that had to change, the state key or workspace the target uses, the R7 diff, and what the user must still supply or restore.

## Agent rules

The constraints the skill holds itself to, and the reasoning behind each:

- **No scope, no start.** But never ask whether the user wants a duplicate, a TEST variant or a rebuild; one configuration does all three through its variables.
- **The resource group, and every in-scope container, is created by the configuration**, captured with `az group show` since `az resource list` does not return it.
- **No data source for anything in scope.** Where an AVM module reads its parent internally, module-level `depends_on` defers it to apply.
- **Prove the rebuild before claiming completion**: plan against an empty scope with `imports.tf` removed, and say plainly when an actual rebuild apply was not run.
- **One root module, one scope.** A second scope in the same repository is a restructure to raise, not a merge to perform.
- **Enumerate child collections and extension resources explicitly.** Never conclude a resource is empty from a summary counter or its absence in `az resource list`.
- **Never suppress errors on discovery commands.** An empty collection and a failed call must not look the same.
- **AVM first, exceptions justified**, and child-resource ownership read from module documentation rather than assumed.
- **Never write import addresses from memory.** Provider type, nesting and `count` vs `for_each` vary by module and version.
- **Check how a module consumes an input before setting it.** An input guarded by `coalesce` behind a non-null default does nothing.
- **Check ForceNew before configuring a value that imported as null.** Asserting one destroys live infrastructure to change nothing.
- **Never invent an undiscoverable value.** Expose it as a sensitive variable and say what supplying the wrong one will do.
- **Decide the backend before the first apply**, and never apply with `-lock=false`.
- **Verify plans mechanically**, against the discovery artefacts, never by reading a diff.
- **Not complete until the post-apply plan is empty.** A clean pre-apply plan is not a completed import.
- **Never let a redeployment touch the source's state, workspace, or provider context**, and never carry its globally-unique names, principal IDs or private DNS links into a target.

## Troubleshooting

| Issue | Likely cause | Recommended action |
|---|---|---|
| Azure CLI authorization errors | Incorrect tenant, subscription, or RBAC permissions | Re-authenticate and verify access |
| Discovery returns no resources | Incorrect scope | Validate the subscription, resource group, or resource IDs |
| Role assignments report zero | Queried per resource group instead of per resource | Iterate every resource ID with `--scope` |
| ARM ID becomes a `C:/Program Files/Git/...` path | Git Bash path conversion | `export MSYS_NO_PATHCONV=1` before the `az` call |
| No AVM module available | Module does not exist yet | Use `azurerm_*` and document the exception |
| Unknown module argument | AVM variable differs from the provider argument | Check the README and `variables.tf` |
| Input set but has no effect | Outranked by a coarser input inside a `coalesce` | Grep the module for how the variable is consumed |
| Import address not found | Incorrect provider type, label, nesting, or index | Inspect the downloaded module source |
| `Resource Import Not Implemented` | The target is an `azapi_update_resource` | Document it as a known non-importable addition |
| Unexpected updates in plan | Live values differ from module defaults | Explicitly set the live values |
| Plan clean before apply, dirty after | Drift hidden behind an import or addition | The post-apply plan is the real test |
| `Resource Group ... was not found` at plan, before anything is created | The group is not in the configuration, or a module reads it with a data source that Terraform resolves at plan time | Model it from `az group show`, reference the module output, add module-level `depends_on` |
| Plan fails after the resource group was deleted | `imports.tf` is still present and its blocks resolve against resources that no longer exist | Delete `imports.tf`; it is a one-shot adoption aid |
| Rebuild fails on a missing required variable | A create-time value was never captured because the import did not need it | Every create-time value needs a variable and a placeholder in `terraform.tfvars.example` |
| Unexplained additions in the plan | AVM telemetry, one pair per module and submodule | Predict the count, or set `enable_telemetry = false` |

## References

- [Azure Verified Modules index (Terraform)](https://github.com/Azure/Azure-Verified-Modules/tree/main/docs/static/module-indexes)
- [Terraform AVM Registry namespace](https://registry.terraform.io/namespaces/Azure)
