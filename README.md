# cloud-infra-to-iaac

A Claude Code skill that reverse-engineers **live Azure infrastructure into maintainable Terraform**, built on Azure CLI discovery and [Azure Verified Modules (AVM)](https://github.com/Azure/Azure-Verified-Modules).

The skill lives at [`.claude/azure-terraform-import/SKILL.md`](.claude/azure-terraform-import/SKILL.md) and is invoked as `azure-terraform-import`.

---

## What it does

You point it at something that already exists in Azure — a subscription, a resource group, or a handful of ARM resource IDs. It discovers what's there, works out how the pieces relate, picks the right AVM module for each resource type, writes the Terraform, derives the exact `import {}` addresses from the module source, and then keeps refining until `terraform plan` is clean.

The goal is not "some Terraform that looks right." It is Terraform that **adopts** the existing resources without recreating them, and where `terraform plan` shows no destroys and no unwanted updates.

## When to use it

- You inherited an environment built by hand in the portal and need it under version control
- You want to codify a subscription or resource group as-is, without a rebuild
- You need to stop configuration drift on resources nobody has a source of truth for
- A `terraform plan` after an import is showing unexpected `~ update` or `- destroy` lines

## Prerequisites

| Requirement | Why |
|---|---|
| Azure CLI, authenticated (`az login`) | All discovery runs through `az` |
| Read access to the target scope | Discovery fails without RBAC on the subscription/RG |
| Terraform CLI | `init` / `validate` / `plan`, and to download module source for address derivation |
| Network access to Terraform Registry + the AVM index on GitHub | Module selection and README lookups |

## Inputs

At least **one** scope is required. Everything else is optional.

| Parameter | Required | Default | Description |
|---|---|---|---|
| `subscription-id` | No | Active CLI context | Subscription-scope discovery |
| `resource-group-name` | No | None | Resource-group-scope discovery |
| `resource-id` | No | None | One or more ARM resource IDs for targeted discovery |

ARM resource IDs are treated strictly as **cloud identifiers**, never as local file paths — they are only ever passed to `az ... --ids`.

---

## Step-by-step functionality

### 1. Collect scope

The skill asks for exactly one scope and stops if none is given. Once a valid scope is present it stops asking follow-up questions — no re-prompting for a subscription when resource IDs were already supplied.

### 2. Authenticate and set context

Runs only what the chosen scope needs:

```bash
az login
az account set --subscription <subscription-id>
az account show --query "{subscriptionId:id, name:name, tenantId:tenantId}" -o json
```

For resource-group or specific-resource scope, `az account set` is skipped if the active context is already correct.

### 3. Run discovery

```bash
az resource list --subscription <subscription-id> -o json        # subscription scope
az resource list --resource-group <resource-group-name> -o json  # resource group scope
az resource show --ids <resource-id> ... -o json                 # specific resources
```

Output is the raw Azure metadata for each resource: `id`, `type`, `name`, `location`, `tags`, `properties`.

### 4. Resolve dependencies

The discovery JSON is parsed to map parent-child relationships (NIC → Subnet → VNet), cross-resource references buried in `properties`, and the ordering Terraform will need.

**Two artefacts are written to `docs/` in the project root:**

- `exported-resources.json` — every discovered resource with its metadata, dependencies and references
- `EXPORTED-ARCHITECTURE.MD` — a human-readable architecture overview of what was found and how it hangs together

This happens **before** any code generation, so the dependency graph is settled first.

### 5. Select AVM modules

AVM modules are preferred over handwritten `azurerm_*` resources wherever one exists; any fallback to a native resource is documented as a gap.

Modules are found via the official AVM indexes (always latest, `main` branch):

- [`TerraformResourceModules.csv`](https://raw.githubusercontent.com/Azure/Azure-Verified-Modules/refs/heads/main/docs/static/module-indexes/TerraformResourceModules.csv)
- [`TerraformPatternModules.csv`](https://raw.githubusercontent.com/Azure/Azure-Verified-Modules/refs/heads/main/docs/static/module-indexes/TerraformPatternModules.csv)
- [`TerraformUtilityModules.csv`](https://raw.githubusercontent.com/Azure/Azure-Verified-Modules/refs/heads/main/docs/static/module-indexes/TerraformUtilityModules.csv)

Or by searching the Terraform Registry for `avm` + resource name and filtering by the **Partner** tag.

### 5a. Read the module README first — mandatory

Before a single line of HCL is written for a module, its README is fetched and read in full:

```text
https://raw.githubusercontent.com/Azure/terraform-azurerm-avm-res-<service>-<resource>/refs/heads/main/README.md
```

Three things are extracted **before** coding:

1. **Required Inputs** — anything listed here (NICs, extensions, subnets, public IPs) is owned *inside* the module. Standalone sibling module blocks for those resources are wrong.
2. **Optional Inputs** — the exact variable names and declared types. These frequently differ from the raw `azurerm` provider's argument names.
3. **Usage examples** — which resource-group identifier the module wants (`parent_id` vs `resource_group_name`), and how child resources are expressed.

Skipping this step is the single most common cause of broken AVM imports. The worked examples in the skill (VM module NIC ownership, TrustedLaunch booleans, `boot_diagnostics` as a `bool`, the AzAPI-backed VNet module's `parent_id`) are given as *patterns of mismatch to expect*, not as facts to apply blindly to every module.

### 6. Generate Terraform

#### 6a. Inspect module source before writing import blocks — mandatory

After `terraform init` downloads the modules, the actual resource addresses are derived from the downloaded source. Import addresses are never written from memory.

```bash
grep "^resource" .terraform/modules/<key>/main*.tf             # azurerm_* vs azapi_resource, and the label
grep "^module"   .terraform/modules/<key>/main*.tf             # nested sub-modules -> extra path segments
grep -n "count\|for_each" .terraform/modules/<key>/main*.tf    # [0] index vs string key
```

Which yields addresses of the shape:

```text
module.<root_key>.module.<child_key>["<map_key>"].<resource_type>.<label>[<index>]
```

Reference patterns the skill carries as templates:

| Resource | Import `to` address pattern |
|---|---|
| AzAPI-backed VNet | `module.<vnet_key>.azapi_resource.vnet` |
| Subnet (nested, count-based) | `module.<vnet_key>.module.subnet["<name>"].azapi_resource.subnet[0]` |
| Linux VM (count-based) | `module.<vm_key>.azurerm_linux_virtual_machine.this[0]` |
| VM NIC | `module.<vm_key>.azurerm_network_interface.virtualmachine_network_interfaces["<nic_key>"]` |
| VM extension (default `deploy_sequence=5`) | `module.<vm_key>.module.extension["<name>"].azurerm_virtual_machine_extension.this` |
| NSG-NIC association | `module.<vm_key>.azurerm_network_interface_security_group_association.this["<nic>-<nsg>"]` |

#### 6b. Files produced

| File | Contents |
|---|---|
| `providers.tf` | `azurerm` provider + version constraints |
| `main.tf` | AVM module blocks with explicit dependencies |
| `variables.tf` | Environment-specific values |
| `outputs.tf` | Key IDs and endpoints |
| `terraform.tfvars.example` | Placeholder values |

Module versions are pinned explicitly.

#### 6c. Diff live properties against module defaults — mandatory

Every non-zero property of each live resource is compared against the default declared in the AVM module's `variables.tf`. Anything that differs is set explicitly in the config — this is where silent drift comes from.

Common offenders:

- **Timeouts** — Public IP `idle_timeout_in_minutes` defaults to `4`, live deployments often run `30`
- **Network policy flags** — subnet `private_endpoint_network_policies` defaults to `Enabled`, existing subnets are often `Disabled`
- **SKU and allocation** — Public IP `sku`, `allocation_method`
- **Availability zones** — VM and Public IP zones
- **Redundancy/replication** on storage and database resources

Full properties are pulled with targeted `az ... show` calls, because `az resource list` omits nested and computed properties:

```bash
az network public-ip show --ids <id> --query "{idleTimeout:idleTimeoutInMinutes, sku:sku.name, zones:zones}" -o json
az network vnet subnet show --ids <id> --query "{privateEndpointPolicies:privateEndpointNetworkPolicies, delegation:delegations}" -o json
```

### 7. Validate

```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan
```

**Definition of done:** the plan shows **0 destroys and 0 unwanted changes**. Telemetry `+ create` resources are acceptable; any `~ update` or `- destroy` against real infrastructure must be resolved before the import is called complete.

---

## What you get back

Every run reports:

1. Scope used (subscription, resource group, or resource IDs)
2. Discovery files created
3. Resource types detected
4. AVM modules selected, with versions
5. Terraform files generated or updated
6. Validation command results
7. Open gaps requiring your input

## Guardrails

These are the rules the skill holds itself to, and the reasoning behind each:

- **No scope, no start.** Discovery against the wrong subscription is worse than no discovery.
- **README before HCL.** Required Inputs decide child-resource ownership; guessing produces "provider configuration not present" errors and duplicated resources.
- **Never assume NICs, extensions or public IPs are standalone.** For any AVM module, treat child resources as parent-owned until the README says otherwise.
- **Never write import addresses from memory.** Provider label, sub-module nesting and `count` vs `for_each` all vary per module and per version.
- **ARM IDs are not file paths.** They belong in `az --ids`, never in `cat`, `ls` or a glob.
- **Don't re-prompt for known scope.** Ask again only when a concrete command fails for want of context.
- **AVM first, fallbacks justified.** Each handwritten `azurerm_*` resource is documented as a gap.

## Troubleshooting

| Problem | Likely cause | Action |
|---|---|---|
| `az` fails with authorization errors | Wrong tenant/subscription, or missing RBAC | Re-run `az login`, verify context, confirm permissions |
| Discovery output is empty | Wrong scope, or nothing in scope | Re-check the scope input and re-run |
| No AVM module for a resource type | Not yet covered by AVM | Use the native `azurerm_*` resource and document the gap |
| `terraform validate` fails | Missing variables or unresolved dependencies | Add the variables and explicit dependencies, re-validate |
| Unknown argument / variable not found | AVM variable name differs from the `azurerm` argument | Check the module README and `variables.tf` |
| Import fails — resource not found at address | Wrong provider label, missing sub-module path, or missing `[0]` | `grep "^resource"` and `grep "^module"` the downloaded module source |
| Plan shows unexpected `~ update` on an imported resource | Live value differs from the module default | Fetch the live property with `az ... show`, set it explicitly |
| "Provider configuration not present" on a child resource | Child declared standalone though the parent module owns it | Check Required Inputs, remove the sibling module, model it inline |
| Nested child import "resource not found" | Missing intermediate module path, wrong map key, or missing index | Rebuild the full nested address from the module source |
| Agent reads an ARM ID as a file path, or loops on scope questions | ID not routed to `--ids`; provided scope not trusted | ARM IDs go to `az --ids` only; stop re-prompting once one scope is valid |

## References

- [Azure Verified Modules index (Terraform)](https://github.com/Azure/Azure-Verified-Modules/tree/main/docs/static/module-indexes)
- [Terraform AVM Registry namespace](https://registry.terraform.io/namespaces/Azure)
